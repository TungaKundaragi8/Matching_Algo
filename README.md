

@GetMapping("/run")
public List<Map<String, Object>> runReconciliation() throws IOException {
    File algoFile = new File("C:\\Users\\h59283\\Documents\\Initial_Margin_EOD_20250407.csv");
    File starFile = new File("C:\\Users\\h59283\\Documents\\STARALGONEW_3428_20250407.csv");

    BufferedReader algoReader = new BufferedReader(new FileReader(algoFile));
    BufferedReader starReader = new BufferedReader(new FileReader(starFile));

    String[] algoHeaders = algoReader.readLine().split(",");
    String[] starHeaders = starReader.readLine().split(",");

    List<Map<String, Object>> comparisonResult = new ArrayList<>();
    String algoLine, starLine;
    int rowNumber = 1;

    while ((algoLine = algoReader.readLine()) != null && (starLine = starReader.readLine()) != null) {
        String[] algoData = algoLine.split(",");
        String[] starData = starLine.split(",");

        Map<String, Object> rowResult = new HashMap<>();
        rowResult.put("Row", rowNumber++);

        List<Map<String, String>> columnComparisons = new ArrayList<>();

        int maxColumns = Math.max(algoHeaders.length, starHeaders.length);
        for (int i = 0; i < maxColumns; i++) {
            String columnName = (i < algoHeaders.length) ? algoHeaders[i] : starHeaders[i];
            String algoValue = (i < algoData.length) ? algoData[i] : "";
            String starValue = (i < starData.length) ? starData[i] : "";

            Map<String, String> columnResult = new LinkedHashMap<>();
            columnResult.put("Column", columnName);
            columnResult.put("ALGO Value", algoValue);
            columnResult.put("STAR Value", starValue);
            columnResult.put("Status", algoValue.equals(starValue) ? "Match" : "Mismatch");

            columnComparisons.add(columnResult);
        }

        rowResult.put("Columns", columnComparisons);
        comparisonResult.add(rowResult);
    }

    algoReader.close();
    starReader.close();

    return comparisonResult;
}
