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





@Service
public class ReconciliationService {

    private List<Map<String, String>> nonExcludedFile1 = new ArrayList<>();
    private List<Map<String, String>> nonExcludedFile2 = new ArrayList<>();
    private List<Map<String, String>> excludedFile1 = new ArrayList<>();

    public Map<String, Object> applyExclusion(
            String file1Path,
            String file2Path,
            String exclusionColumn,
            String exclusionValue
    ) {
        nonExcludedFile1.clear();
        nonExcludedFile2.clear();
        excludedFile1.clear();

        List<Map<String, String>> file1Data = readCsv(file1Path);
        List<Map<String, String>> file2Data = readCsv(file2Path);

        for (Map<String, String> row : file1Data) {
            if (exclusionValue.equalsIgnoreCase(row.getOrDefault(exclusionColumn, ""))) {
                excludedFile1.add(row);
            } else {
                nonExcludedFile1.add(row);
            }
        }

        nonExcludedFile2.addAll(file2Data); // assuming no exclusion needed on file 2

        Map<String, Object> result = new HashMap<>();
        result.put("excludedCount", excludedFile1.size());
        result.put("nonExcludedCount", nonExcludedFile1.size());
        result.put("excludedRecords", excludedFile1);
        result.put("nonExcludedRecords", nonExcludedFile1);
        return result;
    }

    public Map<String, Object> performMatching(
            String type,
            String column1,
            String column2
    ) {
        List<Map<String, String>> matched = new ArrayList<>();
        List<Map<String, String>> unmatched = new ArrayList<>();

        Set<String> seen = new HashSet<>();

        for (Map<String, String> row1 : nonExcludedFile1) {
            String val1 = row1.getOrDefault(column1, "").trim();
            boolean found = false;

            for (Map<String, String> row2 : nonExcludedFile2) {
                String val2 = row2.getOrDefault(column2, "").trim();

                if (val1.equalsIgnoreCase(val2)) {
                    Map<String, String> combined = new HashMap<>();
                    combined.putAll(row1);
                    combined.putAll(row2);
                    matched.add(combined);
                    seen.add(val2);
                    found = true;
                    break;
                }
            }

            if (!found) {
                unmatched.add(row1);
            }
        }

        Map<String, Object> result = new HashMap<>();
        result.put("matchType", type);
        result.put("matchedCount", matched.size());
        result.put("unmatchedCount", unmatched.size());
        result.put("matchedRecords", matched);
        result.put("unmatchedRecords", unmatched);
        return result;
    }

    private List<Map<String, String>> readCsv(String path) {
        List<Map<String, String>> records = new ArrayList<>();
        try (BufferedReader reader = new BufferedReader(new FileReader(path))) {
            String[] headers = reader.readLine().split(",");
            String line;
            while ((line = reader.readLine()) != null) {
                String[] values = line.split(",");
                Map<String, String> row = new HashMap<>();
                for (int i = 0; i < headers.length && i < values.length; i++) {
                    row.put(headers[i].trim(), values[i].trim());
                }
                records.add(row);
            }
        } catch (IOException e) {
            throw new RuntimeException("Failed to read CSV: " + path, e);
        }
        return records;
    }
}










@Service
public class ReconciliationService {

    private List<Map<String, String>> nonExcludedFile1 = new ArrayList<>();
    private List<Map<String, String>> nonExcludedFile2 = new ArrayList<>();
    private List<Map<String, String>> excludedFile1 = new ArrayList<>();

    public Map<String, Object> applyExclusion(
            String file1Path,
            String file2Path,
            String exclusionColumn,
            String exclusionValue
    ) {
        nonExcludedFile1.clear();
        nonExcludedFile2.clear();
        excludedFile1.clear();

        List<Map<String, String>> file1Data = readCsv(file1Path);
        List<Map<String, String>> file2Data = readCsv(file2Path);

        for (Map<String, String> row : file1Data) {
            if (exclusionValue.equalsIgnoreCase(row.getOrDefault(exclusionColumn, ""))) {
                excludedFile1.add(row);
            } else {
                nonExcludedFile1.add(row);
            }
        }

        nonExcludedFile2.addAll(file2Data); // no exclusion for file2

        Map<String, Object> result = new HashMap<>();
        result.put("excludedCount", excludedFile1.size());
        result.put("nonExcludedCount", nonExcludedFile1.size());
        result.put("excludedRecords", excludedFile1);
        result.put("nonExcludedRecords", nonExcludedFile1);
        return result;
    }

    public Map<String, Object> performMatching(String type, String column1, String column2) {
        List<Map<String, String>> matched = new ArrayList<>();
        List<Map<String, String>> unmatched = new ArrayList<>();

        switch (type.toLowerCase()) {
            case "1-1":
                oneToOneMatch(column1, column2, matched, unmatched);
                break;
            case "1-many":
                oneToManyMatch(column1, column2, matched, unmatched);
                break;
            case "many-1":
                manyToOneMatch(column1, column2, matched, unmatched);
                break;
            case "many-many":
                manyToManyMatch(column1, column2, matched, unmatched);
                break;
            default:
                throw new IllegalArgumentException("Invalid match type");
        }

        Map<String, Object> result = new HashMap<>();
        result.put("matchType", type);
        result.put("matchedCount", matched.size());
        result.put("unmatchedCount", unmatched.size());
        result.put("matchedRecords", matched);
        result.put("unmatchedRecords", unmatched);
        return result;
    }

    private void oneToOneMatch(String col1, String col2, List<Map<String, String>> matched, List<Map<String, String>> unmatched) {
        Set<String> used = new HashSet<>();
        for (Map<String, String> row1 : nonExcludedFile1) {
            String val1 = row1.getOrDefault(col1, "").trim();
            boolean found = false;
            for (Map<String, String> row2 : nonExcludedFile2) {
                String val2 = row2.getOrDefault(col2, "").trim();
                if (val1.equalsIgnoreCase(val2) && !used.contains(val2)) {
                    Map<String, String> combined = new HashMap<>();
                    combined.putAll(row1);
                    combined.putAll(row2);
                    matched.add(combined);
                    used.add(val2);
                    found = true;
                    break;
                }
            }
            if (!found) unmatched.add(row1);
        }
    }

    private void oneToManyMatch(String col1, String col2, List<Map<String, String>> matched, List<Map<String, String>> unmatched) {
        for (Map<String, String> row1 : nonExcludedFile1) {
            String val1 = row1.getOrDefault(col1, "").trim();
            boolean found = false;
            for (Map<String, String> row2 : nonExcludedFile2) {
                String val2 = row2.getOrDefault(col2, "").trim();
                if (val1.equalsIgnoreCase(val2)) {
                    Map<String, String> combined = new HashMap<>();
                    combined.putAll(row1);
                    combined.putAll(row2);
                    matched.add(combined);
                    found = true;
                }
            }
            if (!found) unmatched.add(row1);
        }
    }

    private void manyToOneMatch(String col1, String col2, List<Map<String, String>> matched, List<Map<String, String>> unmatched) {
        for (Map<String, String> row2 : nonExcludedFile2) {
            String val2 = row2.getOrDefault(col2, "").trim();
            boolean found = false;
            for (Map<String, String> row1 : nonExcludedFile1) {
                String val1 = row1.getOrDefault(col1, "").trim();
                if (val2.equalsIgnoreCase(val1)) {
                    Map<String, String> combined = new HashMap<>();
                    combined.putAll(row1);
                    combined.putAll(row2);
                    matched.add(combined);
                    found = true;
                }
            }
            if (!found) unmatched.add(row2);
        }
    }

    private void manyToManyMatch(String col1, String col2, List<Map<String, String>> matched, List<Map<String, String>> unmatched) {
        Set<String> matchedKeys = new HashSet<>();
        for (Map<String, String> row1 : nonExcludedFile1) {
            String val1 = row1.getOrDefault(col1, "").trim();
            boolean found = false;
            for (Map<String, String> row2 : nonExcludedFile2) {
                String val2 = row2.getOrDefault(col2, "").trim();
                if (val1.equalsIgnoreCase(val2)) {
                    Map<String, String> combined = new HashMap<>();
                    combined.putAll(row1);
                    combined.putAll(row2);
                    matched.add(combined);
                    matchedKeys.add(val1);
                    found = true;
                }
            }
            if (!found) unmatched.add(row1);
        }
    }

    private List<Map<String, String>> readCsv(String path) {
        List<Map<String, String>> records = new ArrayList<>();
        try (BufferedReader reader = new BufferedReader(new FileReader(path))) {
            String[] headers = reader.readLine().split(",");
            String line;
            while ((line = reader.readLine()) != null) {
                String[] values = line.split(",");
                Map<String, String> row = new HashMap<>();
                for (int i = 0; i < headers.length && i < values.length; i++) {
                    row.put(headers[i].trim(), values[i].trim());
                }
                records.add(row);
            }
        } catch (IOException e) {
            throw new RuntimeException("Failed to read CSV: " + path, e);
        }
        return records;
    }
}






@RestController
@RequestMapping("/reconciliation")
public class ReconciliationController {

    private final ReconciliationService reconciliationService;

    public ReconciliationController(ReconciliationService reconciliationService) {
        this.reconciliationService = reconciliationService;
    }

    @GetMapping("/exclude")
    public ResponseEntity<Map<String, Object>> exclude(
            @RequestParam String file1Path,
            @RequestParam String file2Path,
            @RequestParam String exclusionColumn,
            @RequestParam String exclusionValue
    ) {
        Map<String, Object> result = reconciliationService.applyExclusion(
                file1Path, file2Path, exclusionColumn, exclusionValue
        );
        return ResponseEntity.ok(result);
    }

    @GetMapping("/match")
    public ResponseEntity<Map<String, Object>> match(
            @RequestParam String type,
            @RequestParam String column1,
            @RequestParam String column2
    ) {
        Map<String, Object> result = reconciliationService.performMatching(
                type, column1, column2
        );
        return ResponseEntity.ok(result);
    }
}
........................
@Service
public class ReconciliationService {

    private List<Map<String, String>> nonExcludedFile1;
    private List<Map<String, String>> nonExcludedFile2;

    public Map<String, Object> excludeRecords(String file1Path, String file2Path, String exclusionColumn, String exclusionValue) {
        List<Map<String, String>> file1Records = readCSV(file1Path);
        List<Map<String, String>> file2Records = readCSV(file2Path);

        Predicate<Map<String, String>> excludePredicate = record ->
                exclusionValue.equalsIgnoreCase(record.getOrDefault(exclusionColumn, "").trim());

        List<Map<String, String>> excluded1 = file1Records.stream().filter(excludePredicate).toList();
        List<Map<String, String>> nonExcluded1 = file1Records.stream().filter(excludePredicate.negate()).toList();

        List<Map<String, String>> excluded2 = file2Records.stream().filter(excludePredicate).toList();
        List<Map<String, String>> nonExcluded2 = file2Records.stream().filter(excludePredicate.negate()).toList();

        nonExcludedFile1 = new ArrayList<>(nonExcluded1);
        nonExcludedFile2 = new ArrayList<>(nonExcluded2);

        return Map.of(
                "excludedCountFile1", excluded1.size(),
                "excludedCountFile2", excluded2.size(),
                "nonExcludedCountFile1", nonExcluded1.size(),
                "nonExcludedCountFile2", nonExcluded2.size()
        );
    }

    public Map<String, Object> applyUpdateRules(Map<String, String> updateRules) {
        Function<String, String> updateFunction = value -> {
            if (value == null) return "";
            for (Map.Entry<String, String> entry : updateRules.entrySet()) {
                if (value.contains(entry.getKey())) {
                    return entry.getValue();
                }
            }
            return value;
        };

        for (Map<String, String> row : nonExcludedFile1) {
            updateRules.keySet().forEach(col -> {
                if (row.containsKey(col)) row.put(col, updateFunction.apply(row.get(col)));
            });
        }

        for (Map<String, String> row : nonExcludedFile2) {
            updateRules.keySet().forEach(col -> {
                if (row.containsKey(col)) row.put(col, updateFunction.apply(row.get(col)));
            });
        }

        return Map.of("message", "Update rules applied to non-excluded records.");
    }

    public Map<String, Object> performMatchingCascade(String column1, String column2) {
        List<Map<String, String>> currentFile1 = new ArrayList<>(nonExcludedFile1);
        List<Map<String, String>> currentFile2 = new ArrayList<>(nonExcludedFile2);

        List<Map<String, String>> allMatched = new ArrayList<>();
        List<Map<String, String>> unmatched = new ArrayList<>();

        List<String> types = List.of("1-1", "1-many", "many-1", "many-many");

        for (String type : types) {
            List<Map<String, String>> matched = new ArrayList<>();
            unmatched = new ArrayList<>();

            switch (type) {
                case "1-1" -> oneToOneMatch(column1, column2, currentFile1, currentFile2, matched, unmatched);
                case "1-many" -> oneToManyMatch(column1, column2, currentFile1, currentFile2, matched, unmatched);
                case "many-1" -> manyToOneMatch(column1, column2, currentFile1, currentFile2, matched, unmatched);
                case "many-many" -> manyToManyMatch(column1, column2, currentFile1, currentFile2, matched, unmatched);
            }

            allMatched.addAll(matched);
            currentFile1 = unmatched;
        }

        return Map.of(
                "totalMatched", allMatched.size(),
                "finalUnmatched", unmatched.size(),
                "matchedRecords", allMatched,
                "unmatchedRecords", unmatched
        );
    }

    // Matching Methods
    private void oneToOneMatch(String col1, String col2, List<Map<String, String>> file1, List<Map<String, String>> file2,
                               List<Map<String, String>> matched, List<Map<String, String>> unmatched) {
        Set<String> used = new HashSet<>();
        for (Map<String, String> row1 : file1) {
            String val1 = row1.getOrDefault(col1, "").trim();
            boolean found = false;
            for (Map<String, String> row2 : file2) {
                String val2 = row2.getOrDefault(col2, "").trim();
                if (val1.equalsIgnoreCase(val2) && !used.contains(val2)) {
                    matched.add(combine(row1, row2));
                    used.add(val2);
                    found = true;
                    break;
                }
            }
            if (!found) unmatched.add(row1);
        }
    }

    private void oneToManyMatch(String col1, String col2, List<Map<String, String>> file1, List<Map<String, String>> file2,
                                List<Map<String, String>> matched, List<Map<String, String>> unmatched) {
        for (Map<String, String> row1 : file1) {
            String val1 = row1.getOrDefault(col1, "").trim();
            boolean found = false;
            for (Map<String, String> row2 : file2) {
                String val2 = row2.getOrDefault(col2, "").trim();
                if (val1.equalsIgnoreCase(val2)) {
                    matched.add(combine(row1, row2));
                    found = true;
                }
            }
            if (!found) unmatched.add(row1);
        }
    }

    private void manyToOneMatch(String col1, String col2, List<Map<String, String>> file1, List<Map<String, String>> file2,
                                List<Map<String, String>> matched, List<Map<String, String>> unmatched) {
        for (Map<String, String> row2 : file2) {
            String val2 = row2.getOrDefault(col2, "").trim();
            boolean found = false;
            for (Map<String, String> row1 : file1) {
                String val1 = row1.getOrDefault(col1, "").trim();
                if (val1.equalsIgnoreCase(val2)) {
                    matched.add(combine(row1, row2));
                    found = true;
                }
            }
            if (!found) unmatched.add(row2);
        }
    }

    private void manyToManyMatch(String col1, String col2, List<Map<String, String>> file1, List<Map<String, String>> file2,
                                 List<Map<String, String>> matched, List<Map<String, String>> unmatched) {
        for (Map<String, String> row1 : file1) {
            String val1 = row1.getOrDefault(col1, "").trim();
            boolean found = false;
            for (Map<String, String> row2 : file2) {
                String val2 = row2.getOrDefault(col2, "").trim();
                if (val1.equalsIgnoreCase(val2)) {
                    matched.add(combine(row1, row2));
                    found = true;
                }
            }
            if (!found) unmatched.add(row1);
        }
    }

    private Map<String, String> combine(Map<String, String> r1, Map<String, String> r2) {
        Map<String, String> combined = new HashMap<>();
        combined.putAll(r1);
        combined.putAll(r2);
        return combined;
    }

    private List<Map<String, String>> readCSV(String filePath) {
        List<Map<String, String>> records = new ArrayList<>();
        try (BufferedReader reader = Files.newBufferedReader(Paths.get(filePath));
             CSVParser csvParser = new CSVParser(reader, CSVFormat.DEFAULT.withFirstRecordAsHeader())) {
            for (CSVRecord record : csvParser) {
                Map<String, String> row = new HashMap<>(record.toMap());
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
    private ReconciliationService service;

    @PostMapping("/exclude")
    public ResponseEntity<Map<String, Object>> excludeRecords(
            @RequestParam String file1Path,
            @RequestParam String file2Path,
            @RequestParam String exclusionColumn,
            @RequestParam String exclusionValue) {
        return ResponseEntity.ok(service.excludeRecords(file1Path, file2Path, exclusionColumn, exclusionValue));
    }

    @PostMapping("/update")
    public ResponseEntity<Map<String, Object>> updateColumns(
            @RequestBody Map<String, String> updateRules) {
        return ResponseEntity.ok(service.applyUpdateRules(updateRules));
    }

    @PostMapping("/match")
    public ResponseEntity<Map<String, Object>> performMatching(
            @RequestParam String column1,
            @RequestParam String column2) {
        return ResponseEntity.ok(service.performMatchingCascade(column1, column2));
    }
}



// ReconciliationController.java @RestController @RequestMapping("/reconcile") public class ReconciliationController {

@Autowired
private ReconciliationService reconciliationService;

@GetMapping("/upload")
public ResponseEntity<String> uploadFiles(@RequestParam String file1Path, @RequestParam String file2Path) {
    reconciliationService.loadFiles(file1Path, file2Path);
    return ResponseEntity.ok("Files uploaded successfully");
}

@GetMapping("/update")
public ResponseEntity<Map<String, Object>> updateColumns(@RequestParam String column, @RequestParam String from, @RequestParam String to) {
    Map<String, Object> result = reconciliationService.applyUpdates(column, from, to);
    return ResponseEntity.ok(result);
}

@GetMapping("/exclude")
public ResponseEntity<Map<String, Object>> excludeRecords(@RequestParam String column, @RequestParam String value) {
    Map<String, Object> result = reconciliationService.excludeRecords(column, value);
    return ResponseEntity.ok(result);
}

@GetMapping("/match")
public ResponseEntity<Map<String, Object>> performAllMatching(@RequestParam String column1, @RequestParam String column2) {
    Map<String, Object> result = reconciliationService.matchAll(column1, column2);
    return ResponseEntity.ok(result);
}

}

// ReconciliationService.java @Service public class ReconciliationService {

private List<Map<String, String>> originalFile1;
private List<Map<String, String>> originalFile2;
private List<Map<String, String>> updatedFile1;
private List<Map<String, String>> updatedFile2;
private List<Map<String, String>> nonExcludedFile1;
private List<Map<String, String>> nonExcludedFile2;

public void loadFiles(String file1Path, String file2Path) {
    originalFile1 = readCSV(file1Path);
    originalFile2 = readCSV(file2Path);
    updatedFile1 = new ArrayList<>(originalFile1);
    updatedFile2 = new ArrayList<>(originalFile2);
}

public Map<String, Object> applyUpdates(String column, String from, String to) {
    updatedFile1.forEach(row -> row.computeIfPresent(column, (k, v) -> v.replace(from, to)));
    updatedFile2.forEach(row -> row.computeIfPresent(column, (k, v) -> v.replace(from, to)));
    return Map.of("message", "Updates applied successfully", "updatedCountFile1", updatedFile1.size(), "updatedCountFile2", updatedFile2.size());
}

public Map<String, Object> excludeRecords(String column, String value) {
    Predicate<Map<String, String>> predicate = row -> value.equalsIgnoreCase(row.getOrDefault(column, "").trim());
    List<Map<String, String>> excluded1 = updatedFile1.stream().filter(predicate).toList();
    List<Map<String, String>> excluded2 = updatedFile2.stream().filter(predicate).toList();

    nonExcludedFile1 = updatedFile1.stream().filter(predicate.negate()).collect(Collectors.toList());
    nonExcludedFile2 = updatedFile2.stream().filter(predicate.negate()).collect(Collectors.toList());

    return Map.of(
            "excludedCountFile1", excluded1.size(),
            "excludedRecordsFile1", excluded1,
            "excludedCountFile2", excluded2.size(),
            "excludedRecordsFile2", excluded2,
            "nonExcludedCountFile1", nonExcludedFile1.size(),
            "nonExcludedRecordsFile1", nonExcludedFile1,
            "nonExcludedCountFile2", nonExcludedFile2.size(),
            "nonExcludedRecordsFile2", nonExcludedFile2
    );
}

public Map<String, Object> matchAll(String col1, String col2) {
    List<Map<String, String>> matched = new ArrayList<>();
    List<Map<String, String>> unmatched1 = new ArrayList<>(nonExcludedFile1);
    List<Map<String, String>> unmatched2 = new ArrayList<>(nonExcludedFile2);

    List<String> types = List.of("1-1", "1-many", "many-1", "many-many");
    Map<String, Object> matchResults = new LinkedHashMap<>();

    for (String type : types) {
        List<Map<String, String>> localMatched = new ArrayList<>();
        List<Map<String, String>> stillUnmatched1 = new ArrayList<>();
        List<Map<String, String>> stillUnmatched2 = new ArrayList<>();

        match(unmatched1, unmatched2, col1, col2, type, localMatched, stillUnmatched1, stillUnmatched2);

        matched.addAll(localMatched);
        unmatched1 = stillUnmatched1;
        unmatched2 = stillUnmatched2;

        matchResults.put(type, Map.of(
                "matchedCount", localMatched.size(),
                "matchedRecords", localMatched,
                "unmatchedCountFile1", unmatched1.size(),
                "unmatchedCountFile2", unmatched2.size()
        ));
    }

    matchResults.put("finalUnmatchedFile1", unmatched1);
    matchResults.put("finalUnmatchedFile2", unmatched2);
    return matchResults;
}

private void match(List<Map<String, String>> file1, List<Map<String, String>> file2, String col1, String col2,
                   String type, List<Map<String, String>> matched, List<Map<String, String>> unmatched1,
                   List<Map<String, String>> unmatched2) {

    Set<Integer> usedIndexes = new HashSet<>();

    for (int i = 0; i < file1.size(); i++) {
        Map<String, String> row1 = file1.get(i);
        String val1 = row1.getOrDefault(col1, "").trim();
        boolean found = false;

        for (int j = 0; j < file2.size(); j++) {
            if (usedIndexes.contains(j) && type.equals("1-1")) continue;
            Map<String, String> row2 = file2.get(j);
            String val2 = row2.getOrDefault(col2, "").trim();

            if (val1.equalsIgnoreCase(val2)) {
                Map<String, String> combined = new HashMap<>();
                combined.putAll(row1);
                combined.putAll(row2);
                matched.add(combined);
                usedIndexes.add(j);
                found = true;
                if (type.equals("1-1") || type.equals("many-1")) break;
            }
        }
        if (!found) unmatched1.add(row1);
    }

    for (int j = 0; j < file2.size(); j++) {
        if (!usedIndexes.contains(j)) {
            unmatched2.add(file2.get(j));
        }
    }
}

private List<Map<String, String>> readCSV(String filePath) {
    try (BufferedReader reader = Files.newBufferedReader(Paths.get(filePath));
         CSVParser parser = new CSVParser(reader, CSVFormat.DEFAULT.withFirstRecordAsHeader())) {
        List<Map<String, String>> records = new ArrayList<>();
        for (CSVRecord record : parser) {
            records.add(record.toMap());
        }
        return records;
    } catch (IOException e) {
        throw new RuntimeException("CSV read error: " + filePath, e);
    }
}

}




















@Service
public class ReconciliationService {

    private List<Map<String, String>> originalFile1;
    private List<Map<String, String>> originalFile2;

    private List<Map<String, String>> nonExcludedFile1;
    private List<Map<String, String>> nonExcludedFile2;

    // Unmatched after each matching type:
    private List<Map<String, String>> unmatchedFile1;
    private List<Map<String, String>> unmatchedFile2;

    public void loadFiles(String file1Path, String file2Path) {
        originalFile1 = readCSV(file1Path);
        originalFile2 = readCSV(file2Path);
        nonExcludedFile1 = new ArrayList<>(originalFile1);
        nonExcludedFile2 = new ArrayList<>(originalFile2);
        unmatchedFile1 = new ArrayList<>(nonExcludedFile1);
        unmatchedFile2 = new ArrayList<>(nonExcludedFile2);
    }

    public Map<String, Object> excludeRecords(String column, String value) {
        Predicate<Map<String, String>> predicate = row -> value.equalsIgnoreCase(row.getOrDefault(column, "").trim());

        List<Map<String, String>> excluded1 = nonExcludedFile1.stream().filter(predicate).toList();
        List<Map<String, String>> excluded2 = nonExcludedFile2.stream().filter(predicate).toList();

        nonExcludedFile1 = nonExcludedFile1.stream().filter(predicate.negate()).collect(Collectors.toList());
        nonExcludedFile2 = nonExcludedFile2.stream().filter(predicate.negate()).collect(Collectors.toList());

        // Reset unmatched to non-excluded after exclusion
        unmatchedFile1 = new ArrayList<>(nonExcludedFile1);
        unmatchedFile2 = new ArrayList<>(nonExcludedFile2);

        return Map.of(
                "excludedCountFile1", excluded1.size(),
                "excludedRecordsFile1", excluded1,
                "excludedCountFile2", excluded2.size(),
                "excludedRecordsFile2", excluded2,
                "nonExcludedCountFile1", nonExcludedFile1.size(),
                "nonExcludedRecordsFile1", nonExcludedFile1,
                "nonExcludedCountFile2", nonExcludedFile2.size(),
                "nonExcludedRecordsFile2", nonExcludedFile2
        );
    }

    public Map<String, Object> matchByType(String type, String col1, String col2) {
        List<Map<String, String>> matched = new ArrayList<>();
        List<Map<String, String>> stillUnmatched1 = new ArrayList<>();
        List<Map<String, String>> stillUnmatched2 = new ArrayList<>();

        match(unmatchedFile1, unmatchedFile2, col1, col2, type, matched, stillUnmatched1, stillUnmatched2);

        // Update unmatched lists after this match for next calls
        unmatchedFile1 = stillUnmatched1;
        unmatchedFile2 = stillUnmatched2;

        return Map.of(
                "matchType", type,
                "matchedCount", matched.size(),
                "matchedRecords", matched,
                "unmatchedCountFile1", unmatchedFile1.size(),
                "unmatchedRecordsFile1", unmatchedFile1,
                "unmatchedCountFile2", unmatchedFile2.size(),
                "unmatchedRecordsFile2", unmatchedFile2
        );
    }

    private void match(List<Map<String, String>> file1, List<Map<String, String>> file2, String col1, String col2,
                       String type, List<Map<String, String>> matched, List<Map<String, String>> unmatched1,
                       List<Map<String, String>> unmatched2) {

        Set<Integer> usedIndexes = new HashSet<>();

        for (Map<String, String> row1 : file1) {
            String val1 = row1.getOrDefault(col1, "").trim();
            boolean found = false;

            for (int j = 0; j < file2.size(); j++) {
                if (usedIndexes.contains(j) && type.equals("1-1")) continue;
                Map<String, String> row2 = file2.get(j);
                String val2 = row2.getOrDefault(col2, "").trim();

                if (val1.equalsIgnoreCase(val2)) {
                    Map<String, String> combined = new HashMap<>();
                    combined.putAll(row1);
                    combined.putAll(row2);
                    matched.add(combined);
                    usedIndexes.add(j);
                    found = true;
                    if (type.equals("1-1") || type.equals("many-1")) break;
                }
            }
            if (!found) unmatched1.add(row1);
        }

        for (int j = 0; j < file2.size(); j++) {
            if (!usedIndexes.contains(j)) {
                unmatched2.add(file2.get(j));
            }
        }
    }

    private List<Map<String, String>> readCSV(String filePath) {
        try (BufferedReader reader = Files.newBufferedReader(Paths.get(filePath));
             CSVParser parser = new CSVParser(reader, CSVFormat.DEFAULT.withFirstRecordAsHeader())) {
            List<Map<String, String>> records = new ArrayList<>();
            for (CSVRecord record : parser) {
                records.add(record.toMap());
            }
            return records;
        } catch (IOException e) {
            throw new RuntimeException("CSV read error: " + filePath, e);
        }
    }
}




@RestController
@RequestMapping("/reconcile")
public class ReconciliationController {

    @Autowired
    private ReconciliationService reconciliationService;

    @GetMapping("/upload")
    public ResponseEntity<String> uploadFiles(@RequestParam String file1Path, @RequestParam String file2Path) {
        reconciliationService.loadFiles(file1Path, file2Path);
        return ResponseEntity.ok("Files uploaded successfully");
    }

    @GetMapping("/exclude")
    public ResponseEntity<Map<String, Object>> excludeRecords(@RequestParam String column, @RequestParam String value) {
        Map<String, Object> result = reconciliationService.excludeRecords(column, value);
        return ResponseEntity.ok(result);
    }

    @GetMapping("/match/{type}")
    public ResponseEntity<Map<String, Object>> performMatchingByType(@PathVariable String type,
                                                                     @RequestParam String column1,
                                                                     @RequestParam String column2) {
        Map<String, Object> result = reconciliationService.matchByType(type, column1, column2);
        return ResponseEntity.ok(result);
    }
}
