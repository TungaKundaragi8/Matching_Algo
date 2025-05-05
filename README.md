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
        List<String> algoKeys = new ArrayList<>();
        List<String> starKeys = new ArrayList<>();
        Map<String, List<Integer>> starKeyMap = new HashMap<>();

        try {
            // Read ALGO file
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

            // Read STAR file
            FileReader readerStar = new FileReader(starPath);
            Iterable<CSVRecord> starRecords = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(readerStar);

            int index = 0;
            for (CSVRecord record : starRecords) {
                String key = record.get("CRDS Party Code").replaceAll("\\s+", "").trim()
                        + "3"
                        + record.get("Post Direction").replaceAll("\\s+", "").trim();
                starKeys.add(key);
                starKeyMap.computeIfAbsent(key, k -> new ArrayList<>()).add(index);
                index++;
            }

        } catch (Exception e) {
            e.printStackTrace();
        }

        Set<Integer> matchedStarIndices = new HashSet<>();
        Set<Integer> matchedAlgoIndices = new HashSet<>();
        List<Record> matched = new ArrayList<>();
        List<Record> unmatched = new ArrayList<>();

        // Run the selected matching type first
        performMatching(algoKeys, starKeys, starKeyMap, matchType, matched, unmatched, matchedAlgoIndices, matchedStarIndices);

        // Cascade to other matching types
        List<String> types = List.of("1-1", "1-many", "many-1", "many-many");
        for (String type : types) {
            if (!type.equals(matchType)) {
                performMatching(algoKeys, starKeys, starKeyMap, type, matched, unmatched, matchedAlgoIndices, matchedStarIndices);
            }
        }

        // Add unmatched STAR records
        for (int i = 0; i < starKeys.size(); i++) {
            if (!matchedStarIndices.contains(i)) {
                unmatched.add(new Record("<No Match>", starKeys.get(i), "Mismatch - No Match or Duplicate"));
            }
        }

        return new ReconciliationResult(matched, unmatched);
    }

    private void performMatching(List<String> algoKeys,
                                 List<String> starKeys,
                                 Map<String, List<Integer>> starKeyMap,
                                 String matchType,
                                 List<Record> matched,
                                 List<Record> unmatched,
                                 Set<Integer> matchedAlgoIndices,
                                 Set<Integer> matchedStarIndices) {

        for (int i = 0; i < algoKeys.size(); i++) {
            if (matchedAlgoIndices.contains(i)) continue;

            String algoKey = algoKeys.get(i);
            List<Integer> matchingStarIdxs = starKeyMap.getOrDefault(algoKey, new ArrayList<>());
            List<Integer> availableStarIdxs = new ArrayList<>();

            for (Integer idx : matchingStarIdxs) {
                if (!matchedStarIndices.contains(idx)) {
                    availableStarIdxs.add(idx);
                }
            }

            switch (matchType) {
                case "1-1":
                    if (availableStarIdxs.size() == 1) {
                        int idx = availableStarIdxs.get(0);
                        matched.add(new Record(algoKey, starKeys.get(idx), "1-1 Match"));
                        matchedStarIndices.add(idx);
                        matchedAlgoIndices.add(i);
                    }
                    break;

                case "1-many":
                    if (availableStarIdxs.size() > 1) {
                        for (int idx : availableStarIdxs) {
                            matched.add(new Record(algoKey, starKeys.get(idx), "1-Many Match"));
                            matchedStarIndices.add(idx);
                        }
                        matchedAlgoIndices.add(i);
                    }
                    break;

                case "many-1":
                    // Check how many algos point to the same STAR key
                    long count = algoKeys.stream()
                            .filter(k -> k.equals(algoKey) && !matchedAlgoIndices.contains(algoKeys.indexOf(k)))
                            .count();

                    if (availableStarIdxs.size() == 1 && count > 1) {
                        int idx = availableStarIdxs.get(0);
                        matched.add(new Record(algoKey, starKeys.get(idx), "Many-1 Match"));
                        matchedStarIndices.add(idx);
                        matchedAlgoIndices.add(i);
                    }
                    break;

                case "many-many":
                    if (availableStarIdxs.size() > 1) {
                        for (int idx : availableStarIdxs) {
                            matched.add(new Record(algoKey, starKeys.get(idx), "Many-Many Match"));
                            matchedStarIndices.add(idx);
                        }
                        matchedAlgoIndices.add(i);
                    }
                    break;
            }
        }
    }
}
