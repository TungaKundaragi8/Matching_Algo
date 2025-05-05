// ReconciliationController.java package com.example.reconciliation.controller;

import com.example.reconciliation.service.ReconciliationService; import com.example.reconciliation.model.ReconciliationResult; import org.springframework.beans.factory.annotation.Autowired; import org.springframework.web.bind.annotation.*;

@RestController @RequestMapping("/api") public class ReconciliationController {

@Autowired
private ReconciliationService service;

@GetMapping("/reconcile")
public ReconciliationResult reconcile(@RequestParam String matchType) {
    String algoPath = "C:/path/to/algo.csv";
    String starPath = "C:/path/to/star.csv";
    return service.reconcile(algoPath, starPath, matchType);
}

}

// ReconciliationService.java package com.example.reconciliation.service;

import com.example.reconciliation.model.Record; import com.example.reconciliation.model.ReconciliationResult; import org.apache.commons.csv.CSVFormat; import org.apache.commons.csv.CSVRecord; import org.springframework.stereotype.Service;

import java.io.FileReader; import java.util.*;

@Service public class ReconciliationService {

public ReconciliationResult reconcile(String algoPath, String starPath, String matchType) {
    List<String> algoKeys = new ArrayList<>();
    List<String> starKeys = new ArrayList<>();

    try {
        FileReader readerAlgo = new FileReader(algoPath);
        Iterable<CSVRecord> algoRecords = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(readerAlgo);
        for (CSVRecord record : algoRecords) {
            String key = record.get("Agreement_name")
                    .replace("_RIMR", "3CR")
                    .replace("_RIMP", "3CP")
                    .replaceAll("\\s+", "")
                    .trim();
            algoKeys.add(key);
        }

        FileReader readerStar = new FileReader(starPath);
        Iterable<CSVRecord> starRecords = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(readerStar);
        for (CSVRecord record : starRecords) {
            String key = record.get("CRDS Party Code").replaceAll("\\s+", "").trim() +
                    "3" +
                    record.get("Post Direction").replaceAll("\\s+", "").trim();
            starKeys.add(key);
        }
    } catch (Exception e) {
        e.printStackTrace();
    }

    switch (matchType.toLowerCase()) {
        case "1-1":
            return matchOneToOne(algoKeys, starKeys);
        case "1-many":
            return matchOneToMany(algoKeys, starKeys);
        case "many-1":
            return matchManyToOne(algoKeys, starKeys);
        case "many-many":
            return matchManyToMany(algoKeys, starKeys);
        default:
            throw new IllegalArgumentException("Invalid match type");
    }
}

private ReconciliationResult matchOneToOne(List<String> algoKeys, List<String> starKeys) {
    List<Record> matched = new ArrayList<>();
    List<Record> unmatched = new ArrayList<>();
    Set<Integer> matchedStarIndices = new HashSet<>();

    for (int i = 0; i < algoKeys.size(); i++) {
        String algoKey = algoKeys.get(i);
        int foundIndex = starKeys.indexOf(algoKey);
        if (foundIndex != -1 && !matchedStarIndices.contains(foundIndex)) {
            matched.add(new Record(algoKey, starKeys.get(foundIndex), "Match"));
            matchedStarIndices.add(foundIndex);
        } else {
            unmatched.add(new Record(algoKey, "<No Match>", "Mismatch"));
        }
    }

    for (int i = 0; i < starKeys.size(); i++) {
        if (!matchedStarIndices.contains(i)) {
            unmatched.add(new Record("<No Match>", starKeys.get(i), "Mismatch"));
        }
    }

    return new ReconciliationResult(matched, unmatched);
}

private ReconciliationResult matchOneToMany(List<String> algoKeys, List<String> starKeys) {
    List<Record> matched = new ArrayList<>();
    List<Record> unmatched = new ArrayList<>();
    Map<String, List<Integer>> starMap = new HashMap<>();

    for (int i = 0; i < starKeys.size(); i++) {
        starMap.computeIfAbsent(starKeys.get(i), k -> new ArrayList<>()).add(i);
    }

    for (String algoKey : algoKeys) {
        if (starMap.containsKey(algoKey)) {
            for (Integer idx : starMap.get(algoKey)) {
                matched.add(new Record(algoKey, starKeys.get(idx), "Match"));
            }
        } else {
            unmatched.add(new Record(algoKey, "<No Match>", "Mismatch"));
        }
    }

    return new ReconciliationResult(matched, unmatched);
}

private ReconciliationResult matchManyToOne(List<String> algoKeys, List<String> starKeys) {
    List<Record> matched = new ArrayList<>();
    List<Record> unmatched = new ArrayList<>();
    Map<String, List<Integer>> algoMap = new HashMap<>();

    for (int i = 0; i < algoKeys.size(); i++) {
        algoMap.computeIfAbsent(algoKeys.get(i), k -> new ArrayList<>()).add(i);
    }

    for (String starKey : starKeys) {
        if (algoMap.containsKey(starKey)) {
            for (Integer idx : algoMap.get(starKey)) {
                matched.add(new Record(algoKeys.get(idx), starKey, "Match"));
            }
        } else {
            unmatched.add(new Record("<No Match>", starKey, "Mismatch"));
        }
    }

    return new ReconciliationResult(matched, unmatched);
}

private ReconciliationResult matchManyToMany(List<String> algoKeys, List<String> starKeys) {
    List<Record> matched = new ArrayList<>();
    List<Record> unmatched = new ArrayList<>();
    Set<String> starSet = new HashSet<>(starKeys);

    for (String algoKey : algoKeys) {
        if (starSet.contains(algoKey)) {
            matched.add(new Record(algoKey, algoKey, "Match"));
        } else {
            unmatched.add(new Record(algoKey, "<No Match>", "Mismatch"));
        }
    }

    for (String starKey : starKeys) {
        if (!algoKeys.contains(starKey)) {
            unmatched.add(new Record("<No Match>", starKey, "Mismatch"));
        }
    }

    return new ReconciliationResult(matched, unmatched);
}

}

// Record.java package com.example.reconciliation.model;

public class Record { private String algoKey; private String starKey; private String status;

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

// ReconciliationResult.java package com.example.reconciliation.model;

import java.util.List;

public class ReconciliationResult { private List<Record> matched; private List<Record> unmatched;

public ReconciliationResult(List<Record> matched, List<Record> unmatched) {
    this.matched = matched;
    this.unmatched = unmatched;
}

public List<Record> getMatched() { return matched; }
public void setMatched(List<Record> matched) { this.matched = matched; }

public List<Record> getUnmatched() { return unmatched; }
public void setUnmatched(List<Record> unmatched) { this.unmatched = unmatched; }

}







public ReconciliationResult reconcile(String algoPath, String starPath, String matchType) {
    // Use the matchType value to switch between matching algorithms
    switch (matchType.toLowerCase()) {
        case "one-to-one":
            return oneToOneMatching(algoPath, starPath);
        case "one-to-many":
            return oneToManyMatching(algoPath, starPath);
        case "many-to-one":
            return manyToOneMatching(algoPath, starPath);
        case "many-to-many":
            return manyToManyMatching(algoPath, starPath);
        default:
            throw new IllegalArgumentException("Invalid match type");
    }
}


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
        switch (matchType.toLowerCase()) {
            case "one-to-one":
                return performOneToOne(algoPath, starPath);
            case "one-to-many":
                return performOneToMany(algoPath, starPath);
            case "many-to-one":
                return performManyToOne(algoPath, starPath);
            case "many-to-many":
                return performManyToMany(algoPath, starPath);
            default:
                throw new IllegalArgumentException("Invalid matchType. Use one-to-one, one-to-many, many-to-one, or many-to-many.");
        }
    }

    private ReconciliationResult performOneToOne(String algoPath, String starPath) {
        // Your existing one-to-one matching logic (you can reuse your working implementation)
        return baseReconciliation(algoPath, starPath, "one-to-one");
    }

    private ReconciliationResult performOneToMany(String algoPath, String starPath) {
        return baseReconciliation(algoPath, starPath, "one-to-many");
    }

    private ReconciliationResult performManyToOne(String algoPath, String starPath) {
        return baseReconciliation(algoPath, starPath, "many-to-one");
    }

    private ReconciliationResult performManyToMany(String algoPath, String starPath) {
        return baseReconciliation(algoPath, starPath, "many-to-many");
    }

    private ReconciliationResult baseReconciliation(String algoPath, String starPath, String type) {
        List<String> algoKeys = new ArrayList<>();
        List<String> starKeys = new ArrayList<>();
        List<Record> matched = new ArrayList<>();
        List<Record> unmatched = new ArrayList<>();

        try {
            FileReader readerAlgo = new FileReader(algoPath);
            Iterable<CSVRecord> algoRecords = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(readerAlgo);

            for (CSVRecord record : algoRecords) {
                String key = record.get("Agreement_name")
                        .replace("_RIMR", "3CR")
                        .replace("_RIMP", "3CP")
                        .replaceAll("\\s+", "")
                        .trim();
                algoKeys.add(key);
            }

            FileReader readerStar = new FileReader(starPath);
            Iterable<CSVRecord> starRecords = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(readerStar);
            Map<String, List<Integer>> starKeyMap = new HashMap<>();

            int index = 0;
            for (CSVRecord record : starRecords) {
                String key = record.get("CRDS Party Code").replaceAll("\\s+", "").trim()
                        + "3" + record.get("Post Direction").replaceAll("\\s+", "").trim();
                starKeys.add(key);
                starKeyMap.computeIfAbsent(key, k -> new ArrayList<>()).add(index);
                index++;
            }

            Set<Integer> matchedStarIndices = new HashSet<>();
            Set<Integer> matchedAlgoIndices = new HashSet<>();

            for (int i = 0; i < algoKeys.size(); i++) {
                String algoKey = algoKeys.get(i);
                List<Integer> matchingStarIdxs = starKeyMap.getOrDefault(algoKey, new ArrayList<>());

                if (type.equals("one-to-one")) {
                    if (matchingStarIdxs.size() == 1 && !matchedStarIndices.contains(matchingStarIdxs.get(0))) {
                        matched.add(new Record(algoKey, starKeys.get(matchingStarIdxs.get(0)), "Match"));
                        matchedStarIndices.add(matchingStarIdxs.get(0));
                        matchedAlgoIndices.add(i);
                    } else if (matchingStarIdxs.size() > 1) {
                        for (Integer idx : matchingStarIdxs) {
                            unmatched.add(new Record(algoKey, starKeys.get(idx), "Mismatch - Multiple STAR Keys"));
                        }
                        matchedAlgoIndices.add(i);
                    } else {
                        unmatched.add(new Record(algoKey, "<No Match>", "Mismatch - No Match"));
                    }
                } else if (type.equals("one-to-many")) {
                    if (!matchingStarIdxs.isEmpty()) {
                        for (Integer idx : matchingStarIdxs) {
                            matched.add(new Record(algoKey, starKeys.get(idx), "Match (One-to-Many)"));
                            matchedStarIndices.add(idx);
                        }
                        matchedAlgoIndices.add(i);
                    } else {
                        unmatched.add(new Record(algoKey, "<No Match>", "Mismatch"));
                    }
                } else if (type.equals("many-to-one")) {
                    if (!matchingStarIdxs.isEmpty()) {
                        matched.add(new Record(algoKey, starKeys.get(matchingStarIdxs.get(0)), "Match (Many-to-One)"));
                        matchedStarIndices.add(matchingStarIdxs.get(0));
                        matchedAlgoIndices.add(i);
                    } else {
                        unmatched.add(new Record(algoKey, "<No Match>", "Mismatch"));
                    }
                } else if (type.equals("many-to-many")) {
                    for (Integer idx : matchingStarIdxs) {
                        matched.add(new Record(algoKey, starKeys.get(idx), "Match (Many-to-Many)"));
                        matchedStarIndices.add(idx);
                    }
                    if (matchingStarIdxs.isEmpty()) {
                        unmatched.add(new Record(algoKey, "<No Match>", "Mismatch"));
                    }
                    matchedAlgoIndices.add(i);
                }
            }

            for (int i = 0; i < starKeys.size(); i++) {
                if (!matchedStarIndices.contains(i)) {
                    unmatched.add(new Record("<No Match>", starKeys.get(i), "Unmatched STAR Key"));
                }
            }

        } catch (Exception e) {
            e.printStackTrace();
        }

        return new ReconciliationResult(matched, unmatched);
    }
}

