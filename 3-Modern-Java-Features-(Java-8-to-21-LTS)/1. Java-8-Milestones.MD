# Java 8 Milestones: Lambdas, Functional Interfaces, Streams, and Optional

A comprehensive deep dive into functional programming paradigms introduced in Java 8 that form the backbone of modern Spring Boot business logic and data manipulation pipelines.

---

## 1. Lambda Expressions & Syntax

A lambda expression is an anonymous function (a block of code without a name, return type, or explicit class) that can be passed around as an object.

### Syntax Variations

```java
// 1. Full syntax: Explicit parameter types and block body
(String s) -> { return s.trim().toUpperCase(); }

// 2. Type inference: Parameter type inferred by compiler
(s) -> { return s.trim().toUpperCase(); }

// 3. Single parameter & single expression: Parentheses and braces omitted
s -> s.trim().toUpperCase()

// 4. Multiple parameters
(a, b) -> a.compareTo(b)

// 5. Zero parameters
() -> System.out.println("Processing background tick...")
```

---

## 2. Core Functional Interfaces (`java.util.function`)

A functional interface has exactly **one abstract method** (SAM - Single Abstract Method) and is marked with `@FunctionalInterface`.

| Interface | Method Signature | Use Case Example |
| :--- | :--- | :--- |
| `Predicate<T>` | `boolean test(T t)` | Filtering collections, checking order state |
| `Function<T, R>` | `R apply(T t)` | Transforming DTOs to Entities, mapping types |
| `Consumer<T>` | `void accept(T t)` | Logging, dispatching webhooks, emitting events |
| `Supplier<T>` | `T get()` | Lazy computation, fallback value factories |
| `BiFunction<T, U, R>` | `R apply(T t, U u)` | Merging two domain objects into a single result |

```java
package com.roadmap.java8;

import java.math.BigDecimal;
import java.util.function.Consumer;
import java.util.function.Function;
import java.util.function.Predicate;
import java.util.function.Supplier;

public class FunctionalInterfacesDemo {

    public record Order(String id, BigDecimal total, boolean isVip) {}

    public static void main(String[] args) {
        // 1. Predicate: Is order high value?
        Predicate<Order> isHighValue = o -> o.total().compareTo(new BigDecimal("100.00")) > 0;

        // 2. Function: Extract order ID
        Function<Order, String> idExtractor = Order::id;

        // 3. Consumer: Audit logger
        Consumer<Order> auditLogger = o -> System.out.println("Auditing order: " + o.id());

        // 4. Supplier: Fallback order generator
        Supplier<Order> guestOrderFactory = () -> new Order("GUEST-001", BigDecimal.ZERO, false);

        Order currentOrder = new Order("ORD-8812", new BigDecimal("150.00"), true);

        if (isHighValue.test(currentOrder)) {
            auditLogger.accept(currentOrder);
            System.out.println("Extracted ID: " + idExtractor.apply(currentOrder));
        }
    }
}
```

---

## 3. Method References (`::`)

Method references provide shorthand syntax for lambdas that only forward arguments directly to an existing method.

* **Static method:** `ClassName::staticMethod` $	o$ `(s) -> Integer.parseInt(s)` becomes `Integer::parseInt`
* **Instance method of an arbitrary object:** `ClassName::instanceMethod` $	o$ `(s) -> s.toLowerCase()` becomes `String::toLowerCase`
* **Instance method of a specific object:** `instance::instanceMethod` $	o$ `(x) -> logger.info(x)` becomes `logger::info`
* **Constructor reference:** `ClassName::new` $	o$ `() -> new ArrayList<>()` becomes `ArrayList::new`

---

## 4. The Streams API: Declarative Data Processing

Streams process data declaratively, mimicking database query operations (`SELECT`, `WHERE`, `GROUP BY`) in memory without mutating source collections.

### Anatomy of a Stream Pipeline
$$	ext{Source} \longrightarrow 	ext{Intermediate Operations (Lazy)} \longrightarrow 	ext{Terminal Operation (Eager)}$$

* **Intermediate Operations (Return `Stream<T>`):** `filter()`, `map()`, `flatMap()`, `distinct()`, `sorted()`, `limit()`.
* **Terminal Operations (Trigger execution & close stream):** `collect()`, `forEach()`, `reduce()`, `count()`, `findFirst()`, `anyMatch()`.

```java
package com.roadmap.java8;

import java.math.BigDecimal;
import java.util.*;
import java.util.stream.Collectors;

public class StreamPipelineDemo {

    public record LineItem(String sku, String category, BigDecimal unitPrice, int quantity) {}

    public static void main(String[] args) {
        List<LineItem> cartItems = List.of(
            new LineItem("SKU-BOOK-1", "BOOKS", new BigDecimal("12.50"), 2),
            new LineItem("SKU-BOOK-2", "BOOKS", new BigDecimal("25.00"), 1),
            new LineItem("SKU-ELEC-1", "ELECTRONICS", new BigDecimal("120.00"), 1),
            new LineItem("SKU-CLOTH-1", "APPAREL", new BigDecimal("35.00"), 3),
            new LineItem("SKU-ELEC-2", "ELECTRONICS", new BigDecimal("80.00"), 2)
        );

        // 1. Filter, Map, and Collect to List
        List<String> expensiveSkus = cartItems.stream()
                .filter(item -> item.unitPrice().compareTo(new BigDecimal("30.00")) > 0)
                .map(LineItem::sku)
                .sorted()
                .collect(Collectors.toList());

        System.out.println("Expensive SKUs: " + expensiveSkus);

        // 2. Reduce: Calculate total cart subtotal
        BigDecimal cartSubtotal = cartItems.stream()
                .map(item -> item.unitPrice().multiply(BigDecimal.valueOf(item.quantity())))
                .reduce(BigDecimal.ZERO, BigDecimal::add);

        System.out.println("Total Cart Value: $" + cartSubtotal);

        // 3. GroupingBy: Group SKUs by Category
        Map<String, List<LineItem>> itemsByCategory = cartItems.stream()
                .collect(Collectors.groupingBy(LineItem::category));

        System.out.println("Categories partitioned: " + itemsByCategory.keySet());

        // 4. GroupingBy with downstream collector: Total items count per category
        Map<String, Integer> categoryStockCount = cartItems.stream()
                .collect(Collectors.groupingBy(
                    LineItem::category,
                    Collectors.summingInt(LineItem::quantity)
                ));

        System.out.println("Item count per category: " + categoryStockCount);
    }
}
```

---

## 5. `Optional<T>`: Handling Absence Without `NullPointerException`

`Optional<T>` is a container object used to represent the possible absence of a value, replacing risky `null` checks on method returns.

### Core Methods
* `Optional.of(value)`: Wraps a non-null value (throws NPE if null).
* `Optional.ofNullable(value)`: Wraps a value that might be null.
* `Optional.empty()`: Returns an empty Optional.
* `orElse(fallback)`: Returns value or fallback if empty.
* `orElseGet(Supplier)`: Lazily executes supplier if empty (avoids eager object instantiation).
* `orElseThrow(Supplier)`: Throws exception if empty (standard Spring pattern for 404s).
* `map()` and `flatMap()`: Chains operations safely over the contained value.

```java
package com.roadmap.java8;

import java.util.Optional;

public class OptionalIdiomsDemo {

    public record Customer(String email, String vatNumber) {}

    public static Optional<Customer> findCustomerByEmail(String email) {
        if ("alex@example.com".equalsIgnoreCase(email)) {
            return Optional.of(new Customer("alex@example.com", "FR123456789"));
        }
        return Optional.empty();
    }

    public static void main(String[] args) {
        // Safe transformation with map() and orElse()
        String vat = findCustomerByEmail("alex@example.com")
                .map(Customer::vatNumber)
                .map(String::trim)
                .orElse("TAX-EXEMPT-DEFAULT");

        System.out.println("Resolved VAT: " + vat);

        // Standard Spring Boot pattern: orElseThrow for REST endpoints
        try {
            Customer missing = findCustomerByEmail("unknown@example.com")
                    .orElseThrow(() -> new IllegalArgumentException("Customer record missing"));
        } catch (IllegalArgumentException ex) {
            System.err.println("Expected error: " + ex.getMessage());
        }
    }
}
```

---

## 6. Key Takeaways for Magento / PHP Developers

1. **`array_map` / `array_filter` vs. Streams:** In PHP, functional array functions mutate or allocate fresh copies eagerly at each step. In Java, Stream intermediate operations are **lazy**; iterations execute only once across elements when the terminal operation (`collect`, `reduce`) is called.
2. **`null` Propagation:** In PHP, `?->` (nullsafe operator) is commonly used. In Java, `Optional<T>` is the preferred API return signature for nullable lookups, explicitly declaring to caller methods that a result might be absent.
