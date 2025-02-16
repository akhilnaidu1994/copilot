Below is a complete, production‐ready example using Lombok for getters/setters and SLF4J for logging. The code is organized by package, and each class is provided below. You can copy these files into your Spring Boot project. Make sure you have Lombok configured in your IDE and included in your Maven/Gradle dependencies.

---

## 1. Maven Dependencies

In your **pom.xml**, include these dependencies (among others):

```xml
<dependencies>
    <!-- Spring Boot Starter Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <!-- Spring Boot Starter Data JPA -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <!-- MySQL Connector (or another DB driver) -->
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <scope>runtime</scope>
    </dependency>
    <!-- Lombok -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>1.18.26</version>
        <scope>provided</scope>
    </dependency>
    <!-- Jackson -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
    </dependency>
</dependencies>
```

---

## 2. Database DDL (MySQL Syntax)

Run the following SQL to create the necessary tables:

```sql
-- Table to store source data (for both SOURCE1 and SOURCE2)
CREATE TABLE source_data (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    reconciliation_type VARCHAR(50) NOT NULL, -- e.g. "VAM_BAX", "FLEX_BAX"
    source_type VARCHAR(10) NOT NULL,           -- "SOURCE1" or "SOURCE2"
    transaction_id VARCHAR(255) NOT NULL,
    client_id VARCHAR(50),                      -- Client identifier
    client_name VARCHAR(255),                   -- Client name
    data JSON,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Table to store field-level comparison configurations
CREATE TABLE comparison_config (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    reconciliation_type VARCHAR(50) NOT NULL,  -- e.g. "VAM_BAX", "FLEX_BAX"
    field_name VARCHAR(255) NOT NULL,
    source1_field VARCHAR(255) NOT NULL,
    source2_field VARCHAR(255) NOT NULL,
    comparison_method VARCHAR(50) NOT NULL,     -- e.g. "EQUALS", "SUBSTRING"
    custom_comparison_logic TEXT,               -- e.g. '{"start":0, "end":20}'
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Table to store reconciliation overview per client
CREATE TABLE reconciliation_overview (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    run_id VARCHAR(36) NOT NULL,
    reconciliation_type VARCHAR(50) NOT NULL,
    client_id VARCHAR(50) NOT NULL,
    client_name VARCHAR(255) NOT NULL,
    run_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    total_records_processed INT,
    missing_in_source1 INT,
    missing_in_source2 INT,
    matched_records INT,
    mismatched_records INT,
    additional_info JSON
);

-- Table to store reconciliation details (missing/mismatched records)
CREATE TABLE reconciliation_detail (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    run_id VARCHAR(36) NOT NULL,
    transaction_id VARCHAR(255) NOT NULL,
    issue VARCHAR(50) NOT NULL,   -- e.g., "missing", "mismatch"
    details JSON,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## 3. Java Code

Below are the full classes using Lombok and SLF4J.

### a. Application Class

**ReconciliationApplication.java**
```java
package com.example.reconciliation;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class ReconciliationApplication {
    public static void main(String[] args) {
        SpringApplication.run(ReconciliationApplication.class, args);
    }
}
```

---

### b. Entities

**SourceData.java**
```java
package com.example.reconciliation.entity;

import lombok.Data;
import lombok.NoArgsConstructor;

import javax.persistence.*;
import java.time.LocalDateTime;

@Data
@NoArgsConstructor
@Entity
@Table(name = "source_data")
public class SourceData {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "reconciliation_type", nullable = false)
    private String reconciliationType;

    @Column(name = "source_type", nullable = false)
    private String sourceType; // "SOURCE1" or "SOURCE2"

    @Column(name = "transaction_id", nullable = false)
    private String transactionId;

    @Column(name = "client_id")
    private String clientId;

    @Column(name = "client_name")
    private String clientName;

    @Lob
    @Column(name = "data")
    private String data;

    @Column(name = "created_at")
    private LocalDateTime createdAt = LocalDateTime.now();
}
```

**ComparisonConfig.java**
```java
package com.example.reconciliation.entity;

import lombok.Data;
import lombok.NoArgsConstructor;

import javax.persistence.*;
import java.time.LocalDateTime;

@Data
@NoArgsConstructor
@Entity
@Table(name = "comparison_config")
public class ComparisonConfig {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "reconciliation_type", nullable = false)
    private String reconciliationType;

    @Column(name = "field_name", nullable = false)
    private String fieldName;

    @Column(name = "source1_field", nullable = false)
    private String source1Field;

    @Column(name = "source2_field", nullable = false)
    private String source2Field;

    @Column(name = "comparison_method", nullable = false)
    private String comparisonMethod;

    @Lob
    @Column(name = "custom_comparison_logic")
    private String customComparisonLogic;

    @Column(name = "created_at")
    private LocalDateTime createdAt = LocalDateTime.now();
}
```

**ReconciliationOverview.java**
```java
package com.example.reconciliation.entity;

import lombok.Data;
import lombok.NoArgsConstructor;

import javax.persistence.*;
import java.time.LocalDateTime;

@Data
@NoArgsConstructor
@Entity
@Table(name = "reconciliation_overview")
public class ReconciliationOverview {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "run_id", nullable = false)
    private String runId;

    @Column(name = "reconciliation_type", nullable = false)
    private String reconciliationType;

    @Column(name = "client_id", nullable = false)
    private String clientId;

    @Column(name = "client_name", nullable = false)
    private String clientName;

    @Column(name = "run_date")
    private LocalDateTime runDate = LocalDateTime.now();

    @Column(name = "total_records_processed")
    private int totalRecordsProcessed;

    @Column(name = "missing_in_source1")
    private int missingInSource1;

    @Column(name = "missing_in_source2")
    private int missingInSource2;

    @Column(name = "matched_records")
    private int matchedRecords;

    @Column(name = "mismatched_records")
    private int mismatchedRecords;

    @Lob
    @Column(name = "additional_info")
    private String additionalInfo;
}
```

**ReconciliationDetail.java**
```java
package com.example.reconciliation.entity;

import lombok.Data;
import lombok.NoArgsConstructor;

import javax.persistence.*;
import java.time.LocalDateTime;

@Data
@NoArgsConstructor
@Entity
@Table(name = "reconciliation_detail")
public class ReconciliationDetail {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "run_id", nullable = false)
    private String runId;

    @Column(name = "transaction_id", nullable = false)
    private String transactionId;

    @Column(name = "issue", nullable = false)
    private String issue;  // e.g. "missing", "mismatch"

    @Lob
    @Column(name = "details")
    private String details;

    @Column(name = "created_at")
    private LocalDateTime createdAt = LocalDateTime.now();
}
```

---

### c. Repositories

**SourceDataRepository.java**
```java
package com.example.reconciliation.repository;

import com.example.reconciliation.entity.SourceData;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.List;

public interface SourceDataRepository extends JpaRepository<SourceData, Long> {

    Page<SourceData> findByReconciliationTypeAndSourceType(String reconciliationType, String sourceType, Pageable pageable);

    List<SourceData> findByReconciliationTypeAndSourceTypeAndTransactionIdIn(String reconciliationType, String sourceType, List<String> transactionIds);
}
```

**ComparisonConfigRepository.java**
```java
package com.example.reconciliation.repository;

import com.example.reconciliation.entity.ComparisonConfig;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.List;

public interface ComparisonConfigRepository extends JpaRepository<ComparisonConfig, Long> {
    List<ComparisonConfig> findByReconciliationType(String reconciliationType);
}
```

**ReconciliationOverviewRepository.java**
```java
package com.example.reconciliation.repository;

import com.example.reconciliation.entity.ReconciliationOverview;
import org.springframework.data.jpa.repository.JpaRepository;

public interface ReconciliationOverviewRepository extends JpaRepository<ReconciliationOverview, Long> {
}
```

**ReconciliationDetailRepository.java**
```java
package com.example.reconciliation.repository;

import com.example.reconciliation.entity.ReconciliationDetail;
import org.springframework.data.jpa.repository.JpaRepository;

public interface ReconciliationDetailRepository extends JpaRepository<ReconciliationDetail, Long> {
}
```

---

### d. Strategy Pattern for Field Comparison

**ComparisonStrategy.java**
```java
package com.example.reconciliation.strategy;

import com.example.reconciliation.entity.ComparisonConfig;

public interface ComparisonStrategy {
    boolean compare(String value1, String value2, ComparisonConfig config);
}
```

**EqualsComparisonStrategy.java**
```java
package com.example.reconciliation.strategy;

import com.example.reconciliation.entity.ComparisonConfig;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import java.util.Objects;

@Slf4j
@Component("EQUALS")
public class EqualsComparisonStrategy implements ComparisonStrategy {
    @Override
    public boolean compare(String value1, String value2, ComparisonConfig config) {
        boolean result = Objects.equals(value1, value2);
        log.debug("EqualsComparison: {} vs {} -> {}", value1, value2, result);
        return result;
    }
}
```

**SubstringComparisonStrategy.java**
```java
package com.example.reconciliation.strategy;

import com.example.reconciliation.entity.ComparisonConfig;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

@Slf4j
@Component("SUBSTRING")
public class SubstringComparisonStrategy implements ComparisonStrategy {

    private final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public boolean compare(String value1, String value2, ComparisonConfig config) {
        try {
            if (config.getCustomComparisonLogic() != null && !config.getCustomComparisonLogic().isEmpty()) {
                JsonNode node = objectMapper.readTree(config.getCustomComparisonLogic());
                int start = node.has("start") ? node.get("start").asInt() : 0;
                int end = node.has("end") ? node.get("end").asInt() : value1.length();
                start = Math.max(0, start);
                end = Math.min(end, value1.length());
                String sub1 = value1.substring(start, end);
                int end2 = Math.min(end, value2.length());
                String sub2 = value2.substring(start, end2);
                boolean result = sub1.equals(sub2);
                log.debug("SubstringComparison: [{}] vs [{}] -> {}", sub1, sub2, result);
                return result;
            }
        } catch (Exception e) {
            log.error("Error in SubstringComparisonStrategy: {}", e.getMessage(), e);
        }
        // Fallback logic if custom logic is missing
        boolean result = value1.contains(value2) || value2.contains(value1);
        log.debug("SubstringComparison fallback: {} vs {} -> {}", value1, value2, result);
        return result;
    }
}
```

**ComparisonStrategyFactory.java**
```java
package com.example.reconciliation.strategy;

import org.springframework.stereotype.Service;

import java.util.Map;

@Service
public class ComparisonStrategyFactory {

    private final Map<String, ComparisonStrategy> strategyMap;

    public ComparisonStrategyFactory(Map<String, ComparisonStrategy> strategyMap) {
        this.strategyMap = strategyMap;
    }

    public ComparisonStrategy getStrategy(String type) {
        return strategyMap.getOrDefault(type.toUpperCase(), strategyMap.get("EQUALS"));
    }
}
```

---

### e. ReconciliationService.java

**ReconciliationService.java**
```java
package com.example.reconciliation.service;

import com.example.reconciliation.entity.*;
import com.example.reconciliation.repository.*;
import com.example.reconciliation.strategy.ComparisonStrategy;
import com.example.reconciliation.strategy.ComparisonStrategyFactory;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.*;
import java.util.stream.Collectors;
import java.util.UUID;

@Slf4j
@Service
public class ReconciliationService {

    private final SourceDataRepository sourceDataRepo;
    private final ComparisonConfigRepository configRepo;
    private final ReconciliationOverviewRepository overviewRepo;
    private final ReconciliationDetailRepository detailRepo;
    private final ComparisonStrategyFactory strategyFactory;
    private final ObjectMapper objectMapper = new ObjectMapper();

    public ReconciliationService(SourceDataRepository sourceDataRepo,
                                 ComparisonConfigRepository configRepo,
                                 ReconciliationOverviewRepository overviewRepo,
                                 ReconciliationDetailRepository detailRepo,
                                 ComparisonStrategyFactory strategyFactory) {
        this.sourceDataRepo = sourceDataRepo;
        this.configRepo = configRepo;
        this.overviewRepo = overviewRepo;
        this.detailRepo = detailRepo;
        this.strategyFactory = strategyFactory;
    }

    // Helper class to aggregate reconciliation results per client.
    public static class ClientAggregation {
        private final String clientId;
        private final String clientName;
        private final Set<String> allTransactionIds = new HashSet<>();
        private final Set<String> processedTxIds = new HashSet<>();
        private int missingInSource1 = 0;
        private int missingInSource2 = 0;
        private int matchedRecords = 0;
        private int mismatchedRecords = 0;

        public ClientAggregation(String clientId, String clientName) {
            this.clientId = clientId;
            this.clientName = clientName;
        }
        public String getClientId() { return clientId; }
        public String getClientName() { return clientName; }
        public Set<String> getAllTransactionIds() { return allTransactionIds; }
        public Set<String> getProcessedTxIds() { return processedTxIds; }
        public int getMissingInSource1() { return missingInSource1; }
        public void incrementMissingInSource1() { missingInSource1++; }
        public int getMissingInSource2() { return missingInSource2; }
        public void incrementMissingInSource2() { missingInSource2++; }
        public int getMatchedRecords() { return matchedRecords; }
        public void incrementMatchedRecords() { matchedRecords++; }
        public int getMismatchedRecords() { return mismatchedRecords; }
        public void incrementMismatchedRecords() { mismatchedRecords++; }
    }

    @Transactional
    public void runReconciliation(String reconciliationType) {
        String runId = UUID.randomUUID().toString();
        int batchSize = 1000;
        int pageNumber = 0;

        // Map to aggregate results per client.
        Map<String, ClientAggregation> clientAggMap = new HashMap<>();

        log.info("Starting reconciliation run {} for type {}", runId, reconciliationType);

        // Load comparison configurations.
        List<ComparisonConfig> configs = configRepo.findByReconciliationType(reconciliationType);

        // Process SOURCE1 in batches.
        Page<SourceData> source1Page;
        do {
            source1Page = sourceDataRepo.findByReconciliationTypeAndSourceType(
                    reconciliationType, "SOURCE1", PageRequest.of(pageNumber, batchSize));
            List<SourceData> source1List = source1Page.getContent();
            log.info("Processing SOURCE1 batch {} with {} records", pageNumber, source1List.size());

            if (!source1List.isEmpty()) {
                List<String> txIds = source1List.stream()
                        .map(SourceData::getTransactionId)
                        .collect(Collectors.toList());

                List<SourceData> source2List = sourceDataRepo
                        .findByReconciliationTypeAndSourceTypeAndTransactionIdIn(
                                reconciliationType, "SOURCE2", txIds);
                Map<String, SourceData> source2Map = source2List.stream()
                        .collect(Collectors.toMap(SourceData::getTransactionId, r -> r));

                for (SourceData s1 : source1List) {
                    String txId = s1.getTransactionId();
                    String clientId = s1.getClientId();
                    String clientName = s1.getClientName();

                    ClientAggregation agg = clientAggMap.computeIfAbsent(clientId,
                            id -> new ClientAggregation(clientId, clientName));

                    agg.getAllTransactionIds().add(txId);
                    agg.getProcessedTxIds().add(txId);

                    SourceData s2 = source2Map.get(txId);
                    if (s2 == null) {
                        agg.incrementMissingInSource2();
                        saveDetail(runId, txId, "missing", "{\"source\":\"SOURCE2\",\"reason\":\"Record not found\"}");
                        continue;
                    }

                    try {
                        JsonNode json1 = objectMapper.readTree(s1.getData());
                        JsonNode json2 = objectMapper.readTree(s2.getData());

                        boolean recordMatched = true;
                        Map<String, String> fieldDifferences = new HashMap<>();

                        for (ComparisonConfig config : configs) {
                            String value1 = getFieldValue(json1, config.getSource1Field());
                            String value2 = getFieldValue(json2, config.getSource2Field());

                            if (value1 == null || value2 == null) {
                                recordMatched = false;
                                fieldDifferences.put(config.getFieldName(),
                                        String.format("Field missing (SOURCE1: %s, SOURCE2: %s)", value1, value2));
                                continue;
                            }

                            ComparisonStrategy strategy = strategyFactory.getStrategy(config.getComparisonMethod());
                            boolean match = strategy.compare(value1, value2, config);
                            if (!match) {
                                recordMatched = false;
                                fieldDifferences.put(config.getFieldName(),
                                        String.format("Mismatch (SOURCE1: '%s', SOURCE2: '%s')", value1, value2));
                            }
                        }

                        if (recordMatched) {
                            agg.incrementMatchedRecords();
                        } else {
                            agg.incrementMismatchedRecords();
                            saveDetail(runId, txId, "mismatch", objectMapper.writeValueAsString(fieldDifferences));
                        }
                    } catch (Exception e) {
                        agg.incrementMismatchedRecords();
                        log.error("Error processing transaction {}: {}", txId, e.getMessage(), e);
                        saveDetail(runId, txId, "mismatch", "{\"error\":\"" + e.getMessage() + "\"}");
                    }
                }
            }
            pageNumber++;
        } while (source1Page.hasNext());

        // Process SOURCE2 in batches to capture records missing in SOURCE1.
        pageNumber = 0;
        Page<SourceData> source2Page;
        do {
            source2Page = sourceDataRepo.findByReconciliationTypeAndSourceType(
                    reconciliationType, "SOURCE2", PageRequest.of(pageNumber, batchSize));
            List<SourceData> source2List = source2Page.getContent();
            log.info("Processing SOURCE2 batch {} with {} records", pageNumber, source2List.size());

            for (SourceData s2 : source2List) {
                String txId = s2.getTransactionId();
                String clientId = s2.getClientId();
                String clientName = s2.getClientName();

                ClientAggregation agg = clientAggMap.computeIfAbsent(clientId,
                        id -> new ClientAggregation(clientId, clientName));
                agg.getAllTransactionIds().add(txId);
                if (!agg.getProcessedTxIds().contains(txId)) {
                    agg.incrementMissingInSource1();
                    saveDetail(runId, txId, "missing", "{\"source\":\"SOURCE1\",\"reason\":\"Record not found\"}");
                }
            }
            pageNumber++;
        } while (source2Page.hasNext());

        // Save an overview record per client.
        for (ClientAggregation agg : clientAggMap.values()) {
            ReconciliationOverview overview = new ReconciliationOverview();
            overview.setRunId(runId);
            overview.setReconciliationType(reconciliationType);
            overview.setClientId(agg.getClientId());
            overview.setClientName(agg.getClientName());
            overview.setTotalRecordsProcessed(agg.getAllTransactionIds().size());
            overview.setMissingInSource1(agg.getMissingInSource1());
            overview.setMissingInSource2(agg.getMissingInSource2());
            overview.setMatchedRecords(agg.getMatchedRecords());
            overview.setMismatchedRecords(agg.getMismatchedRecords());

            // Store summary additional info (do not store huge lists)
            Map<String, Object> extraInfo = new HashMap<>();
            extraInfo.put("processedTxCount", agg.getAllTransactionIds().size());
            try {
                overview.setAdditionalInfo(objectMapper.writeValueAsString(extraInfo));
            } catch (Exception ex) {
                overview.setAdditionalInfo("{}");
            }
            overviewRepo.save(overview);
            log.info("Saved overview for client {}: {} records processed", agg.getClientId(), agg.getAllTransactionIds().size());
        }

        log.info("Reconciliation run {} completed.", runId);
    }

    private String getFieldValue(JsonNode json, String fieldName) {
        JsonNode node = json.get(fieldName);
        return (node != null && !node.isNull()) ? node.asText() : null;
    }

    private void saveDetail(String runId, String txId, String issue, String detailsJson) {
        try {
            ReconciliationDetail detail = new ReconciliationDetail();
            detail.setRunId(runId);
            detail.setTransactionId(txId);
            detail.setIssue(issue);
            detail.setDetails(detailsJson);
            detailRepo.save(detail);
        } catch (Exception e) {
            log.error("Failed to save detail for transaction {}: {}", txId, e.getMessage(), e);
        }
    }
}
```

---

### f. ReconciliationController.java

**ReconciliationController.java**
```java
package com.example.reconciliation.controller;

import com.example.reconciliation.service.ReconciliationService;
import lombok.extern.slf4j.Slf4j;
import org.springframework.web.bind.annotation.*;

@Slf4j
@RestController
@RequestMapping("/api")
public class ReconciliationController {

    private final ReconciliationService reconciliationService;

    public ReconciliationController(ReconciliationService reconciliationService) {
        this.reconciliationService = reconciliationService;
    }

    @PostMapping("/reconcile/{reconciliationType}")
    public String reconcile(@PathVariable String reconciliationType) {
        try {
            log.info("Received reconciliation request for type: {}", reconciliationType);
            reconciliationService.runReconciliation(reconciliationType);
            return "Reconciliation run completed successfully for type: " + reconciliationType;
        } catch (Exception e) {
            log.error("Error during reconciliation: {}", e.getMessage(), e);
            return "Error during reconciliation: " + e.getMessage();
        }
    }
}
```

---

## 4. Running the Application

1. **Configure your database** using the `application.properties` (or `application.yml`) file:
   ```properties
   spring.datasource.url=jdbc:mysql://localhost:3306/your_database
   spring.datasource.username=your_username
   spring.datasource.password=your_password
   spring.jpa.hibernate.ddl-auto=update
   spring.jpa.show-sql=true
   logging.level.root=INFO
   logging.level.com.example.reconciliation=DEBUG
   ```
2. **Run the application** (e.g., using `mvn spring-boot:run`).
3. **Trigger a reconciliation run** by sending a POST request (using curl or Postman):
   ```bash
   curl -X POST http://localhost:8080/api/reconcile/VAM_BAX
   ```

---

## Summary

This complete example uses Lombok to reduce boilerplate code for getters/setters, SLF4J for logging, and good design patterns (strategy, factory, and aggregation) to build a production‐ready reconciliation framework. Adjust the batch size, logging levels, and error handling as needed for your production environment.



## DB data

-- Generate Comparison Configurations for reconciliation type "VAM_BAX"
INSERT INTO comparison_config (reconciliation_type, field_name, source1_field, source2_field, comparison_method, custom_comparison_logic)
VALUES ('VAM_BAX', 'account_name', 'account_name', 'account_name', 'SUBSTRING', '{"start":0,"end":20}');

INSERT INTO comparison_config (reconciliation_type, field_name, source1_field, source2_field, comparison_method, custom_comparison_logic)
VALUES ('VAM_BAX', 'balance', 'balance', 'balance', 'EQUALS', NULL);

-- Generate 1,000,000 records for SOURCE1
INSERT INTO source_data (reconciliation_type, source_type, transaction_id, client_id, client_name, data)
SELECT 
    'VAM_BAX' AS reconciliation_type,
    'SOURCE1' AS source_type,
    'TX' || gs AS transaction_id,
    'client' || ((gs % 100) + 1) AS client_id,  -- 100 distinct clients
    'Client ' || ((gs % 100) + 1) AS client_name,
    json_build_object(
         'account_id', gs,
         'account_name', 'Account ' || gs,
         'balance', round(random() * 10000::numeric, 2),
         'status', CASE WHEN random() > 0.9 THEN 'inactive' ELSE 'active' END
    ) AS data
FROM generate_series(1, 1000000) gs;

-- Generate SOURCE2 records:
-- Approximately 95% of the records (simulate missing ~5%),
-- and simulate mismatches by altering the account_name and balance for some rows.
INSERT INTO source_data (reconciliation_type, source_type, transaction_id, client_id, client_name, data)
SELECT 
    'VAM_BAX' AS reconciliation_type,
    'SOURCE2' AS source_type,
    'TX' || gs AS transaction_id,
    'client' || ((gs % 100) + 1) AS client_id,
    'Client ' || ((gs % 100) + 1) AS client_name,
    json_build_object(
         'account_id', gs,
         'account_name', 
              CASE WHEN random() > 0.85 
                   THEN 'Account ' || gs || '_mismatch' 
                   ELSE 'Account ' || gs 
              END,
         'balance', 
              CASE WHEN random() > 0.80 
                   THEN round(random() * 10000::numeric, 2) 
                   ELSE round(random() * 10000 + 50, 2) 
              END,
         'status', CASE WHEN random() > 0.95 THEN 'inactive' ELSE 'active' END
    ) AS data
FROM generate_series(1, 1000000) gs
WHERE random() > 0.05;
