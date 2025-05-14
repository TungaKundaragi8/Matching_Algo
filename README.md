@Service
public class ReconciliationService {

    private List<Map<String, String>> nonExcludedFile1;
    private List<Map<String, String>> nonExcludedFile2;

    public Map<String, Object> excludeRecords(String file1Path, String file2Path, String exclusionColumn, String exclusionValue) {
        List<Map<String, String>> file1Records = readCSV(file1Path);
        List<Map<String, String>> file2Records = readCSV(file2Path);

        Predicate<Map<String, String>> excludePredicate = record ->
                exclusionValue.equalsIgnoreCase(record.getOrDefault(exclusionColumn, "").trim());

        Map<String, Object> response = new HashMap<>();
        List<Map<String, String>> excluded1 = file1Records.stream().filter(excludePredicate).toList();
        List<Map<String, String>> nonExcluded1 = file1Records.stream().filter(excludePredicate.negate()).toList();

        List<Map<String, String>> excluded2 = file2Records.stream().filter(excludePredicate).toList();
        List<Map<String, String>> nonExcluded2 = file2Records.stream().filter(excludePredicate.negate()).toList();

        nonExcludedFile1 = nonExcluded1;
        nonExcludedFile2 = nonExcluded2;

        response.put("exclusionSummary", Map.of(
                "excludedCountFile1", excluded1.size(),
                "nonExcludedCountFile1", nonExcluded1.size(),
                "excludedCountFile2", excluded2.size(),
                "nonExcludedCountFile2", nonExcluded2.size()
        ));
        response.put("excludedRecordsFile1", excluded1);
        response.put("excludedRecordsFile2", excluded2);
        response.put("nonExcludedRecordsFile1", nonExcluded1);
        response.put("nonExcludedRecordsFile2", nonExcluded2);

        return response;
    }

    public Map<String, Object> matchRecords(String matchType) {
        List<Map<String, String>> file1 = new ArrayList<>(nonExcludedFile1);
        List<Map<String, String>> file2 = new ArrayList<>(nonExcludedFile2);

        List<Map<String, Object>> matched = new ArrayList<>();
        List<Map<String, Object>> unmatchedFile1 = new ArrayList<>(file1);
        List<Map<String, Object>> unmatchedFile2 = new ArrayList<>(file2);

        for (Map<String, String> rec1 : file1) {
            for (Map<String, String> rec2 : file2) {
                if (rec1.equals(rec2)) {
                    matched.add(Map.of("file1", rec1, "file2", rec2, "status", "MATCHED"));
                    unmatchedFile1.remove(rec1);
                    unmatchedFile2.remove(rec2);
                    break;
                }
            }
        }

        Map<String, Object> result = new HashMap<>();
        result.put("matchType", matchType);
        result.put("matchedCount", matched.size());
        result.put("unmatchedFile1Count", unmatchedFile1.size());
        result.put("unmatchedFile2Count", unmatchedFile2.size());
        result.put("matchedRecords", matched);
        result.put("unmatchedFile1Records", unmatchedFile1);
        result.put("unmatchedFile2Records", unmatchedFile2);

        return result;
    }

    private List<Map<String, String>> readCSV(String filePath) {
        List<Map<String, String>> records = new ArrayList<>();
        try (BufferedReader reader = Files.newBufferedReader(Paths.get(filePath));
             CSVParser csvParser = new CSVParser(reader, CSVFormat.DEFAULT.withFirstRecordAsHeader())) {
            for (CSVRecord record : csvParser) {
                Map<String, String> row = new HashMap<>();
                record.toMap().forEach(row::put);
                records.add(row);
            }
        } catch (IOException e) {
            throw new RuntimeException("Failed to read file: " + filePath, e);
        }
        return records;
    }
}







@RestController
@RequestMapping("/reconciliation")
public class ReconciliationController {

    @Autowired
    private ReconciliationService reconciliationService;

    @GetMapping("/exclude")
    public ResponseEntity<Map<String, Object>> excludeRecords(
            @RequestParam String file1Path,
            @RequestParam String file2Path,
            @RequestParam String exclusionColumn,
            @RequestParam String exclusionValue
    ) {
        Map<String, Object> result = reconciliationService.excludeRecords(file1Path, file2Path, exclusionColumn, exclusionValue);
        return ResponseEntity.ok(result);
    }

    @GetMapping("/match")
    public ResponseEntity<Map<String, Object>> matchRecords(
            @RequestParam String matchType // "1-1", "1-many", "many-1", "many-many"
    ) {
        Map<String, Object> result = reconciliationService.matchRecords(matchType);
        return ResponseEntity.ok(result);
    }
}






@GetMapping("/reconciliation/match/{type}")
public ResponseEntity<?> matchRecords(
    @PathVariable("type") String matchType,
    @RequestParam("column1") String column1,  // from ALGO
    @RequestParam("column2") String column2   // from STAR
) {
    reconciliationService.performMatching(matchType, column1, column2);
    return ResponseEntity.ok("Matching complete");
}





public void performMatching(String matchType, String column1, String column2) {
    List<AlgoRecord> algoRecords = algoRepository.findAllNonExcluded();
    List<StarRecord> starRecords = starRepository.findAllNonExcluded();

    Map<String, AlgoRecord> algoMap = new HashMap<>();
    for (AlgoRecord a : algoRecords) {
        String key = getFieldValue(a, column1);
        algoMap.put(key, a);
    }

    for (StarRecord s : starRecords) {
        String key = getFieldValue(s, column2);
        if (algoMap.containsKey(key)) {
            // Save as matched
        } else {
            // Save as unmatched
        }
    }
}

