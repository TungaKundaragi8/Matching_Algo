import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.io.*;
import java.util.*;

@RestController
@RequestMapping("/reconciliation")
public class reconciliation {

    @GetMapping("/run")
    public ResponseEntity<List<Map<String, String>>> runReconciliation() throws IOException {
        File algoFile = new File("C:\\Practise\\Initial_Margin_EOD_20250307.csv");
        File starFile = new File("C:\\Practise\\STARALGONEW_3428_20250307_1.csv");

        List<String[]> algoRows = readCSV(algoFile);
        List<String[]> starRows = readCSV(starFile);

        int agreementNameIndex = getColumnIndex(algoRows.get(0), "Agreement_name");
        int crdsIndex = getColumnIndex(starRows.get(0), "CRDS Party Code");
        int postDirIndex = getColumnIndex(starRows.get(0), "Post Direction");

        List<String> algoKeys = new ArrayList<>();
        List<String> starKeys = new ArrayList<>();
        Map<String, List<Integer>> starKeyMap = new HashMap<>();

        // Build keys for ALGO
        for (int i = 1; i < algoRows.size(); i++) {
            String raw = algoRows.get(i)[agreementNameIndex].trim();
            String key = raw.replace("_RIMR", "3CR")
                            .replace("_RIMP", "3CP")
                            .replace(" ", "")
                            .strip();
            algoKeys.add(key);
        }

        // Build keys for STAR
        for (int i = 1; i < starRows.size(); i++) {
            String crds = starRows.get(i)[crdsIndex].trim().replace(" ", "");
            String post = starRows.get(i)[postDirIndex].trim().replace(" ", "");
            String key = crds + "3" + post;
            starKeys.add(key);
            starKeyMap.computeIfAbsent(key, k -> new ArrayList<>()).add(i);
        }

        List<Map<String, String>> result = new ArrayList<>();
        Set<Integer> matchedStarIndices = new HashSet<>();

        // Match logic
        for (int i = 0; i < algoKeys.size(); i++) {
            String algoKey = algoKeys.get(i);
            List<Integer> matches = starKeyMap.getOrDefault(algoKey, new ArrayList<>());

            if (matches.size() == 1 && !matchedStarIndices.contains(matches.get(0))) {
                result.add(Map.of(
                        "ALGO Key", algoKey,
                        "STAR Key", starKeys.get(matches.get(0)),
                        "Status", "Match"
                ));
                matchedStarIndices.add(matches.get(0));
            } else if (matches.size() > 1) {
                for (int idx : matches) {
                    result.add(Map.of(
                            "ALGO Key", algoKey,
                            "STAR Key", starKeys.get(idx),
                            "Status", "Mismatch - Multiple STAR Keys"
                    ));
                }
            } else {
                result.add(Map.of(
                        "ALGO Key", algoKey,
                        "STAR Key", "<No Match>",
                        "Status", "Mismatch - No Match"
                ));
            }
        }

        // Check for unmatched STAR entries
        for (int i = 0; i < starKeys.size(); i++) {
            if (!matchedStarIndices.contains(i)) {
                String starKey = starKeys.get(i);
                boolean foundInAlgo = algoKeys.contains(starKey);
                if (!foundInAlgo || starKeyMap.get(starKey).size() > 1) {
                    result.add(Map.of(
                            "ALGO Key", "<No Match>",
                            "STAR Key", starKey,
                            "Status", "Mismatch - No Match or Duplicate"
                    ));
                }
            }
        }

        return ResponseEntity.ok(result);
    }

    private List<String[]> readCSV(File file) throws IOException {
        List<String[]> rows = new ArrayList<>();
        try (BufferedReader reader = new BufferedReader(new FileReader(file))) {
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
}

