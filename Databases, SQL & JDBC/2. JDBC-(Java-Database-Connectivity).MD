# JDBC & Connection Pooling (HikariCP)

A comprehensive guide to low-level database persistence in Java: `java.sql` architecture, `PreparedStatement` execution, `ResultSet` object mapping, and high-performance connection pool configuration using HikariCP.

---

## 1. JDBC Architecture: Drivers, Connections & Statements

Java Database Connectivity (JDBC) is the baseline abstraction API that allows Java applications to communicate with relational database engines over TCP/IP sockets [cite: 1, 3].

```
+-------------------------------------------------------------+
| Java Application Code                                       |
|  [ Connection, PreparedStatement, ResultSet ]               |
+-------------------------------------------------------------+
                               |
                               v
+-------------------------------------------------------------+
| JDBC API (java.sql / javax.sql)                             |
|  [ DriverManager / DataSource ]                             |
+-------------------------------------------------------------+
                               |
                               v
+-------------------------------------------------------------+
| JDBC Driver Implementation (e.g., org.postgresql.Driver)   |
|  [ Translates JDBC calls into proprietary database wire protocol ]
+-------------------------------------------------------------+
                               |
                               v
                     Database Engine (PostgreSQL / MySQL)
```

---

## 2. `Statement` vs. `PreparedStatement` & SQL Injection Prevention

* **`Statement` (Unsafe for dynamic user input):** The SQL query string is sent to the database and compiled at runtime. Concatenating parameters into raw strings creates direct SQL injection vulnerabilities.
* **`PreparedStatement` (Production Standard):** The SQL statement template is pre-compiled by the database engine with parameter placeholders (`?`). Values are bound separately via binary network parameters, making SQL injection structurally impossible.

```java
package com.roadmap.jdbc;

import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;

public class PreparedStatementDemo {

    // SAFE: Parameters are bound via pre-compiled binary placeholders
    public static void findCustomerOrders(Connection conn, String customerEmail) throws SQLException {
        String sql = """
                SELECT id, order_number, total_amount, status
                FROM orders
                WHERE customer_email = ? AND status != ?
                """;

        try (PreparedStatement pstmt = conn.prepareStatement(sql)) {
            pstmt.setString(1, customerEmail); // 1-based index parameter binding
            pstmt.setString(2, "CANCELLED");

            try (ResultSet rs = pstmt.executeQuery()) {
                while (rs.next()) {
                    long id = rs.getLong("id");
                    String orderNumber = rs.getString("order_number");
                    double total = rs.getDouble("total_amount");
                    String status = rs.getString("status");

                    System.out.printf("Order #%s (ID: %d): $%.2f [%s]%n", 
                            orderNumber, id, total, status);
                }
            }
        }
    }
}
```

---

## 3. `ResultSet` Mapping & Handling Nullable Values

A `ResultSet` maintains a database cursor pointing to one row of query results.

> **Gotcha Alert:** In JDBC, primitive getters like `rs.getInt("column")` or `rs.getLong("column")` return `0` if the database cell holds `NULL`. To check if a database column was actually null, invoke `rs.wasNull()` immediately after reading the field.

```java
package com.roadmap.jdbc;

import java.math.BigDecimal;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.time.Instant;

public class ResultSetMapper {

    public record OrderSummary(
            Long id,
            String orderNumber,
            BigDecimal amount,
            Integer discountPercentage, // Nullable column
            Instant createdAt
    ) {}

    public static OrderSummary mapRowToOrder(ResultSet rs) throws SQLException {
        long id = rs.getLong("id");
        String orderNumber = rs.getString("order_number");
        BigDecimal amount = rs.getBigDecimal("total_amount");

        // Safe nullable handling for integers
        int rawDiscount = rs.getInt("discount_percentage");
        Integer discount = rs.wasNull() ? null : rawDiscount;

        Instant createdAt = rs.getTimestamp("created_at").toInstant();

        return new OrderSummary(id, orderNumber, amount, discount, createdAt);
    }
}
```

---

## 4. Connection Pooling Mechanics: HikariCP

Creating a new physical TCP socket connection to a database per request requires TLS negotiation, authentication handshakes, and process allocation—taking 20ms to 100ms per call.

A **Connection Pool** pre-allocates a pool of live, persistent connections, renting them to threads during operations and recycling them immediately upon `close()`.

```
Incoming Request Thread
         |
         | Rent Connection (.getConnection())
         v
+-------------------------------------------------------------+
| HikariCP Connection Pool (Default in Spring Boot)           |
|                                                             |
|   [ Conn-1: Active ]       [ Conn-2: Idle ]                 |
|   [ Conn-3: Idle ]         [ Conn-4: Active ]               |
+-------------------------------------------------------------+
         |
         | Return Connection (conn.close() -> returns to pool)
         v
Worker Thread Continues
```

### HikariCP Setup Example

```java
package com.roadmap.jdbc;

import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;

import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;

public class HikariPoolManager {

    private static HikariDataSource dataSource;

    public static DataSource getDataSource() {
        if (dataSource == null) {
            HikariConfig config = new HikariConfig();
            config.setJdbcUrl("jdbc:postgresql://localhost:5432/ecommerce_db");
            config.setUsername("app_user");
            config.setPassword("secure_password");

            // Pool Sizing & Health Configurations
            config.setMaximumPoolSize(10); // Standard sizing: (CPU Cores * 2) + Disk Count
            config.setMinimumIdle(5);
            config.setIdleTimeout(300000); // 5 minutes
            config.setConnectionTimeout(20000); // 20 seconds wait before failure
            config.setMaxLifetime(1800000); // 30 minutes (recycles sockets before OS cutoff)
            config.setPoolName("OrderService-HikariPool");

            dataSource = new HikariDataSource(config);
        }
        return dataSource;
    }

    public static void executeFastQuery() {
        DataSource ds = getDataSource();

        // Calling close() on pooled Connection releases it back to HikariCP (does NOT disconnect)
        try (Connection conn = ds.getConnection();
             PreparedStatement pstmt = conn.prepareStatement("SELECT 1");
             ResultSet rs = pstmt.executeQuery()) {

            if (rs.next()) {
                System.out.println("Database heartbeat: " + rs.getInt(1));
            }
        } catch (SQLException e) {
            System.err.println("Database query failed: " + e.getMessage());
        }
    }
}
```

---

## 5. Key Takeaways for Magento / PHP Developers

1. **Persistent Connection Pools vs. PHP-FPM:** In PHP, each request boots independently; connection pooling requires external proxy software like **PgBouncer** or **ProxySQL**. In Spring Boot, HikariCP runs **in-process** inside the JVM, maintaining warm, active connection sockets continuously [cite: 1, 3].
2. **`close()` Semantics:** Calling `$connection = null` in PHP triggers garbage collection and socket teardown. In Java with HikariCP, `connection.close()` calls a proxy wrapper method that resets transaction state and returns the physical TCP connection to the pool for reuse [cite: 1, 3].
