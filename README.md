@GetMapping("/run")
public ResponseEntity<Map<String, Object>> runReconciliation() throws IOException {
    File algoFile = new File("C:\\Practise\\Initial_Margin_EOD_20250307.csv");
    File starFile = new File("C:\\Practise\\STARALGONEW_3428_20250307_1.csv");

    List<String[]> algoRows = readCSV(algoFile);
    List<String[]> starRows = readCSV(starFile);

    int agreementNameIndex = getColumnIndex(algoRows.get(0), "Agreement_name");
    int crdsIndex = getColumnIndex(starRows.get(0), "CRDS Party Code");
    int postDirIndex = getColumnIndex(starRows.get(0), "Post Direction");

    Map<String, Integer> algoKeyToIndex = new LinkedHashMap<>();
    Map<String, List<Integer>> starKeyToIndices = new HashMap<>();

    for (int i = 1; i < algoRows.size(); i++) {
        String agreement = algoRows.get(i)[agreementNameIndex].trim()
                .replace("_RIMR", "3CR")
                .replace("_RIMP", "3CP")
                .replaceAll("\\s", "");

        algoKeyToIndex.put(agreement, i);
    }

    for (int i = 1; i < starRows.size(); i++) {
        String crds = starRows.get(i)[crdsIndex].trim().replaceAll("\\s", "");
        String post = starRows.get(i)[postDirIndex].trim().replaceAll("\\s", "");
        String key = crds + "3" + post;

        starKeyToIndices.computeIfAbsent(key, k -> new ArrayList<>()).add(i);
    }

    List<Map<String, String>> result = new ArrayList<>();
    Set<Integer> matchedStarIndices = new HashSet<>();
    int matchedCount = 0;
    int unmatchedCount = 0;

    for (Map.Entry<String, Integer> entry : algoKeyToIndex.entrySet()) {
        String algoKey = entry.getKey();
        int algoIndex = entry.getValue();
        List<Integer> starIndices = starKeyToIndices.get(algoKey);

        Map<String, String> map = new LinkedHashMap<>();
        map.put("ALGO Key", algoKey);

        if (starIndices == null || starIndices.isEmpty()) {
            map.put("STAR Key", "<No Match>");
            map.put("Status", "Mismatch - No STAR entry");
            unmatchedCount++;
        } else if (starIndices.size() == 1 && !matchedStarIndices.contains(starIndices.get(0))) {
            int matchedIndex = starIndices.get(0);
            map.put("STAR Key", algoKey);
            map.put("Status", "Match");
            matchedStarIndices.add(matchedIndex);
            matchedCount++;
        } else {
            map.put("STAR Key", algoKey);
            map.put("Status", "Mismatch - Multiple STAR entries or already matched");
            unmatchedCount++;
        }

        result.add(map);
    }

    for (Map.Entry<String, List<Integer>> entry : starKeyToIndices.entrySet()) {
        String starKey = entry.getKey();
        for (int idx : entry.getValue()) {
            if (!matchedStarIndices.contains(idx) && !algoKeyToIndex.containsKey(starKey)) {
                Map<String, String> map = new LinkedHashMap<>();
                map.put("ALGO Key", "<No Match>");
                map.put("STAR Key", starKey);
                map.put("Status", "Mismatch - STAR entry unmatched");
                result.add(map);
                unmatchedCount++;
            }
        }
    }

    Map<String, Object> finalResponse = new LinkedHashMap<>();
    finalResponse.put("Matched Records", matchedCount);
    finalResponse.put("Unmatched Records", unmatchedCount);
    finalResponse.put("Details", result);

    return ResponseEntity.ok(finalResponse);
}
