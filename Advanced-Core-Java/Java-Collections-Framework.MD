# Java Collections Framework

A comprehensive guide to the Java Collections Framework (`java.util`), covering internal mechanics, complexity analysis, thread safety, and best practices for enterprise backend development.

---

## 1. Hierarchy & Core Interfaces

The Java Collections Framework separates collections of elements (`Collection`) from key-value stores (`Map`):

```
                        +------------------+
                        |  Iterable<E>     |
                        +------------------+
                                 |
                        +------------------+
                        |  Collection<E>   |
                        +------------------+
                                 |
        +------------------------+------------------------+
        |                        |                        |
+----------------+       +----------------+       +----------------+
|    List<E>     |       |     Set<E>     |       |    Queue<E>    |
+----------------+       +----------------+       +----------------+
        |                        |                        |
  - ArrayList              - HashSet                - PriorityQueue
  - LinkedList             - LinkedHashSet          - ArrayDeque
                           - TreeSet (SortedSet)    - LinkedList (Deque)

----------------------------------------------------------------------
Distinct Key-Value Hierarchy (Does NOT inherit from Collection):

                        +------------------+
                        |    Map<K, V>     |
                        +------------------+
                                 |
  +------------------------------+------------------------------+
  |                              |                              |
- HashMap                  - LinkedHashMap                - TreeMap (SortedMap)
- ConcurrentHashMap
```

---

## 2. Lists: `ArrayList` vs. `LinkedList` vs. `CopyOnWriteArrayList`

| Characteristic | `ArrayList` | `LinkedList` | `CopyOnWriteArrayList` |
| :--- | :--- | :--- | :--- |
| **Internal Structure** | Resizable dynamic array | Doubly linked list | Array (cloned on mutation) |
| **Random Access (`get(i)`)** | $O(1)$ | $O(n)$ | $O(1)$ |
| **Insertion / Deletion** | $O(n)$ (resizing/shifting) | $O(1)$ (if pointer known) | $O(n)$ (clones entire array) |
| **Memory Overhead** | Low (contiguous memory) | High (node pointers) | High during writes |
| **Thread Safety** | Not thread-safe | Not thread-safe | Thread-safe (read-heavy loads) |

> **Production Standard:** Default to `ArrayList` for 99% of list use cases. CPU cache locality makes `ArrayList` faster than `LinkedList` even for many middle insertions.

```java
package com.roadmap.collections;

import java.util.ArrayList;
import java.util.List;

public class ListDemo {

    public static void main(String[] args) {
        // Pre-size capacity when item count is roughly known to avoid array resizing
        List<String> skuList = new ArrayList<>(100);
        skuList.add("SKU-APPAREL-001");
        skuList.add("SKU-APPAREL-002");

        // Fast O(1) indexed lookup
        String firstSku = skuList.get(0);
        System.out.println("First item: " + firstSku);

        // Immutable List factory (Java 9+)
        List<String> fixedRoles = List.of("ADMIN", "MANAGER", "CUSTOMER");
        // fixedRoles.add("SUPPORT"); // Throws UnsupportedOperationException
    }
}
```

---

## 3. Sets: `HashSet` vs. `LinkedHashSet` vs. `TreeSet`

Sets enforce element uniqueness using `equals()` and `hashCode()`.

* **`HashSet`**: Backed by a `HashMap`. Provides $O(1)$ lookup and insertion. **Unordered**.
* **`LinkedHashSet`**: Extends `HashSet` with a doubly linked list running through all entries. Maintains **insertion order**.
* **`TreeSet`**: Backed by a Red-Black tree. Elements are sorted by natural order or a custom `Comparator`. Provides $O(\log n)$ operations.

```java
package com.roadmap.collections;

import java.util.*;

public class SetDemo {

    public static void main(String[] args) {
        // 1. HashSet: Unique elements, non-deterministic order
        Set<String> categories = new HashSet<>();
        categories.add("Electronics");
        categories.add("Fashion");
        categories.add("Electronics"); // Duplicate ignored

        // 2. TreeSet: Natural alphabetical ordering
        Set<String> sortedTags = new TreeSet<>(Set.of("promo", "sale", "clearance", "bogo"));
        System.out.println("TreeSet sorted: " + sortedTags); // [bogo, clearance, promo, sale]
    }
}
```

---

## 4. Maps: `HashMap` Internal Mechanics & `ConcurrentHashMap`

### How `HashMap` Works Under the Hood

1. **Bucket Array:** `HashMap` maintains an array of `Node<K, V>` (buckets) with a default initial capacity of 16 and a load factor of 0.75.
2. **Hash & Index Computation:** The key's hash code is spread via bitwise operations:
   $$	ext{index} = (n - 1) \ \& \ 	ext{hash}$$
3. **Collision Handling (Chaining):**
   * Multiple keys mapped to the same index form a linked list.
   * **Treeification (Java 8+):** If a bucket holds 8 or more entries and total table capacity is at least 64, the linked list converts to a balanced Red-Black tree, reducing worst-case lookup from $O(n)$ to $O(\log n)$.
4. **Resizing:** When `size > capacity * 0.75`, the internal array doubles in size and re-indexes elements.

### `HashMap` vs. `ConcurrentHashMap`

* `HashMap` is **not thread-safe**. Concurrent modifications during a resize can lead to corruption or infinite loops.
* `ConcurrentHashMap` allows concurrent reads without locking and uses bucket-level locking (synchronized node heads or CAS operations) for writes, avoiding global table locks.

```java
package com.roadmap.collections;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class MapDemo {

    public static void main(String[] args) {
        // Thread-safe map for caching / in-memory tracking
        Map<String, Integer> inventoryCache = new ConcurrentHashMap<>();

        // Atomic compute-if-absent pattern
        inventoryCache.computeIfAbsent("SKU-990", key -> loadStockFromDb(key));

        // Atomic mutation
        inventoryCache.merge("SKU-990", 5, Integer::sum);
        System.out.println("Updated stock: " + inventoryCache.get("SKU-990"));
    }

    private static int loadStockFromDb(String sku) {
        return 20; // Simulated DB lookup
    }
}
```

---

## 5. Sorting: `Comparable<T>` vs. `Comparator<T>`

* **`Comparable<T>` (Natural Ordering):** Implemented directly on the domain class via `compareTo(T o)`.
* **`Comparator<T>` (Custom / Strategy Ordering):** Separate comparator classes or lambda expressions passed at sorting time.

```java
package com.roadmap.collections;

import java.math.BigDecimal;
import java.util.*;

public class SortingDemo {

    public record Product(String sku, BigDecimal price, int stock) implements Comparable<Product> {

        // Natural order: ascending by SKU
        @Override
        public int compareTo(Product other) {
            return this.sku.compareTo(other.sku);
        }
    }

    public static void main(String[] args) {
        List<Product> products = new ArrayList<>(List.of(
            new Product("PROD-B", new BigDecimal("49.99"), 5),
            new Product("PROD-A", new BigDecimal("99.00"), 20),
            new Product("PROD-C", new BigDecimal("19.50"), 10)
        ));

        // 1. Natural Sort (Comparable)
        Collections.sort(products);
        System.out.println("Natural sort by SKU: " + products);

        // 2. Custom Sort with Comparator chaining (Java 8+)
        Comparator<Product> priceThenStock = Comparator
                .comparing(Product::price)
                .thenComparing(Product::stock, Comparator.reverseOrder());

        products.sort(priceThenStock);
        System.out.println("Sorted by Price then Stock: " + products);
    }
}
```

---

## 6. Key Takeaways for Magento / PHP Developers

1. **Associative Arrays vs. Concrete Types:** In PHP, an array serves simultaneously as a list, hash map, stack, and queue. In Java, pick the exact collection based on access patterns (`ArrayList` vs. `HashMap` vs. `ArrayDeque`).
2. **Key Mutability in Maps:** In PHP, keys are strings or ints. In Java, keys can be arbitrary objects. If a key's mutable fields change after insertion into a `HashMap`, its `hashCode()` changes, making the entry unretrievable (memory leak). Use immutable types (`String`, `Long`, or `record`) as keys.
