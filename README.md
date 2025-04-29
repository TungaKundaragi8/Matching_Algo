



@GetMapping("/run")
public ResponseEntity<Map<String, Object>> runReconciliation() throws IOException {
    File algoFile = new File("C:\\Practise\\Initial_Margin_EOD_20250307.csv");
    File starFile = new File("C:\\Practise\\STARALGONEW_3428_20250307_1.csv");

    List<String[]> algoRows = readCSV(algoFile);
    List<String[]> starRows = readCSV(starFile);

    int agreementIndex = getColumnIndex(algoRows.get(0), "Agreement_name");
    int crdsIndex = getColumnIndex(starRows.get(0), "CRDS Party Code");
    int postDirIndex = getColumnIndex(starRows.get(0), "Post Direction");

    Map<String, Integer> algoKeys = new LinkedHashMap<>();
    Map<String, List<Integer>> starKeys = new HashMap<>();

    for (int i = 1; i < algoRows.size(); i++) {
        String agreement = algoRows.get(i)[agreementIndex].trim();
        String suffix = agreement.endsWith("_RIMR") ? "3CR" :
                        agreement.endsWith("_RIMP") ? "3CP" : "UNKNOWN";
        String algoKey = agreement.replace("_RIMR", "").replace("_RIMP", "").replaceAll("\\s", "") + suffix;
        algoKeys.put(algoKey, i);
    }

    for (int i = 1; i < starRows.size(); i++) {
        String crds = starRows.get(i)[crdsIndex].trim().replaceAll("\\s", "");
        String post = starRows.get(i)[postDirIndex].trim().replaceAll("\\s", "");
        String starKey = crds + post;
        starKeys.computeIfAbsent(starKey, k -> new ArrayList<>()).add(i);
    }

    List<Map<String, String>> results = new ArrayList<>();
    Set<Integer> matchedStarIndices = new HashSet<>();
    int matched = 0, unmatched = 0;

    for (Map.Entry<String, Integer> entry : algoKeys.entrySet()) {
        String key = entry.getKey();
        int algoIdx = entry.getValue();
        List<Integer> starIdxList = starKeys.getOrDefault(key, new ArrayList<>());

        Map<String, String> map = new LinkedHashMap<>();
        map.put("ALGO Key", key);

        if (starIdxList.size() == 1 && !matchedStarIndices.contains(starIdxList.get(0))) {
            int starIdx = starIdxList.get(0);
            map.put("STAR Key", key);
            map.put("Status", "Match");
            matchedStarIndices.add(starIdx);
            matched++;
        } else if (starIdxList.isEmpty()) {
            map.put("STAR Key", "<No Match>");
            map.put("Status", "Mismatch - No STAR entry");
            unmatched++;
        } else {
            map.put("STAR Key", key);
            map.put("Status", "Mismatch - Multiple STAR entries or already matched");
            unmatched++;
        }

        results.add(map);
    }

    for (Map.Entry<String, List<Integer>> entry : starKeys.entrySet()) {
        String key = entry.getKey();
        for (int idx : entry.getValue()) {
            if (!matchedStarIndices.contains(idx) && !algoKeys.containsKey(key)) {
                Map<String, String> map = new LinkedHashMap<>();
                map.put("ALGO Key", "<No Match>");
                map.put("STAR Key", key);
                map.put("Status", "Mismatch - STAR entry unmatched");
                results.add(map);
                unmatched++;
            }
        }
    }

    Map<String, Object> response = new LinkedHashMap<>();
    response.put("Matched Records", matched);
    response.put("Unmatched Records", unmatched);
    response.put("Details", results);

    return ResponseEntity.ok(response);
}
