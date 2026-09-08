# Spring Boot Core Features & Practical Examples

A hands-on, code-first guide covering the primary features that define **Spring Boot**, complete with architecture explanations and real-world code examples.

---

## Table of Contents
1. [Auto-Configuration (`@EnableAutoConfiguration`)](#1-auto-configuration)
2. [Starter Dependencies (Opinionated POMs)](#2-starter-dependencies)
3. [Embedded Web Servers (No External WAR Deployments)](#3-embedded-web-servers)
4. [Externalized Configuration & Type-Safe Properties](#4-externalized-configuration--type-safe-properties)
5. [Spring Data JPA Repositories (Derived & Custom Queries)](#5-spring-data-jpa-repositories)
6. [Declarative REST APIs & Global Validation](#6-declarative-rest-apis--global-validation)
7. [Centralized Exception Handling (`@RestControllerAdvice`)](#7-centralized-exception-handling)
8. [Production-Ready Metrics & Health (`Spring Boot Actuator`)](#8-production-ready-metrics--health-actuator)
9. [Scheduled Tasks & Async Execution (`@Scheduled`, `@Async`)](#9-scheduled-tasks--async-execution)
10. [Event-Driven Architecture (`ApplicationEventPublisher`)](#10-event-driven-architecture)

---

## 1. Auto-Configuration

### Concept
Spring Boot detects libraries on your classpath (like HikariCP, Hibernate, or Jackson) and automatically wires up default beans so you don't write hundreds of lines of XML or Java `@Bean` declarations.

If you declare your own bean, Spring Boot's conditional bean (`@ConditionalOnMissingBean`) backs off automatically.

### Example: Customizing Jackson Object Mapper

```java
package com.example.demo.config;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;

@Configuration
public class JacksonConfig {

    // Overriding the default Spring Boot Jackson ObjectMapper
    @Bean
    @Primary
    public ObjectMapper objectMapper() {
        return new ObjectMapper()
                .registerModule(new JavaTimeModule())
                .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
    }
}

```

---

## 2. Starter Dependencies

### Concept
Instead of managing dozens of individual, mismatched library versions, Spring Boot provides curated starters (`spring-boot-starter-*`). Starters aggregate tested, compatible dependencies into a single package.

### Example: Minimal Maven `pom.xml`

```xml
<dependencies>
    <!-- Web: Embedded Tomcat, Spring MVC, REST support, Jackson JSON -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Persistence: Hibernate, Spring Data JPA, HikariCP Connection Pool -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <!-- Declarative Validation: Jakarta Validation API & Hibernate Validator -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>

    <!-- PostgreSQL Driver (version is auto-managed by parent POM) -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>

```

---

## 3. Embedded Web Servers

### Concept
Traditional Java applications were packaged as `.war` files and deployed into separate containers (like standalone Tomcat, WildFly, or JBoss). Spring Boot embeds the server directly inside the `.jar` file. Applications start via a standard `public static void main` method.

### Example: Switching from Tomcat to Undertow & Port Config

```xml
<!-- In pom.xml: Exclude Tomcat and include Undertow -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>
</dependency>

```

```yaml
# In application.yml
server:
  port: 8081
  servlet:
    context-path: /api/v1
  compression:
    enabled: true
    mime-types: application/json,text/html

```

---

## 4. Externalized Configuration & Type-Safe Properties

### Concept
Spring Boot decouples environment variables and property files from application logic. Using `@ConfigurationProperties`, properties are converted directly into strongly-typed Java objects.

### Example: Payment Gateway Configuration

```yaml
# application.yml
app:
  payment:
    provider: stripe
    api-key: "sk_test_51Mz..."
    timeout-ms: 5000
    retry:
      max-attempts: 3
      backoff-multiplier: 1.5

```

```java
package com.example.demo.config;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Positive;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.validation.annotation.Validated;

@ConfigurationProperties(prefix = "app.payment")
@Validated
public record PaymentProperties(
        @NotBlank String provider,
        @NotBlank String apiKey,
        @Positive int timeoutMs,
        Retry retry
) {
    public record Retry(
            @Positive int maxAttempts,
            @Positive double backoffMultiplier
    ) {}
}

```

```java
// Injecting and using the validated properties
@Service
public class StripePaymentService {
    private final PaymentProperties paymentProperties;

    public StripePaymentService(PaymentProperties paymentProperties) {
        this.paymentProperties = paymentProperties;
    }

    public void processPayment() {
        System.out.println("Connecting to: " + paymentProperties.provider());
        System.out.println("Max retries: " + paymentProperties.retry().maxAttempts());
    }
}

```

---

## 5. Spring Data JPA Repositories

### Concept
Spring Data eliminates boilerplate data access objects. By defining an interface that extends `JpaRepository`, Spring creates queries automatically at runtime from your method names.

### Example: Entity and Derived Repository

```java
package com.example.demo.order;

import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.Instant;

@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String orderNumber;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private OrderStatus status;

    @Column(nullable = false)
    private BigDecimal totalAmount;

    @Column(nullable = false, updatable = false)
    private Instant createdAt = Instant.now();

    // Getters and Setters omitted for brevity
}

```

```java
package com.example.demo.order;

import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.math.BigDecimal;
import java.util.Optional;

@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {

    // Derived Query: Spring generates SQL automatically
    Optional<Order> findByOrderNumber(String orderNumber);

    // Filter by status with pagination
    Page<Order> findByStatus(OrderStatus status, Pageable pageable);

    // Custom JPQL Query
    @Query("SELECT o FROM Order o WHERE o.totalAmount >= :minTotal AND o.status = 'COMPLETED'")
    Page<Order> findHighValueCompletedOrders(@Param("minTotal") BigDecimal minTotal, Pageable pageable);
}

```

---

## 6. Declarative REST APIs & Global Validation

### Concept
Build REST endpoints with HTTP mapping annotations and automatically validate incoming request bodies via Jakarta Bean Validation annotations.

### Example: Product DTO & REST Controller

```java
package com.example.demo.catalog;

import jakarta.validation.constraints.*;
import java.math.BigDecimal;

public record CreateProductRequest(
        @NotBlank(message = "Product title must not be blank")
        @Size(min = 3, max = 120, message = "Title must be between 3 and 120 chars")
        String title,

        @NotBlank(message = "SKU is required")
        @Pattern(regexp = "^[A-Z0-9_-]{4,20}$", message = "SKU must be uppercase alphanumeric")
        String sku,

        @NotNull(message = "Price is mandatory")
        @DecimalMin(value = "0.01", message = "Price must be greater than zero")
        BigDecimal price,

        @Min(value = 0, message = "Stock cannot be negative")
        int stockQuantity
) {}

```

```java
package com.example.demo.catalog;

import jakarta.validation.Valid;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/products")
public class ProductController {

    private final ProductService productService;

    public ProductController(ProductService productService) {
        this.productService = productService;
    }

    @PostMapping
    public ResponseEntity<ProductResponse> createProduct(@Valid @RequestBody CreateProductRequest request) {
        ProductResponse response = productService.createProduct(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }

    @GetMapping("/{id}")
    public ResponseEntity<ProductResponse> getProductById(@PathVariable Long id) {
        return ResponseEntity.ok(productService.getProductById(id));
    }
}

```

---

## 7. Centralized Exception Handling

### Concept
Instead of wrapping individual controller endpoints in repetitive `try-catch` blocks, use `@RestControllerAdvice` to translate domain exceptions into uniform, client-friendly HTTP error responses.

### Example: Global Error Handler with Problem Details

```java
package com.example.demo.exception;

import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.net.URI;
import java.time.Instant;
import java.util.HashMap;
import java.util.Map;

@RestControllerAdvice
public class GlobalExceptionHandler {

    // Handles validation failures (e.g. invalid @RequestBody)
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail handleValidationException(MethodArgumentNotValidException ex) {
        ProblemDetail problem = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        problem.setTitle("Validation Failed");
        problem.setType(URI.create("https://api.example.com/errors/validation-failed"));

        Map<String, String> errors = new HashMap<>();
        for (FieldError fieldError : ex.getBindingResult().getFieldErrors()) {
            errors.put(fieldError.getField(), fieldError.getDefaultMessage());
        }

        problem.setProperty("invalidFields", errors);
        problem.setProperty("timestamp", Instant.now());
        return problem;
    }

    // Handles resource-not-found exceptions
    @ExceptionHandler(ResourceNotFoundException.class)
    public ProblemDetail handleNotFound(ResourceNotFoundException ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
        problem.setTitle("Resource Not Found");
        problem.setProperty("timestamp", Instant.now());
        return problem;
    }
}

```

---

## 8. Production-Ready Metrics & Health (Actuator)

### Concept
`spring-boot-starter-actuator` exposes operational endpoints for monitoring, uptime probes (Kubernetes liveness and readiness), thread dumps, and application metrics without manual setup.

### Example: Actuator Configuration & Custom Health Indicator

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: always
      probes:
        enabled: true

```

```java
package com.example.demo.health;

import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;

@Component
public class WarehouseServiceHealthIndicator implements HealthIndicator {

    @Override
    public Health health() {
        boolean warehouseApiReachable = checkWarehouseConnection();

        if (warehouseApiReachable) {
            return Health.up()
                    .withDetail("warehouseService", "Available")
                    .withDetail("latencyMs", 42)
                    .build();
        }

        return Health.down()
                .withDetail("warehouseService", "Unreachable")
                .withDetail("error", "HTTP 503 Gateway Timeout")
                .build();
    }

    private boolean checkWarehouseConnection() {
        // Ping service logic
        return true;
    }
}

```

---

## 9. Scheduled Tasks & Async Execution

### Concept
Spring Boot provides native annotations for background tasks and asynchronous operations without requiring heavy third-party task runners for routine jobs.

### Example: Scheduled Cron Job and Asynchronous Worker

```java
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.annotation.EnableScheduling;

@SpringBootApplication
@EnableScheduling
@EnableAsync
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

```

```java
package com.example.demo.task;

import org.springframework.scheduling.annotation.Async;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;

import java.time.Instant;
import java.util.concurrent.CompletableFuture;

@Service
public class TaskService {

    // Runs every day at 02:00 AM server time
    @Scheduled(cron = "0 0 2 * * ?")
    public void runDailyReportCleanup() {
        System.out.println("Running daily cleanup at: " + Instant.now());
    }

    // Offloads work to a separate worker thread pool
    @Async
    public CompletableFuture<Void> sendOrderConfirmationEmail(String customerEmail, Long orderId) {
        // Simulating third-party email API call
        System.out.println("Sending email to " + customerEmail + " on thread: " + Thread.currentThread().getName());
        return CompletableFuture.completedFuture(null);
    }
}

```

---

## 10. Event-Driven Architecture

### Concept
Spring's internal event bus allows domain events to be published and consumed within the same JVM process, keeping business logic clean and decoupled.

### Example: Publishing & Handling Order Events

```java
// Event Record
public record OrderPlacedEvent(Long orderId, String customerEmail, Instant occurredAt) {}

```

```java
// Event Publisher (inside your order business logic)
@Service
public class OrderService {

    private final ApplicationEventPublisher eventPublisher;

    public OrderService(ApplicationEventPublisher eventPublisher) {
        this.eventPublisher = eventPublisher;
    }

    @Transactional
    public void completeCheckout(Long orderId, String email) {
        // Persist order...
        System.out.println("Order persisted. Publishing domain event...");
        
        eventPublisher.publishEvent(new OrderPlacedEvent(orderId, email, Instant.now()));
    }
}

```

```java
// Event Listeners (Independent components handling side effects)
@Component
public class NotificationEventListener {

    // Runs asynchronously so checkout is not blocked
    @Async
    @EventListener
    public void onOrderPlaced(OrderPlacedEvent event) {
        System.out.println("Received order placed event for: " + event.orderId());
        // Trigger customer email, push notifications, etc.
    }
}

```
=====================================================
Summary of Features Covered with Examples:
Auto-Configuration (@EnableAutoConfiguration): How Spring Boot's conditional beans work, plus overriding a default bean (e.g., custom Jackson ObjectMapper).

Starter Dependencies: Structure of a clean pom.xml using opinionated starters (web, data-jpa, validation).

Embedded Web Servers: Replacing default Tomcat with Undertow, plus context-path and compression configuration.

Externalized Configuration: Binding YAML values to a validated Java record using @ConfigurationProperties.

Spring Data JPA Repositories: Entities, derived query generation, and custom @Query (JPQL) with pagination.

Declarative REST APIs & Validation: Building REST controllers using DTO records and Jakarta validation annotations (@NotBlank, @DecimalMin, etc.).

Centralized Exception Handling: Global exception handler (@RestControllerAdvice) producing RFC 7807 ProblemDetail responses.

Spring Boot Actuator: Configuring management endpoints and implementing a custom HealthIndicator.

Scheduled Tasks & Async Execution: Enabling and using @Scheduled (cron jobs) and @Async (thread-pool offloading).

In-Process Event-Driven Architecture: Decoupling workflows using ApplicationEventPublisher and @EventListener.
