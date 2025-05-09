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
    private List<Record> excludedRecords = new ArrayList<>();

    public ReconciliationResult excludeAndTransform(String algoPath, String starPath) {
        try {
            CSVParser algoParser = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(new FileReader(algoPath));
            CSVParser starParser = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(new FileReader(starPath));

            List<CSVRecord> algoRecords = algoParser.getRecords();
            List<CSVRecord> starRecords = starParser.getRecords();

            nonExcludedAlgo.clear();
            nonExcludedStar.clear();
            excludedRecords.clear();

            for (CSVRecord record : starRecords) {
                String maturityDate = record.get("Maturity Date").trim().toLowerCase();
                if (maturityDate.contains("1900")) {
                    String starKey = record.get("CRDS Party Code").replaceAll("\\s+", "") +
                                     "3" + record.get("Post Direction").replaceAll("\\s+", "");
                    excludedRecords.add(new Record("<Excluded>", starKey, "Excluded by Maturity Date"));
                } else {
                    nonExcludedStar.add(record);
                }
            }

            nonExcludedAlgo.addAll(algoRecords);

            return new ReconciliationResult(
                Collections.emptyList(),
                Collections.emptyList(),
                excludedRecords
            );

        } catch (Exception e) {
            e.printStackTrace();
            return new ReconciliationResult(Collections.emptyList(), Collections.emptyList(), Collections.emptyList());
        }
    }

    public ReconciliationResult match(String type) {
        List<Record> matched = new ArrayList<>();
        List<Record> unmatched = new ArrayList<>();

        List<String> algoKeys = new ArrayList<>();
        for (CSVRecord record : nonExcludedAlgo) {
            String key = record.get("Agreement_name")
                    .replace("_RIMP", "3CP")
                    .replace("_RIMR", "3CR")
                    .replaceAll("\\s+", "").trim();
            algoKeys.add(key);
        }

        Map<String, List<Integer>> starKeyMap = new HashMap<>();
        List<String> starKeys = new ArrayList<>();
        for (int i = 0; i < nonExcludedStar.size(); i++) {
            CSVRecord record = nonExcludedStar.get(i);
            String key = record.get("CRDS Party Code").replaceAll("\\s+", "") +
                         "3" + record.get("Post Direction").replaceAll("\\s+", "").trim();
            starKeys.add(key);
            starKeyMap.computeIfAbsent(key, k -> new ArrayList<>()).add(i);
        }

        Set<Integer> matchedAlgoIndexes = new HashSet<>();
        Set<Integer> matchedStarIndexes = new HashSet<>();

        for (int i = 0; i < algoKeys.size(); i++) {
            String algoKey = algoKeys.get(i);
            List<Integer> matchIndexes = starKeyMap.getOrDefault(algoKey, new ArrayList<>());

            switch (type.toLowerCase()) {
                case "one-to-one":
                    if (matchIndexes.size() == 1 && !matchedStarIndexes.contains(matchIndexes.get(0))) {
                        matched.add(new Record(algoKey, starKeys.get(matchIndexes.get(0)), "Match 1-1"));
                        matchedAlgoIndexes.add(i);
                        matchedStarIndexes.add(matchIndexes.get(0));
                    } else {
                        unmatched.add(new Record(algoKey, "<No Match>", "Mismatch"));
                    }
                    break;

                case "one-to-many":
                    if (!matchIndexes.isEmpty()) {
                        for (Integer idx : matchIndexes) {
                            matched.add(new Record(algoKey, starKeys.get(idx), "Match 1-M"));
                            matchedStarIndexes.add(idx);
                        }
                        matchedAlgoIndexes.add(i);
                    } else {
                        unmatched.add(new Record(algoKey, "<No Match>", "Mismatch"));
                    }
                    break;

                case "many-to-one":
                    if (!matchIndexes.isEmpty()) {
                        matched.add(new Record(algoKey, starKeys.get(matchIndexes.get(0)), "Match M-1"));
                        matchedAlgoIndexes.add(i);








                        public ReconciliationResult excludeAndTransform(String algoPath, String starPath) {
    try {
        CSVParser algoParser = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(new FileReader(algoPath));
        CSVParser starParser = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(new FileReader(starPath));

        List<CSVRecord> algoRecords = algoParser.getRecords();
        List<CSVRecord> starRecords = starParser.getRecords();

        nonExcludedAlgo.clear();
        nonExcludedStar.clear();
        excludedRecords.clear();

        for (CSVRecord record : starRecords) {
            String maturityDate = record.get("Maturity Date").trim().toLowerCase();
            if (maturityDate.contains("1900")) {
                excludedRecords.add(new Record(
                    "<Excluded>",
                    record.get("CRDS Party Code").replaceAll("\\s+", "") +
                    "3" + record.get("Post Direction").replaceAll("\\s+", ""),
                    "Excluded by Maturity Date"));
            } else {
                nonExcludedStar.add(record);
            }
        }

        nonExcludedAlgo.addAll(algoRecords);

        // Build transformed non-excluded STAR records to return
        List<Record> transformedNonExcluded = new ArrayList<>();
        for (CSVRecord record : nonExcludedStar) {
            String starKey = record.get("CRDS Party Code").replaceAll("\\s+", "") +
                             "3" + record.get("Post Direction").replaceAll("\\s+", "").trim();
            transformedNonExcluded.add(new Record("<Non-Excluded>", starKey, "Eligible for Matching"));
        }

        // Include count as a separate record for clarity
        List<Record> countInfo = new ArrayList<>();
        countInfo.add(new Record("Excluded Count", String.valueOf(excludedRecords.size()), "Excluded"));

        return new ReconciliationResult(
            Collections.emptyList(),           // matched
            transformedNonExcluded,           // show non-excluded STAR records
            countInfo                          // show only the count of excluded
        );

    } catch (Exception e) {
        e.printStackTrace();
        return new ReconciliationResult(Collections.emptyList(), Collections.emptyList(), Collections.emptyList());
    }
}
                        matchedStarIndexes.add(matchIndexes.get(0));
                    } else {
                        unmatched.add(new Record(algoKey, "<No Match>", "Mismatch"));
                    }
                    break;

                case "many-to-many":
                    if (!matchIndexes.isEmpty()) {
                        for (Integer idx : matchIndexes) {
                            matched.add(new Record(algoKey, starKeys.get(idx), "Match M-M"));
                            matchedStarIndexes.add(idx);
                        }
                        matchedAlgoIndexes.add(i);
                    } else {
                        unmatched.add(new Record(algoKey, "<No Match>", "Mismatch"));
                    }
                    break;

                default:
                    unmatched.add(new Record(algoKey, "<Invalid Type>", "Error"));
            }
        }

        // Update for next round of matching
        List<CSVRecord> nextAlgo = new ArrayList<>();
        for (int i = 0; i < nonExcludedAlgo.size(); i++) {
            if (!matchedAlgoIndexes.contains(i)) nextAlgo.add(nonExcludedAlgo.get(i));
        }

        List<CSVRecord> nextStar = new ArrayList<>();
        for (int i = 0; i < nonExcludedStar.size(); i++) {
            if (!matchedStarIndexes.contains(i)) nextStar.add(nonExcludedStar.get(i));
        }

        nonExcludedAlgo = nextAlgo;
        nonExcludedStar = nextStar;

        return new ReconciliationResult(matched, unmatched, Collections.emptyList());
    }
}





package com.example.reconciliation.controller;

import com.example.reconciliation.model.ReconciliationResult;
import com.example.reconciliation.service.ReconciliationService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/reconcile")
public class ReconciliationController {

    @Autowired
    private ReconciliationService reconciliationService;

    private final String algoPath = "C:/path/to/algo.csv";
    private final String starPath = "C:/path/to/star.csv";

    @GetMapping("/exclude")
    public ReconciliationResult exclude() {
        return reconciliationService.excludeAndTransform(algoPath, starPath);
    }

    @GetMapping("/match/{type}")
    public ReconciliationResult match(@PathVariable String type) {
        return reconciliationService.match(type);
    }
}





public ReconciliationResult excludeAndTransform(String algoPath, String starPath) {
    try {
        CSVParser algoParser = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(new FileReader(algoPath));
        CSVParser starParser = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(new FileReader(starPath));

        List<CSVRecord> algoRecords = algoParser.getRecords();
        List<CSVRecord> starRecords = starParser.getRecords();

        nonExcludedAlgo.clear();
        nonExcludedStar.clear();
        excludedRecords.clear();

        for (CSVRecord record : starRecords) {
            String maturityDate = record.get("Maturity Date").trim().toLowerCase();
            if (maturityDate.contains("1900")) {
                excludedRecords.add(new Record(
                    "<Excluded>",
                    record.get("CRDS Party Code").replaceAll("\\s+", "") +
                    "3" + record.get("Post Direction").replaceAll("\\s+", ""),
                    "Excluded by Maturity Date"));
            } else {
                nonExcludedStar.add(record);
            }
        }

        nonExcludedAlgo.addAll(algoRecords);

        // Transform non-excluded records into Result format to return
        List<Record> transformedNonExcluded = new ArrayList<>();
        for (CSVRecord record : nonExcludedStar) {
            String starKey = record.get("CRDS Party Code").replaceAll("\\s+", "") +
                             "3" + record.get("Post Direction").replaceAll("\\s+", "");
            transformedNonExcluded.add(new Record("<Non-Excluded>", starKey, "Kept for Matching"));
        }

        return new ReconciliationResult(
            Collections.emptyList(),            // No matched
            transformedNonExcluded,            // Show non-excluded
            Collections.singletonList(         // Represent count in one record
                new Record("Excluded Count", String.valueOf(excludedRecords.size()), "Excluded Records")
            )
        );

    } catch (Exception e) {
        e.printStackTrace();
        return new ReconciliationResult(Collections.emptyList(), Collections.emptyList(), Collections.emptyList());
    }
}





public ReconciliationResult excludeAndTransform(String algoPath, String starPath) {
    try {
        CSVParser algoParser = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(new FileReader(algoPath));
        CSVParser starParser = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(new FileReader(starPath));

        List<CSVRecord> algoRecords = algoParser.getRecords();
        List<CSVRecord> starRecords = starParser.getRecords();

        nonExcludedAlgo.clear();
        nonExcludedStar.clear();
        excludedRecords.clear();

        for (CSVRecord record : starRecords) {
            String maturityDate = record.get("Maturity Date").trim().toLowerCase();
            if (maturityDate.contains("1900")) {
                excludedRecords.add(new Record(
                    "<Excluded>",
                    record.get("CRDS Party Code").replaceAll("\\s+", "") +
                    "3" + record.get("Post Direction").replaceAll("\\s+", ""),
                    "Excluded by Maturity Date"));
            } else {
                nonExcludedStar.add(record);
            }
        }

        nonExcludedAlgo.addAll(algoRecords);

        // Prepare transformed non-excluded STAR records
        List<Record> transformedNonExcluded = new ArrayList<>();
        for (CSVRecord record : nonExcludedStar) {
            String starKey = record.get("CRDS Party Code").replaceAll("\\s+", "") +
                             "3" + record.get("Post Direction").replaceAll("\\s+", "").trim();
            transformedNonExcluded.add(new Record("<Non-Excluded>", starKey, "Eligible for Matching"));
        }

        // Add count info to the excluded list
        List<Record> countSummary = new ArrayList<>();
        countSummary.add(new Record("Excluded Count", String.valueOf(excludedRecords.size()), "Summary"));
        countSummary.add(new Record("Non-Excluded Count", String.valueOf(nonExcludedStar.size()), "Summary"));

        return new ReconciliationResult(
            Collections.emptyList(),       // matched
            transformedNonExcluded,       // show non-excluded STAR records
            countSummary                   // show counts
        );

    } catch (Exception e) {
        e.printStackTrace();
        return new ReconciliationResult(Collections.emptyList(), Collections.emptyList(), Collections.emptyList());
    }
}





package com.example.reconciliation.service;

import com.example.reconciliation.model.Record; import com.example.reconciliation.model.ReconciliationResult; import org.apache.commons.csv.CSVFormat; import org.apache.commons.csv.CSVParser; import org.apache.commons.csv.CSVRecord; import org.springframework.stereotype.Service;

import java.io.FileReader; import java.util.*;

@Service public class ReconciliationService {

private List<CSVRecord> nonExcludedAlgo = new ArrayList<>();
private List<CSVRecord> nonExcludedStar = new ArrayList<>();
private List<Record> transformedNonExcludedRecords = new ArrayList<>();
private int excludedCount = 0;

public ReconciliationResult excludeAndTransform(String algoPath, String starPath, String maturityDateInput) {
    try {
        CSVParser algoParser = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(new FileReader(algoPath));
        CSVParser starParser = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(new FileReader(starPath));

        List<CSVRecord> algoRecords = algoParser.getRecords();
        List<CSVRecord> starRecords = starParser.getRecords();

        nonExcludedAlgo.clear();
        nonExcludedStar.clear();
        transformedNonExcludedRecords.clear();
        excludedCount = 0;

        for (CSVRecord record : starRecords) {
            String maturityDate = record.get("Maturity Date").trim();
            if (maturityDate.equalsIgnoreCase(maturityDateInput.trim())) {
                excludedCount++;
            } else {
                nonExcludedStar.add(record);
            }
        }

        nonExcludedAlgo.addAll(algoRecords);

        for (CSVRecord algo : nonExcludedAlgo) {
            String algoKey = algo.get("Agreement_name")
                    .replace("_RIMP", "3CP")
                    .replace("_RIMR", "3CR")
                    .replaceAll("\\s+", "").trim();
            transformedNonExcludedRecords.add(new Record(algoKey, "<ALGO>", "NonExcluded"));
        }

        for (CSVRecord star : nonExcludedStar) {
            String starKey = star.get("CRDS Party Code").replaceAll("\\s+", "") +
                    "3" + star.get("Post Direction").replaceAll("\\s+", "").trim();
            transformedNonExcludedRecords.add(new Record("<STAR>", starKey, "NonExcluded"));
        }

        return new ReconciliationResult(
                transformedNonExcludedRecords,
                Collections.emptyList(),
                List.of(new Record("Excluded Count", String.valueOf(excludedCount),
                        "NonExcluded Count: " + transformedNonExcludedRecords.size()))
        );

    } catch (Exception e) {
        e.printStackTrace();
        return new ReconciliationResult(Collections.emptyList(), Collections.emptyList(), Collections.emptyList());
    }
}

public ReconciliationResult match(String type) {
    List<Record> matched = new ArrayList<>();
    List<Record> unmatched = new ArrayList<>();

    List<String> algoKeys = new ArrayList<>();
    for (CSVRecord record : nonExcludedAlgo) {
        String key = record.get("Agreement_name")
                .replace("_RIMP", "3CP")
                .replace("_RIMR", "3CR")
                .replaceAll("\\s+", "").trim();
        algoKeys.add(key);
    }

    Map<String, List<Integer>> starKeyMap = new HashMap<>();
    List<String> starKeys = new ArrayList<>();
    for (int i = 0; i < nonExcludedStar.size(); i++) {
        CSVRecord record = nonExcludedStar.get(i);
        String key = record.get("CRDS Party Code").replaceAll("\\s+", "") +
                "3" + record.get("Post Direction").replaceAll("\\s+", "").trim();
        starKeys.add(key);
        starKeyMap.computeIfAbsent(key, k -> new ArrayList<>()).add(i);
    }

    Set<Integer> matchedAlgoIndexes = new HashSet<>();
    Set<Integer> matchedStarIndexes = new HashSet<>();

    for (int i = 0; i < algoKeys.size(); i++) {
        String algoKey = algoKeys.get(i);
        List<Integer> matchIndexes = starKeyMap.getOrDefault(algoKey, new ArrayList<>());

        switch (type.toLowerCase()) {
            case "one-to-one":
                if (matchIndexes.size() == 1 && !matchedStarIndexes.contains(matchIndexes.get(0))) {
                    matched.add(new Record(algoKey, starKeys.get(matchIndexes.get(0)), "Match 1-1"));
                    matchedAlgoIndexes.add(i);
                    matchedStarIndexes.add(matchIndexes.get(0));
                } else {
                    unmatched.add(new Record(algoKey, "<No Match>", "Mismatch"));
                }
                break;

            case "one-to-many":
                if (!matchIndexes.isEmpty()) {
                    for (Integer idx : matchIndexes) {
                        matched.add(new Record(algoKey, starKeys.get(idx), "Match 1-M"));
                        matchedStarIndexes.add(idx);
                    }
                    matchedAlgoIndexes.add(i);
                } else {
                    unmatched.add(new Record(algoKey, "<No Match>", "Mismatch"));
                }
                break;

            case "many-to-one":
                if (!matchIndexes.isEmpty()) {
                    matched.add(new Record(algoKey, starKeys.get(matchIndexes.get(0)), "Match M-1"));
                    matchedAlgoIndexes.add(i);
                    matchedStarIndexes.add(matchIndexes.get(0));
                } else {
                    unmatched.add(new Record(algoKey, "<No Match>", "Mismatch"));
                }
                break;

            case "many-to-many":
                if (!matchIndexes.isEmpty()) {
                    for (Integer idx : matchIndexes) {
                        matched.add(new Record(algoKey, starKeys.get(idx), "Match M-M"));
                        matchedStarIndexes.add(idx);
                    }
                    matchedAlgoIndexes.add(i);
                } else {
                    unmatched.add(new Record(algoKey, "<No Match>", "Mismatch"));
                }
                break;

            default:
                unmatched.add(new Record(algoKey, "<Invalid Type>", "Error"));
        }
    }

    List<CSVRecord> nextAlgo = new ArrayList<>();
    for (int i = 0; i < nonExcludedAlgo.size(); i++) {
        if (!matchedAlgoIndexes.contains(i)) nextAlgo.add(nonExcludedAlgo.get(i));
    }
    List<CSVRecord> nextStar = new ArrayList<>();
    for (int i = 0; i < nonExcludedStar.size(); i++) {
        if (!matchedStarIndexes.contains(i)) nextStar.add(nonExcludedStar.get(i));
    }

    nonExcludedAlgo = nextAlgo;
    nonExcludedStar = nextStar;

    return new ReconciliationResult(matched, unmatched, Collections.emptyList());
}

}






package com.example.reconciliation.model;

import java.util.List;

public class ReconciliationResult {
    private List<Record> matched;
    private List<Record> unmatched;
    private List<Record> nonExcluded;
    private int excludedCount;
    private int nonExcludedCount;

    public ReconciliationResult(List<Record> matched, List<Record> unmatched,
                                List<Record> nonExcluded, int excludedCount, int nonExcludedCount) {
        this.matched = matched;
        this.unmatched = unmatched;
        this.nonExcluded = nonExcluded;
        this.excludedCount = excludedCount;
        this.nonExcludedCount = nonExcludedCount;
    }

    public List<Record> getMatched() {
        return matched;
    }

    public List<Record> getUnmatched() {
        return unmatched;
    }

    public List<Record> getNonExcluded() {
        return nonExcluded;
    }

    public int getExcludedCount() {
        return excludedCount;
    }

    public int getNonExcludedCount() {
        return nonExcludedCount;
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

    private List<CSVRecord> nonExcludedAlgo = new ArrayList<>();
    private List<CSVRecord> nonExcludedStar = new ArrayList<>();
    private int excludedCount = 0;

    public ReconciliationResult excludeAndTransform(String algoPath, String starPath, String exclusionDate) {
        try {
            CSVParser algoParser = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(new FileReader(algoPath));
            CSVParser starParser = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(new FileReader(starPath));

            List<CSVRecord> algoRecords = algoParser.getRecords();
            List<CSVRecord> starRecords = starParser.getRecords();

            nonExcludedAlgo.clear();
            nonExcludedStar.clear();
            excludedCount = 0;

            for (CSVRecord record : starRecords) {
                String maturityDate = record.get("Maturity Date").trim();
                if (maturityDate.equalsIgnoreCase(exclusionDate)) {
                    excludedCount++;
                } else {
                    nonExcludedStar.add(record);
                }
            }

            nonExcludedAlgo.addAll(algoRecords);

            List<Record> nonExcludedList = new ArrayList<>();
            for (CSVRecord record : nonExcludedStar) {
                String starKey = record.get("CRDS Party Code").replaceAll("\\s+", "") +
                                 "3" + record.get("Post Direction").replaceAll("\\s+", "");
                nonExcludedList.add(new Record("<NonExcluded>", starKey, "Included"));
            }

            return new ReconciliationResult(
                    Collections.emptyList(),
                    Collections.emptyList(),
                    nonExcludedList,
                    excludedCount,
                    nonExcludedList.size()
            );

        } catch (Exception e) {
            e.printStackTrace();
            return new ReconciliationResult(Collections.emptyList(), Collections.emptyList(), Collections.emptyList(), 0, 0);
        }
    }

    public ReconciliationResult match(String type) {
        List<Record> matched = new ArrayList<>();
        List<Record> unmatched = new ArrayList<>();

        List<String> algoKeys = new ArrayList<>();
        for (CSVRecord record : nonExcludedAlgo) {
            String key = record.get("Agreement_name")
                    .replace("_RIMP", "3CP")
                    .replace("_RIMR", "3CR")
                    .replaceAll("\\s+", "").trim();
            algoKeys.add(key);
        }

        Map<String, List<Integer>> starKeyMap = new HashMap<>();
        List<String> starKeys = new ArrayList<>();
        for (int i = 0; i < nonExcludedStar.size(); i++) {
            CSVRecord record = nonExcludedStar.get(i);
            String key = record.get("CRDS Party Code").replaceAll("\\s+", "") +
                         "3" + record.get("Post Direction").replaceAll("\\s+", "").trim();
            starKeys.add(key);
            starKeyMap.computeIfAbsent(key, k -> new ArrayList<>()).add(i);
        }

        Set<Integer> matchedAlgoIndexes = new HashSet<>();
        Set<Integer> matchedStarIndexes = new HashSet<>();

        for (int i = 0; i < algoKeys.size(); i++) {
            String algoKey = algoKeys.get(i);
            List<Integer> matchIndexes = starKeyMap.getOrDefault(algoKey, new ArrayList<>());

            switch (type.toLowerCase()) {
                case "one-to-one":
                    if (matchIndexes.size() == 1 && !matchedStarIndexes.contains(matchIndexes.get(0))) {
                        matched.add(new Record(algoKey, starKeys.get(matchIndexes.get(0)), "Match 1-1"));
                        matchedAlgoIndexes.add(i);
                        matchedStarIndexes.add(matchIndexes.get(0));
                    } else {
                        unmatched.add(new Record(algoKey, "<No Match>", "Mismatch"));
                    }
                    break;

                case "one-to-many":
                    if (!matchIndexes.isEmpty()) {
                        for (Integer idx : matchIndexes) {
                            matched.add(new Record(algoKey, starKeys.get(idx), "Match 1-M"));
                            matchedStarIndexes.add(idx);
                        }
                        matchedAlgoIndexes.add(i);
                    } else {
                        unmatched.add(new Record(algoKey, "<No Match>", "Mismatch"));
                    }
                    break;

                case "many-to-one":
                    if (!matchIndexes.isEmpty()) {
                        matched.add(new Record(algoKey, starKeys.get(matchIndexes.get(0)), "Match M-1"));
                        matchedAlgoIndexes.add(i);
                        matchedStarIndexes.add(matchIndexes.get(0));
                    } else {
                        unmatched.add(new Record(algoKey, "<No Match>", "Mismatch"));
                    }
                    break;

                case "many-to-many":
                    if (!matchIndexes.isEmpty()) {
                        for (Integer idx : matchIndexes) {
                            matched.add(new Record(algoKey, starKeys.get(idx), "Match M-M"));
                            matchedStarIndexes.add(idx);
                        }
                        matchedAlgoIndexes.add(i);
                    } else {
                        unmatched.add(new Record(algoKey, "<No Match>", "Mismatch"));
                    }
                    break;

                default:
                    unmatched.add(new Record(algoKey, "<Invalid Type>", "Error"));
            }
        }

        // Keep unmatched records for next match
        List<CSVRecord> nextAlgo = new ArrayList<>();
        for (int i = 0; i < nonExcludedAlgo.size(); i++) {
            if (!matchedAlgoIndexes.contains(i)) nextAlgo.add(nonExcludedAlgo.get(i));
        }

        List<CSVRecord> nextStar = new ArrayList<>();
        for (int i = 0; i < nonExcludedStar.size(); i++) {
            if (!matchedStarIndexes.contains(i)) nextStar.add(nonExcludedStar.get(i));
        }

        nonExcludedAlgo = nextAlgo;
        nonExcludedStar = nextStar;

        return new ReconciliationResult(matched, unmatched, Collections.emptyList(), excludedCount, nonExcludedStar.size());
    }
}package com.example.reconciliation.controller;

import com.example.reconciliation.model.ReconciliationResult;
import com.example.reconciliation.service.ReconciliationService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/reconcile")
public class ReconciliationController {

    @Autowired
    private ReconciliationService service;

    @PostMapping("/exclude")
    public ReconciliationResult exclude(
        @RequestParam String algoPath,
        @RequestParam String starPath,
        @RequestParam String exclusionDate) {
        return service.excludeAndTransform(algoPath, starPath, exclusionDate);
    }

    @PostMapping("/match/{type}")
    public ReconciliationResult match(@PathVariable String type) {
        return service.match(type);
    }
}
















package com.example.reconciliation.controller;

import com.example.reconciliation.model.ReconciliationResult;
import com.example.reconciliation.service.ReconciliationService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/reconcile")
public class ReconciliationController {

    @Autowired
    private ReconciliationService service;

    @GetMapping("/exclude")
    public ReconciliationResult exclude(@RequestParam String algoPath,
                                        @RequestParam String starPath,
                                        @RequestParam String exclusionDate) {
        return service.excludeAndTransform(algoPath, starPath, exclusionDate);
    }

    @GetMapping("/match/{type}")
    public ReconciliationResult match(@PathVariable String type,
                                      @RequestParam String algoPath,
                                      @RequestParam String starPath) {
        return service.match(type, algoPath, starPath);
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

    private List<CSVRecord> nonExcludedAlgo = new ArrayList<>();
    private List<CSVRecord> nonExcludedStar = new ArrayList<>();
    private int excludedCount = 0;

    public ReconciliationResult excludeAndTransform(String algoPath, String starPath, String exclusionDate) {
        try {
            CSVParser algoParser = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(new FileReader(algoPath));
            CSVParser starParser = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(new FileReader(starPath));

            List<CSVRecord> algoRecords = algoParser.getRecords();
            List<CSVRecord> starRecords = starParser.getRecords();

            nonExcludedAlgo.clear();
            nonExcludedStar.clear();
            excludedCount = 0;

            for (CSVRecord record : starRecords) {
                String maturityDate = record.get("Maturity Date").trim().toLowerCase();
                if (maturityDate.contains(exclusionDate.toLowerCase())) {
                    excludedCount++;
                } else {
                    nonExcludedStar.add(record);
                }
            }

            nonExcludedAlgo.addAll(algoRecords);

            List<Record> nonExcludedRecords = new ArrayList<>();
            for (CSVRecord record : nonExcludedStar) {
                String key = record.get("CRDS Party Code").replaceAll("\\s+", "") +
                        "3" + record.get("Post Direction").replaceAll("\\s+", "");
                nonExcludedRecords.add(new Record("<Non-Excluded>", key, "Included for Matching"));
            }

            return new ReconciliationResult(
                    Collections.emptyList(),
                    nonExcludedRecords,
                    Collections.singletonList(new Record("Excluded Count", String.valueOf(excludedCount),
                            "Non-Excluded Count: " + nonExcludedRecords.size()))
            );

        } catch (Exception e) {
            e.printStackTrace();
            return new ReconciliationResult(Collections.emptyList(), Collections.emptyList(), Collections.emptyList());
        }
    }

    public ReconciliationResult match(String type, String algoPath, String starPath) {
        List<Record> matched = new ArrayList<>();
        List<Record> unmatched = new ArrayList<>();

        try {
            List<String> algoKeys = new ArrayList<>();
            for (CSVRecord record : nonExcludedAlgo) {
                String key = record.get("Agreement_name")
                        .replace("_RIMP", "3CP")
                        .replace("_RIMR", "3CR")
                        .replaceAll("\\s+", "").trim();
                algoKeys.add(key);
            }

            Map<String, List<Integer>> starKeyMap = new HashMap<>();
            List<String> starKeys = new ArrayList<>();
            for (int i = 0; i < nonExcludedStar.size(); i++) {
                CSVRecord record = nonExcludedStar.get(i);
                String key = record.get("CRDS Party Code").replaceAll("\\s+", "") +
                        "3" + record.get("Post Direction").replaceAll("\\s+", "").trim();
                starKeys.add(key);
                starKeyMap.computeIfAbsent(key, k -> new ArrayList<>()).add(i);
            }

            Set<Integer> matchedAlgoIndexes = new HashSet<>();
            Set<Integer> matchedStarIndexes = new HashSet<>();

            for (int i = 0; i < algoKeys.size(); i++) {
                String algoKey = algoKeys.get(i);
                List<Integer> matchIndexes = starKeyMap.getOrDefault(algoKey, new ArrayList<>());

                switch (type.toLowerCase()) {
                    case "one-to-one":
                        if (matchIndexes.size() == 1 && !matchedStarIndexes.contains(matchIndexes.get(0))) {
                            matched.add(new Record(algoKey, starKeys.get(matchIndexes.get(0)), "Match 1-1"));
                            matchedAlgoIndexes.add(i);
                            matchedStarIndexes.add(matchIndexes.get(0));
                        } else {
                            unmatched.add(new Record(algoKey, "<No Match>", "Mismatch"));
                        }
                        break;

                    case "one-to-many":
                        if (!matchIndexes.isEmpty()) {
                            for (Integer idx : matchIndexes) {
                                matched.add(new Record(algoKey, starKeys.get(idx), "Match 1-M"));
                                matchedStarIndexes.add(idx);
                            }
                            matchedAlgoIndexes.add(i);
                        } else {
                            unmatched.add(new Record(algoKey, "<No Match>", "Mismatch"));
                        }
                        break;

                    case "many-to-one":
                        if (!matchIndexes.isEmpty()) {
                            matched.add(new Record(algoKey, starKeys.get(matchIndexes.get(0)), "Match M-1"));
                            matchedAlgoIndexes.add(i);
                            matchedStarIndexes.add(matchIndexes.get(0));
                        } else {
                            unmatched.add(new Record(algoKey, "<No Match>", "Mismatch"));
                        }
                        break;

                    case "many-to-many":
                        if (!matchIndexes.isEmpty()) {
                            for (Integer idx : matchIndexes) {
                                matched.add(new Record(algoKey, starKeys.get(idx), "Match M-M"));
                                matchedStarIndexes.add(idx);
                            }
                            matchedAlgoIndexes.add(i);
                        } else {
                            unmatched.add(new Record(algoKey, "<No Match>", "Mismatch"));
                        }
                        break;

                    default:
                        unmatched.add(new Record(algoKey, "<Invalid Type>", "Error"));
                }
            }

            // Update for next round of matching
            List<CSVRecord> nextAlgo = new ArrayList<>();
            for (int i = 0; i < nonExcludedAlgo.size(); i++) {
                if (!matchedAlgoIndexes.contains(i)) nextAlgo.add(nonExcludedAlgo.get(i));
            }

            List<CSVRecord> nextStar = new ArrayList<>();
            for (int i = 0; i < nonExcludedStar.size(); i++) {
                if (!matchedStarIndexes.contains(i)) nextStar.add(nonExcludedStar.get(i));
            }

            nonExcludedAlgo = nextAlgo;
            nonExcludedStar = nextStar;

            return new ReconciliationResult(matched, unmatched,
                    Collections.singletonList(new Record("Remaining Unmatched Count", String.valueOf(unmatched.size()),
                            "Matched Count: " + matched.size())));

        } catch (Exception e) {
            e.printStackTrace();
            return new ReconciliationResult(Collections.emptyList(), Collections.emptyList(), Collections.emptyList());
        }
    }
}package com.example.reconciliation.model;

import java.util.List;

public class ReconciliationResult {
    private List<Record> matchedRecords;
    private List<Record> unmatchedRecords;
    private List<Record> summary;

    public ReconciliationResult() {}

    public ReconciliationResult(List<Record> matchedRecords, List<Record> unmatchedRecords, List<Record> summary) {
        this.matchedRecords = matchedRecords;
        this.unmatchedRecords = unmatchedRecords;
        this.summary = summary;
    }

    public List<Record> getMatchedRecords() {
        return matchedRecords;
    }

    public void setMatchedRecords(List<Record> matchedRecords) {
        this.matchedRecords = matchedRecords;
    }

    public List<Record> getUnmatchedRecords() {
        return unmatchedRecords;
    }

    public void setUnmatchedRecords(List<Record> unmatchedRecords) {
        this.unmatchedRecords = unmatchedRecords;
    }

    public List<Record> getSummary() {
        return summary;
    }

    public void setSummary(List<Record> summary) {
        this.summary = summary;
    }
}




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


package com.example.demo.service;

import com.example.demo.model.Record;
import com.example.demo.model.ReconciliationResult;
import org.springframework.stereotype.Service;

import java.io.BufferedReader;
import java.io.FileReader;
import java.time.LocalDate;
import java.time.format.DateTimeFormatter;
import java.util.*;
import java.util.stream.Collectors;

@Service
public class ReconciliationService {

    private List<Record> nonExcludedAlgo = new ArrayList<>();
    private List<Record> nonExcludedStar = new ArrayList<>();
    private List<Record> excludedStar = new ArrayList<>();

    public ReconciliationResult excludeAndTransform(String algoPath, String starPath) {
        nonExcludedAlgo.clear();
        nonExcludedStar.clear();
        excludedStar.clear();

        try (BufferedReader algoReader = new BufferedReader(new FileReader(algoPath));
             BufferedReader starReader = new BufferedReader(new FileReader(starPath))) {

            // Skip header
            algoReader.readLine();
            starReader.readLine();

            // Process ALGO
            String line;
            while ((line = algoReader.readLine()) != null) {
                String[] parts = line.split(",", -1);
                String key = parts[0].replace("_RIMP", "_3CP").replace("_RIMR", "_3CR");
                nonExcludedAlgo.add(new Record(key, "", ""));
            }

            // Process STAR
            DateTimeFormatter formatter = DateTimeFormatter.ofPattern("M/d/yyyy");

            while ((line = starReader.readLine()) != null) {
                String[] parts = line.split(",", -1);
                String crdsPartyCode = parts[0];
                String postDirection = parts[1];
                String maturityDateStr = parts[2];

                String key = crdsPartyCode + "3" + postDirection;

                LocalDate maturityDate = LocalDate.parse(maturityDateStr, formatter);
                if (maturityDate.equals(LocalDate.of(1900, 1, 1))) {
                    excludedStar.add(new Record("", key, "EXCLUDED"));
                } else {
                    nonExcludedStar.add(new Record("", key, ""));
                }
            }

        } catch (Exception e) {
            e.printStackTrace();
        }

        // Return non-excluded records and counts
        List<Record> summary = new ArrayList<>();
        summary.add(new Record("Non-Excluded ALGO", String.valueOf(nonExcludedAlgo.size()), ""));
        summary.add(new Record("Non-Excluded STAR", String.valueOf(nonExcludedStar.size()), ""));
        summary.add(new Record("Excluded STAR", String.valueOf(excludedStar.size()), ""));

        return new ReconciliationResult(new ArrayList<>(), new ArrayList<>(), summary);
    }

    public ReconciliationResult match(String type) {
        List<Record> matched = new ArrayList<>();
        List<Record> unmatched = new ArrayList<>();

        Set<String> starKeys = nonExcludedStar.stream()
                .map(Record::getStarKey)
                .collect(Collectors.toSet());

        for (Record algoRecord : nonExcludedAlgo) {
            String algoKey = algoRecord.getAlgoKey();
            if (starKeys.contains(algoKey)) {
                matched.add(new Record(algoKey, algoKey, "MATCHED"));
            } else {
                unmatched.add(new Record(algoKey, "", "UNMATCHED"));
            }
        }

        return new ReconciliationResult(matched, unmatched, new ArrayList<>());
    }
}
