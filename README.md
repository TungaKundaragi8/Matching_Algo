package com.example.demo.controller;

import com.example.demo.service.ReconciliationService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/api")
public class ReconciliationController {

    @Autowired
    private ReconciliationService service;

    @GetMapping("/update")
    public Map<String, Object> updateFile(
            @RequestParam String filePath,
            @RequestParam Map<String, String> updates,
            @RequestParam(required = false) List<String> appendColumns) {
        return service.updateRecords(filePath, updates, appendColumns);
    }

    @GetMapping("/exclude")
    public Map<String, Object> excludeRecords(
            @RequestParam String filePath,
            @RequestParam String column,
            @RequestParam String value) {
        return service.excludeRecords(filePath, column, value);
    }

    @GetMapping("/match")
    public Map<String, Object> matchFiles(
            @RequestParam String file1,
            @RequestParam String file2,
            @RequestParam String matchType,
            @RequestParam String col1,
            @RequestParam String col2) {
        return service.matchRecords(file1, file2, matchType, col1, col2);
    }
}
package com.example.demo.service;

import org.apache.commons.csv.*;
import org.springframework.stereotype.Service;

import java.io.*;
import java.nio.file.*;
import java.util.*;
import java.util.stream.Collectors;

@Service
public class ReconciliationService {

    private List<Map<String, String>> unmatchedRecords = new ArrayList<>();
    private boolean isFirstMatch = true;

    public Map<String, Object> updateRecords(String filePath, Map<String, String> updates, List<String> appendCols) {
        Map<String, Object> result = new HashMap<>();
        try (Reader reader = Files.newBufferedReader(Paths.get(filePath))) {
            CSVParser parser = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(reader);
            List<CSVRecord> records = parser.getRecords();
            List<Map<String, String>> updated = new ArrayList<>();

            for (CSVRecord record : records) {
                Map<String, String> row = new HashMap<>(record.toMap());
                boolean modified = false;

                for (String key : updates.keySet()) {
                    String[] parts = key.split(":");
                    String column = parts[0];
                    String oldVal = parts.length > 1 ? parts[1] : null;
                    String newValue = updates.get(key);

                    if (row.containsKey(column)) {
                        String currentVal = row.get(column);
                        boolean shouldUpdate = (oldVal == null || currentVal.equals(oldVal));

                        if (shouldUpdate) {
                            if (appendCols != null && appendCols.contains(column)) {
                                row.put(column, currentVal + newValue);
                            } else {
                                row.put(column, newValue);
                            }
                            modified = true;
                        }
                    }
                }
                if (modified) updated.add(row);
            }

            result.put("updatedRecords", updated);
            result.put("updatedCount", updated.size());
        } catch (Exception e) {
            result.put("error", e.getMessage());
        }
        return result;
    }

    public Map<String, Object> excludeRecords(String filePath, String column, String value) {
        Map<String, Object> result = new HashMap<>();
        try (Reader reader = Files.newBufferedReader(Paths.get(filePath))) {
            CSVParser parser = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(reader);
            List<CSVRecord> records = parser.getRecords();

            List<Map<String, String>> excluded = new ArrayList<>();
            List<Map<String, String>> nonExcluded = new ArrayList<>();

            for (CSVRecord record : records) {
                Map<String, String> row = record.toMap();
                if (row.getOrDefault(column, "").equals(value)) {
                    excluded.add(row);
                } else {
                    nonExcluded.add(row);
                }
            }

            unmatchedRecords = new ArrayList<>(nonExcluded);
            isFirstMatch = true;

            result.put("excludedCount", excluded.size());
            result.put("excludedRecords", excluded);
            result.put("nonExcludedCount", nonExcluded.size());
            result.put("nonExcludedRecords", nonExcluded);
        } catch (Exception e) {
            result.put("error", e.getMessage());
        }
        return result;
    }

    public Map<String, Object> matchRecords(String file1, String file2, String matchType, String col1, String col2) {
        Map<String, Object> result = new HashMap<>();
        try {
            List<Map<String, String>> records1;
            if (isFirstMatch) {
                records1 = unmatchedRecords;
                isFirstMatch = false;
            } else {
                records1 = unmatchedRecords;
            }

            List<Map<String, String>> records2 = readCsv(file2);
            List<Map<String, String>> matched = new ArrayList<>();
            List<Map<String, String>> unmatched = new ArrayList<>();

            switch (matchType.toLowerCase()) {
                case "1-1":
                    for (Map<String, String> r1 : records1) {
                        boolean found = false;
                        for (Map<String, String> r2 : records2) {
                            if (r1.get(col1).equals(r2.get(col2))) {
                                matched.add(mergeRecords(r1, r2));
                                found = true;
                                break;
                            }
                        }
                        if (!found) unmatched.add(r1);
                    }
                    break;
                case "1-many":
                    for (Map<String, String> r1 : records1) {
                        boolean found = false;
                        for (Map<String, String> r2 : records2) {
                            if (r1.get(col1).equals(r2.get(col2))) {
                                matched.add(mergeRecords(r1, r2));
                                found = true;
                            }
                        }
                        if (!found) unmatched.add(r1);
                    }
                    break;
                case "many-1":
                    for (Map<String, String> r2 : records2) {
                        boolean found = false;
                        for (Map<String, String> r1 : records1) {
                            if (r1.get(col1).equals(r2.get(col2))) {
                                matched.add(mergeRecords(r1, r2));
                                found = true;
                            }
                        }
                        if (!found) unmatched.add(r2);
                    }
                    break;
                case "many-many":
                    for (Map<String, String> r1 : records1) {
                        for (Map<String, String> r2 : records2) {
                            if (r1.get(col1).equals(r2.get(col2))) {
                                matched.add(mergeRecords(r1, r2));
                            }
                        }
                    }
                    Set<Map<String, String>> matchedSet = new HashSet<>(matched);
                    unmatched = records1.stream().filter(r -> !matchedSet.contains(r)).collect(Collectors.toList());
                    break;
                default:
                    result.put("error", "Invalid match type");
                    return result;
            }

            unmatchedRecords = unmatched;

            result.put("matchedCount", matched.size());
            result.put("matchedRecords", matched);
            result.put("unmatchedCount", unmatched.size());
            result.put("unmatchedRecords", unmatched);
        } catch (Exception e) {
            result.put("error", e.getMessage());
        }
        return result;
    }

    private List<Map<String, String>> readCsv(String path) throws IOException {
        try (Reader reader = Files.newBufferedReader(Paths.get(path))) {
            CSVParser parser = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(reader);
            return parser.getRecords().stream().map(CSVRecord::toMap).collect(Collectors.toList());
        }
    }

    private Map<String, String> mergeRecords(Map<String, String> r1, Map<String, String> r2) {
        Map<String, String> merged = new HashMap<>(r1);
        for (Map.Entry<String, String> entry : r2.entrySet()) {
            merged.putIfAbsent("file2_" + entry.getKey(), entry.getValue());
        }
        return merged;
    }
}









// pom.xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId>
  <artifactId>oracle-to-mongo-metadata</artifactId>
  <version>1.0.0</version>
  <packaging>jar</packaging>

  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-mongodb</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-jdbc</artifactId>
    </dependency>
    <dependency>
      <groupId>com.oracle.database.jdbc</groupId>
      <artifactId>ojdbc8</artifactId>
      <version>19.3.0.0</version>
    </dependency>
  </dependencies>

  <properties>
    <java.version>17</java.version>
  </properties>
</project>

// application.properties
spring.datasource.url=jdbc:oracle:thin:@localhost:1521:xe
spring.datasource.username=your_oracle_username
spring.datasource.password=your_oracle_password
spring.datasource.driver-class-name=oracle.jdbc.OracleDriver
spring.data.mongodb.uri=mongodb://localhost:27017/metadata_db

// TableMetadata.java
package com.example.model;

import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;
import java.util.List;

@Document("table_metadata")
public class TableMetadata {
    @Id
    private String id;
    private String tableName;
    private List<ColumnMetadata> columns;

    public static class ColumnMetadata {
        private String columnName;
        private String dataType;

        public String getColumnName() { return columnName; }
        public void setColumnName(String columnName) { this.columnName = columnName; }
        public String getDataType() { return dataType; }
        public void setDataType(String dataType) { this.dataType = dataType; }
    }

    public String getId() { return id; }
    public void setId(String id) { this.id = id; }
    public String getTableName() { return tableName; }
    public void setTableName(String tableName) { this.tableName = tableName; }
    public List<ColumnMetadata> getColumns() { return columns; }
    public void setColumns(List<ColumnMetadata> columns) { this.columns = columns; }
}

// TableMetadataRepository.java
package com.example.repository;

import com.example.model.TableMetadata;
import org.springframework.data.mongodb.repository.MongoRepository;

public interface TableMetadataRepository extends MongoRepository<TableMetadata, String> {
}

// MetadataService.java
package com.example.service;

import com.example.model.TableMetadata;
import com.example.repository.TableMetadataRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import javax.sql.DataSource;
import java.sql.*;
import java.util.ArrayList;
import java.util.List;

@Service
public class MetadataService {
    @Autowired
    private DataSource dataSource;
    @Autowired
    private TableMetadataRepository repository;

    public void extractMetadataAndSaveToMongo(String tableName) {
        try (Connection connection = dataSource.getConnection()) {
            DatabaseMetaData metaData = connection.getMetaData();
            ResultSet columns = metaData.getColumns(null, null, tableName.toUpperCase(), null);
            List<TableMetadata.ColumnMetadata> columnList = new ArrayList<>();

            while (columns.next()) {
                TableMetadata.ColumnMetadata column = new TableMetadata.ColumnMetadata();
                column.setColumnName(columns.getString("COLUMN_NAME"));
                column.setDataType(columns.getString("TYPE_NAME"));
                columnList.add(column);
            }

            TableMetadata metadata = new TableMetadata();
            metadata.setTableName(tableName.toUpperCase());
            metadata.setColumns(columnList);
            repository.save(metadata);
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
}

// MetadataController.java
package com.example.controller;

import com.example.service.MetadataService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/metadata")
public class MetadataController {
    @Autowired
    private MetadataService metadataService;

    @GetMapping("/extract/{tableName}")
    public String extractMetadata(@PathVariable String tableName) {
        metadataService.extractMetadataAndSaveToMongo(tableName);
        return "Metadata for table " + tableName + " stored in MongoDB.";
    }
}

// OracleToMongoMetadataApplication.java
package com.example;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class OracleToMongoMetadataApplication {
    public static void main(String[] args) {
        SpringApplication.run(OracleToMongoMetadataApplication.class, args);
    }
}





// pom.xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId>
  <artifactId>oracle-to-mongo-metadata</artifactId>
  <version>1.0.0</version>
  <packaging>jar</packaging>

  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-mongodb</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-jdbc</artifactId>
    </dependency>
    <dependency>
      <groupId>com.oracle.database.jdbc</groupId>
      <artifactId>ojdbc8</artifactId>
      <version>19.3.0.0</version>
    </dependency>
  </dependencies>

  <properties>
    <java.version>17</java.version>
  </properties>
</project>

// application.properties
spring.datasource.url=jdbc:oracle:thin:@localhost:1521:xe
spring.datasource.username=your_oracle_username
spring.datasource.password=your_oracle_password
spring.datasource.driver-class-name=oracle.jdbc.OracleDriver
spring.data.mongodb.uri=mongodb://localhost:27017/metadata_db

// model/TableMetadata.java
package com.example.model;

import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;
import java.util.List;

@Document("table_metadata")
public class TableMetadata {
    @Id
    private String id;
    private String tableName;
    private List<ColumnMetadata> columns;

    public static class ColumnMetadata {
        private String columnName;
        private String dataType;

        public String getColumnName() { return columnName; }
        public void setColumnName(String columnName) { this.columnName = columnName; }
        public String getDataType() { return dataType; }
        public void setDataType(String dataType) { this.dataType = dataType; }
    }

    public String getId() { return id; }
    public void setId(String id) { this.id = id; }
    public String getTableName() { return tableName; }
    public void setTableName(String tableName) { this.tableName = tableName; }
    public List<ColumnMetadata> getColumns() { return columns; }
    public void setColumns(List<ColumnMetadata> columns) { this.columns = columns; }
}

// repository/TableMetadataRepository.java
package com.example.repository;

import com.example.model.TableMetadata;
import org.springframework.data.mongodb.repository.MongoRepository;

public interface TableMetadataRepository extends MongoRepository<TableMetadata, String> {
}

// service/MetadataService.java
package com.example.service;

import com.example.model.TableMetadata;
import com.example.repository.TableMetadataRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import javax.sql.DataSource;
import java.sql.*;
import java.util.ArrayList;
import java.util.List;

@Service
public class MetadataService {
    @Autowired
    private DataSource dataSource;
    @Autowired
    private TableMetadataRepository repository;

    public void extractMetadataAndSaveToMongo(String tableName) {
        try (Connection connection = dataSource.getConnection()) {
            DatabaseMetaData metaData = connection.getMetaData();
            ResultSet columns = metaData.getColumns(null, null, tableName.toUpperCase(), null);
            List<TableMetadata.ColumnMetadata> columnList = new ArrayList<>();

            while (columns.next()) {
                TableMetadata.ColumnMetadata column = new TableMetadata.ColumnMetadata();
                column.setColumnName(columns.getString("COLUMN_NAME"));
                column.setDataType(columns.getString("TYPE_NAME"));
                columnList.add(column);
            }

            TableMetadata metadata = new TableMetadata();
            metadata.setTableName(tableName.toUpperCase());
            metadata.setColumns(columnList);
            repository.save(metadata);
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
}

// controller/MetadataController.java
package com.example.controller;

import com.example.service.MetadataService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

import java.util.Map;

@RestController
@RequestMapping("/metadata")
public class MetadataController {
    @Autowired
    private MetadataService metadataService;

    @PostMapping("/extract")
    public String extractMetadata(@RequestBody Map<String, String> body) {
        String tableName = body.get("tableName");
        if (tableName == null || tableName.isBlank()) {
            return "Error: tableName is required in request body.";
        }
        metadataService.extractMetadataAndSaveToMongo(tableName);
        return "Metadata for table '" + tableName + "' stored in MongoDB.";
    }
}

// OracleToMongoMetadataApplication.java
package com.example;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class OracleToMongoMetadataApplication {
    public static void main(String[] args) {
        SpringApplication.run(OracleToMongoMetadataApplication.class, args);
    }
}
