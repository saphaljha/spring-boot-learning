# Java Architecture & Basics

A comprehensive deep dive into the JVM architecture, memory model, data types, control flow, and execution fundamentals, prepared for backend developers transitioning to the Java ecosystem.

---

## 1. JVM, JRE, and JDK Architecture

Understanding how Java executes code is foundational to writing efficient backend services and diagnosing performance bottlenecks.

```
+-----------------------------------------------------------------------+
| JDK (Java Development Kit)                                            |
|  [ javac, jdb, jar, jshell, profiling tools ]                         |
|                                                                       |
|  +-----------------------------------------------------------------+  |
|  | JRE (Java Runtime Environment)                                  |  |
|  |  [ Core Java Class Libraries (java.base, java.sql, etc.) ]      |  |
|  |                                                                 |  |
|  |  +-----------------------------------------------------------+  |  |
|  |  | JVM (Java Virtual Machine)                                |  |  |
|  |  |  - ClassLoader Subsystem (Loading, Linking, Initialization)|  |  |
|  |  |  - Runtime Data Areas (Method, Heap, Stack, PC, Native)   |  |  |
|  |  |  - Execution Engine (Interpreter, JIT Compiler, GC)       |  |  |
|  |  +-----------------------------------------------------------+  |  |
|  +-----------------------------------------------------------------+  |
+-----------------------------------------------------------------------+
```

### Key Components

1. **JDK (Java Development Kit):**
   * The complete developer suite containing compiler tools (`javac`), archive tools (`jar`), debugging utilities, and the JRE.
2. **JRE (Java Runtime Environment):**
   * The minimum runtime distribution needed to execute a compiled Java program. It bundles the standard libraries and the JVM.
3. **JVM (Java Virtual Machine):**
   * The execution engine that executes compiled Java bytecode (`.class` files). It is platform-dependent, but provides the platform-independent "Write Once, Run Anywhere" guarantee.

### The Internal Mechanics of the JVM

* **ClassLoader Subsystem:**
  * **Loading:** Loads `.class` files into memory using three built-in loaders: Bootstrap ClassLoader, Extension/Platform ClassLoader, and Application/System ClassLoader.
  * **Linking:** Verifies bytecode security, prepares static variables with default values, and resolves symbolic memory references.
  * **Initialization:** Runs static initializers and assigns explicit static values.
* **JVM Runtime Data Areas:**
  * **Heap:** Shared across all threads. Stores all runtime objects and instance variables. Managed by the Garbage Collector (GC).
  * **Stack (Per-Thread):** Stores stack frames containing local variables, partial results, and method invocation/return data. Each thread has its own call stack.
  * **Method Area / Metaspace:** Shared across threads. Stores class metadata, runtime constant pool, and static fields. (Metaspace uses native off-heap memory since Java 8).
  * **Program Counter (PC) Register:** Stores the address of the currently executing JVM instruction for each thread.
  * **Native Method Stack:** Supports C/C++ native system libraries called via JNI (Java Native Interface).
* **Execution Engine:**
  * **Interpreter:** Reads and interprets bytecode line by line.
  * **JIT (Just-In-Time) Compiler:** Monitors code execution ("hot spots") and compiles frequently executed bytecode directly into native machine instructions (C1/C2 compilers) for near-native performance.
  * **Garbage Collector (GC):** Automatically tracks unreachable objects and reclaims heap memory (e.g., G1GC, ZGC).

---

## 2. Data Types, Primitive Wrappers, and Type Casting

Java is statically and strongly typed. Every variable must have a declared type before use.

### Primitive vs. Non-Primitive Data Types

| Type | Bits / Bytes | Default Value | Range / Description |
| :--- | :--- | :--- | :--- |
| `byte` | 8 bits (1 byte) | `0` | -128 to 127 |
| `short` | 16 bits (2 bytes) | `0` | -32,768 to 32,767 |
| `int` | 32 bits (4 bytes) | `0` | -2,147,483,648 to 2,147,483,647 |
| `long` | 64 bits (8 bytes) | `0L` | -9,223,372,036,854,775,808 to 9,223,372,036,854,775,807 |
| `float` | 32 bits (4 bytes) | `0.0f` | Single-precision IEEE 754 floating-point |
| `double` | 64 bits (8 bytes) | `0.0d` | Double-precision IEEE 754 floating-point |
| `boolean`| 1 bit (JVM dependent)| `false` | `true` or `false` |
| `char` | 16 bits (2 bytes) | `'\u0000'` | Unicode characters (0 to 65,535) |

### Primitive Wrappers & Autoboxing

Each primitive has an object wrapper in `java.lang` (e.g., `int` -> `Integer`, `boolean` -> `Boolean`).
* **Autoboxing:** Automatic conversion from primitive to wrapper object.
* **Unboxing:** Automatic extraction of the primitive value from the wrapper.

> **Backend Tip:** Always use primitives for high-performance loops to avoid GC overhead. Use Wrappers when declaring entities/DTO fields where values can be nullable (`null` vs `0`).

### Type Casting (Widening vs. Narrowing)

* **Widening (Implicit):** Converting a smaller type to a larger type without data loss: `int` -> `long` -> `float` -> `double`.
* **Narrowing (Explicit):** Converting a larger type to a smaller type. Requires an explicit cast and may truncate data.

### Code Example: Data Types & Casting

```java
package com.roadmap.basics;

public class DataTypesAndCastingDemo {

    public static void main(String[] args) {
        // 1. Primitive Declarations
        int orderCount = 42;
        long transactionId = 9876543210L; // Suffix 'L' required for long literals
        double itemPrice = 19.99;
        float discountRate = 0.05f;      // Suffix 'f' required for float literals
        char currencySymbol = '$';
        boolean isProcessed = false;

        // 2. Widening Casting (Implicit - safe)
        double totalWithTaxes = orderCount * itemPrice; // int promoted to double automatically
        System.out.println("Total calculated: " + totalWithTaxes);

        // 3. Narrowing Casting (Explicit - potential data truncation)
        double rawScore = 98.75;
        int roundedScore = (int) rawScore; // Truncates decimal part (.75 dropped)
        System.out.println("Truncated Score: " + roundedScore);

        long largeValue = 130L;
        byte overflowByte = (byte) largeValue; // byte max is 127; wraps to -126
        System.out.println("Overflowed byte: " + overflowByte);

        // 4. Autoboxing and Unboxing
        Integer boxedCount = orderCount;        // Autoboxing: primitive int -> Integer object
        int unboxedCount = boxedCount;          // Unboxing: Integer object -> primitive int
        
        // Handling null wrapper values safely
        Integer nullableStock = null;
        try {
            int currentStock = nullableStock;   // Throws NullPointerException!
        } catch (NullPointerException ex) {
            System.out.println("Safely caught unboxing NPE on null wrapper reference.");
        }
    }
}
```

---

## 3. Operators and Control Flow

Control flow structures govern logical execution paths. Modern Java includes traditional statements as well as modern pattern-matching expressions.

### Conditional Statements

* `if`, `else if`, `else`: Branching based on boolean conditions.
* Traditional `switch`: Statement with manual `break` commands.
* **Enhanced `switch` expressions (Java 14+):** Clean arrow syntax (`->`), returns values directly, and prevents fall-through bugs.

### Looping Structures

* `while`: Condition evaluated before entry.
* `do-while`: Executes at least once before condition evaluation.
* Standard `for`: Loop with counter initialization, condition, and increment.
* Enhanced `for-each`: Iterates sequentially over arrays or `Iterable` collections.

### Code Example: Control Flow & Switch Expressions

```java
package com.roadmap.basics;

public class ControlFlowDemo {

    public enum OrderStatus {
        PENDING,
        SHIPPED,
        DELIVERED,
        CANCELLED
    }

    public static void main(String[] args) {
        OrderStatus currentStatus = OrderStatus.SHIPPED;

        // 1. Enhanced Switch Expression (returns value directly)
        int estimatedDeliveryDays = switch (currentStatus) {
            case PENDING -> 5;
            case SHIPPED -> 2;
            case DELIVERED -> 0;
            case CANCELLED -> -1;
        };
        System.out.println("Estimated delivery days: " + estimatedDeliveryDays);

        // 2. Multi-case match in Switch Expression
        String priority = switch (currentStatus) {
            case PENDING, SHIPPED -> "High Priority";
            case DELIVERED, CANCELLED -> "Normal Priority";
        };
        System.out.println("Order priority: " + priority);

        // 3. Traditional Loops vs Enhanced For-Each
        int[] retryIntervalsSeconds = { 1, 2, 4, 8, 16 };

        System.out.println("Processing retry backoff schedule:");
        for (int interval : retryIntervalsSeconds) {
            if (interval > 8) {
                System.out.println("Interval " + interval + "s exceeds safety threshold. Aborting.");
                break; // Break exits the loop immediately
            }
            if (interval == 2) {
                System.out.println("Skipping diagnostic interval: " + interval + "s");
                continue; // Continue skips current iteration
            }
            System.out.println("Configured backoff: " + interval + "s");
        }
    }
}
```

---

## 4. Arrays and Command-Line Arguments

Arrays are fixed-length, contiguous memory blocks storing elements of the same type.

### Characteristics of Java Arrays

* Array sizing is **fixed** once instantiated.
* Zero-indexed (`0` to `array.length - 1`).
* Boundary checking is enforced at runtime (`ArrayIndexOutOfBoundsException`).
* Multi-dimensional arrays are represented as "arrays of arrays" (allowing jagged arrays).

### Command-Line Arguments (`String[] args`)

The `main` entrypoint accepts an array of strings passed to the JVM at startup:
```bash
java com.roadmap.basics.CommandLineRunner env=production port=8080
```

### Code Example: Arrays & Arguments Processing

```java
package com.roadmap.basics;

import java.util.Arrays;

public class ArraysAndArgumentsDemo {

    public static void main(String[] args) {
        // 1. Handling Command-Line Arguments (args)
        System.out.println("Total arguments passed: " + args.length);
        for (int i = 0; i < args.length; i++) {
            System.out.println("Argument [" + i + "]: " + args[i]);
        }

        // 2. Array Instantiation and Initialization
        int[] scores = new int[3]; // Allocates memory with default values [0, 0, 0]
        scores[0] = 95;
        scores[1] = 88;
        scores[2] = 72;

        // Literal array instantiation
        String[] authorizedRoles = { "ADMIN", "MANAGER", "SUPPORT" };

        // 3. Array Utilities (java.util.Arrays)
        Arrays.sort(scores);
        System.out.println("Sorted scores: " + Arrays.toString(scores));

        int searchIndex = Arrays.binarySearch(scores, 88);
        System.out.println("Index of score 88: " + searchIndex);

        // 4. Multi-dimensional Arrays (Matrix)
        int[][] matrix = {
            { 1, 2, 3 },
            { 4, 5, 6 }
        };

        System.out.println("Matrix element row 1, col 2: " + matrix[1][2]); // Prints 6
    }
}
```

---

## 5. Summary & Key Takeaways for Magento / PHP Developers

1. **Process Isolation vs JVM Lifetime:** In PHP, the runtime boots and tears down per request. In Java, the JVM stays running. Memory leaks accumulate if references remain open.
2. **Type Safety:** Variables cannot change types dynamically. Type mismatches cause compilation errors rather than runtime notices.
3. **No Global State via Files:** In PHP, config files or `di.xml` can be loaded dynamically. In Java, class definitions are loaded and verified by the ClassLoader during JVM initialization.

============================================================================================================================================================================================================

What’s Included in this Module
Architecture Diagram & Breakdown: JDK vs. JRE vs. JVM, ClassLoader phases, Memory Areas (Heap, Stack, Metaspace), and Execution Engine (Interpreter, JIT, GC).

Data Types, Wrappers & Casting: Memory footprints, widening vs. narrowing casts, and handling nullable wrappers (preventing unboxing NullPointerException).

Control Flow & Enhanced Switch: Modern Java switch expressions (->), condition matching, and iteration controls.

Arrays & CLI Arguments: Array bounds, memory layouts, java.util.Arrays utility methods, and String[] args processing.

PHP/Magento Mental Bridges: Key differences in memory lifecycle, runtime execution, and class loading.

Whenever you are ready, let me know if you would like to proceed to the next topic: Core-Java-Foundations/Object-Oriented-Programming.

