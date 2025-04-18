package com.example.demo.controller;

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
        File algoFile = new File("C:\\path\\to\\algo.csv"); // Update path
        File starFile = new File("C:\\path\\to\\star.csv"); // Update path

        List<String[]> algoRows = readCSV(algoFile);
        List<String[]> starRows = readCSV(starFile);

        List<Map<String, Object>> result = new ArrayList<>();

        int maxLen = Math.max(algoRows.size(), starRows.size());
        for (int i = 0; i < maxLen; i++) {
            Map<String, Object> rowResult = new HashMap<>();
            String[] algoRow = i < algoRows.size() ? algoRows.get(i) : new String[0];
            String[] starRow = i < starRows.size() ? starRows.get(i) : new String[0];

            rowResult.put("Row", i + 1);

            List<Map<String, String>> columns = new ArrayList<>();
            int maxCols = Math.max(algoRow.length, starRow.length);

            for (int j = 0; j < maxCols; j++) {
                String algoVal = j < algoRow.length ? algoRow[j] : "Missing";
                String starVal = j < starRow.length ? starRow[j] : "Missing";
                String status = algoVal.equals(starVal) ? "Match" : "Mismatch";

                Map<String, String> colResult = new HashMap<>();
                colResult.put("Column", "Column " + (j + 1));
                colResult.put("Algo Value", algoVal);
                colResult.put("Star Value", starVal);
                colResult.put("Status", status);

                columns.add(colResult);
            }

            rowResult.put("Columns", columns);
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
                rows.add(line.split(",", -1)); // include empty strings
            }
        }

        return rows;
    }
}
