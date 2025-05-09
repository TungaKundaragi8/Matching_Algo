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
