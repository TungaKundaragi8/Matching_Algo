# Matching_Algo

// Project: csv-compare-app (Spring Boot Backend)

// ===================== // 1. CsvCompareApplication.java // ===================== package com.example.csvcompare;

import org.springframework.boot.SpringApplication; import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication public class CsvCompareApplication { public static void main(String[] args) { SpringApplication.run(CsvCompareApplication.class, args); } }

// ===================== // 2. model/FileUpload.java // ===================== package com.example.csvcompare.model;

import jakarta.persistence.*; import java.time.LocalDateTime;

@Entity public class FileUpload { @Id @GeneratedValue(strategy = GenerationType.IDENTITY) private Long id;

private String fileName;
private LocalDateTime uploadTime;

// Getters and Setters
public Long getId() { return id; }
public void setId(Long id) { this.id = id; }
public String getFileName() { return fileName; }
public void setFileName(String fileName) { this.fileName = fileName; }
public LocalDateTime getUploadTime() { return uploadTime; }
public void setUploadTime(LocalDateTime uploadTime) { this.uploadTime = uploadTime; }

}

// ===================== // 3. model/CsvRowData.java // ===================== package com.example.csvcompare.model;

import jakarta.persistence.*;

@Entity public class CsvRowData { @Id @GeneratedValue(strategy = GenerationType.IDENTITY) private Long id;

private String content;

@ManyToOne
private FileUpload file;

// Getters and Setters
public Long getId() { return id; }
public void setId(Long id) { this.id = id; }
public String getContent() { return content; }
public void setContent(String content) { this.content = content; }
public FileUpload getFile() { return file; }
public void setFile(FileUpload file) { this.file = file; }

}

// ===================== // 4. model/ComparisonResult.java // ===================== package com.example.csvcompare.model;

import jakarta.persistence.*;

@Entity public class ComparisonResult { @Id @GeneratedValue(strategy = GenerationType.IDENTITY) private Long id;

private String rowFromFile1;
private String rowFromFile2;
private boolean isMatch;

@ManyToOne
private FileUpload file1;

@ManyToOne
private FileUpload file2;

// Getters and Setters
public Long getId() { return id; }
public void setId(Long id) { this.id = id; }
public String getRowFromFile1() { return rowFromFile1; }
public void setRowFromFile1(String rowFromFile1) { this.rowFromFile1 = rowFromFile1; }
public String getRowFromFile2() { return rowFromFile2; }
public void setRowFromFile2(String rowFromFile2) { this.rowFromFile2 = rowFromFile2; }
public boolean isMatch() { return isMatch; }
public void setMatch(boolean match) { isMatch = match; }
public FileUpload getFile1() { return file1; }
public void setFile1(FileUpload file1) { this.file1 = file1; }
public FileUpload getFile2() { return file2; }
public void setFile2(FileUpload file2) { this.file2 = file2; }

}

