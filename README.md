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





import org.apache.commons.csv.CSVFormat;
import org.apache.commons.csv.CSVParser;
import org.apache.commons.csv.CSVRecord;
import org.springframework.stereotype.Service;

import java.io.BufferedReader;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.*;
import java.util.stream.Collectors;

@Service
public class ReconciliationService {

    private List<Map<String, String>> originalFile1 = new ArrayList<>();
    private List<Map<String, String>> originalFile2 = new ArrayList<>();

    private List<Map<String, String>> excludedFile1 = new ArrayList<>();
    private List<Map<String, String>> excludedFile2 = new ArrayList<>();

    private List<Map<String, String>> nonExcludedFile1 = new ArrayList<>();
    private List<Map<String, String>> nonExcludedFile2 = new ArrayList<>();

    // Load CSV files
    public void loadFiles(String file1Path, String file2Path) {
        originalFile1 = readCSV(file1Path);
        originalFile2 = readCSV(file2Path);
        // Clear previous states
        excludedFile1.clear();
        excludedFile2.clear();
        nonExcludedFile1.clear();
        nonExcludedFile2.clear();
    }

    // Update rule: apply multiple replacements on a specific column in both datasets
    public Map<String, Object> applyUpdateRule(String column, Map<String, String> replacements) {
        for (Map<String, String> record : originalFile1) {
            String val = record.getOrDefault(column, "");
            for (Map.Entry<String, String> entry : replacements.entrySet()) {
                val = val.replace(entry.getKey(), entry.getValue());
            }
            record.put(column, val);
        }
        for (Map<String, String> record : originalFile2) {
            String val = record.getOrDefault(column, "");
            for (Map.Entry<String, String> entry : replacements.entrySet()) {
                val = val.replace(entry.getKey(), entry.getValue());
            }
            record.put(column, val);
        }

        return Map.of(
                "message", "Update rule applied successfully",
                "updatedCountFile1", originalFile1.size(),
                "updatedCountFile2", originalFile2.size()
        );
    }

    // Exclusion rule: exclude records where column == excludeValue
    public Map<String, Object> applyExclusionRule(String column, String excludeValue) {
        excludedFile1 = originalFile1.stream()
                .filter(r -> excludeValue.equalsIgnoreCase(r.getOrDefault(column, "").trim()))
                .collect(Collectors.toList());

        excludedFile2 = originalFile2.stream()
                .filter(r -> excludeValue.equalsIgnoreCase(r.getOrDefault(column, "").trim()))
                .collect(Collectors.toList());

        nonExcludedFile1 = originalFile1.stream()
                .filter(r -> !excludeValue.equalsIgnoreCase(r.getOrDefault(column, "").trim()))
                .collect(Collectors.toList());

        nonExcludedFile2 = originalFile2.stream()
                .filter(r -> !excludeValue.equalsIgnoreCase(r.getOrDefault(column, "").trim()))
                .collect(Collectors.toList());

        return Map.of(
                "excludedCountFile1", excludedFile1.size(),
                "excludedRecordsFile1", excludedFile1,
                "excludedCountFile2", excludedFile2.size(),
                "excludedRecordsFile2", excludedFile2,
                "nonExcludedCountFile1", nonExcludedFile1.size(),
                "nonExcludedRecordsFile1", nonExcludedFile1,
                "nonExcludedCountFile2", nonExcludedFile2.size(),
                "nonExcludedRecordsFile2", nonExcludedFile2
        );
    }

    // Helper method to read CSV as List<Map<String,String>>
    private List<Map<String, String>> readCSV(String filePath) {
        try (BufferedReader reader = Files.newBufferedReader(Paths.get(filePath));
             CSVParser parser = new CSVParser(reader, CSVFormat.DEFAULT.withFirstRecordAsHeader())) {
            List<Map<String, String>> records = new ArrayList<>();
            for (CSVRecord record : parser) {
                records.add(record.toMap());
            }
            return records;
        } catch (IOException e) {
            throw new RuntimeException("Failed to read CSV: " + filePath, e);
        }
    }

    // Getters for excluded and non-excluded records (optional)
    public List<Map<String, String>> getExcludedFile1() {
        return excludedFile1;
    }

    public List<Map<String, String>> getExcludedFile2() {
        return excludedFile2;
    }

    public List<Map<String, String>> getNonExcludedFile1() {
        return nonExcludedFile1;
    }

    public List<Map<String, String>> getNonExcludedFile2() {
        return nonExcludedFile2;
    }
}
....................................................
// File: ReconciliationController.java package com.example.reconciliation.controller;

import com.example.reconciliation.model.Record; import com.example.reconciliation.service.ReconciliationService; import org.springframework.beans.factory.annotation.Autowired; import org.springframework.web.bind.annotation.*;

import java.util.List; import java.util.Map;

@RestController @RequestMapping("/reconcile") public class ReconciliationController {

@Autowired
private ReconciliationService reconciliationService;

@GetMapping("/load")
public String loadFiles(@RequestParam String file1Path, @RequestParam String file2Path) {
    reconciliationService.loadFiles(file1Path, file2Path);
    return "Files loaded and initial data prepared.";
}

@GetMapping("/update")
public String applyUpdates(@RequestParam Map<String, String> updates) {
    reconciliationService.applyUpdateRules(updates);
    return "Update rules applied.";
}

@GetMapping("/exclude")
public String applyExclusions(@RequestParam List<String> exclusionColumns, @RequestParam List<String> exclusionValues) {
    reconciliationService.applyExclusionRules(exclusionColumns, exclusionValues);
    return "Exclusion rules applied.";
}

@GetMapping("/match/one-to-one")
public Map<String, Object> matchOneToOne(@RequestParam List<String> matchColumns) {
    return reconciliationService.matchOneToOne(matchColumns);
}

@GetMapping("/match/one-to-many")
public Map<String, Object> matchOneToMany(@RequestParam List<String> matchColumns) {
    return reconciliationService.matchOneToMany(matchColumns);
}

@GetMapping("/match/many-to-one")
public Map<String, Object> matchManyToOne(@RequestParam List<String> matchColumns) {
    return reconciliationService.matchManyToOne(matchColumns);
}

@GetMapping("/match/many-to-many")
public Map<String, Object> matchManyToMany(@RequestParam List<String> matchColumns) {
    return reconciliationService.matchManyToMany(matchColumns);
}

}

// File: ReconciliationService.java package com.example.reconciliation.service;

import com.example.reconciliation.model.Record; import org.springframework.stereotype.Service;

import java.io.BufferedReader; import java.io.FileReader; import java.util.*; import java.util.stream.Collectors;

@Service public class ReconciliationService { private List<Record> file1Records = new ArrayList<>(); private List<Record> file2Records = new ArrayList<>(); private List<Record> excludedRecords = new ArrayList<>(); private List<Record> matchedRecords = new ArrayList<>(); private List<Record> unmatchedRecords = new ArrayList<>();

public void loadFiles(String path1, String path2) {
    file1Records = loadCsv(path1);
    file2Records = loadCsv(path2);
    excludedRecords.clear();
    matchedRecords.clear();
    unmatchedRecords.clear();
}

public List<Record> loadCsv(String path) {
    List<Record> records = new ArrayList<>();
    try (BufferedReader reader = new BufferedReader(new FileReader(path))) {
        String[] headers = reader.readLine().split(",");
        String line;
        while ((line = reader.readLine()) != null) {
            String[] values = line.split(",");
            Map<String, String> map = new HashMap<>();
            for (int i = 0; i < headers.length; i++) {
                map.put(headers[i], values[i]);
            }
            records.add(new Record(map));
        }
    } catch (Exception e) {
        e.printStackTrace();
    }
    return records;
}

public void applyUpdateRules(Map<String, String> updates) {
    for (Record r : file1Records) {
        updates.forEach((col, val) -> {
            if (r.getData().containsKey(col)) {
                r.getData().put(col, val);
            }
        });
    }
    for (Record r : file2Records) {
        updates.forEach((col, val) -> {
            if (r.getData().containsKey(col)) {
                r.getData().put(col, val);
            }
        });
    }
}

public void applyExclusionRules(List<String> columns, List<String> values) {
    file2Records = file2Records.stream().filter(r -> {
        for (int i = 0; i < columns.size(); i++) {
            if (r.getData().get(columns.get(i)).equals(values.get(i))) {
                excludedRecords.add(r);
                return false;
            }
        }
        return true;
    }).collect(Collectors.toList());
}

public Map<String, Object> matchOneToOne(List<String> matchColumns) {
    matchedRecords.clear();
    unmatchedRecords.clear();
    List<Record> remainingFile2 = new ArrayList<>(file2Records);

    for (Record r1 : file1Records) {
        Optional<Record> match = remainingFile2.stream().filter(r2 -> matchColumns.stream()
                .allMatch(c -> r1.getData().get(c).equals(r2.getData().get(c)))).findFirst();
        if (match.isPresent()) {
            matchedRecords.add(r1);
            remainingFile2.remove(match.get());
        } else {
            unmatchedRecords.add(r1);
        }
    }

    Map<String, Object> result = new HashMap<>();
    result.put("matchedCount", matchedRecords.size());
    result.put("unmatchedCount", unmatchedRecords.size());
    result.put("excludedCount", excludedRecords.size());
    result.put("matchedRecords", matchedRecords);
    result.put("unmatchedRecords", unmatchedRecords);
    result.put("excludedRecords", excludedRecords);
    return result;
}

public Map<String, Object> matchOneToMany(List<String> matchColumns) {
    // Similar to matchOneToOne, but allows r1 to match multiple r2s
    // Implement similarly by collecting all matches
    return Map.of("status", "one-to-many not yet implemented");
}

public Map<String, Object> matchManyToOne(List<String> matchColumns) {
    // Reverse of one-to-many
    return Map.of("status", "many-to-one not yet implemented");
}

public Map<String, Object> matchManyToMany(List<String> matchColumns) {
    // Match all combinations that meet criteria
    return Map.of("status", "many-to-many not yet implemented");
}

}

// File: Record.java package com.example.reconciliation.model;

import java.util.Map;

public class Record { private Map<String, String> data;

public Record(Map<String, String> data) {
    this.data = data;
}

public Map<String, String> getData() {
    return data;
}

public void setData(Map<String, String> data) {
    this.data = data;
}

@Override
public String toString() {
    return data.toString();
}

}






















package com.example.reconciliation.service;

import com.example.reconciliation.model.Record;
import org.springframework.stereotype.Service;

import java.io.BufferedReader;
import java.io.FileReader;
import java.util.*;
import java.util.stream.Collectors;

@Service
public class ReconciliationService {
    private List<Record> file1Records = new ArrayList<>();
    private List<Record> file2Records = new ArrayList<>();
    private List<Record> excludedRecords = new ArrayList<>();
    private List<Record> matchedRecords = new ArrayList<>();
    private List<Record> unmatchedRecords = new ArrayList<>();

    public Map<String, Object> loadFiles(String path1, String path2) {
        file1Records = loadCsv(path1);
        file2Records = loadCsv(path2);

        // Clear others on load
        excludedRecords.clear();
        matchedRecords.clear();
        unmatchedRecords.clear();

        Map<String, Object> response = new HashMap<>();
        response.put("message", "Files loaded successfully.");
        response.put("file1RecordCount", file1Records.size());
        response.put("file1Records", file1Records);
        response.put("file2RecordCount", file2Records.size());
        response.put("file2Records", file2Records);

        return response;
    }

    private List<Record> loadCsv(String path) {
        List<Record> records = new ArrayList<>();
        try (BufferedReader reader = new BufferedReader(new FileReader(path))) {
            String headerLine = reader.readLine();
            if (headerLine == null) return records;

            String[] headers = headerLine.split(",");
            String line;
            while ((line = reader.readLine()) != null) {
                String[] values = line.split(",", -1);
                if (values.length != headers.length) {
                    // skip malformed row or log error if needed
                    continue;
                }
                Map<String, String> map = new LinkedHashMap<>();
                for (int i = 0; i < headers.length; i++) {
                    map.put(headers[i].trim(), values[i].trim());
                }
                records.add(new Record(map));
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return records;
    }

    public Map<String, Object> applyUpdateRules(Map<String, String> updates) {
        // Apply updates on file1
        for (Record r : file1Records) {
            updates.forEach((col, val) -> {
                if (r.getData().containsKey(col)) {
                    r.getData().put(col, val);
                }
            });
        }
        // Apply updates on file2
        for (Record r : file2Records) {
            updates.forEach((col, val) -> {
                if (r.getData().containsKey(col)) {
                    r.getData().put(col, val);
                }
            });
        }
        Map<String, Object> response = new HashMap<>();
        response.put("message", "Update rules applied.");
        response.put("file1UpdatedRecords", file1Records);
        response.put("file2UpdatedRecords", file2Records);
        return response;
    }

    public Map<String, Object> applyExclusionRules(List<String> columns, List<String> values) {
        excludedRecords.clear();

        List<Record> filtered = file2Records.stream().filter(r -> {
            for (int i = 0; i < columns.size(); i++) {
                String col = columns.get(i);
                String val = values.get(i);
                if (val.equals(r.getData().get(col))) {
                    excludedRecords.add(r);
                    return false; // exclude this record
                }
            }
            return true;
        }).collect(Collectors.toList());

        file2Records = filtered;

        Map<String, Object> response = new HashMap<>();
        response.put("message", "Exclusion rules applied.");
        response.put("excludedRecordsCount", excludedRecords.size());
        response.put("excludedRecords", excludedRecords);
        response.put("remainingFile2RecordsCount", file2Records.size());
        response.put("remainingFile2Records", file2Records);
        return response;
    }

    public Map<String, Object> matchOneToOne(List<String> matchColumns) {
        matchedRecords.clear();
        unmatchedRecords.clear();

        List<Record> file2Copy = new ArrayList<>(file2Records);

        for (Record r1 : file1Records) {
            Optional<Record> match = file2Copy.stream().filter(r2 -> matchColumns.stream()
                    .allMatch(c -> Objects.equals(r1.getData().get(c), r2.getData().get(c)))).findFirst();

            if (match.isPresent()) {
                matchedRecords.add(r1);
                file2Copy.remove(match.get());
            } else {
                unmatchedRecords.add(r1);
            }
        }

        Map<String, Object> response = new HashMap<>();
        response.put("matchedCount", matchedRecords.size());
        response.put("unmatchedCount", unmatchedRecords.size());
        response.put("excludedCount", excludedRecords.size());

        response.put("matchedRecords", matchedRecords);
        response.put("unmatchedRecords", unmatchedRecords);
        response.put("excludedRecords", excludedRecords);

        response.put("file1RecordCount", file1Records.size());
        response.put("file2RecordCount", file2Records.size());
        response.put("file1Records", file1Records);
        response.put("file2Records", file2Records);

        return response;
    }

}




package com.example.reconciliation.controller;

import com.example.reconciliation.model.Record;
import com.example.reconciliation.service.ReconciliationService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/reconcile")
public class ReconciliationController {

    @Autowired
    private ReconciliationService reconciliationService;

    @GetMapping("/load")
    public Map<String, Object> loadFiles(@RequestParam String file1Path, @RequestParam String file2Path) {
        return reconciliationService.loadFiles(file1Path, file2Path);
    }

    @PostMapping("/update")
    public Map<String, Object> applyUpdates(@RequestBody Map<String, String> updates) {
        return reconciliationService.applyUpdateRules(updates);
    }

    @PostMapping("/exclude")
    public Map<String, Object> applyExclusions(@RequestBody Map<String, List<String>> exclusionRules) {
        // Expecting JSON: { "columns": [...], "values": [...] }
        List<String> columns = exclusionRules.get("columns");
        List<String> values = exclusionRules.get("values");
        return reconciliationService.applyExclusionRules(columns, values);
    }

    @PostMapping("/match/one-to-one")
    public Map<String, Object> matchOneToOne(@RequestBody List<String> matchColumns) {
        return reconciliationService.matchOneToOne(matchColumns);
    }

}



















public Map<String, Object> applyUpdateRules(Map<String, String> rules) {
    // Each entry in rules is like: rule1=AGREEMENT_NAME:_RIMP:_3CP
    for (String ruleKey : rules.keySet()) {
        String rule = rules.get(ruleKey);
        // Split rule into 3 parts: columnName, fromValue, toValue
        String[] parts = rule.split(":", 3);
        if (parts.length == 3) {
            String col = parts[0];
            String fromVal = parts[1];
            String toVal = parts[2];

            // Update file1Records
            for (Record r : file1Records) {
                Map<String, String> data = r.getData();
                if (data.containsKey(col) && data.get(col).contains(fromVal)) {
                    data.put(col, data.get(col).replace(fromVal, toVal));
                }
            }
            // Update file2Records
            for (Record r : file2Records) {
                Map<String, String> data = r.getData();
                if (data.containsKey(col) && data.get(col).contains(fromVal)) {
                    data.put(col, data.get(col).replace(fromVal, toVal));
                }
            }
        }
    }

    Map<String, Object> response = new HashMap<>();
    response.put("message", "Update rules applied.");
    response.put("file1UpdatedRecords", file1Records);
    response.put("file2UpdatedRecords", file2Records);
    return response;
}







2. Controller:

@GetMapping("/update")
public Map<String, Object> applyUpdates(@RequestParam Map<String, String> rules) {
    return reconciliationService.applyAdvancedUpdateRules(rules);
}


---

3. Service Method:

public Map<String, Object> applyAdvancedUpdateRules(Map<String, String> rules) {
    for (String ruleKey : rules.keySet()) {
        String rule = rules.get(ruleKey);
        if (rule.contains(":=")) {
            String[] parts = rule.split(":=", 2);
            String targetCol = parts[0].trim();
            String expression = parts[1].trim();

            applyExpressionToRecords(file1Records, targetCol, expression);
            applyExpressionToRecords(file2Records, targetCol, expression);
        }
    }

    Map<String, Object> response = new HashMap<>();
    response.put("message", "Expression-based updates applied.");
    response.put("file1UpdatedRecords", file1Records);
    response.put("file2UpdatedRecords", file2Records);
    return response;
}


---

4. Helper Method (Apply expression):

private void applyExpressionToRecords(List<Record> records, String targetCol, String expression) {
    for (Record r : records) {
        StringBuilder newValue = new StringBuilder();
        String[] tokens = expression.split("\\+");

        for (String token : tokens) {
            token = token.trim();
            if (r.getData().containsKey(token)) {
                newValue.append(r.getData().get(token));
            } else {
                newValue.append(token); // literal value like "3"
            }
        }

        r.getData().put(targetCol, newValue.toString());
    }
}




public Map<String, Object> applyAdvancedUpdateRules(Map<String, String> rules) {
    for (String ruleKey : rules.keySet()) {
        String rule = rules.get(ruleKey);
        if (rule.contains(":=")) {
            String[] parts = rule.split(":=", 2);
            String targetCol = parts[0].trim();
            String expression = parts[1].trim();

            applyExpressionToRecords(file1Records, targetCol, expression);
            applyExpressionToRecords(file2Records, targetCol, expression);
        }
    }

    Map<String, Object> response = new HashMap<>();
    response.put("message", "Expression-based updates applied.");
    response.put("file1UpdatedRecords", file1Records);
    response.put("file2UpdatedRecords", file2Records);
    return response;
}


@GetMapping("/update")
public Map<String, Object> applyUpdates(@RequestParam Map<String, String> rules) {
    return reconciliationService.applyAdvancedUpdateRules(rules);
}







@GetMapping("/exclude")
public Map<String, Object> applyExclusions(@RequestParam List<String> columns, @RequestParam List<String> values) {
    return reconciliationService.applyExclusionRules(columns, values);
}


http://localhost:8080/exclude?columns=Name,Age&values=John%20Doe,25









public Map<String, Object> applyUpdates(Map<String, String> fromValues, Map<String, String> toValues) {
    fromValues.forEach((column, fromValue) -> {
        String toValue = toValues.getOrDefault(column, fromValue);  // If `to` is not provided, keep the original value
        updatedFile1.forEach(row -> row.computeIfPresent(column, (k, v) -> v.equals(fromValue) ? toValue : v));
        updatedFile2.forEach(row -> row.computeIfPresent(column, (k, v) -> v.equals(fromValue) ? toValue : v));
    });

    return Map.of(
        "updatedCountFile1", updatedFile1.size(),
        "updatedCountFile2", updatedFile2.size(),
        "UpdatedRecordsFile1", updatedFile1,
        "UpdatedRecordsFile2", updatedFile2
    );
}




private String resolvePlaceholders(String template, Map<String, String> rowData) {
    Pattern pattern = Pattern.compile("\\{(\\w+)}");
    Matcher matcher = pattern.matcher(template);
    StringBuffer result = new StringBuffer();

    while (matcher.find()) {
        String colName = matcher.group(1);
        String replacement = rowData.getOrDefault(colName, "");
        matcher.appendReplacement(result, Matcher.quoteReplacement(replacement));
    }
    matcher.appendTail(result);
    return result.toString();
}


................................
@RestController
@RequestMapping("/api")
public class RecordController {

    private List<Map<String, String>> updatedData = new ArrayList<>();
    private List<Map<String, String>> nonExcluded = new ArrayList<>();

    // 1. Update Endpoint
    @PostMapping("/update")
    public Map<String, Object> updateRecords(
            @RequestParam("file") MultipartFile file,
            @RequestParam("column") String column,
            @RequestParam("oldValue") String oldValue,
            @RequestParam("newValue") String newValue) throws IOException {

        updatedData = parseCsv(file);
        int count = 0;

        for (Map<String, String> row : updatedData) {
            if (row.containsKey(column) && row.get(column).equals(oldValue)) {
                row.put(column, newValue);
                count++;
            }
        }

        Map<String, Object> response = new LinkedHashMap<>();
        response.put("updatedCount", count);
        response.put("updatedRecords", updatedData);
        return response;
    }

    // 2. Exclude Endpoint
    @PostMapping("/exclude")
    public Map<String, Object> excludeRecords(
            @RequestParam("column") String column,
            @RequestParam("value") String value) {

        List<Map<String, String>> excluded = updatedData.stream()
                .filter(row -> value.equals(row.get(column)))
                .collect(Collectors.toList());

        nonExcluded = updatedData.stream()
                .filter(row -> !value.equals(row.get(column)))
                .collect(Collectors.toList());

        Map<String, Object> response = new LinkedHashMap<>();
        response.put("excludedCount", excluded.size());
        response.put("excludedRecords", excluded);
        response.put("nonExcludedCount", nonExcluded.size());
        response.put("nonExcludedRecords", nonExcluded);
        return response;
    }

    // 3. Match Endpoint
    @PostMapping("/match")
    public Map<String, Object> matchFiles(
            @RequestParam("file1") MultipartFile file1,
            @RequestParam("file2") MultipartFile file2,
            @RequestParam("column1") String column1,
            @RequestParam("column2") String column2,
            @RequestParam("matchType") String matchType) throws IOException {

        List<Map<String, String>> data1 = (nonExcluded.isEmpty()) ? parseCsv(file1) : nonExcluded;
        List<Map<String, String>> data2 = parseCsv(file2);

        Map<String, List<Map<String, String>>> map2 = data2.stream()
                .collect(Collectors.groupingBy(row -> row.get(column2)));

        List<Map<String, Object>> matched = new ArrayList<>();
        List<Map<String, String>> unmatched = new ArrayList<>();

        for (Map<String, String> row1 : data1) {
            String val = row1.get(column1);
            List<Map<String, String>> matches = map2.get(val);

            if (matches != null && !matches.isEmpty()) {
                if ("1-1".equalsIgnoreCase(matchType)) {
                    matched.add(Map.of("file1", row1, "file2", matches.get(0)));
                } else {
                    for (Map<String, String> m : matches) {
                        matched.add(Map.of("file1", row1, "file2", m));
                    }
                }
            } else {
                unmatched.add(row1);
            }
        }

        Map<String, Object> response = new LinkedHashMap<>();
        response.put("matchType", matchType);
        response.put("matchedCount", matched.size());
        response.put("matchedRecords", matched);
        response.put("unmatchedCount", unmatched.size());
        response.put("unmatchedRecords", unmatched);
        return response;
    }

    // CSV Parser
    private List<Map<String, String>> parseCsv(MultipartFile file) throws IOException {
        List<Map<String, String>> records = new ArrayList<>();
        try (Reader reader = new InputStreamReader(file.getInputStream())) {
            Iterable<CSVRecord> csvRecords = CSVFormat.DEFAULT
                    .withFirstRecordAsHeader()
                    .parse(reader);
            for (CSVRecord record : csvRecords) {
                records.add(record.toMap());
            }
        }
        return records;
    }
}














package com.example.csvprocessor;

import org.springframework.web.bind.annotation.*;
import org.springframework.core.io.ClassPathResource;
import java.io.*;
import java.nio.file.*;
import java.util.*;
import java.util.stream.Collectors;

@RestController
@RequestMapping("/api")
public class CsvController {

    private List<Map<String, String>> readCsv(String filename) throws IOException {
        InputStream inputStream = new ClassPathResource("static/files/" + filename).getInputStream();
        BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream));
        String[] headers = reader.readLine().split(",");
        List<Map<String, String>> data = new ArrayList<>();
        String line;
        while ((line = reader.readLine()) != null) {
            String[] values = line.split(",");
            Map<String, String> row = new HashMap<>();
            for (int i = 0; i < headers.length && i < values.length; i++) {
                row.put(headers[i], values[i]);
            }
            data.add(row);
        }
        return data;
    }

    private String toCsv(List<Map<String, String>> rows) {
        if (rows.isEmpty()) return "";
        List<String> headers = new ArrayList<>(rows.get(0).keySet());
        StringBuilder sb = new StringBuilder(String.join(",", headers)).append("\n");
        for (Map<String, String> row : rows) {
            sb.append(headers.stream().map(h -> row.getOrDefault(h, "")).collect(Collectors.joining(","))).append("\n");
        }
        return sb.toString();
    }

    @GetMapping("/update")
    public Map<String, Object> update(
            @RequestParam String file,
            @RequestParam String column,
            @RequestParam String oldValue,
            @RequestParam String newValue
    ) throws IOException {
        List<Map<String, String>> data = readCsv(file);
        int updated = 0;
        for (Map<String, String> row : data) {
            if (row.containsKey(column) && row.get(column).equals(oldValue)) {
                row.put(column, newValue);
                updated++;
            }
        }
        Map<String, Object> response = new HashMap<>();
        response.put("updatedCount", updated);
        response.put("updatedRecords", data);
        return response;
    }

    List<Map<String, String>> lastUpdated = new ArrayList<>();
    List<Map<String, String>> lastNonExcluded = new ArrayList<>();

    @GetMapping("/exclude")
    public Map<String, Object> exclude(
            @RequestParam String file,
            @RequestParam String column,
            @RequestParam String value
    ) throws IOException {
        lastUpdated = readCsv(file);
        List<Map<String, String>> excluded = new ArrayList<>();
        List<Map<String, String>> nonExcluded = new ArrayList<>();

        for (Map<String, String> row : lastUpdated) {
            if (row.getOrDefault(column, "").equals(value)) {
                excluded.add(row);
            } else {
                nonExcluded.add(row);
            }
        }

        lastNonExcluded = nonExcluded;

        Map<String, Object> response = new HashMap<>();
        response.put("excludedCount", excluded.size());
        response.put("nonExcludedCount", nonExcluded.size());
        response.put("excludedRecords", excluded);
        response.put("nonExcludedRecords", nonExcluded);
        return response;
    }

    @GetMapping("/match/{type}")
    public Map<String, Object> match(
            @PathVariable String type,
            @RequestParam String file1,
            @RequestParam String file2,
            @RequestParam String column1,
            @RequestParam String column2
    ) throws IOException {
        List<Map<String, String>> data1 = lastNonExcluded.isEmpty() ? readCsv(file1) : lastNonExcluded;
        List<Map<String, String>> data2 = readCsv(file2);

        Map<String, Object> response = new HashMap<>();
        List<Map<String, String>> matched = new ArrayList<>();
        List<Map<String, String>> unmatched1 = new ArrayList<>();
        List<Map<String, String>> unmatched2 = new ArrayList<>(data2);

        if (type.equalsIgnoreCase("1-1")) {
            for (Map<String, String> row1 : data1) {
                boolean found = false;
                for (Map<String, String> row2 : data2) {
                    if (row1.get(column1).equals(row2.get(column2))) {
                        matched.add(row1);
                        unmatched2.remove(row2);
                        found = true;
                        break;
                    }
                }
                if (!found) unmatched1.add(row1);
            }
        }

        // Extend logic for 1-many, many-1, many-many if needed

        response.put("matchedCount", matched.size());
        response.put("matchedRecords", matched);
        response.put("unmatchedInFile1", unmatched1);
        response.put("unmatchedInFile2", unmatched2);
        return response;
    }
}
