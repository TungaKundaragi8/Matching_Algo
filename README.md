package com.example.reconciliation.service;

import com.example.reconciliation.model.Record; import com.example.reconciliation.model.ReconciliationResult; import org.apache.commons.csv.CSVFormat; import org.apache.commons.csv.CSVRecord; import org.springframework.stereotype.Service;

import java.io.FileReader; import java.time.LocalDate; import java.time.format.DateTimeFormatter; import java.util.*;

@Service public class ReconciliationService {

public ReconciliationResult reconcile(String algoPath, String starPath, String matchType) {
    List<Record> matched = new ArrayList<>();
    List<Record> unmatched = new ArrayList<>();
    List<Record> excluded = new ArrayList<>();

    try {
        FileReader readerAlgo = new FileReader(algoPath);
        Iterable<CSVRecord> algoIterable = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(readerAlgo);
        List<CSVRecord> algoRecords = new ArrayList<>();
        for (CSVRecord record : algoIterable) {
            algoRecords.add(record);
        }

        FileReader readerStar = new FileReader(starPath);
        Iterable<CSVRecord> starIterable = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(readerStar);
        List<CSVRecord> starRecords = new ArrayList<>();
        for (CSVRecord record : starIterable) {
            // Exclusion rule: Maturity Date = 01-Jan-1900
            String maturityDateStr = record.get("Maturity Date").trim();
            LocalDate maturityDate = LocalDate.parse(maturityDateStr, DateTimeFormatter.ofPattern("dd-MMM-yyyy", Locale.ENGLISH));
            if (maturityDate.equals(LocalDate.of(1900, 1, 1))) {
                excluded.add(new Record("", record.get("CRDS Party Code"), "Excluded - Invalid Maturity Date"));
            } else {
                starRecords.add(record);
            }
        }

        if (matchType.equalsIgnoreCase("one-to-one")) {
            performOneToOneMatch(algoRecords, starRecords, matched, unmatched);
        }
        // Future: add one-to-many, many-to-one, many-to-many

    } catch (Exception e) {
        e.printStackTrace();
    }

    return new ReconciliationResult(matched, unmatched, excluded);
}

private void performOneToOneMatch(List<CSVRecord> algoRecords, List<CSVRecord> starRecords, List<Record> matched, List<Record> unmatched) {
    Map<String, Integer> starKeyIndexMap = new HashMap<>();
    List<String> starKeys = new ArrayList<>();

    for (int i = 0; i < starRecords.size(); i++) {
        CSVRecord record = starRecords.get(i);
        String key = record.get("CRDS Party Code").replaceAll("\\s+", "") +
                "3" +
                record.get("Post Direction").replaceAll("\\s+", "");
        starKeys.add(key);
        starKeyIndexMap.put(key, i);
    }

    Set<Integer> matchedStarIndices = new HashSet<>();

    for (int i = 0; i < algoRecords.size(); i++) {
        CSVRecord algoRecord = algoRecords.get(i);
        String algoKey = algoRecord.get("Agreement_name")
                .replace("_RIMR", "3CR")
                .replace("_RIMP", "3CP")
                .replaceAll("\\s+", "")
                .trim();

        Integer matchedIdx = starKeyIndexMap.get(algoKey);
        if (matchedIdx != null && !matchedStarIndices.contains(matchedIdx)) {
            matched.add(new Record(algoKey, starKeys.get(matchedIdx), "Match"));
            matchedStarIndices.add(matchedIdx);
        } else {
            unmatched.add(new Record(algoKey, "<No Match>", "Mismatch - No Match or Duplicate"));
        }
    }
}

}
