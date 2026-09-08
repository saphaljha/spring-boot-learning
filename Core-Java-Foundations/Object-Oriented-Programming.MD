# Object-Oriented Programming (OOP) in Java

A comprehensive deep dive into OOP concepts in Java, memory allocation dynamics, and architectural patterns, tailored for developers transitioning from PHP/Magento.

---

## 1. Classes, Objects, and JVM Memory Allocation

* **Class**: The blueprint or template that defines field states and method behaviors. Loaded into **Metaspace** (Method Area) by the ClassLoader.
* **Object (Instance)**: A concrete instantiation of a class. Always allocated dynamically on the **Heap**.
* **Reference Variable**: A local pointer located on the **Thread Stack** holding the heap address of the instantiated object.

```
Thread Stack (Fast, Per-Thread)               JVM Heap (Shared Across Threads)
+-------------------------------+              +-------------------------------------+
| void processOrder()           |              | Order Object Instance               |
|                               |              |  - id: 101L                         |
| [ orderRef ] -----------------|------------->|  - orderNumber: "ORD-991"           |
| (memory address: 0x7ffd2a)    |              |  - totalAmount: $120.50             |
+-------------------------------+              +-------------------------------------+
```

> **Magento vs. Java Memory Note:** In PHP, objects live and die with the HTTP request. In Java, objects remain on the heap until the Garbage Collector (GC) discovers they are unreachable from any active thread root.

---

## 2. Constructors, Chaining, and `this` / `super`

Constructors initialize an object's instance state. Unlike PHP (`__construct`), Java constructors match the exact class name and have no return type.

* `this`: Refers to the current class instance. Can call another constructor in the same class via `this(...)` (must be the first line).
* `super`: Refers to the immediate parent class. Invokes parent constructors via `super(...)` or parent methods via `super.method()`.

```java
package com.roadmap.oop;

public class Order {
    private final Long id;
    private final String orderNumber;
    private String status;

    // Default / Overloaded Constructor
    public Order(Long id, String orderNumber) {
        this(id, orderNumber, "PENDING"); // Constructor chaining via this()
    }

    // Primary Constructor
    public Order(Long id, String orderNumber, String status) {
        this.id = id;
        this.orderNumber = orderNumber;
        this.status = status;
    }

    public Long getId() {
        return id;
    }

    public String getOrderNumber() {
        return orderNumber;
    }

    public String getStatus() {
        return status;
    }
}
```

---

## 3. Encapsulation and Access Modifiers

Encapsulation hides internal object state and enforces integrity by mediating access through public behaviors (getters/setters/domain methods).

| Modifier | Same Class | Same Package | Subclass (Any Package) | World (Everywhere) |
| :--- | :---: | :---: | :---: | :---: |
| `private` | **Yes** | No | No | No |
| *(default / package-private)* | **Yes** | **Yes** | No | No |
| `protected` | **Yes** | **Yes** | **Yes** | No |
| `public` | **Yes** | **Yes** | **Yes** | **Yes** |

> **Best Practice:** Fields should be `private final` by default. Expose mutation only through explicit domain methods (e.g., `cancelOrder()`) rather than generic public setters.

---

## 4. Inheritance & Polymorphism

Java supports single-class inheritance (`extends`) and multiple-interface implementation (`implements`).

* **Compile-Time Polymorphism (Method Overloading):** Multiple methods in the same class with identical names but differing parameter lists (different types or counts). Resolved at compile time.
* **Runtime Polymorphism (Method Overriding):** A child class provides a specialized implementation of a method declared in its parent. Resolved at runtime via virtual method tables (Dynamic Method Dispatch).

```java
package com.roadmap.oop;

import java.math.BigDecimal;

// Base Class
public abstract class PaymentMethod {
    private final String providerName;

    protected PaymentMethod(String providerName) {
        this.providerName = providerName;
    }

    public String getProviderName() {
        return providerName;
    }

    // Abstract method: must be overridden by concrete subclasses
    public abstract boolean processTransaction(BigDecimal amount);

    // Concrete shared logic
    public void logAudit(BigDecimal amount, boolean success) {
        System.out.println("[" + providerName + "] Transaction of $" + amount + " - Success: " + success);
    }
}
```

```java
package com.roadmap.oop;

import java.math.BigDecimal;

// Child Class 1
public class CreditCardPayment extends PaymentMethod {
    private final String maskedCardNumber;

    public CreditCardPayment(String maskedCardNumber) {
        super("CreditCardGateway"); // Invoking parent constructor
        this.maskedCardNumber = maskedCardNumber;
    }

    @Override
    public boolean processTransaction(BigDecimal amount) {
        // Concrete payment execution
        System.out.println("Charging card " + maskedCardNumber + " for amount: $" + amount);
        logAudit(amount, true);
        return true;
    }

    // Method Overload (Compile-Time Polymorphism)
    public boolean processTransaction(BigDecimal amount, String currency) {
        System.out.println("Processing in foreign currency: " + currency);
        return processTransaction(amount);
    }
}
```

```java
package com.roadmap.oop;

import java.math.BigDecimal;

// Child Class 2
public class PayPalPayment extends PaymentMethod {
    private final String payerEmail;

    public PayPalPayment(String payerEmail) {
        super("PayPalRestAPI");
        this.payerEmail = payerEmail;
    }

    @Override
    public boolean processTransaction(BigDecimal amount) {
        System.out.println("Charging PayPal wallet of " + payerEmail + " for: $" + amount);
        logAudit(amount, true);
        return true;
    }
}
```

---

## 5. Abstraction: Abstract Classes vs. Interfaces

Both abstractions decouple specification from implementation, but they serve distinct architectural roles:

| Dimension | Abstract Class | Interface |
| :--- | :--- | :--- |
| **Primary Intent** | Code sharing & state inheritance among related objects (is-a). | Defining a contract or capability independent of hierarchy (can-do). |
| **State / Fields** | Can have instance variables, mutable fields, and constructors. | Only `public static final` constants (no mutable instance state). |
| **Multiple Inheritance** | No (single `extends` only). | Yes (a class can implement multiple interfaces). |
| **Methods** | Abstract, concrete, and protected methods allowed. | Abstract methods, `default` methods, and `static` utility methods. |

### Interface Contract Example

```java
package com.roadmap.oop;

import java.math.BigDecimal;

public interface Refundable {
    // Implicitly public static final
    String REFUND_POLICY_VERSION = "2.1";

    // Implicitly public abstract
    boolean issueRefund(String transactionId, BigDecimal amount);

    // Default method (Java 8+): provides fallback behavior without breaking implementers
    default void logRefundInitiated(String transactionId) {
        System.out.println("Refund initiated for transaction: " + transactionId + " under policy " + REFUND_POLICY_VERSION);
    }
}
```

```java
package com.roadmap.oop;

import java.math.BigDecimal;

// Combining Abstract Class inheritance and Interface capabilities
public class RefundableCreditCardPayment extends CreditCardPayment implements Refundable {

    public RefundableCreditCardPayment(String maskedCardNumber) {
        super(maskedCardNumber);
    }

    @Override
    public boolean issueRefund(String transactionId, BigDecimal amount) {
        logRefundInitiated(transactionId);
        System.out.println("Reversing $" + amount + " to card.");
        return true;
    }
}
```

---

## 6. Putting It Together: Runtime Polymorphism in Practice

This executable demo illustrates how interfaces and runtime polymorphism allow services to process transactions without knowing concrete implementation details:

```java
package com.roadmap.oop;

import java.math.BigDecimal;
import java.util.List;

public class OOPExecutionDemo {

    public static void main(String[] args) {
        // Polymorphic list referencing the base abstract type
        List<PaymentMethod> checkoutOptions = List.of(
            new CreditCardPayment("4111-XXXX-XXXX-1111"),
            new PayPalPayment("buyer@example.com")
        );

        BigDecimal invoiceTotal = new BigDecimal("89.99");

        // Runtime Dispatch: The JVM determines the exact method to invoke at runtime
        for (PaymentMethod payment : checkoutOptions) {
            System.out.println("--- Processing via: " + payment.getProviderName() + " ---");
            payment.processTransaction(invoiceTotal);
        }

        // Capability checking via interface
        RefundableCreditCardPayment refundablePayment = new RefundableCreditCardPayment("5500-XXXX-XXXX-2222");
        if (refundablePayment instanceof Refundable refundable) { // Java 16+ pattern matching
            refundable.issueRefund("TXN-90210", new BigDecimal("45.00"));
        }
    }
}
```

---

## 7. Key Takeaways for Magento / PHP Developers

1. **Virtual Method Calls by Default:** In Java, all non-static, non-private, non-final methods are virtual (can be overridden). In PHP, methods are also virtual by default, but Java resolves them with high-performance polymorphic inline caches in the JIT compiler.
2. **`final` Keyword:**
   * A `final` class cannot be extended (similar to `final class` in PHP).
   * A `final` method cannot be overridden.
   * A `final` variable can only be assigned once (immutable reference pointer).
3. **No Dynamic Class Loading via Strings:** In Magento, classes are often resolved dynamically via factory strings like `$this->orderFactory->create(...)`. In Java, dependencies and polymorphic types are bound to explicit interfaces and compiled type signatures.
