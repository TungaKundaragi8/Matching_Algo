// Record.java package com.example.reconciliation.model;

public class Record { private String algoRecord; private String starRecord; private String status;

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

// ReconciliationResult.java package com.example.reconciliation.model;

import java.util.List;

public class ReconciliationResult { private List<Record> matched; private List<Record> unmatched; private List<Record> excluded; private int matchedCount; private int unmatchedCount; private int excludedCount;

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

// ReconciliationService.java package com.example.reconciliation.service;

import com.example.reconciliation.model.Record; import com.example.reconciliation.model.ReconciliationResult; import org.apache.commons.csv.CSVFormat; import org.apache.commons.csv.CSVRecord; import org.springframework.stereotype.Service;

import java.io.FileReader; import java.util.*;

@Service public class ReconciliationService {

public ReconciliationResult reconcile(String algoPath, String starPath, String matchType) {
    try {
        FileReader readerAlgo = new FileReader(algoPath);
        List<CSVRecord> algoRecords = (List<CSVRecord>) CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(readerAlgo);

        FileReader readerStar = new FileReader(starPath);
        List<CSVRecord> starRecords = (List<CSVRecord>) CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(readerStar);

        List<CSVRecord> excluded = new ArrayList<>();
        List<CSVRecord> filteredStar = applyExclusionRule(starRecords, excluded);

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

    for (CSVRecord excludedRecord : excludedStar) {
        String excludedKey = excludedRecord.get("CRDS Party Code").replaceAll("\\s+", "") +
                "3" + excludedRecord.get("Post Direction").replaceAll("\\s+", "");
        excluded.add(new Record("<Excluded>", excludedKey, "Excluded by Rule (Maturity Date = 01-JAN-1900)"));
    }

    return new ReconciliationResult(matched, unmatched, excluded);
}

}
