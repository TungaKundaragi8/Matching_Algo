Repository Interfaces

public interface AlgoRepository extends JpaRepository<AlgoRecord, Long> {}

public interface StarRepository extends JpaRepository<StarRecord, Long> {}

public interface ReconciliationResultRepository extends JpaRepository<ReconciliationResultEntity, Long> {}


---

3. Service: ReconciliationService.java

@Service
public class ReconciliationService {

    @Autowired
    private AlgoRepository algoRepository;

    @Autowired
    private StarRepository starRepository;

    @Autowired
    private ReconciliationResultRepository resultRepository;

    public void reconcile(String matchType) {
        List<AlgoRecord> algoList = algoRepository.findAll();
        List<StarRecord> starList = starRepository.findAll();

        List<StarRecord> filteredStars = new ArrayList<>();

        for (StarRecord star : starList) {
            star.setPostDirection(calculatePostDirection(star));
            String starKey = buildStarKey(star);

            if ("01-JAN-1900".equalsIgnoreCase(star.getMaturityDate().trim())) {
                saveResult("<Excluded>", starKey, "Excluded by Rule (Maturity Date = 01-JAN-1900)");
            } else {
                filteredStars.add(star);
            }
        }

        matchAndStoreResults(algoList, filteredStars, matchType.toLowerCase());
    }

    private void matchAndStoreResults(List<AlgoRecord> algoList, List<StarRecord> starList, String matchType) {
        Map<String, List<StarRecord>> starKeyMap = new HashMap<>();
        for (StarRecord star : starList) {
            String key = buildStarKey(star);
            starKeyMap.computeIfAbsent(key, k -> new ArrayList<>()).add(star);
        }

        Set<String> matchedStarKeys = new HashSet<>();

        for (AlgoRecord algo : algoList) {
            String algoKey = buildAlgoKey(algo);
            List<StarRecord> matchingStars = starKeyMap.getOrDefault(algoKey, new ArrayList<>());

            if (matchingStars.isEmpty()) {
                saveResult(algoKey, "<No Match>", "Mismatch");
                continue;
            }

            switch (matchType) {
                case "one-to-one":
                    if (matchingStars.size() == 1 && !matchedStarKeys.contains(buildStarKey(matchingStars.get(0)))) {
                        saveResult(algoKey, buildStarKey(matchingStars.get(0)), "Match");
                        matchedStarKeys.add(buildStarKey(matchingStars.get(0)));
                    } else {
                        saveResult(algoKey, "<No Match>", "Mismatch");
                    }
                    break;

                case "one-to-many":
                    for (StarRecord star : matchingStars) {
                        saveResult(algoKey, buildStarKey(star), "Match (One-to-Many)");
                        matchedStarKeys.add(buildStarKey(star));
                    }
                    break;

                case "many-to-one":
                    saveResult(algoKey, buildStarKey(matchingStars.get(0)), "Match (Many-to-One)");
                    matchedStarKeys.add(buildStarKey(matchingStars.get(0)));
                    break;

                case "many-to-many":
                    for (StarRecord star : matchingStars) {
                        saveResult(algoKey, buildStarKey(star), "Match (Many-to-Many)");
                        matchedStarKeys.add(buildStarKey(star));
                    }
                    break;
            }
        }

        // Unmatched STAR records
        for (StarRecord star : starList) {
            String starKey = buildStarKey(star);
            if (!matchedStarKeys.contains(starKey)) {
                saveResult("<No Match>", starKey, "Unmatched STAR Key");
            }
        }
    }

    private String calculatePostDirection(StarRecord record) {
        // Example logic: you can customize based on your columns
        String code = record.getCrdsPartyCode().trim();
        return code.endsWith("B") ? "BUY" : "SELL";
    }

    private String buildAlgoKey(AlgoRecord record) {
        return record.getAgreementName()
                .replace("_RIMR", "3CR")
                .replace("_RIMP", "3CP")
                .replaceAll("\\s+", "")
                .trim();
    }

    private String buildStarKey(StarRecord record) {
        return record.getCrdsPartyCode().replaceAll("\\s+", "").trim()
                + "3" + record.getPostDirection().replaceAll("\\s+", "").trim();
    }

    private void saveResult(String algoKey, String starKey, String status) {
        ReconciliationResultEntity result = new ReconciliationResultEntity();
        result.setAlgoKey(algoKey);
        result.setStarKey(starKey);
        result.setStatus(status);
        resultRepository.save(result);
    }
}







package com.example.reconciliation.controller;

import com.example.reconciliation.service.ReconciliationService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/reconciliation")
public class ReconciliationController {

    @Autowired
    private ReconciliationService reconciliationService;

    @PostMapping
    public String reconcile(@RequestParam String matchType) {
        try {
            reconciliationService.reconcile(matchType);
            return "Reconciliation completed successfully with match type: " + matchType;
        } catch (Exception e) {
            e.printStackTrace();
            return "Reconciliation failed: " + e.getMessage();
        }
    }
}
