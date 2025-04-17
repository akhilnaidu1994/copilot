```Java

/* =====================================================================
 * Oracle MCP Server – COMPLETE, RUNNABLE PROJECT
 * ---------------------------------------------------------------------
 * Build  : Maven 3.9+  |  Java 21  |  Spring Boot 3.3.x
 * Purpose: Expose Oracle DB metadata & sample data to any MCP‑capable LLM
 * ===================================================================== */

// ----------------------------------------------------------------------
// pom.xml  (place in the project root)
// ----------------------------------------------------------------------
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>dev.oramcp</groupId>
    <artifactId>oracle-mcp-server</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <properties>
        <java.version>21</java.version>
        <spring.boot.version>3.3.0</spring.boot.version>
        <spring.ai.version>1.0.0-M2</spring.ai.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring.boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <!-- Spring Boot core & JDBC -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jdbc</artifactId>
        </dependency>

        <!-- Spring AI MCP Server starter -->
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-mcp-server-boot-starter</artifactId>
            <version>${spring.ai.version}</version>
        </dependency>

        <!-- Oracle JDBC driver  ➜  copy ojdbc11.jar into lib/  (license) -->
        <dependency>
            <groupId>com.oracle.database.jdbc</groupId>
            <artifactId>ojdbc11</artifactId>
            <version>23.4.0.0</version>
            <scope>system</scope>
            <systemPath>${project.basedir}/lib/ojdbc11.jar</systemPath>
        </dependency>

        <!-- Optional: Lombok for getters/equals (remove if unwanted) -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <release>${java.version}</release>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>

// ----------------------------------------------------------------------
// src/main/resources/application.yml
// ----------------------------------------------------------------------
spring:
  datasource:
    url: jdbc:oracle:thin:@//localhost:1521/ORCLCDB
    username: ${ORACLE_USER}
    password: ${ORACLE_PASS}
    driver-class-name: oracle.jdbc.OracleDriver
  ai:
    mcp:
      server:
        path: /mcp     # endpoint ➜ http://host:8080/mcp
        transport: http

server:
  port: 8080

logging:
  level:
    dev.oramcp: INFO

/* =====================================================================
 * Java Sources (src/main/java)
 * ===================================================================== */

// dev/oramcp/OracleMcpServerApplication.java
package dev.oramcp;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class OracleMcpServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(OracleMcpServerApplication.class, args);
    }
}

// ----------------------------------------------------------------------
// DTOs (records)  ➜  dev/oramcp/dto/*
// ----------------------------------------------------------------------
package dev.oramcp.dto;

public record ColumnInfo(
        String name,
        String dataType,
        Integer dataLength,
        Integer dataPrecision,
        Integer dataScale,
        boolean nullable,
        String defaultValue,
        String comment
) {}

package dev.oramcp.dto;

public record ConstraintInfo(
        String name,
        String type,
        String referencedTable,
        String referencedColumn
) {}

package dev.oramcp.dto;

import java.util.List;

public record TableInfo(
        String owner,
        String name,
        String comment,
        List<ColumnInfo> columns,
        List<ConstraintInfo> constraints,
        long rowCount
) {}

// ----------------------------------------------------------------------
// Service layer ➜ dev/oramcp/service/OracleMetadataService.java
// ----------------------------------------------------------------------
package dev.oramcp.service;

import dev.oramcp.dto.*;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.concurrent.*;

@Service
public class OracleMetadataService {
    private final JdbcTemplate jdbc;
    private final ExecutorService exec = Executors.newVirtualThreadPerTaskExecutor();

    public OracleMetadataService(JdbcTemplate jdbc) { this.jdbc = jdbc; }

    /* -----------------------  TOP‑LEVEL APIS  ----------------------- */
    public List<String> listSchemas() {
        return jdbc.queryForList("SELECT username FROM all_users ORDER BY username", String.class);
    }

    public List<String> listTables(String schema) {
        return jdbc.queryForList(
            "SELECT table_name FROM all_tables WHERE owner = ? ORDER BY table_name",
            String.class, schema.toUpperCase());
    }

    public TableInfo tableInfo(String schema, String table) {
        CompletableFuture<List<ColumnInfo>> colsF =
                CompletableFuture.supplyAsync(() -> columns(schema, table), exec);
        CompletableFuture<List<ConstraintInfo>> consF =
                CompletableFuture.supplyAsync(() -> constraints(schema, table), exec);
        CompletableFuture<Long> rowsF =
                CompletableFuture.supplyAsync(() -> rowCount(schema, table), exec);
        CompletableFuture<String> commentF =
                CompletableFuture.supplyAsync(() -> tableComment(schema, table), exec);

        return new TableInfo(
                schema.toUpperCase(),
                table.toUpperCase(),
                commentF.join(),
                colsF.join(),
                consF.join(),
                rowsF.join());
    }

    public List<java.util.Map<String,Object>> sampleRows(String schema, String table, int limit) {
        return jdbc.queryForList("SELECT * FROM " + schema + "." + table + " FETCH FIRST " + limit + " ROWS ONLY");
    }

    /* -----------------------  INTERNAL QUERIES  ----------------------- */
    private List<ColumnInfo> columns(String schema, String table) {
        return jdbc.query("""
            SELECT col.column_name,
                   col.data_type,
                   col.data_length,
                   col.data_precision,
                   col.data_scale,
                   col.nullable,
                   col.data_default,
                   comm.comments
            FROM all_tab_columns col
            LEFT JOIN all_col_comments comm
                   ON col.owner = comm.owner
                  AND col.table_name = comm.table_name
                  AND col.column_name = comm.column_name
            WHERE col.owner = ?
              AND col.table_name = ?
            ORDER BY col.column_id
        """, (rs, i) -> new ColumnInfo(
                rs.getString("column_name"),
                rs.getString("data_type"),
                (Integer) rs.getObject("data_length"),
                (Integer) rs.getObject("data_precision"),
                (Integer) rs.getObject("data_scale"),
                "Y".equals(rs.getString("nullable")),
                rs.getString("data_default"),
                rs.getString("comments")
        ), schema.toUpperCase(), table.toUpperCase());
    }

    private List<ConstraintInfo> constraints(String schema, String table) {
        return jdbc.query("""
            SELECT ac.constraint_name,
                   ac.constraint_type,
                   r.table_name   AS ref_table,
                   acc.column_name AS ref_column
            FROM all_constraints ac
            JOIN all_cons_columns acc
              ON ac.owner = acc.owner AND ac.constraint_name = acc.constraint_name
            LEFT JOIN all_constraints r
              ON ac.r_owner = r.owner AND ac.r_constraint_name = r.constraint_name
            WHERE ac.owner = ? AND ac.table_name = ?
        """, (rs, i) -> new ConstraintInfo(
                rs.getString("constraint_name"),
                rs.getString("constraint_type"),
                rs.getString("ref_table"),
                rs.getString("ref_column")
        ), schema.toUpperCase(), table.toUpperCase());
    }

    private long rowCount(String schema, String table) {
        Long estimate = jdbc.queryForObject(
            "SELECT num_rows FROM all_tables WHERE owner = ? AND table_name = ?",
            Long.class, schema.toUpperCase(), table.toUpperCase());
        if (estimate != null && estimate > 0) return estimate;
        return jdbc.queryForObject("SELECT COUNT(*) FROM " + schema + "." + table, Long.class);
    }

    private String tableComment(String schema, String table) {
        return jdbc.queryForObject(
            "SELECT comments FROM all_tab_comments WHERE owner = ? AND table_name = ?",
            String.class, schema.toUpperCase(), table.toUpperCase());
    }
}

// ----------------------------------------------------------------------
// MCP Tools  ➜  dev/oramcp/tools/*
// ----------------------------------------------------------------------
package dev.oramcp.tools;

import dev.oramcp.service.OracleMetadataService;
import org.springframework.ai.mcp.tools.McpTool;
import org.springframework.ai.mcp.tools.McpToolMethod;
import java.util.List;

@McpTool(name = "listSchemas", description = "Returns all Oracle schemas accessible to the user")
public class ListSchemasTool {
    private final OracleMetadataService svc;
    public ListSchemasTool(OracleMetadataService svc) { this.svc = svc; }
    @McpToolMethod public List<String> invoke() { return svc.listSchemas(); }
}

package dev.oramcp.tools;

import dev.oramcp.service.OracleMetadataService;
import org.springframework.ai.mcp.tools.McpTool;
import org.springframework.ai.mcp.tools.McpToolMethod;
import java.util.List;

@McpTool(name = "listTables", description = "Return all tables for the given schema")
public class ListTablesTool {
    private final OracleMetadataService svc;
    public ListTablesTool(OracleMetadataService svc) { this.svc = svc; }
    @McpToolMethod public List<String> invoke(String schema) { return svc.listTables(schema); }
}

package dev.oramcp.tools;

import dev.oramcp.dto.TableInfo;
import dev.oramcp.service.OracleMetadataService;
import org.springframework.ai.mcp.tools.McpTool;
import org.springframework.ai.mcp.tools.McpToolMethod;

@McpTool(name = "tableInfo", description = "Detailed metadata (columns, constraints, row count) for a table")
public class TableInfoTool {
    private final OracleMetadataService svc;
    public TableInfoTool(OracleMetadataService svc) { this.svc = svc; }
    @McpToolMethod public TableInfo invoke(String schema, String table) { return svc.tableInfo(schema, table); }
}

package dev.oramcp.tools;

import dev.oramcp.service.OracleMetadataService;
import org.springframework.ai.mcp.tools.McpTool;
import org.springframework.ai.mcp.tools.McpToolMethod;
import java.util.List;
import java.util.Map;

@McpTool(name = "sampleRows", description = "Return up to 'limit' sample rows from the specified table")
public class SampleRowsTool {
    private final OracleMetadataService svc;
    public SampleRowsTool(OracleMetadataService svc) { this.svc = svc; }
    @McpToolMethod public List<Map<String,Object>> invoke(String schema, String table, int limit) {
        return svc.sampleRows(schema, table, limit);
    }
}

package dev.oramcp.tools;

import dev.oramcp.dto.TableInfo;
import dev.oramcp.service.OracleMetadataService;
import org.springframework.ai.mcp.tools.McpTool;
import org.springframework.ai.mcp.tools.McpToolMethod;
import java.util.*;

@McpTool(name = "databaseContext", description = "Return full <schema,<table,TableInfo>> map for selected schemas")
public class DatabaseContextTool {
    private final OracleMetadataService svc;
    public DatabaseContextTool(OracleMetadataService svc) { this.svc = svc; }

    @McpToolMethod
    public Map<String, Map<String, TableInfo>> invoke(List<String> includeSchemas) {
        Map<String, Map<String, TableInfo>> ctx = new LinkedHashMap<>();
        List<String> schemas = (includeSchemas == null || includeSchemas.isEmpty()) ? svc.listSchemas() : includeSchemas;
        for (String schema : schemas) {
            Map<String, TableInfo> tables = new LinkedHashMap<>();
            for (String tbl : svc.listTables(schema)) {
                tables.put(tbl, svc.tableInfo(schema, tbl));
            }
            ctx.put(schema, tables);
        }
        return ctx;
    }
}

/* ---------------------------------------------------------------------
 * HOW TO RUN
 * ---------------------------------------------------------------------
 * 1) Place your Oracle JDBC jar in ./lib/ojdbc11.jar  (or edit pom.xml)
 * 2) Export creds & start:
 *      export ORACLE_USER=hr
 *      export ORACLE_PASS=hrpwd
 *      mvn spring-boot:run
 * 3) The MCP endpoint is now http://localhost:8080/mcp
 * 4) In ChatGPT ▸ "Add MCP server" ▸ paste the URL and try:
 *      - listSchemas
 *      - listTables {"schema":"HR"}
 *      - tableInfo {"schema":"HR","table":"EMPLOYEES"}
 *
 * Enjoy!  ─────────────────────────────────────────────────────────── */

```
