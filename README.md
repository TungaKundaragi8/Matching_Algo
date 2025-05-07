1. AlgoRecord.java

@Entity
@Table(name = "ALGO_RECORDS")
public class AlgoRecord {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "AGREEMENT_NAME")
    private String agreementName;

    // Getters and setters
    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getAgreementName() {
        return agreementName;
    }

    public void setAgreementName(String agreementName) {
        this.agreementName = agreementName;
    }
}


---

2. StarRecord.java

@Entity
@Table(name = "STAR_RECORDS")
public class StarRecord {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "CRDS_PARTY_CODE")
    private String partyCode;

    @Column(name = "MATURITY_DATE")
    private String maturityDate;

    @Transient
    private String postDirection;

    // Getters and setters
    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getPartyCode() {
        return partyCode;
    }

    public void setPartyCode(String partyCode) {
        this.partyCode = partyCode;
    }

    public String getMaturityDate() {
        return maturityDate;
    }

    public void setMaturityDate(String maturityDate) {
        this.maturityDate = maturityDate;
    }

    public String getPostDirection() {
        return partyCode != null ? partyCode + "3" : null;
    }

    public void setPostDirection(String postDirection) {
        this.postDirection = postDirection;
    }
}


---

3. ReconciliationResultEntity.java

@Entity
@Table(name = "RECONCILIATION_RESULTS")
public class ReconciliationResultEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String algoKey;
    private String starKey;
    private String status;

    // Getters and setters
    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getAlgoKey() {
        return algoKey;
    }

    public void setAlgoKey(String algoKey) {
        this.algoKey = algoKey;
    }

    public String getStarKey() {
        return starKey;
    }

    public void setStarKey(String starKey) {
        this.starKey = starKey;
    }

    public String getStatus() {
        return status;
    }

    public void setStatus(String status) {
        this.status = status;
    }
}


---

4. ExcludedRecordEntity.java

@Entity
@Table(name = "EXCLUDED_RECORDS")
public class ExcludedRecordEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String algoKey;
    private String starKey;
    private String reason;

    // Getters and setters
    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getAlgoKey() {
        return algoKey;
    }

    public void setAlgoKey(String algoKey) {
        this.algoKey = algoKey;
    }

    public String getStarKey() {
        return starKey;
    }

    public void setStarKey(String starKey) {
        this.starKey = starKey;
    }

    public String getReason() {
        return reason;
    }

    public void setReason(String reason) {
        this.reason = reason;
    }
}


---

5. Repositories

public interface AlgoRepository extends JpaRepository<AlgoRecord, Long> {}

public interface StarRepository extends JpaRepository<StarRecord, Long> {}

public interface ReconciliationRepository extends JpaRepository<ReconciliationResultEntity, Long> {}

public interface ExcludedRepository extends JpaRepository<ExcludedRecordEntity, Long> {}







@Service
public class ReconciliationService {

    @Autowired
    private AlgoRepository algoRepository;

    @Autowired
    private StarRepository starRepository;

    @Autowired
    private ReconciliationRepository reconciliationRepository;

    @Autowired
    private ExcludedRepository excludedRepository;

    public void performReconciliation() {
        List<AlgoRecord> algoRecords = algoRepository.findAll();
        List<StarRecord> starRecords = starRepository.findAll();

        // Step 1: Exclude STAR records with Maturity Date = 1900-01-01
        List<StarRecord> validStarRecords = new ArrayList<>();
        for (StarRecord star : starRecords) {
            if ("1900-01-01".equals(star.getMaturityDate())) {
                // Save to excluded table with null ALGO key
                ExcludedRecordEntity excluded = new ExcludedRecordEntity();
                excluded.setAlgoKey(null);
                excluded.setStarKey(String.valueOf(star.getId()));
                excluded.setReason("Maturity date is 1900-01-01");
                excludedRepository.save(excluded);
            } else {
                validStarRecords.add(star);
            }
        }

        // Step 2: Simple 1-1 match by agreementName and postDirection
        for (AlgoRecord algo : algoRecords) {
            boolean matched = false;

            for (StarRecord star : validStarRecords) {
                if (algo.getAgreementName() != null &&
                    algo.getAgreementName().equalsIgnoreCase(star.getPostDirection())) {

                    // Save matched record
                    ReconciliationResultEntity result = new ReconciliationResultEntity();
                    result.setAlgoKey(String.valueOf(algo.getId()));
                    result.setStarKey(String.valueOf(star.getId()));
                    result.setStatus("Matched");
                    reconciliationRepository.save(result);

                    matched = true;
                    break;
                }
            }

            if (!matched) {
                // Save unmatched algo record
                ReconciliationResultEntity result = new ReconciliationResultEntity();
                result.setAlgoKey(String.valueOf(algo.getId()));
                result.setStarKey(null);
                result.setStatus("Unmatched");
                reconciliationRepository.save(result);
            }
        }

        // Save remaining unmatched STAR records
        for (StarRecord star : validStarRecords) {
            boolean matched = reconciliationRepository
                    .findAll()
                    .stream()
                    .anyMatch(r -> r.getStarKey() != null && r.getStarKey().equals(String.valueOf(star.getId())));
            if (!matched) {
                ReconciliationResultEntity result = new ReconciliationResultEntity();
                result.setAlgoKey(null);
                result.setStarKey(String.valueOf(star.getId()));
                result.setStatus("Unmatched");
                reconciliationRepository.save(result);
            }
        }
    }
}






@RestController
@RequestMapping("/api/reconciliation")
public class ReconciliationController {

    @Autowired
    private ReconciliationService reconciliationService;

    @PostMapping("/run")
    public ResponseEntity<String> runReconciliation() {
        reconciliationService.performReconciliation();
        return ResponseEntity.ok("Reconciliation complete.");
    }
}













@Service
public class ReconciliationService {

    @Autowired
    private AlgoRecordRepository algoRepo;
    @Autowired
    private StarRecordRepository starRepo;
    @Autowired
    private ReconciliationResultRepository resultRepo;
    @Autowired
    private ExcludedRecordRepository excludedRepo;

    public void reconcile(String matchType) {
        List<AlgoRecord> algoList = algoRepo.findAll();
        List<StarRecord> starList = starRepo.findAll();

        Map<String, String> algoMap = new HashMap<>();
        for (AlgoRecord a : algoList) {
            String transformed = a.getAgreementName().replace("_RIMR", "_3CR").replace("_RIMP", "_3CP").trim();
            algoMap.put(transformed, a.getAgreementName());
        }

        Map<String, String> starMap = new HashMap<>();
        List<String> excludedKeys = new ArrayList<>();

        for (StarRecord s : starList) {
            if ("01-JAN-1900".equalsIgnoreCase(s.getMaturityDate())) {
                String key = s.getCrdsPartyCode().trim() + "3" + s.getPostDirection().trim();
                excludedKeys.add(key);

                ExcludedRecordEntity e = new ExcludedRecordEntity();
                e.setAlgoKey("N/A");
                e.setStarKey(key);
                e.setReason("Maturity Date = 01-JAN-1900");
                excludedRepo.save(e);
            } else {
                String key = s.getCrdsPartyCode().trim() + "3" + s.getPostDirection().trim();
                starMap.put(key, s.getCrdsPartyCode());
            }
        }

        Set<String> matchedStar = new HashSet<>();
        for (Map.Entry<String, String> algo : algoMap.entrySet()) {
            List<String> matchingStars = starMap.keySet().stream()
                    .filter(k -> k.equals(algo.getKey()))
                    .collect(Collectors.toList());

            switch (matchType.toLowerCase()) {
                case "one-to-one":
                    if (matchingStars.size() == 1 && !matchedStar.contains(matchingStars.get(0))) {
                        resultRepo.save(newResult(algo.getKey(), matchingStars.get(0), "Match"));
                        matchedStar.add(matchingStars.get(0));
                    } else {
                        resultRepo.save(newResult(algo.getKey(), "N/A", "Mismatch"));
                    }
                    break;
                case "one-to-many":
                    if (!matchingStars.isEmpty()) {
                        for (String s : matchingStars) {
                            resultRepo.save(newResult(algo.getKey(), s, "Match (1-many)"));
                            matchedStar.add(s);
                        }
                    } else {
                        resultRepo.save(newResult(algo.getKey(), "N/A", "Mismatch"));
                    }
                    break;
                case "many-to-one":
                    if (!matchingStars.isEmpty()) {
                        resultRepo.save(newResult(algo.getKey(), matchingStars.get(0), "Match (many-1)"));
                        matchedStar.add(matchingStars.get(0));
                    } else {
                        resultRepo.save(newResult(algo.getKey(), "N/A", "Mismatch"));
                    }
                    break;
                case "many-to-many":
                    if (!matchingStars.isEmpty()) {
                        for (String s : matchingStars) {
                            resultRepo.save(newResult(algo.getKey(), s, "Match (many-many)"));
                            matchedStar.add(s);
                        }
                    } else {
                        resultRepo.save(newResult(algo.getKey(), "N/A", "Mismatch"));
                    }
                    break;
            }
        }

        // Add unmatched STAR records
        for (String s : starMap.keySet()) {
            if (!matchedStar.contains(s)) {
                resultRepo.save(newResult("N/A", s, "Unmatched STAR"));
            }
        }
    }

    private ReconciliationResultEntity newResult(String algoKey, String starKey, String status) {
        ReconciliationResultEntity r = new ReconciliationResultEntity();
        r.setAlgoKey(algoKey);
        r.setStarKey(starKey);
        r.setStatus(status);
        return r;
    }
}








// AlgoRecord.java import jakarta.persistence.*;

@Entity @Table(name = "AGREEMENT_NAME") public class AlgoRecord { @Id private Long id;

@Column(name = "AGREEMENT_NAME")
private String agreementName;

@Column(name = "POST_DIRECTION")
private String postDirection;

public Long getId() {
    return id;
}

public void setId(Long id) {
    this.id = id;
}

public String getAgreementName() {
    return agreementName;
}

public void setAgreementName(String agreementName) {
    this.agreementName = agreementName;
}

public String getPostDirection() {
    return postDirection;
}

public void setPostDirection(String postDirection) {
    this.postDirection = postDirection;
}

}

// StarRecord.java import jakarta.persistence.*;

@Entity @Table(name = "CRDS_PARTY_CODE") public class StarRecord { @Id private Long id;

@Column(name = "CRDS_PARTY_CODE")
private String crdsPartyCode;

@Column(name = "POST_DIRECTION")
private String postDirection;

@Column(name = "MATURITY_DATE")
private String maturityDate;

public Long getId() {
    return id;
}

public void setId(Long id) {
    this.id = id;
}

public String getCrdsPartyCode() {
    return crdsPartyCode;
}

public void setCrdsPartyCode(String crdsPartyCode) {
    this.crdsPartyCode = crdsPartyCode;
}

public String getPostDirection() {
    return postDirection;
}

public void setPostDirection(String postDirection) {
    this.postDirection = postDirection;
}

public String getMaturityDate() {
    return maturityDate;
}

public void setMaturityDate(String maturityDate) {
    this.maturityDate = maturityDate;
}

}

// ReconciliationResultEntity.java import jakarta.persistence.*;

@Entity @Table(name = "RECONCILIATION_RESULTS") public class ReconciliationResultEntity { @Id @GeneratedValue(strategy = GenerationType.IDENTITY) private Long id;

private String algoKey;
private String starKey;
private String status;

public Long getId() {
    return id;
}

public void setId(Long id) {
    this.id = id;
}

public String getAlgoKey() {
    return algoKey;
}

public void setAlgoKey(String algoKey) {
    this.algoKey = algoKey;
}

public String getStarKey() {
    return starKey;
}

public void setStarKey(String starKey) {
    this.starKey = starKey;
}

public String getStatus() {
    return status;
}

public void setStatus(String status) {
    this.status = status;
}

}

// ExcludedRecordEntity.java import jakarta.persistence.*;

@Entity @Table(name = "EXCLUDED_RECORDS") public class ExcludedRecordEntity { @Id @GeneratedValue(strategy = GenerationType.IDENTITY) private Long id;

private String algoKey;
private String starKey;
private String reason;

public Long getId() {
    return id;
}

public void setId(Long id) {
    this.id = id;
}

public String getAlgoKey() {
    return algoKey;
}

public void setAlgoKey(String algoKey) {
    this.algoKey = algoKey;
}

public String getStarKey() {
    return starKey;
}

public void setStarKey(String starKey) {
    this.starKey = starKey;
}

public String getReason() {
    return reason;
}

public void setReason(String reason) {
    this.reason = reason;
}

}

// Repositories package com.example.reconciliation.repository;

import com.example.reconciliation.entity.AlgoRecord; import com.example.reconciliation.entity.StarRecord; import com.example.reconciliation.entity.ExcludedRecordEntity; import com.example.reconciliation.entity.ReconciliationResultEntity; import org.springframework.data.jpa.repository.JpaRepository; import org.springframework.stereotype.Repository;

@Repository public interface AlgoRecordRepository extends JpaRepository<AlgoRecord, Long> { }

@Repository public interface StarRecordRepository extends JpaRepository<StarRecord, Long> { }

@Repository public interface ExcludedRecordRepository extends JpaRepository<ExcludedRecordEntity, Long> { }

@Repository public interface ReconciliationResultRepository extends JpaRepository<ReconciliationResultEntity, Long> { }

// Service package com.example.reconciliation.service;

import com.example.reconciliation.entity.; import com.example.reconciliation.repository.; import org.springframework.beans.factory.annotation.Autowired; import org.springframework.stereotype.Service; import java.util.*; import java.util.stream.Collectors;

@Service public class ReconciliationService {

@Autowired
private AlgoRecordRepository algoRecordRepository;

@Autowired
private StarRecordRepository starRecordRepository;

@Autowired
private ExcludedRecordRepository excludedRecordRepository;

@Autowired
private ReconciliationResultRepository reconciliationResultRepository;

public void reconcile(String matchType) {
    List<AlgoRecord> algoRecords = algoRecordRepository.findAll();
    List<StarRecord> starRecords = starRecordRepository.findAll();

    List<ExcludedRecordEntity> excludedRecords = new ArrayList<>();
    List<StarRecord> filteredStarRecords = new ArrayList<>();

    for (StarRecord star : starRecords) {
        if ("01-JAN-1900".equalsIgnoreCase(star.getMaturityDate())) {
            ExcludedRecordEntity excluded = new ExcludedRecordEntity();
            excluded.setStarkey(star.getCrdsPartyCode());
            excluded.setExcludedValue(star.getMaturityDate());
            excluded.setStatus("Excluded: Maturity Date is 01-JAN-1900");
            excludedRecords.add(excluded);
        } else {
            filteredStarRecords.add(star);
        }
    }

    excludedRecordRepository.saveAll(excludedRecords);

    List<ReconciliationResultEntity> results = new ArrayList<>();

    for (AlgoRecord algo : algoRecords) {
        String algoKey = transformPostDirection(algo.getAgreementName());
        boolean matchFound = false;

        List<StarRecord> matchedStarRecords = filteredStarRecords.stream()
                .filter(star -> algoKey.equals(generateStarKey(star)))
                .collect(Collectors.toList());

        if (!matchedStarRecords.isEmpty()) {
            switch (matchType.toLowerCase()) {
                case "one-to-one":
                    if (matchedStarRecords.size() == 1) {
                        results.add(createResult(algoKey, generateStarKey(matchedStarRecords.get(0)), "Matched"));
                        matchFound = true;
                    }
                    break;
                case "one-to-many":
                    for (StarRecord star : matchedStarRecords) {
                        results.add(createResult(algoKey, generateStarKey(star), "Matched One-to-Many"));
                    }
                    matchFound = true;
                    break;
                case "many-to-one":
                    results.add(createResult(algoKey, generateStarKey(matchedStarRecords.get(0)), "Matched Many-to-One"));
                    matchFound = true;
                    break;
                case "many-to-many":
                    for (StarRecord star : matchedStarRecords) {
                        results.add(createResult(algoKey, generateStarKey(star), "Matched Many-to-Many"));
                    }
                    matchFound = true;
                    break;
            }
        }

        if (!matchFound) {
            results.add(createResult(algoKey, "", "Unmatched"));
        }
    }

    reconciliationResultRepository.saveAll(results);
}

private String transformPostDirection(String name) {
    return name.replace("_RIMP", "_3CP").replace("_RIMR", "_3CR").replaceAll("\\s+", "").trim();
}

private String generateStarKey(StarRecord star) {
    return star.getCrdsPartyCode().replaceAll("\\s+", "") + "3" + star.getPostDirection().replaceAll("\\s+", "").trim();
}

private ReconciliationResultEntity createResult(String algoKey, String starKey, String status) {
    ReconciliationResultEntity result = new ReconciliationResultEntity();
    result.setAlgoKey(algoKey);
    result.setStarKey(starKey);
    result.setStatus(status);
    return result;
}

}




















// File: Record.java package com.example.reconciliation.model;

public class Record { private String algoKey; private String starKey; private String status;

public Record() {}

public Record(String algoKey, String starKey, String status) {
    this.algoKey = algoKey;
    this.starKey = starKey;
    this.status = status;
}

public String getAlgoKey() {
    return algoKey;
}

public void setAlgoKey(String algoKey) {
    this.algoKey = algoKey;
}

public String getStarKey() {
    return starKey;
}

public void setStarKey(String starKey) {
    this.starKey = starKey;
}

public String getStatus() {
    return status;
}

public void setStatus(String status) {
    this.status = status;
}

}

// File: AlgoRecord.java package com.example.reconciliation.entity;

import jakarta.persistence.*;

@Entity @Table(name = "ALGO_RECORDS") public class AlgoRecord {

@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;

@Column(name = "AGREEMENT_NAME")
private String agreementName;

// Getters and Setters
public Long getId() {
    return id;
}

public void setId(Long id) {
    this.id = id;
}

public String getAgreementName() {
    return agreementName;
}

public void setAgreementName(String agreementName) {
    this.agreementName = agreementName;
}

}

// File: StarRecord.java package com.example.reconciliation.entity;

import jakarta.persistence.*;

@Entity @Table(name = "STAR_RECORDS") public class StarRecord {

@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;

@Column(name = "CRDS_PARTY_CODE")
private String crdsPartyCode;

@Column(name = "POST_DIRECTION")
private String postDirection;

@Column(name = "MATURITY_DATE")
private String maturityDate;

// Getters and Setters
public Long getId() {
    return id;
}

public void setId(Long id) {
    this.id = id;
}

public String getCrdsPartyCode() {
    return crdsPartyCode;
}

public void setCrdsPartyCode(String crdsPartyCode) {
    this.crdsPartyCode = crdsPartyCode;
}

public String getPostDirection() {
    return postDirection;
}

public void setPostDirection(String postDirection) {
    this.postDirection = postDirection;
}

public String getMaturityDate() {
    return maturityDate;
}

public void setMaturityDate(String maturityDate) {
    this.maturityDate = maturityDate;
}

}

// File: ExcludedRecord.java package com.example.reconciliation.entity;

import jakarta.persistence.*;

@Entity @Table(name = "EXCLUDED_RECORDS") public class ExcludedRecord {

@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;
private String algoKey;
private String starKey;
private String reason;

// Getters and Setters
public Long getId() {
    return id;
}

public void setId(Long id) {
    this.id = id;
}

public String getAlgoKey() {
    return algoKey;
}

public void setAlgoKey(String algoKey) {
    this.algoKey = algoKey;
}

public String getStarKey() {
    return starKey;
}

public void setStarKey(String starKey) {
    this.starKey = starKey;
}

public String getReason() {
    return reason;
}

public void setReason(String reason) {
    this.reason = reason;
}

}

// File: ResultRecord.java package com.example.reconciliation.entity;

import jakarta.persistence.*;

@Entity @Table(name = "RECONCILIATION_RESULTS") public class ResultRecord { @Id @GeneratedValue(strategy = GenerationType.IDENTITY) private Long id; private String algoKey; private String starKey; private String status;

// Getters and Setters
public Long getId() {
    return id;
}

public void setId(Long id) {
    this.id = id;
}

public String getAlgoKey() {
    return algoKey;
}

public void setAlgoKey(String algoKey) {
    this.algoKey = algoKey;
}

public String getStarKey() {
    return starKey;
}

public void setStarKey(String starKey) {
    this.starKey = starKey;
}

public String getStatus() {
    return status;
}

public void setStatus(String status) {
    this.status = status;
}

}

// File: AlgoRecordRepository.java package com.example.reconciliation.repository;

import com.example.reconciliation.entity.AlgoRecord; import org.springframework.data.jpa.repository.JpaRepository;

public interface AlgoRecordRepository extends JpaRepository<AlgoRecord, Long> {}

// File: StarRecordRepository.java package com.example.reconciliation.repository;

import com.example.reconciliation.entity.StarRecord; import org.springframework.data.jpa.repository.JpaRepository;

public interface StarRecordRepository extends JpaRepository<StarRecord, Long> {}

// File: ExcludedRecordRepository.java package com.example.reconciliation.repository;

import com.example.reconciliation.entity.ExcludedRecord; import org.springframework.data.jpa.repository.JpaRepository;

public interface ExcludedRecordRepository extends JpaRepository<ExcludedRecord, Long> {}

// File: ResultRecordRepository.java package com.example.reconciliation.repository;

import com.example.reconciliation.entity.ResultRecord; import org.springframework.data.jpa.repository.JpaRepository;

public interface ResultRecordRepository extends JpaRepository<ResultRecord, Long> {}

// File: ReconciliationService.java package com.example.reconciliation.service;

import com.example.reconciliation.entity.; import com.example.reconciliation.repository.; import org.springframework.stereotype.Service;

import java.util.*;

@Service public class ReconciliationService {

private final AlgoRecordRepository algoRepo;
private final StarRecordRepository starRepo;
private final ExcludedRecordRepository excludedRepo;
private final ResultRecordRepository resultRepo;

public ReconciliationService(AlgoRecordRepository algoRepo, StarRecordRepository starRepo,
                             ExcludedRecordRepository excludedRepo, ResultRecordRepository resultRepo) {
    this.algoRepo = algoRepo;
    this.starRepo = starRepo;
    this.excludedRepo = excludedRepo;
    this.resultRepo = resultRepo;
}

public void runExclusionAndSave() {
    List<AlgoRecord> algoList = algoRepo.findAll();
    List<StarRecord> starList = starRepo.findAll();

    Map<Long, String> algoKeyMap = new HashMap<>();
    for (AlgoRecord algo : algoList) {
        String key = algo.getAgreementName().replace("_RIMP", "_3CP").replace("_RIMR", "_3CR").trim();
        algoKeyMap.put(algo.getId(), key);
    }

    for (StarRecord star : starList) {
        String starKey = star.getCrdsPartyCode().trim() + "3" + star.getPostDirection().trim();
        if ("01-JAN-1900".equalsIgnoreCase(star.getMaturityDate().trim())) {
            ExcludedRecord excluded = new ExcludedRecord();
            excluded.setStarKey(starKey);
            excluded.setAlgoKey("<Not Linked>");
            excluded.setReason("Maturity Date = 01-JAN-1900");
            excludedRepo.save(excluded);
        }
    }
}

public void performMatching(String matchType) {
    List<AlgoRecord> algoList = algoRepo.findAll();
    List<StarRecord> starList = starRepo.findAll();
    List<ExcludedRecord> excluded = excludedRepo.findAll();
    Set<String> excludedStarKeys = new HashSet<>();

    for (ExcludedRecord ex : excluded) {
        excludedStarKeys.add(ex.getStarKey());
    }

    Map<String, Long> starMap = new HashMap<>();
    for (StarRecord star : starList) {
        String key = star.getCrdsPartyCode().trim() + "3" + star.getPostDirection().trim();
        if (!excludedStarKeys.contains(key)) {
            starMap.put(key, star.getId());
        }
    }

    for (AlgoRecord algo : algoList) {
        String algoKey = algo.getAgreementName().replace("_RIMP", "_3CP").replace("_RIMR", "_3CR").trim();
        if (starMap.containsKey(algoKey)) {
            ResultRecord r = new ResultRecord();
            r.setAlgoKey(algoKey);
            r.setStarKey(algoKey);
            r.setStatus("Matched");
            resultRepo.save(r);
        } else {
            ResultRecord r = new ResultRecord();
            r.setAlgoKey(algoKey);
            r.setStarKey("<No Match>");
            r.setStatus("Unmatched");
            resultRepo.save(r);
        }
    }
}

}

// File: ReconciliationController.java package com.example.reconciliation.controller;

import com.example.reconciliation.service.ReconciliationService; import org.springframework.web.bind.annotation.*;

@RestController @RequestMapping("/api/reconciliation") public class ReconciliationController {

private final ReconciliationService service;

public ReconciliationController(ReconciliationService service) {
    this.service = service;
}

@GetMapping("/exclude")
public String runExclusion() {
    service.runExclusionAndSave();
    return "Exclusion rule applied and excluded records saved.";
}

@GetMapping("/match/{type}")
public String runMatching(@PathVariable("type") String type) {
    service.performMatching(type);
    return "Matching completed with type: " + type;
}

}








// AlgoRecord.java @Entity @Table(name = "Initial_margin_EOD") public class AlgoRecord { @Id @Column(name = "AGREEMENT_NAME") private String agreementName;

// Getters and Setters
public String getAgreementName() { return agreementName; }
public void setAgreementName(String agreementName) { this.agreementName = agreementName; }

}

// StarRecord.java @Entity @Table(name = "StarAlgo") public class StarRecord { @Id @Column(name = "CRDS_PARTY_CODE") private String crdsPartyCode;

@Column(name = "POST_DIRECTION")
private String postDirection;

@Column(name = "MATURITY_DATE")
private Timestamp maturityDate;

// Getters and Setters
public String getCrdsPartyCode() { return crdsPartyCode; }
public void setCrdsPartyCode(String crdsPartyCode) { this.crdsPartyCode = crdsPartyCode; }

public String getPostDirection() { return postDirection; }
public void setPostDirection(String postDirection) { this.postDirection = postDirection; }

public Timestamp getMaturityDate() { return maturityDate; }
public void setMaturityDate(Timestamp maturityDate) { this.maturityDate = maturityDate; }

}

// NonExcludedAlgo.java @Entity public class NonExcludedAlgo { @Id private String agreementName; private String transformedAgreementName;

// Getters and Setters
public String getAgreementName() { return agreementName; }
public void setAgreementName(String agreementName) { this.agreementName = agreementName; }

public String getTransformedAgreementName() { return transformedAgreementName; }
public void setTransformedAgreementName(String transformedAgreementName) { this.transformedAgreementName = transformedAgreementName; }

}

// NonExcludedStar.java @Entity public class NonExcludedStar { @Id private String crdsPartyCode;

private String postDirection;
private Timestamp maturityDate;
private String transformedPartyCode;

// Getters and Setters
public String getCrdsPartyCode() { return crdsPartyCode; }
public void setCrdsPartyCode(String crdsPartyCode) { this.crdsPartyCode = crdsPartyCode; }

public String getPostDirection() { return postDirection; }
public void setPostDirection(String postDirection) { this.postDirection = postDirection; }

public Timestamp getMaturityDate() { return maturityDate; }
public void setMaturityDate(Timestamp maturityDate) { this.maturityDate = maturityDate; }

public String getTransformedPartyCode() { return transformedPartyCode; }
public void setTransformedPartyCode(String transformedPartyCode) { this.transformedPartyCode = transformedPartyCode; }

}

// Repositories @Repository public interface AlgoRecordRepository extends JpaRepository<AlgoRecord, String> {}

@Repository public interface StarRecordRepository extends JpaRepository<StarRecord, String> {}

@Repository public interface NonExcludedAlgoRepository extends JpaRepository<NonExcludedAlgo, String> {}

@Repository public interface NonExcludedStarRepository extends JpaRepository<NonExcludedStar, String> {}

// ReconciliationService.java @Service public class ReconciliationService { @Autowired private AlgoRecordRepository algoRepo;

@Autowired
private StarRecordRepository starRepo;

@Autowired
private NonExcludedAlgoRepository nonExcludedAlgoRepo;

@Autowired
private NonExcludedStarRepository nonExcludedStarRepo;

public void transformAndFilter() {
    List<AlgoRecord> algoRecords = algoRepo.findAll();
    List<StarRecord> starRecords = starRepo.findAll();

    List<NonExcludedAlgo> nonExcludedAlgos = new ArrayList<>();
    List<NonExcludedStar> nonExcludedStars = new ArrayList<>();

    for (AlgoRecord algo : algoRecords) {
        String transformed = algo.getAgreementName()
                .replace("_RIMP", "_3CP")
                .replace("_RIMR", "_3CR");
        NonExcludedAlgo neAlgo = new NonExcludedAlgo();
        neAlgo.setAgreementName(algo.getAgreementName());
        neAlgo.setTransformedAgreementName(transformed);
        nonExcludedAlgos.add(neAlgo);
    }

    for (StarRecord star : starRecords) {
        if (star.getMaturityDate() != null &&
            star.getMaturityDate().toLocalDateTime().toLocalDate().equals(LocalDate.of(1900, 1, 1))) {
            continue;
        }
        String transformed = star.getCrdsPartyCode() + "3" + star.getPostDirection();
        NonExcludedStar neStar = new NonExcludedStar();
        neStar.setCrdsPartyCode(star.getCrdsPartyCode());
        neStar.setPostDirection(star.getPostDirection());
        neStar.setMaturityDate(star.getMaturityDate());
        neStar.setTransformedPartyCode(transformed);
        nonExcludedStars.add(neStar);
    }

    nonExcludedAlgoRepo.saveAll(nonExcludedAlgos);
    nonExcludedStarRepo.saveAll(nonExcludedStars);
}

// Add matchOneToOne(), matchOneToMany(), etc. if needed

}package com.example.reconciliation.controller;

import com.example.reconciliation.service.ReconciliationService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/reconciliation")
public class ReconciliationController {

    @Autowired
    private ReconciliationService reconciliationService;

    // Run exclusion + transformation logic
    @PostMapping("/transform-and-exclude")
    public String transformAndExclude() {
        reconciliationService.performTransformationAndExclusion();
        return "Transformation and exclusion completed. Non-excluded records stored.";
    }

    // Run one-to-one matching on non-excluded records
    @GetMapping("/match/one-to-one")
    public String oneToOneMatching() {
        reconciliationService.matchOneToOne();
        return "One-to-one matching completed.";
    }

    // You can later add other matching types here like /match/one-to-many etc.
}

✅✅✅✅✅
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

    public ReconciliationResult reconcile(String algoPath, String starPath, String action, String matchType) {
        try {
            FileReader readerAlgo = new FileReader(algoPath);
            List<CSVRecord> algoRecords = (List<CSVRecord>) CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(readerAlgo);

            FileReader readerStar = new FileReader(starPath);
            List<CSVRecord> starRecords = (List<CSVRecord>) CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(readerStar);

            List<CSVRecord> excluded = new ArrayList<>();
            List<CSVRecord> filteredStar = starRecords;

            if ("exclude".equalsIgnoreCase(action)) {
                filteredStar = applyExclusionRule(starRecords, excluded);
            }

            return match(algoRecords, filteredStar, matchType.toLowerCase(), excluded);

        } catch (Exception e) {
            e.printStackTrace();
            return new ReconciliationResult(Collections.emptyList(), Collections.emptyList(), Collections.emptyList());
        }
    }

    private List<CSVRecord> applyExclusionRule(List<CSVRecord> starRecords, List<CSVRecord> excluded) {
        List<CSVRecord> filtered = new ArrayList<>();
        for (CSVRecord record : starRecords) {
            String maturityDateStr = record.get("Maturity Date").trim();
            if (!maturityDateStr.equalsIgnoreCase("01-JAN-1900")) {
                filtered.add(record);
            } else {
                excluded.add(record);
            }
        }
        return filtered;
    }

    private ReconciliationResult match(List<CSVRecord> algoRecords, List<CSVRecord> starRecords, String type, List<CSVRecord> excludedStar) {
        List<Record> matched = new ArrayList<>();
        List<Record> unmatched = new ArrayList<>();
        List<Record> excluded = new ArrayList<>();

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
                    if (!matchingStarIdxs.isEmpty()) {
                        for (Integer idx : matchingStarIdxs) {
                            matched.add(new Record(algoKey, starKeys.get(idx), "Match (Many-to-Many)"));
                            matchedStarIndices.add(idx);
                        }
                        matchedAlgoIndices.add(i);
                    } else {
                        unmatched.add(new Record(algoKey, "<No Match>", "Mismatch"));
                    }
                    break;
                default:
                    unmatched.add(new Record(algoKey, "<No Match>", "Unknown Matching Type"));
            }
        }

        for (int i = 0; i < starKeys.size(); i++) {
            if (!matchedStarIndices.contains(i)) {
                unmatched.add(new Record("<No Match>", starKeys.get(i), "Unmatched STAR Key"));
            }
        }

        for (CSVRecord excludedRecord : excludedStar) {
            String excludedKey = excludedRecord.get("CRDS Party Code").replaceAll("\\s+", "") +
                    "3" + excludedRecord.get("Post Direction").replaceAll("\\s+", "");
            excluded.add(new Record("<Excluded>", excludedKey, "Excluded by Rule (Maturity Date = 01-JAN-1900)"));
        }

        return new ReconciliationResult(matched, unmatched, excluded);
    }
}




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

    @GetMapping
    public ReconciliationResult reconcile(
            @RequestParam String algoPath,
            @RequestParam String starPath,
            @RequestParam String action,
            @RequestParam String matchType
    ) {
        return reconciliationService.reconcile(algoPath, starPath, action, matchType);
    }
}
















package com.example.demo.service;

import com.example.demo.model.ReconciliationResult;
import org.apache.commons.csv.CSVFormat;
import org.apache.commons.csv.CSVRecord;
import org.springframework.stereotype.Service;

import java.io.FileReader;
import java.io.Reader;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

@Service
public class ReconciliationService {

    public ReconciliationResult reconcile(String algoPath, String starPath, String action, String matchType) {
        try {
            Reader readerAlgo = new FileReader(algoPath);
            Reader readerStar = new FileReader(starPath);

            // Convert Iterable<CSVRecord> to List<CSVRecord>
            List<CSVRecord> algoRecords = new ArrayList<>();
            CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(readerAlgo).forEach(algoRecords::add);

            List<CSVRecord> starRecords = new ArrayList<>();
            CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(readerStar).forEach(starRecords::add);

            List<CSVRecord> excluded = new ArrayList<>();
            List<CSVRecord> filteredStar = starRecords;

            if ("exclude".equalsIgnoreCase(action)) {
                filteredStar = applyExclusionRule(starRecords, excluded);
            }

            return match(algoRecords, filteredStar, matchType.toLowerCase());

        } catch (Exception e) {
            e.printStackTrace();
            return new ReconciliationResult(Collections.emptyList(), Collections.emptyList());
        }
    }

    private List<CSVRecord> applyExclusionRule(List<CSVRecord> records, List<CSVRecord> excluded) {
        List<CSVRecord> result = new ArrayList<>();
        for (CSVRecord record : records) {
            String maturityDate = record.get("MATURITY_DATE");
            if ("1900-01-01".equals(maturityDate)) {
                excluded.add(record);
            } else {
                result.add(record);
            }
        }
        return result;
    }

    private ReconciliationResult match(List<CSVRecord> algo, List<CSVRecord> star, String matchType) {
        // Simple 1-1 matching stub
        List<CSVRecord> matched = new ArrayList<>();
        List<CSVRecord> unmatched = new ArrayList<>(algo);

        for (CSVRecord algoRec : algo) {
            for (CSVRecord starRec : star) {
                if (algoRec.get("ID").equals(starRec.get("ID"))) {
                    matched.add(algoRec);
                    unmatched.remove(algoRec);
                    break;
                }
            }
        }

        return new ReconciliationResult(matched, unmatched);
    }
}




package com.example.demo.controller;

import com.example.demo.model.ReconciliationResult;
import com.example.demo.service.ReconciliationService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/reconcile")
public class ReconciliationController {

    @Autowired
    private ReconciliationService reconciliationService;

    @GetMapping("/run")
    public ReconciliationResult runReconciliation() {
        // Provide absolute or relative file paths here
        String algoFilePath = "C:/Users/YourUser/Desktop/input/algo.csv";
        String starFilePath = "C:/Users/YourUser/Desktop/input/star.csv";

        String action = "exclude";         // or "none"
        String matchType = "1-1";          // use match type like "1-1", "1-many", etc.

        return reconciliationService.reconcile(algoFilePath, starFilePath, action, matchType);
    }
}









package com.example.reconciliation.service;

import com.example.reconciliation.model.Record;
import com.example.reconciliation.model.ReconciliationResult;
import org.apache.commons.csv.CSVFormat;
import org.apache.commons.csv.CSVParser;
import org.apache.commons.csv.CSVRecord;
import org.springframework.stereotype.Service;

import java.io.FileReader;
import java.util.*;

@Service
public class ReconciliationService {

    public ReconciliationResult reconcile(String algoFilePath, String starFilePath, String action, String matchType) {
        try {
            CSVParser algoParser = CSVFormat.DEFAULT
                    .withFirstRecordAsHeader()
                    .parse(new FileReader(algoFilePath));

            CSVParser starParser = CSVFormat.DEFAULT
                    .withFirstRecordAsHeader()
                    .parse(new FileReader(starFilePath));

            List<CSVRecord> algoRecords = algoParser.getRecords();
            List<CSVRecord> starRecords = starParser.getRecords();

            List<CSVRecord> excluded = new ArrayList<>();
            List<CSVRecord> filteredStar = starRecords;

            if ("exclude".equalsIgnoreCase(action)) {
                filteredStar = applyExclusionRule(starRecords, excluded);
            }

            return match(algoRecords, filteredStar, matchType.toLowerCase(), excluded);

        } catch (Exception e) {
            e.printStackTrace();
            return new ReconciliationResult(Collections.emptyList(), Collections.emptyList(), Collections.emptyList());
        }
    }

    private List<CSVRecord> applyExclusionRule(List<CSVRecord> starRecords, List<CSVRecord> excluded) {
        List<CSVRecord> filtered = new ArrayList<>();
        for (CSVRecord record : starRecords) {
            String maturityDateStr = record.get("Maturity Date").trim();
            if (!maturityDateStr.equalsIgnoreCase("01-JAN-1900")) {
                filtered.add(record);
            } else {
                excluded.add(record);
            }
        }
        return filtered;
    }

    private ReconciliationResult match(List<CSVRecord> algoRecords, List<CSVRecord> starRecords, String type, List<CSVRecord> excludedStar) {
        List<Record> matched = new ArrayList<>();
        List<Record> unmatched = new ArrayList<>();
        List<Record> excluded = new ArrayList<>();

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
                    if (!matchingStarIdxs.isEmpty()) {
                        for (Integer idx : matchingStarIdxs) {
                            matched.add(new Record(algoKey, starKeys.get(idx), "Match (Many-to-Many)"));
                            matchedStarIndices.add(idx);
                        }
                        matchedAlgoIndices.add(i);
                    } else {
                        unmatched.add(new Record(algoKey, "<No Match>", "Mismatch"));
                    }
                    break;
                default:
                    unmatched.add(new Record(algoKey, "<No Match>", "Unknown Matching Type"));
            }
        }

        for (int i = 0; i < starKeys.size(); i++) {
            if (!matchedStarIndices.contains(i)) {
                unmatched.add(new Record("<No Match>", starKeys.get(i), "Unmatched STAR Key"));
            }
        }

        for (CSVRecord excludedRecord : excludedStar) {
            String excludedKey = excludedRecord.get("CRDS Party Code").replaceAll("\\s+", "") +
                    "3" + excludedRecord.get("Post Direction").replaceAll("\\s+", "");
            excluded.add(new Record("<Excluded>", excludedKey, "Excluded by Rule (Maturity Date = 01-JAN-1900)"));
        }

        return new ReconciliationResult(matched, unmatched, excluded);
    }
}

@GetMapping("/run")
public ReconciliationResult runReconciliation() {
    String algoPath = "C:/path/to/algo.csv";
    String starPath = "C:/path/to/star.csv";
    return reconciliationService.reconcile(algoPath, starPath, "exclude", "one-to-one");
}











Here is the full updated Spring Boot Reconciliation implementation with:

/exclude endpoint – reads CSVs, applies exclusion and stores non-excluded records in memory.

/match/{type} endpoint – performs matching on in-memory stored data (1-1, 1-many, many-1, many-many).



---

✅ 1. ReconciliationController.java

package com.example.reconciliation.controller;

import com.example.reconciliation.model.ReconciliationResult;
import com.example.reconciliation.service.ReconciliationService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/reconciliation")
public class ReconciliationController {

    @Autowired
    private ReconciliationService reconciliationService;

    @GetMapping("/exclude")
    public String applyExclusion() {
        String algoPath = "C:/path/to/algo.csv"; // Update path
        String starPath = "C:/path/to/star.csv"; // Update path

        reconciliationService.excludeAndStore(algoPath, starPath);
        return "Exclusion complete. Now call /reconciliation/match/{type} to perform matching.";
    }

    @GetMapping("/match/{type}")
    public ReconciliationResult match(@PathVariable String type) {
        return reconciliationService.matchOnStoredData(type.toLowerCase());
    }
}


---

✅ 2. ReconciliationService.java

package com.example.reconciliation.service;

import com.example.reconciliation.model.Record;
import com.example.reconciliation.model.ReconciliationResult;
import org.apache.commons.csv.CSVFormat;
import org.apache.commons.csv.CSVParser;
import org.apache.commons.csv.CSVRecord;
import org.springframework.stereotype.Service;

import java.io.FileReader;
import java.util.*;

@Service
public class ReconciliationService {

    private List<CSVRecord> nonExcludedAlgo = new ArrayList<>();
    private List<CSVRecord> nonExcludedStar = new ArrayList<>();
    private List<CSVRecord> excludedStar = new ArrayList<>();

    public void excludeAndStore(String algoPath, String starPath) {
        try {
            CSVParser algoParser = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(new FileReader(algoPath));
            CSVParser starParser = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(new FileReader(starPath));

            List<CSVRecord> algoRecords = algoParser.getRecords();
            List<CSVRecord> starRecords = starParser.getRecords();

            this.nonExcludedAlgo = algoRecords;
            this.excludedStar = new ArrayList<>();
            this.nonExcludedStar = applyExclusionRule(starRecords, this.excludedStar);

        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public ReconciliationResult matchOnStoredData(String matchType) {
        return match(nonExcludedAlgo, nonExcludedStar, matchType, excludedStar);
    }

    private List<CSVRecord> applyExclusionRule(List<CSVRecord> starRecords, List<CSVRecord> excluded) {
        List<CSVRecord> filtered = new ArrayList<>();
        for (CSVRecord record : starRecords) {
            String maturityDate = record.get("Maturity Date").trim();
            if (!maturityDate.contains("1900")) {
                filtered.add(record);
            } else {
                excluded.add(record);
            }
        }
        return filtered;
    }

    private ReconciliationResult match(List<CSVRecord> algoRecords, List<CSVRecord> starRecords, String type, List<CSVRecord> excludedStar) {
        List<Record> matched = new ArrayList<>();
        List<Record> unmatched = new ArrayList<>();
        List<Record> excluded = new ArrayList<>();

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
                    if (!matchingStarIdxs.isEmpty()) {
                        for (Integer idx : matchingStarIdxs) {
                            matched.add(new Record(algoKey, starKeys.get(idx), "Match (Many-to-Many)"));
                            matchedStarIndices.add(idx);
                        }
                        matchedAlgoIndices.add(i);
                    } else {
                        unmatched.add(new Record(algoKey, "<No Match>", "Mismatch"));
                    }
                    break;
                default:
                    unmatched.add(new Record(algoKey, "<No Match>", "Unknown Matching Type"));
            }
        }

        for (int i = 0; i < starKeys.size(); i++) {
            if (!matchedStarIndices.contains(i)) {
                unmatched.add(new Record("<No Match>", starKeys.get(i), "Unmatched STAR Key"));
            }
        }

        for (CSVRecord excludedRecord : excludedStar) {
            String excludedKey = excludedRecord.get("CRDS Party Code").replaceAll("\\s+", "") +
                    "3" + excludedRecord.get("Post Direction").replaceAll("\\s+", "");
            excluded.add(new Record("<Excluded>", excludedKey, "Excluded by Rule (Maturity Date = 1900)"));
        }

        return new ReconciliationResult(matched, unmatched, excluded);
    }
}


---

✅ 3. Record.java

package com.example.reconciliation.model;

public class Record {
    private String algoKey;
    private String starKey;
    private String status;

    public Record() {}

    public Record(String algoKey, String starKey, String status) {
        this.algoKey = algoKey;
        this.starKey = starKey;
        this.status = status;
    }

    public String getAlgoKey() { return algoKey; }
    public void setAlgoKey(String algoKey) { this.algoKey = algoKey; }

    public String getStarKey() { return starKey; }
    public void setStarKey(String starKey) { this.starKey = starKey; }

    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }
}


---

✅ 4. ReconciliationResult.java

package com.example.reconciliation.model;

import java.util.List;

public class ReconciliationResult {
    private List<Record> matched;
    private List<Record> unmatched;
    private List<Record> excluded;

    public ReconciliationResult() {}

    public ReconciliationResult(List<Record> matched, List<Record> unmatched, List<Record> excluded) {
        this.matched = matched;
        this.unmatched = unmatched;
        this.excluded = excluded;
    }

    public List<Record> getMatched() { return matched; }
    public void setMatched(List<Record> matched) { this.matched = matched; }

    public List<Record> getUnmatched() { return unmatched; }
    public void setUnmatched(List<Record> unmatched) { this.unmatched = unmatched; }

    public List<Record> getExcluded() { return excluded; }
    public void setExcluded(List<Record> excluded) { this.excluded = excluded; }
}


---

✅ Example Calls

GET http://localhost:8080/reconciliation/exclude → applies exclusion, stores transformed data.

GET http://localhost:8080/reconciliation/match/one-to-one → performs matching on stored data.



---

Let me know if you want this to also store to a database or return counts along with the data.



public class ReconciliationResult {
    private List<Record> matched;
    private List<Record> unmatched;
    private List<Record> excluded;
    private int matchedCount;
    private int unmatchedCount;
    private int excludedCount;

    // Constructors
    public ReconciliationResult() {}

    public ReconciliationResult(List<Record> matched, List<Record> unmatched, List<Record> excluded) {
        this.matched = matched;
        this.unmatched = unmatched;
        this.excluded = excluded;
        this.matchedCount = matched != null ? matched.size() : 0;
        this.unmatchedCount = unmatched != null ? unmatched.size() : 0;
        this.excludedCount = excluded != null ? excluded.size() : 0;
    }

    // Getters and setters
    public List<Record> getMatched() { return matched; }
    public void setMatched(List<Record> matched) {
        this.matched = matched;
        this.matchedCount = matched != null ? matched.size() : 0;
    }

    public List<Record> getUnmatched() { return unmatched; }
    public void setUnmatched(List<Record> unmatched) {
        this.unmatched = unmatched;
        this.unmatchedCount = unmatched != null ? unmatched.size() : 0;
    }

    public List<Record> getExcluded() { return excluded; }
    public void setExcluded(List<Record> excluded) {
        this.excluded = excluded;
        this.excludedCount = excluded != null ? excluded.size() : 0;
    }

    public int getMatchedCount() { return matchedCount; }
    public int getUnmatchedCount() { return unmatchedCount; }
    public int getExcludedCount() { return excludedCount; }
}
