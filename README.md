package com.example.reconciliation.service;

import com.example.reconciliation.model.Record; import com.example.reconciliation.model.ReconciliationResult; import org.apache.commons.csv.CSVFormat; import org.apache.commons.csv.CSVRecord; import org.springframework.stereotype.Service;

import java.io.FileReader; import java.util.*; import java.util.stream.Collectors;

@Service public class ReconciliationService {

public ReconciliationResult reconcile(String algoPath, String starPath, String matchType) {
    List<Record> matched = new ArrayList<>();
    List<Record> unmatched = new ArrayList<>();
    List<Record> excluded = new ArrayList<>();

    try {
        List<CSVRecord> algoRecords = readCsv(algoPath);
        List<CSVRecord> starRecords = readCsv(starPath);

        // Apply exclusion rule
        List<CSVRecord> filteredStarRecords = new ArrayList<>();
        for (CSVRecord record : starRecords) {
            String maturityDate = record.get("Maturity Date").trim();
            if (maturityDate.equals("Jan  1 1900 12:00:00:000AM")) {
                excluded.add(new Record("<Excluded>", record.toString(), "Excluded - Maturity Date Rule"));
            } else {
                filteredStarRecords.add(record);
            }
        }

        // Perform selected matching
        switch (matchType.toLowerCase()) {
            case "one-to-one":
                performOneToOneMatch(algoRecords, filteredStarRecords, matched, unmatched);
                break;
            case "one-to-many":
                performOneToManyMatch(algoRecords, filteredStarRecords, matched, unmatched);
                break;
            case "many-to-one":
                performManyToOneMatch(algoRecords, filteredStarRecords, matched, unmatched);
                break;
            case "many-to-many":
                performManyToManyMatch(algoRecords, filteredStarRecords, matched, unmatched);
                break;
            default:
                throw new IllegalArgumentException("Invalid match type");
        }

    } catch (Exception e) {
        e.printStackTrace();
    }

    return new ReconciliationResult(matched, unmatched, excluded);
}

private List<CSVRecord> readCsv(String path) throws Exception {
    FileReader reader = new FileReader(path);
    return (List<CSVRecord>) CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(reader);
}

private void performOneToOneMatch(List<CSVRecord> algoRecords, List<CSVRecord> starRecords,
                                  List<Record> matched, List<Record> unmatched) {
    Map<String, CSVRecord> starMap = new HashMap<>();
    Set<String> matchedStarKeys = new HashSet<>();

    for (CSVRecord record : starRecords) {
        String key = record.get("CRDS Party Code").trim() + "3" + record.get("Post Direction").trim();
        starMap.putIfAbsent(key, record);
    }

    for (CSVRecord record : algoRecords) {
        String algoKey = record.get("Agreement_name").replace("_RIMR", "3CR")
                .replace("_RIMP", "3CP").replaceAll("\\s+", "").trim();
        if (starMap.containsKey(algoKey) && !matchedStarKeys.contains(algoKey)) {
            matched.add(new Record(algoKey, algoKey, "Matched - 1:1"));
            matchedStarKeys.add(algoKey);
        } else {
            unmatched.add(new Record(algoKey, "<No Match>", "Unmatched - 1:1"));
        }
    }

    for (String starKey : starMap.keySet()) {
        if (!matchedStarKeys.contains(starKey)) {
            unmatched.add(new Record("<No Match>", starKey, "Unmatched STAR - 1:1"));
        }
    }
}

private void performOneToManyMatch(List<CSVRecord> algoRecords, List<CSVRecord> starRecords,
                                   List<Record> matched, List<Record> unmatched) {
    // Simplified logic
    Map<String, List<String>> starMap = new HashMap<>();
    for (CSVRecord record : starRecords) {
        String key = record.get("CRDS Party Code").trim();
        starMap.computeIfAbsent(key, k -> new ArrayList<>()).add(record.toString());
    }

    for (CSVRecord record : algoRecords) {
        String key = record.get("Agreement_name").replace("_RIMR", "3CR")
                .replace("_RIMP", "3CP").replaceAll("\\s+", "").trim();
        String starKey = key.substring(0, key.length() - 3); // assuming 3CR/3CP suffix
        List<String> matches = starMap.getOrDefault(starKey, Collections.emptyList());
        if (!matches.isEmpty()) {
            matched.add(new Record(key, matches.toString(), "Matched - 1:Many"));
        } else {
            unmatched.add(new Record(key, "<No Match>", "Unmatched - 1:Many"));
        }
    }
}

private void performManyToOneMatch(List<CSVRecord> algoRecords, List<CSVRecord> starRecords,
                                   List<Record> matched, List<Record> unmatched) {
    // Inverse of one-to-many
    performOneToManyMatch(starRecords, algoRecords, matched, unmatched);
}

private void performManyToManyMatch(List<CSVRecord> algoRecords, List<CSVRecord> starRecords,
                                    List<Record> matched, List<Record> unmatched) {
    Set<String> starSet = starRecords.stream().map(r ->
            r.get("CRDS Party Code").trim() + "3" + r.get("Post Direction").trim()
    ).collect(Collectors.toSet());

    for (CSVRecord record : algoRecords) {
        String key = record.get("Agreement_name").replace("_RIMR", "3CR")
                .replace("_RIMP", "3CP").replaceAll("\\s+", "").trim();
        if (starSet.contains(key)) {
            matched.add(new Record(key, key, "Matched - Many:Many"));
        } else {
            unmatched.add(new Record(key, "<No Match>", "Unmatched - Many:Many"));
        }
    }
}

}
