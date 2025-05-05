1. Record.java

package com.example.reconciliation.model;

public class Record {
    private String algoRecord;
    private String starRecord;
    private String status;

    public Record(String algoRecord, String starRecord, String status) {
        this.algoRecord = algoRecord;
        this.starRecord = starRecord;
        this.status = status;
    }

    public String getAlgoRecord() {
        return algoRecord;
    }

    public String getStarRecord() {
        return starRecord;
    }

    public String getStatus() {
        return status;
    }
}


---

2. ReconciliationResult.java

package com.example.reconciliation.model;

import java.util.List;

public class ReconciliationResult {
    private List<Record> matched;
    private List<Record> unmatched;
    private List<Record> excluded;
    private int matchedCount;
    private int unmatchedCount;
    private int excludedCount;

    public ReconciliationResult(List<Record> matched, List<Record> unmatched, List<Record> excluded) {
        this.matched = matched;
        this.unmatched = unmatched;
        this.excluded = excluded;
        this.matchedCount = matched.size();
        this.unmatchedCount = unmatched.size();
        this.excludedCount = excluded.size();
    }

    public List<Record> getMatched() {
        return matched;
    }

    public List<Record> getUnmatched() {
        return unmatched;
    }

    public List<Record> getExcluded() {
        return excluded;
    }

    public int getMatchedCount() {
        return matchedCount;
    }

    public int getUnmatchedCount() {
        return unmatchedCount;
    }

    public int getExcludedCount() {
        return excludedCount;
    }
}


---

3. ReconciliationService.java

package com.example.reconciliation.service;

import com.example.reconciliation.model.Record;
import com.example.reconciliation.model.ReconciliationResult;
import org.apache.commons.csv.CSVFormat;
import org.apache.commons.csv.CSVRecord;
import org.springframework.stereotype.Service;

import java.io.FileReader;
import java.util.*;

@Service
public class ReconciliationService {

    public ReconciliationResult reconcile(String algoPath, String starPath, String matchType) {
        try {
            FileReader readerAlgo = new FileReader(algoPath);
            List<CSVRecord> algoRecords = (List<CSVRecord>) CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(readerAlgo);

            FileReader readerStar = new FileReader(starPath);
            List<CSVRecord> starRecords = (List<CSVRecord>) CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(readerStar);

            // 1. Apply exclusion rule
            List<CSVRecord> excluded = new ArrayList<>();
            List<CSVRecord> filteredStar = applyExclusionRule(starRecords, excluded);

            // 2. Convert excluded to Record list
            List<Record> excludedRecords = new ArrayList<>();
            for (CSVRecord record : excluded) {
                String key = record.get("CRDS Party Code").replaceAll("\\s+", "").trim() +
                        "3" + record.get("Post Direction").replaceAll("\\s+", "").trim();
                excludedRecords.add(new Record("<Excluded>", key, "Excluded by Rule (Maturity Date = Jan 1 1900)"));
            }

            // 3. Perform matching
            ReconciliationResult result = match(algoRecords, filteredStar, matchType.toLowerCase());

            // 4. Return result including excluded records
            return new ReconciliationResult(result.getMatched(), result.getUnmatched(), excludedRecords);

        } catch (Exception e) {
            e.printStackTrace();
            return new ReconciliationResult(Collections.emptyList(), Collections.emptyList(), Collections.emptyList());
        }
    }

    private List<CSVRecord> applyExclusionRule(List<CSVRecord> starRecords, List<CSVRecord> excluded) {
        List<CSVRecord> filtered = new ArrayList<>();
        for (CSVRecord record : starRecords) {
            String maturityDate = record.get("Maturity Date").trim();
            if (maturityDate.startsWith("Jan 1 1900")) {
                excluded.add(record);
            } else {
                filtered.add(record);
            }
        }
        return filtered;
    }

    private ReconciliationResult match(List<CSVRecord> algoRecords, List<CSVRecord> starRecords, String type) {
        List<Record> matched = new ArrayList<>();
        List<Record> unmatched = new ArrayList<>();

        List<String> algoKeys = new ArrayList<>();
        for (CSVRecord record : algoRecords) {
            String key = record.get("Agreement_name")
                    .replace("_RIMR", "3CR")
                    .replace("_RIMP", "3CP")
                    .replaceAll("\\s+", "")
                    .trim();
            algoKeys.add(key);
        }

        Map<String, List<Integer>> starKeyMap = new HashMap<>();
        List<String> starKeys = new ArrayList<>();
        int index = 0;
        for (CSVRecord record : starRecords) {
            String key = record.get("CRDS Party Code").replaceAll("\\s+", "").trim() +
                    "3" + record.get("Post Direction").replaceAll("\\s+", "").trim();
            starKeys.add(key);
            starKeyMap.computeIfAbsent(key, k -> new ArrayList<>()).add(index);
            index++;
        }

        Set<Integer> matchedStarIndices = new HashSet<>();
        Set<Integer> matchedAlgoIndices = new HashSet<>();

        for (int i = 0; i < algoKeys.size(); i++) {
            String algoKey = algoKeys.get(i);
            List<Integer> matchingStarIdxs = starKeyMap.getOrDefault(algoKey, new ArrayList<>());

            switch (type) {
                case "one-to-one":
                    if (matchingStarIdxs.size() == 1 && !matchedStarIndices.contains(matchingStarIdxs.get(0))) {
                        matched.add(new Record(algoKey, starKeys.get(matchingStarIdxs.get(0)), "Match"));
                        matchedStarIndices.add(matchingStarIdxs.get(0));
                        matchedAlgoIndices.add(i);
                    } else {
                        unmatched.add(new Record(algoKey, "<No Match>", "Mismatch"));
                    }
                    break;

                case "one-to-many":
                    if (!matchingStarIdxs.isEmpty()) {
                        for (Integer idx : matchingStarIdxs) {
                            matched.add(new Record(algoKey, starKeys.get(idx), "Match (One-to-Many)"));
                            matchedStarIndices.add(idx);
                        }
                        matchedAlgoIndices.add(i);
                    } else {
                        unmatched.add(new Record(algoKey, "<No Match>", "Mismatch"));
                    }
                    break;

                case "many-to-one":
                    if (!matchingStarIdxs.isEmpty()) {
                        matched.add(new Record(algoKey, starKeys.get(matchingStarIdxs.get(0)), "Match (Many-to-One)"));
                        matchedStarIndices.add(matchingStarIdxs.get(0));
                        matchedAlgoIndices.add(i);
                    } else {
                        unmatched.add(new Record(algoKey, "<No Match>", "Mismatch"));
                    }
                    break;

                case "many-to-many":
                    for (Integer idx : matchingStarIdxs) {
                        matched.add(new Record(algoKey, starKeys.get(idx), "Match (Many-to-Many)"));
                        matchedStarIndices.add(idx);
                    }
                    if (matchingStarIdxs.isEmpty()) {
                        unmatched.add(new Record(algoKey, "<No Match>", "Mismatch"));
                    }
                    matchedAlgoIndices.add(i);
                    break;
            }
        }

        for (int i = 0; i < starKeys.size(); i++) {
            if (!matchedStarIndices.contains(i)) {
                unmatched.add(new Record("<No Match>", starKeys.get(i), "Unmatched STAR Key"));
            }
        }

        return new ReconciliationResult(matched, unmatched, Collections.emptyList());
    }
}




...................................................m
3. Entity Classes

AlgoRecordEntity.java

package com.example.reconciliation.entity;

import jakarta.persistence.*;

@Entity
@Table(name = "ALGO_RECORDS")
public class AlgoRecordEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "AGREEMENT_NAME")
    private String agreementName;

    // Getters and setters
}

StarRecordEntity.java

package com.example.reconciliation.entity;

import jakarta.persistence.*;

@Entity
@Table(name = "STAR_RECORDS")
public class StarRecordEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "CRDS_PARTY_CODE")
    private String crdsPartyCode;

    @Column(name = "POST_DIRECTION")
    private String postDirection;

    @Column(name = "MATURITY_DATE")
    private String maturityDate;

    // Getters and setters
}


---

4. Repository Interfaces

AlgoRecordRepository.java

package com.example.reconciliation.repository;

import com.example.reconciliation.entity.AlgoRecordEntity;
import org.springframework.data.jpa.repository.JpaRepository;

public interface AlgoRecordRepository extends JpaRepository<AlgoRecordEntity, Long> {
}

StarRecordRepository.java

package com.example.reconciliation.repository;

import com.example.reconciliation.entity.StarRecordEntity;
import org.springframework.data.jpa.repository.JpaRepository;

public interface StarRecordRepository extends JpaRepository<StarRecordEntity, Long> {
}


---

5. Service Update

ReconciliationService.java

Use the same logic you already shared earlier, but replace the reconcile method like this:

@Autowired
private AlgoRecordRepository algoRepo;

@Autowired
private StarRecordRepository starRepo;

public ReconciliationResult reconcile(String matchType) {
    List<AlgoRecordEntity> algoEntities = algoRepo.findAll();
    List<StarRecordEntity> starEntities = starRepo.findAll();

    List<String> algoKeys = algoEntities.stream()
            .map(a -> a.getAgreementName()
                .replace("_RIMR", "3CR")
                .replace("_RIMP", "3CP")
                .replaceAll("\\s+", "")
                .trim())
            .toList();

    List<String> starKeys = new ArrayList<>();
    List<StarRecordEntity> filtered = new ArrayList<>();
    List<StarRecordEntity> excluded = new ArrayList<>();

    for (StarRecordEntity star : starEntities) {
        if (star.getMaturityDate().toLowerCase().contains("jan 1 1900")) {
            excluded.add(star);
        } else {
            filtered.add(star);
            starKeys.add(star.getCrdsPartyCode().replaceAll("\\s+", "") +
                         "3" + star.getPostDirection().replaceAll("\\s+", ""));
        }
    }

    // Then pass algoKeys and starKeys to your existing matching logic unchanged
}

Everything else in your ReconciliationService (matching logic, etc.) can remain as-is.


---

6. Controller

package com.example.reconciliation.controller;

import com.example.reconciliation.model.ReconciliationResult;
import com.example.reconciliation.service.ReconciliationService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/reconciliation")
public class ReconciliationController {

    @Autowired
    private ReconciliationService reconciliationService;

    @GetMapping("/{matchType}")
    public ReconciliationResult reconcile(@PathVariable String matchType) {
        return reconciliationService.reconcile(matchType);
    }
}
