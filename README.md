package com.example.demo.controller;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.io.*;
import java.nio.file.Files;
import java.util.*;

@RestController
@RequestMapping("/reconciliation")
public class ReconciliationController {

    @GetMapping("/run")
    public ResponseEntity<List<Map<String, Object>>> runReconciliation() throws IOException {
        File algoFile = new File("C:\\Practise\\Initial_Margin_EOD_20250307.csv");
        File starFile = new File("C:\\Practise\\STARGALGONEW_3428_20250307_1.csv");

        List<String[]> algoRows = readCSV(algoFile);
        List<String[]> starRows = readCSV(starFile);

        if (algoRows.isEmpty() || starRows.isEmpty()) {
            return ResponseEntity.status(HttpStatus.BAD_REQUEST)
                    .body(List.of(Map.of("error", "One or both CSV files are empty or missing")));
        }

        int agreementNameIndex = getColumnIndex(algoRows.get(0), "Agreement_name");
        int crdsCodeIndex = getColumnIndex(starRows.get(0), "CRDS Party Code");

        if (agreementNameIndex == -1 || crdsCodeIndex == -1) {
            return ResponseEntity.status(HttpStatus.BAD_REQUEST)
                    .body(List.of(Map.of("error", "Required column not found in one or both files")));
        }

        List<Map<String, Object>> result = new ArrayList<>();

        for (int i = 1; i < algoRows.size(); i++) {
            String[] algoRow = algoRows.get(i);
            if (agreementNameIndex >= algoRow.length) continue;

            String agreementName = algoRow[agreementNameIndex];
            String transformedCrdsName = null;
            String[] matchingStarRow = null;

            // Search in STAR file
            for (String[] starRow : starRows) {
                if (crdsCodeIndex >= starRow.length) continue;
                String crdsName = starRow[crdsCodeIndex];
                transformedCrdsName = applyTransformations(agreementName, crdsName);

                if (transformedCrdsName.equals(agreementName)) {
                    matchingStarRow = starRow;
                    break;
                }
            }

            Map<String, Object> rowResult = new LinkedHashMap<>();
            rowResult.put("Agreement Name", agreementName);
            rowResult.put("Transformed CRDS Name", transformedCrdsName);
            if (matchingStarRow != null) {
                rowResult.put("Match Status", "Match");
                rowResult.put("Matching Row", Arrays.toString(matchingStarRow));
            } else {
                rowResult.put("Match Status", "No Match");
            }

            result.add(rowResult);
        }

        return ResponseEntity.ok(result);
    }

    private List<String[]> readCSV(File file) throws IOException {
        List<String[]> rows = new ArrayList<>();
        if (!file.exists()) return rows;

        try (BufferedReader br = Files.newBufferedReader(file.toPath())) {
            String line;
            while ((line = br.readLine()) != null) {
                rows.add(line.split(",", -1)); // Preserve empty values
            }
        }
        return rows;
    }

    private int getColumnIndex(String[] headers, String columnName) {
        for (int i = 0; i < headers.length; i++) {
            if (headers[i].trim().equalsIgnoreCase(columnName)) {
                return i;
            }
        }
        return -1;
    }

    private String applyTransformations(String agreementName, String crdsName) {
        if (agreementName.contains("-RIMP") && crdsName.equals("3CR")) {
            return "3CRP";
        } else if (agreementName.contains("-RIMP") && crdsName.equals("3CP")) {
            return "3CPP";
        }
        return crdsName;
    }
}
