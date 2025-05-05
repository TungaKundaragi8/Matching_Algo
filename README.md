package com.example.demo.service;

import com.example.demo.model.ReconciliationResult; import com.example.demo.model.Record; import org.apache.commons.csv.CSVFormat; import org.apache.commons.csv.CSVParser; import org.apache.commons.csv.CSVRecord; import org.springframework.stereotype.Service; import org.springframework.web.multipart.MultipartFile;

import java.io.BufferedReader; import java.io.InputStreamReader; import java.io.Reader; import java.time.LocalDate; import java.time.LocalDateTime; import java.time.format.DateTimeFormatter; import java.util.*; import java.util.stream.Collectors;

@Service public class ReconciliationService {

private final DateTimeFormatter exclusionFormatter = DateTimeFormatter.ofPattern("MMM d yyyy hh:mm:ss:SSSa", Locale.ENGLISH);

public ReconciliationResult reconcile(MultipartFile file1, MultipartFile file2, String matchType) {
    try {
        List<Record> file1Records = parseCSV(file1);
        List<Record> file2Records = parseCSV(file2);

        // Apply exclusion: remove STAR records with Maturity Date = Jan 1 1900 12:00:00:000AM
        List<Record> excluded = file2Records.stream()
            .filter(r -> isExcluded(r.getMaturityDate()))
            .collect(Collectors.toList());

        file2Records.removeAll(excluded);

        List<Record> matched = new ArrayList<>();
        List<Record> unmatched = new ArrayList<>(file1Records);

        switch (matchType) {
            case "1-1":
                performOneToOneMatching(unmatched, file2Records, matched);
                break;
            case "1-many":
                performOneToManyMatching(unmatched, file2Records, matched);
                break;
            case "many-1":
                performManyToOneMatching(unmatched, file2Records, matched);
                break;
            case "many-many":
                performManyToManyMatching(unmatched, file2Records, matched);
                break;
            default:
                throw new IllegalArgumentException("Invalid match type: " + matchType);
        }

        return new ReconciliationResult(matched, unmatched, excluded);
    } catch (Exception e) {
        throw new RuntimeException("Error during reconciliation: " + e.getMessage(), e);
    }
}

private boolean isExcluded(String maturityDate) {
    try {
        LocalDate date = LocalDateTime.parse(maturityDate.trim(), exclusionFormatter).toLocalDate();
        return date.equals(LocalDate.of(1900, 1, 1));
    } catch (Exception e) {
        return false;
    }
}

private List<Record> parseCSV(MultipartFile file) throws Exception {
    Reader reader = new BufferedReader(new InputStreamReader(file.getInputStream()));
    CSVParser csvParser = new CSVParser(reader, CSVFormat.DEFAULT.withFirstRecordAsHeader());

    List<Record> records = new ArrayList<>();
    for (CSVRecord csvRecord : csvParser) {
        records.add(new Record(
                csvRecord.get("ID"),
                csvRecord.get("Amount"),
                csvRecord.get("Date"),
                csvRecord.get("MaturityDate")
        ));
    }
    return records;
}

private void performOneToOneMatching(List<Record> unmatched, List<Record> source, List<Record> matched) {
    Iterator<Record> itr = unmatched.iterator();
    while (itr.hasNext()) {
        Record rec1 = itr.next();
        Optional<Record> match = source.stream()
                .filter(rec2 -> rec1.equals(rec2))
                .findFirst();
        match.ifPresent(matched::add);
        match.ifPresent(source::remove);
        if (match.isPresent()) itr.remove();
    }
}

private void performOneToManyMatching(List<Record> unmatched, List<Record> source, List<Record> matched) {
    for (Iterator<Record> it = unmatched.iterator(); it.hasNext(); ) {
        Record record = it.next();
        List<Record> matches = source.stream()
                .filter(s -> s.equals(record))
                .collect(Collectors.toList());
        if (!matches.isEmpty()) {
            matched.addAll(matches);
            source.removeAll(matches);
            it.remove();
        }
    }
}

private void performManyToOneMatching(List<Record> unmatched, List<Record> source, List<Record> matched) {
    for (Record s : new ArrayList<>(source)) {
        List<Record> matches = unmatched.stream()
                .filter(r -> r.equals(s))
                .collect(Collectors.toList());
        if (!matches.isEmpty()) {
            matched.addAll(matches);
            unmatched.removeAll(matches);
            source.remove(s);
        }
    }
}

private void performManyToManyMatching(List<Record> unmatched, List<Record> source, List<Record> matched) {
    List<Record> matches = unmatched.stream()
            .filter(source::contains)
            .collect(Collectors.toList());
    matched.addAll(matches);
    unmatched.removeAll(matches);
    source.removeAll(matches);
}

}
