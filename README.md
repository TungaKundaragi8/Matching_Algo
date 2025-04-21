Here's an updated version of the code that matches the `agreement_name` column in the first file with the `CODE` column in the second file and applies the specified transformations:

```
package com.example.demo.controller;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.io.*;
import java.nio.file.Files;
import java.util.*;

@RestController
@RequestMapping("/reconciliation")
public class ReconciliationController {

    @GetMapping("/run")
    public ResponseEntity<List<Map<String, Object>>> runReconciliation() throws IOException {
        File algoFile = new File("C:\\path\\to\\algo.csv");
        File starFile = new File("C:\\path\\to\\star.csv");

        List<String[]> algoRows = readCSV(algoFile);
        List<String[]> starRows = readCSV(starFile);

        // Get the index of the agreement_name column in algo file
        int agreementNameIndex = getColumnIndex(algoRows.get(0), "agreement_name");

        // Get the index of the CODE column in star file
        int codeIndex = getColumnIndex(starRows.get(0), "CODE");

        List<Map<String, Object>> result = new ArrayList<>();

        // Iterate over algo rows
        for (int i = 1; i < algoRows.size(); i++) {
            String[] algoRow = algoRows.get(i);
            String agreementName = algoRow[agreementNameIndex];

            // Apply transformations
            String transformedAgreementName = applyTransformations(agreementName);

            // Find matching row in star file
            String[] matchingStarRow = findMatchingRow(starRows, codeIndex, transformedAgreementName);

            Map<String, Object> rowResult = new HashMap<>();
            rowResult.put("Agreement Name", agreementName);
            rowResult.put("Transformed Agreement Name", transformedAgreementName);

            if (matchingStarRow != null) {
                rowResult.put("Match Status", "Match");
                rowResult.put("Star Row", Arrays.toString(matchingStarRow));
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
                rows.add(line.split(",", -1));
            }
        }
        return rows;
    }

    private int getColumnIndex(String[] headers, String columnName) {
        for (int i = 0; i < headers.length; i++) {
            if (headers[i].trim().equals(columnName)) {
                return i;
            }
        }
        return -1;
    }

    private String applyTransformations(String agreementName) {
        if (agreementName.contains("_RIMR")) {
            return agreementName.replace("_RIMR", "_3CR");
        } else if (agreementName.contains("_RIMP")) {
            return agreementName.replace("_RIMP", "_3CP");
        }
        return agreementName;
    }

    private String[] findMatchingRow(List<String[]> starRows, int codeIndex, String transformedAgreementName) {
        for (int i = 1; i < starRows.size(); i++) {
            String[] starRow = starRows.get(i);
            if (starRow[codeIndex].equals(transformedAgreementName)) {
                return starRow;
            }
        }
        return null;
    }
}
```

This updated code:

1. Reads the CSV files and gets the indices of the `agreement_name` and `CODE` columns.
2. Iterates over the rows in the first file, applies the transformations to the `agreement_name` column, and finds the matching row in the second file based on the transformed value.
3. Creates a result map for each row, indicating whether a match was found and providing the matching row data if applicable.
