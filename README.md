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
