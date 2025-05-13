@Service
public class ReconciliationService {

    private Map<String, CSVRecord> nonExcludedStarRecords = new HashMap<>();
    private Map<String, CSVRecord> excludedStarRecords = new HashMap<>();
    private Map<String, CSVRecord> algoRecordsMap = new HashMap<>();

    public void excludeByMaturity(String algoPath, String starPath) throws IOException {
        CSVParser algoParser = CSVFormat.DEFAULT.withFirstRecordAsHeader()
                .parse(new FileReader(algoPath));
        CSVParser starParser = CSVFormat.DEFAULT.withFirstRecordAsHeader()
                .parse(new FileReader(starPath));

        nonExcludedStarRecords.clear();
        excludedStarRecords.clear();
        algoRecordsMap.clear();

        for (CSVRecord record : starParser.getRecords()) {
            String maturityDate = record.get("Maturity Date").trim().toLowerCase();
            String starKey = record.get("CRDS Party Code").replaceAll("\\s", "") +
                             "3" + record.get("Post Direction").replaceAll("\\s", "");

            if (maturityDate.contains("1900")) {
                excludedStarRecords.put(starKey, record);
            } else {
                nonExcludedStarRecords.put(starKey, record);
            }
        }

        for (CSVRecord record : algoParser.getRecords()) {
            String transformedKey = record.get("AGREEMENT_NAME")
                    .replace("_RIMP", "_3CP")
                    .replace("_RIMR", "_3CR")
                    .replaceAll("\\s", "");
            algoRecordsMap.put(transformedKey, record);
        }
    }

    public ReconciliationResult performReconciliation(String matchType) {
        Map<String, CSVRecord> matched = new HashMap<>();
        Map<String, CSVRecord> unmatchedAlgo = new HashMap<>(algoRecordsMap);
        Map<String, CSVRecord> unmatchedStar = new HashMap<>(nonExcludedStarRecords);

        switch (matchType.toLowerCase()) {
            case "1-1":
                performOneToOneMatch(unmatchedAlgo, unmatchedStar, matched);
                break;
            case "1-many":
                performOneToManyMatch(unmatchedAlgo, unmatchedStar, matched);
                break;
            case "many-1":
                performManyToOneMatch(unmatchedAlgo, unmatchedStar, matched);
                break;
            case "many-many":
                performManyToManyMatch(unmatchedAlgo, unmatchedStar, matched);
                break;
            case "cascade":
                performOneToOneMatch(unmatchedAlgo, unmatchedStar, matched);
                performOneToManyMatch(unmatchedAlgo, unmatchedStar, matched);
                performManyToOneMatch(unmatchedAlgo, unmatchedStar, matched);
                performManyToManyMatch(unmatchedAlgo, unmatchedStar, matched);
                break;
            default:
                throw new IllegalArgumentException("Unsupported match type: " + matchType);
        }

        return new ReconciliationResult(
                matched.values(),
                unmatchedAlgo.values(),
                unmatchedStar.values(),
                excludedStarRecords.values()
        );
    }

    private void performOneToOneMatch(Map<String, CSVRecord> algo, Map<String, CSVRecord> star, Map<String, CSVRecord> matched) {
        Iterator<Map.Entry<String, CSVRecord>> iterator = algo.entrySet().iterator();
        while (iterator.hasNext()) {
            Map.Entry<String, CSVRecord> entry = iterator.next();
            String key = entry.getKey();
            if (star.containsKey(key)) {
                matched.put(key, entry.getValue());
                star.remove(key);
                iterator.remove();
            }
        }
    }

    private void performOneToManyMatch(Map<String, CSVRecord> algo, Map<String, CSVRecord> star, Map<String, CSVRecord> matched) {
        // Example logic: pick first star match for each algo
        for (String key : new HashSet<>(algo.keySet())) {
            String prefix = key.substring(0, key.length() - 3); // just as an example
            for (String starKey : star.keySet()) {
                if (starKey.startsWith(prefix)) {
                    matched.put(key, algo.get(key));
                    star.remove(starKey);
                    algo.remove(key);
                    break;
                }
            }
        }
    }

    private void performManyToOneMatch(Map<String, CSVRecord> algo, Map<String, CSVRecord> star, Map<String, CSVRecord> matched) {
        // Reverse: find star keys that match any part of algo keys
        for (String starKey : new HashSet<>(star.keySet())) {
            String prefix = starKey.substring(0, starKey.length() - 3); // example logic
            for (String algoKey : algo.keySet()) {
                if (algoKey.startsWith(prefix)) {
                    matched.put(algoKey, algo.get(algoKey));
                    star.remove(starKey);
                    algo.remove(algoKey);
                    break;
                }
            }
        }
    }

    private void performManyToManyMatch(Map<String, CSVRecord> algo, Map<String, CSVRecord> star, Map<String, CSVRecord> matched) {
        // Brute force match if key contains similar part
        for (String ak : new HashSet<>(algo.keySet())) {
            for (String sk : new HashSet<>(star.keySet())) {
                if (ak.contains(sk) || sk.contains(ak)) {
                    matched.put(ak, algo.get(ak));
                    star.remove(sk);
                    algo.remove(ak);
                    break;
                }
            }
        }
    }
}



public class ReconciliationResult {
    private Collection<CSVRecord> matchedRecords;
    private Collection<CSVRecord> unmatchedAlgo;
    private Collection<CSVRecord> unmatchedStar;
    private Collection<CSVRecord> excludedRecords;

    public ReconciliationResult(Collection<CSVRecord> matchedRecords,
                                Collection<CSVRecord> unmatchedAlgo,
                                Collection<CSVRecord> unmatchedStar,
                                Collection<CSVRecord> excludedRecords) {
        this.matchedRecords = matchedRecords;
        this.unmatchedAlgo = unmatchedAlgo;
        this.unmatchedStar = unmatchedStar;
        this.excludedRecords = excludedRecords;
    }

    // Getters and setters
}
