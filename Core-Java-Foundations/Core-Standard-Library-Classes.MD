# Core Built-in Classes: String, Wrappers, and Object

A comprehensive deep dive into Java's standard library fundamentals: String immutability and memory optimization, Wrapper class behaviors, and the essential contracts of `java.lang.Object`.

---

## 1. `String`, `StringBuilder`, and `StringBuffer`

### String Immutability & String Constant Pool (SCP)

In Java, `String` objects are immutable. Once created, their internal byte array cannot be changed.

```
JVM Heap
+-------------------------------------------------------------------------+
|                                                                         |
|   +-----------------------------------------------------------------+   |
|   | String Constant Pool (SCP)                                      |   |
|   |                                                                 |   |
|   |   "customer" <----------------------+  (Reuses literal)         |   |
|   |        ^                            |                           |   |
|   +--------|----------------------------|---------------------------+   |
|            |                            |                               |
|   [ s1 = "customer" ]          [ s2 = "customer" ]                      |
|                                                                         |
|   [ s3 = new String("customer") ] ---> Separate Heap Object             |
|                                        (Points internally to SCP value) |
+-------------------------------------------------------------------------+
```

* **String Constant Pool:** A specialized storage area in the JVM Heap. When a string literal is declared (`String s = "apple"`), the JVM checks the SCP first. If present, it reuses the reference instead of allocating new heap space.
* **Why Immutability Matters:**
  * **Thread Safety:** Strings can be shared across multiple threads without synchronization.
  * **Security:** Sensitive parameters (database URLs, usernames, network sockets) cannot be altered maliciously after validation.
  * **Hash Code Caching:** The hash code (`hashCode()`) is calculated once and cached, making Strings fast keys in `HashMap` collections.

### String vs. StringBuilder vs. StringBuffer

| Feature | `String` | `StringBuilder` | `StringBuffer` |
| :--- | :--- | :--- | :--- |
| **Mutability** | Immutable | Mutable | Mutable |
| **Thread Safety** | Safe (read-only) | Not Thread-Safe | Thread-Safe (synchronized) |
| **Performance** | Slow for repeated concatenation | Fastest (single thread) | Slower due to lock overhead |
| **Recommended Use** | Standard values, DTO fields | High-volume string loops | Legacy multi-threaded apps |

```java
package com.roadmap.coreclasses;

public class StringConcatenationDemo {

    public static void main(String[] args) {
        // Anti-pattern: Creates 1000 intermediate String objects in Heap/SCP
        String result = "";
        for (int i = 0; i < 1000; i++) {
            result += i; // Inefficient: allocates and discards strings continuously
        }

        // Production standard: Mutates a single internal char/byte buffer
        StringBuilder builder = new StringBuilder(1000); // Pre-sized capacity
        for (int i = 0; i < 1000; i++) {
            builder.append(i);
        }
        String efficientResult = builder.toString();
        System.out.println("Result length: " + efficientResult.length());
    }
}
```

---

## 2. Wrapper Classes and Autoboxing / Unboxing

Wrapper classes provide object representations for primitive types.

| Primitive | Wrapper Class | Primitive | Wrapper Class |
| :--- | :--- | :--- | :--- |
| `byte` | `java.lang.Byte` | `float` | `java.lang.Float` |
| `short` | `java.lang.Short` | `double` | `java.lang.Double` |
| `int` | `java.lang.Integer` | `char` | `java.lang.Character` |
| `long` | `java.lang.Long` | `boolean` | `java.lang.Boolean` |

### Autoboxing, Unboxing, and the Integer Cache

* **Autoboxing:** Compiler converts primitive $	o$ wrapper (e.g., `Integer.valueOf(10)`).
* **Unboxing:** Compiler converts wrapper $	o$ primitive (e.g., `boxed.intValue()`).
* **Integer Cache:** The JVM caches `Integer` objects between `-128` and `127` by default.

```java
package com.roadmap.coreclasses;

public class WrapperComparisonDemo {

    public static void main(String[] args) {
        // Values within [-128, 127] reuse cached instances
        Integer a = 127;
        Integer b = 127;
        System.out.println("127 == 127: " + (a == b));       // true (same cached instance)

        // Values outside the cache create distinct heap objects
        Integer x = 128;
        Integer y = 128;
        System.out.println("128 == 128: " + (x == y));       // false (different heap addresses!)
        System.out.println("128 equals: " + x.equals(y));    // true (compares numeric values)
    }
}
```

> **Critical Rule:** Never use `==` to compare Wrapper objects. Always use `.equals()` to avoid subtle bugs when values exceed $127$.

---

## 3. The `Object` Class Foundation

Every class in Java directly or indirectly extends `java.lang.Object`.

### Key Methods to Understand

1. `public boolean equals(Object obj)`: Determines logical equivalence. Default implementation checks reference equality (`this == obj`).
2. `public int hashCode()`: Returns an integer hash value for bucket indexing in hash collections (`HashMap`, `HashSet`).
3. `public String toString()`: Returns string representation (default: `ClassName@hexHashCode`).
4. `protected Object clone()`: Creates a field-by-field copy (shallow copy).
5. `public final Class<?> getClass()`: Returns runtime type descriptor for reflection.

---

## 4. The `equals()` and `hashCode()` Contract

If you override `equals()`, you **must** override `hashCode()`. Breaking this contract causes hash-based collections (`HashMap`, `HashSet`) to fail silently.

### The Contract Rules

1. **Consistency:** If `o1.equals(o2)` is `true`, then `o1.hashCode() == o2.hashCode()` **must** always be `true`.
2. **Unequal Objects:** If `o1.equals(o2)` is `false`, their `hashCode()` values do *not* have to be distinct (though distinct values reduce hash collisions and improve lookup speed).
3. **Internal State Stability:** The fields used to calculate `hashCode()` must be identical to the fields evaluated in `equals()`.

### Implementation Example: Clean Domain Entity

```java
package com.roadmap.coreclasses;

import java.util.Objects;

public class CustomerProfile {

    private final Long id;
    private final String email;
    private final String taxIdentifier;

    public CustomerProfile(Long id, String email, String taxIdentifier) {
        this.id = id;
        this.email = email;
        this.taxIdentifier = taxIdentifier;
    }

    public Long getId() {
        return id;
    }

    public String getEmail() {
        return email;
    }

    public String getTaxIdentifier() {
        return taxIdentifier;
    }

    @Override
    public boolean equals(Object o) {
        // 1. Check identity (same memory address)
        if (this == o) return true;

        // 2. Check null and runtime type compatibility
        if (o == null || getClass() != o.getClass()) return false;

        // 3. Cast and compare significant business identity fields
        CustomerProfile that = (CustomerProfile) o;
        return Objects.equals(id, that.id) &&
               Objects.equals(email, that.email) &&
               Objects.equals(taxIdentifier, that.taxIdentifier);
    }

    @Override
    public int hashCode() {
        // Generates hash based on the exact same fields as equals()
        return Objects.hash(id, email, taxIdentifier);
    }

    @Override
    public String toString() {
        return "CustomerProfile{" +
                "id=" + id +
                ", email='" + email + ''' +
                ", taxIdentifier='" + taxIdentifier + ''' +
                '}';
    }
}
```

---

## 5. Practical Failure: What Breaks When the Contract is Ignored?

```java
package com.roadmap.coreclasses;

import java.util.HashSet;
import java.util.Set;

public class HashCodeFailureDemo {

    static class BadCustomer {
        private final String email;

        public BadCustomer(String email) {
            this.email = email;
        }

        // Overrides equals() but forgets hashCode()!
        @Override
        public boolean equals(Object o) {
            if (this == o) return true;
            if (o == null || getClass() != o.getClass()) return false;
            BadCustomer that = (BadCustomer) o;
            return email.equals(that.email);
        }
    }

    public static void main(String[] args) {
        Set<BadCustomer> customers = new HashSet<>();
        customers.add(new BadCustomer("alex@example.com"));

        // Looking up an equal object
        boolean found = customers.contains(new BadCustomer("alex@example.com"));

        // Prints false! The HashSet routed the lookup to the wrong hash bucket
        // because the default Object.hashCode() returned different memory-based hashes.
        System.out.println("Customer recognized in Set: " + found);
    }
}
```

---

## 6. Key Takeaways for Magento / PHP Developers

1. **String Concatenation in Loops:** PHP's `.` operator reallocates memory smoothly, but in Java, chaining `+` inside large loops generates discarded heap objects. Always use `StringBuilder` for loop concatenation.
2. **Value Equality vs. Identity:** In PHP, `==` performs loose type-juggling and `===` performs strict type checking. In Java:
   * `==` checks reference address (pointer equality) for objects.
   * `.equals()` checks logical value equality.
3. **Null vs. Zero:** Never rely on PHP-style falsey evaluations (e.g., `0 == null`). A primitive `int` cannot be `null`, while an uninitialized wrapper `Integer` defaults to `null`.
