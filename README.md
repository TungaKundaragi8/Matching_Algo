package com.example.reconciliation.controller;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;
import org.springframework.http.HttpStatus;

import java.io.*;
import java.util.*;

@RestController
@RequestMapping("/reconciliation")
public class ReconciliationController {

    @PostMapping("/upload")
    public ResponseEntity<List<Map<String, Object>>> uploadAndReconcile(
            @RequestParam("algoFile") MultipartFile algoFile,
            @RequestParam("starFile") MultipartFile starFile) {

        try {
            List<String[]> algoRows = readCSV(algoFile);
            List<String[]> starRows = readCSV(starFile);

            if (algoRows.isEmpty() || starRows.isEmpty()) {
                return ResponseEntity.status(HttpStatus.BAD_REQUEST)
                        .body(List.of(Map.of("error", "One or both files are empty")));
            }

            int agreementIndex = getColumnIndex(algoRows.get(0), "Agreement_name");
            int crdsIndex = getColumnIndex(starRows.get(0), "CRDS Party Code");

            if (agreementIndex == -1 || crdsIndex == -1) {
                return ResponseEntity.status(HttpStatus.BAD_REQUEST)
                        .body(List.of(Map.of("error", "Required columns not found")));
            }

            List<Map<String, Object>> results = new ArrayList<>();
            Set<Integer> matchedStarIndices = new HashSet<>();

            for (int i = 1; i < algoRows.size(); i++) {
                String[] algoRow = algoRows.get(i);
                if (agreementIndex >= algoRow.length) continue;

                String agreementName = algoRow[agreementIndex].trim();
                String expectedSuffix = getExpectedCrdsSuffix(agreementName);
                String[] matchedStar = null;
                String matchedCrds = null;

                for (int j = 1; j < starRows.size(); j++) {
                    if (matchedStarIndices.contains(j)) continue;

                    String[] starRow = starRows.get(j);
                    if (crdsIndex >= starRow.length) continue;

                    String crdsPartyCode = starRow[crdsIndex].trim();

                    if (crdsPartyCode.endsWith(expectedSuffix)) {
                        matchedStar = starRow;
                        matchedCrds = crdsPartyCode;
                        matchedStarIndices.add(j);
                        break;
                    }
                }

                Map<String, Object> map = new LinkedHashMap<>();
                map.put("Agreement Name", agreementName);
                map.put("Expected CRDS Suffix", expectedSuffix);
                map.put("Matched CRDS", matchedCrds != null ? matchedCrds : "No Match");
                map.put("Match Status", matchedStar != null ? "Match" : "No Match");
                if (matchedStar != null) {
                    map.put("Matched STAR Row", Arrays.toString(matchedStar));
                }

                results.add(map);
            }

            return ResponseEntity.ok(results);

        } catch (Exception e) {
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                    .body(List.of(Map.of("error", "Processing failed: " + e.getMessage())));
        }
    }

    private List<String[]> readCSV(MultipartFile file) throws IOException {
        List<String[]> rows = new ArrayList<>();
        try (BufferedReader reader = new BufferedReader(new InputStreamReader(file.getInputStream()))) {
            String line;
            while ((line = reader.readLine()) != null) {
                rows.add(line.split(",", -1));
            }
        }
        return rows;
    }

    private int getColumnIndex(String[] headers, String columnName) {
        for (int i = 0; i < headers.length; i++) {
            if (headers[i].trim().equalsIgnoreCase(columnName)) return i;
        }
        return -1;
    }

    private String getExpectedCrdsSuffix(String agreementName) {
        if (agreementName.endsWith("_RIMP")) return "_3CP";
        else if (agreementName.endsWith("_RIMR")) return "_3CR";
        else return "_UNKNOWN";
    }
}
