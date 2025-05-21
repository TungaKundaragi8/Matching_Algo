package com.example.demo.controller;

import com.example.demo.service.ReconciliationService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/api")
public class ReconciliationController {

    @Autowired
    private ReconciliationService service;

    @GetMapping("/update")
    public Map<String, Object> updateFile(
            @RequestParam String filePath,
            @RequestParam Map<String, String> updates,
            @RequestParam(required = false) List<String> appendColumns) {
        return service.updateRecords(filePath, updates, appendColumns);
    }

    @GetMapping("/exclude")
    public Map<String, Object> excludeRecords(
            @RequestParam String filePath,
            @RequestParam String column,
            @RequestParam String value) {
        return service.excludeRecords(filePath, column, value);
    }

    @GetMapping("/match")
    public Map<String, Object> matchFiles(
            @RequestParam String file1,
            @RequestParam String file2,
            @RequestParam String matchType,
            @RequestParam String col1,
            @RequestParam String col2) {
        return service.matchRecords(file1, file2, matchType, col1, col2);
    }
}
package com.example.demo.service;

import org.apache.commons.csv.*;
import org.springframework.stereotype.Service;

import java.io.*;
import java.nio.file.*;
import java.util.*;
import java.util.stream.Collectors;

@Service
public class ReconciliationService {

    private List<Map<String, String>> unmatchedRecords = new ArrayList<>();
    private boolean isFirstMatch = true;

    public Map<String, Object> updateRecords(String filePath, Map<String, String> updates, List<String> appendCols) {
        Map<String, Object> result = new HashMap<>();
        try (Reader reader = Files.newBufferedReader(Paths.get(filePath))) {
            CSVParser parser = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(reader);
            List<CSVRecord> records = parser.getRecords();
            List<Map<String, String>> updated = new ArrayList<>();

            for (CSVRecord record : records) {
                Map<String, String> row = new HashMap<>(record.toMap());
                boolean modified = false;

                for (String key : updates.keySet()) {
                    String[] parts = key.split(":");
                    String column = parts[0];
                    String oldVal = parts.length > 1 ? parts[1] : null;
                    String newValue = updates.get(key);

                    if (row.containsKey(column)) {
                        String currentVal = row.get(column);
                        boolean shouldUpdate = (oldVal == null || currentVal.equals(oldVal));

                        if (shouldUpdate) {
                            if (appendCols != null && appendCols.contains(column)) {
                                row.put(column, currentVal + newValue);
                            } else {
                                row.put(column, newValue);
                            }
                            modified = true;
                        }
                    }
                }
                if (modified) updated.add(row);
            }

            result.put("updatedRecords", updated);
            result.put("updatedCount", updated.size());
        } catch (Exception e) {
            result.put("error", e.getMessage());
        }
        return result;
    }

    public Map<String, Object> excludeRecords(String filePath, String column, String value) {
        Map<String, Object> result = new HashMap<>();
        try (Reader reader = Files.newBufferedReader(Paths.get(filePath))) {
            CSVParser parser = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(reader);
            List<CSVRecord> records = parser.getRecords();

            List<Map<String, String>> excluded = new ArrayList<>();
            List<Map<String, String>> nonExcluded = new ArrayList<>();

            for (CSVRecord record : records) {
                Map<String, String> row = record.toMap();
                if (row.getOrDefault(column, "").equals(value)) {
                    excluded.add(row);
                } else {
                    nonExcluded.add(row);
                }
            }

            unmatchedRecords = new ArrayList<>(nonExcluded);
            isFirstMatch = true;

            result.put("excludedCount", excluded.size());
            result.put("excludedRecords", excluded);
            result.put("nonExcludedCount", nonExcluded.size());
            result.put("nonExcludedRecords", nonExcluded);
        } catch (Exception e) {
            result.put("error", e.getMessage());
        }
        return result;
    }

    public Map<String, Object> matchRecords(String file1, String file2, String matchType, String col1, String col2) {
        Map<String, Object> result = new HashMap<>();
        try {
            List<Map<String, String>> records1;
            if (isFirstMatch) {
                records1 = unmatchedRecords;
                isFirstMatch = false;
            } else {
                records1 = unmatchedRecords;
            }

            List<Map<String, String>> records2 = readCsv(file2);
            List<Map<String, String>> matched = new ArrayList<>();
            List<Map<String, String>> unmatched = new ArrayList<>();

            switch (matchType.toLowerCase()) {
                case "1-1":
                    for (Map<String, String> r1 : records1) {
                        boolean found = false;
                        for (Map<String, String> r2 : records2) {
                            if (r1.get(col1).equals(r2.get(col2))) {
                                matched.add(mergeRecords(r1, r2));
                                found = true;
                                break;
                            }
                        }
                        if (!found) unmatched.add(r1);
                    }
                    break;
                case "1-many":
                    for (Map<String, String> r1 : records1) {
                        boolean found = false;
                        for (Map<String, String> r2 : records2) {
                            if (r1.get(col1).equals(r2.get(col2))) {
                                matched.add(mergeRecords(r1, r2));
                                found = true;
                            }
                        }
                        if (!found) unmatched.add(r1);
                    }
                    break;
                case "many-1":
                    for (Map<String, String> r2 : records2) {
                        boolean found = false;
                        for (Map<String, String> r1 : records1) {
                            if (r1.get(col1).equals(r2.get(col2))) {
                                matched.add(mergeRecords(r1, r2));
                                found = true;
                            }
                        }
                        if (!found) unmatched.add(r2);
                    }
                    break;
                case "many-many":
                    for (Map<String, String> r1 : records1) {
                        for (Map<String, String> r2 : records2) {
                            if (r1.get(col1).equals(r2.get(col2))) {
                                matched.add(mergeRecords(r1, r2));
                            }
                        }
                    }
                    Set<Map<String, String>> matchedSet = new HashSet<>(matched);
                    unmatched = records1.stream().filter(r -> !matchedSet.contains(r)).collect(Collectors.toList());
                    break;
                default:
                    result.put("error", "Invalid match type");
                    return result;
            }

            unmatchedRecords = unmatched;

            result.put("matchedCount", matched.size());
            result.put("matchedRecords", matched);
            result.put("unmatchedCount", unmatched.size());
            result.put("unmatchedRecords", unmatched);
        } catch (Exception e) {
            result.put("error", e.getMessage());
        }
        return result;
    }

    private List<Map<String, String>> readCsv(String path) throws IOException {
        try (Reader reader = Files.newBufferedReader(Paths.get(path))) {
            CSVParser parser = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(reader);
            return parser.getRecords().stream().map(CSVRecord::toMap).collect(Collectors.toList());
        }
    }

    private Map<String, String> mergeRecords(Map<String, String> r1, Map<String, String> r2) {
        Map<String, String> merged = new HashMap<>(r1);
        for (Map.Entry<String, String> entry : r2.entrySet()) {
            merged.putIfAbsent("file2_" + entry.getKey(), entry.getValue());
        }
        return merged;
    }
}
