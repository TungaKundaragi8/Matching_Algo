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
