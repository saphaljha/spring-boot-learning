# Java Multithreading & Concurrency

A comprehensive guide to multithreading, concurrency models, thread synchronization, thread pools, and atomic operations in Java, tailored for enterprise backend development.

---

## 1. Concurrency vs. Parallelism & The Java Thread Model

* **Concurrency:** Managing multiple tasks simultaneously by switching between execution contexts (interleaving on single or multi-core CPUs).
* **Parallelism:** Executing multiple computational operations at the exact same physical instant on multiple CPU cores.
* **OS Threads vs. JVM Threads:** In standard Java (pre-Java 21 platform threads), each `java.lang.Thread` directly maps $1:1$ to an underlying Operating System kernel thread.

```
+-------------------------------------------------------------+
| Java Application Runtime                                    |
|                                                             |
|   Thread-1               Thread-2               Thread-3    |
|   [Call Stack]           [Call Stack]           [Call Stack]|
|        |                      |                      |      |
+--------|----------------------|----------------------|------+
         |                      |                      |
         v                      v                      v
   OS Kernel Thread       OS Kernel Thread       OS Kernel Thread
+-------------------------------------------------------------+
| Host Operating System / CPU Cores                           |
+-------------------------------------------------------------+
```

---

## 2. Creating and Executing Threads

Java offers three primary ways to define threaded tasks:

1. **Extending `Thread`:** Direct inheritance (inflexible due to Java's single inheritance model).
2. **Implementing `Runnable`:** Decouples the task logic from thread execution (preferred for void tasks).
3. **Implementing `Callable<V>`:** Supports returning computed results (`Future<V>`) and throwing checked exceptions.

```java
package com.roadmap.concurrency;

import java.util.concurrent.*;

public class ThreadCreationDemo {

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        // 1. Runnable (Fire-and-forget task)
        Runnable logTask = () -> System.out.println("Processing audit on thread: " + Thread.currentThread().getName());
        Thread thread = new Thread(logTask, "AuditWorker-1");
        thread.start();

        // 2. Callable with ExecutorService (Task returning a value)
        Callable<Integer> calculationTask = () -> {
            TimeUnit.MILLISECONDS.sleep(100);
            return 42 * 2;
        };

        ExecutorService executor = Executors.newSingleThreadExecutor();
        Future<Integer> futureResult = executor.submit(calculationTask);

        // Blocking get() call with timeout safeguard
        try {
            Integer result = futureResult.get(500, TimeUnit.MILLISECONDS);
            System.out.println("Callable computed result: " + result);
        } catch (TimeoutException e) {
            futureResult.cancel(true);
            System.err.println("Task timed out.");
        } finally {
            executor.shutdown();
        }
    }
}
```

---

## 3. Thread Synchronization: `synchronized` and `volatile`

When multiple threads concurrently mutate shared state on the JVM Heap, race conditions occur without proper memory visibility and mutual exclusion guarantees.

### The `volatile` Keyword: Visibility Guarantee

* In modern architectures, CPU cores cache variables in high-speed L1/L2 caches.
* Without `volatile`, changes made by Core 1 to a shared variable might not be flushed immediately to main memory, keeping Core 2 reading stale values.
* **`volatile` ensures:** All reads and writes go directly to main memory (happens-before relationship), but it **does not** guarantee atomicity for compound operations like `count++`.

### The `synchronized` Block: Mutual Exclusion + Visibility

* Acquires the intrinsic lock (monitor) of an object.
* Only one thread can execute a synchronized block/method guarded by that monitor at any time.

```java
package com.roadmap.concurrency;

public class SynchronizationDemo {

    private int counter = 0;
    private final Object lock = new Object();

    // Synchronized instance method: locks on 'this'
    public synchronized void incrementMethod() {
        counter++;
    }

    // Synchronized block: locks on private monitor (preferred for fine-grained control)
    public void incrementBlock() {
        synchronized (lock) {
            counter++;
        }
    }

    public synchronized int getCounter() {
        return counter;
    }
}
```

---

## 4. Atomic Variables & Lock-Free Programming

For single-variable mutations, hardware-level **Compare-And-Swap (CAS)** instructions provide thread safety without the heavy context-switching overhead of thread locks.

Java provides atomic wrappers in `java.util.concurrent.atomic`:
* `AtomicInteger`, `AtomicLong`, `AtomicBoolean`
* `AtomicReference<V>`
* High-throughput counters: `LongAdder`, `LongAccumulator`

```java
package com.roadmap.concurrency;

import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.LongAdder;

public class AtomicDemo {

    // Lock-free thread-safe counter using hardware-level CAS
    private final AtomicInteger activeRequests = new AtomicInteger(0);

    // Highly optimized for write-heavy concurrent scenarios
    private final LongAdder totalProcessedRequests = new LongAdder();

    public void onRequestReceived() {
        activeRequests.incrementAndGet();
        totalProcessedRequests.increment();
    }

    public void onRequestCompleted() {
        activeRequests.decrementAndGet();
    }

    public int getActiveRequests() {
        return activeRequests.get();
    }

    public long getTotalProcessed() {
        return totalProcessedRequests.sum();
    }
}
```

---

## 5. Thread Pools with `ExecutorService` & `ThreadPoolExecutor`

Manually spinning up `new Thread()` in backend servers causes severe resource exhaustion under traffic spikes. Production applications manage threads through fixed, managed pools.

### Anatomy of `ThreadPoolExecutor`

```java
ThreadPoolExecutor(
    int corePoolSize,               // Baseline permanent threads
    int maximumPoolSize,            // Maximum capacity during traffic bursts
    long keepAliveTime,             // Lifetime of idle threads above core size
    TimeUnit unit,                  // Time unit for keepAliveTime
    BlockingQueue<Runnable> workQueue, // Buffer holding pending tasks
    ThreadFactory threadFactory,    // Factory creating customized named threads
    RejectedExecutionHandler handler // Strategy when queue is full and max threads reached
)
```

### Production Thread Pool Example

```java
package com.roadmap.concurrency;

import java.util.concurrent.*;

public class ProductionThreadPoolDemo {

    public static void main(String[] args) {
        int coreThreads = 4;
        int maxThreads = 8;
        int queueCapacity = 100;

        // Bounded queue prevents OutOfMemoryError under heavy ingestion
        BlockingQueue<Runnable> taskQueue = new ArrayBlockingQueue<>(queueCapacity);

        // Custom thread factory for actionable logging and thread dumps
        ThreadFactory threadFactory = new ThreadFactory() {
            private int counter = 1;
            @Override
            public Thread newThread(Runnable r) {
                Thread t = new Thread(r, "OrderWorker-" + counter++);
                t.setDaemon(false);
                return t;
            }
        };

        // Rejection Policy: CallerRunsPolicy applies backpressure on the calling thread
        RejectedExecutionHandler rejectionHandler = new ThreadPoolExecutor.CallerRunsPolicy();

        ThreadPoolExecutor executor = new ThreadPoolExecutor(
                coreThreads,
                maxThreads,
                60L, TimeUnit.SECONDS,
                taskQueue,
                threadFactory,
                rejectionHandler
        );

        // Submit concurrent jobs
        for (int i = 1; i <= 10; i++) {
            final int taskId = i;
            executor.execute(() -> {
                System.out.println("Running task " + taskId + " on " + Thread.currentThread().getName());
            });
        }

        // Graceful shutdown pattern
        executor.shutdown();
        try {
            if (!executor.awaitTermination(5, TimeUnit.SECONDS)) {
                executor.shutdownNow();
            }
        } catch (InterruptedException ex) {
            executor.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }
}
```

---

## 6. Concurrency Utilities: `CountDownLatch`

`CountDownLatch` blocks one or more threads until a set of parallel sub-tasks executing in other threads completes.

```java
package com.roadmap.concurrency;

import java.util.concurrent.*;

public class CountDownLatchDemo {

    public static void main(String[] args) throws InterruptedException {
        int subServicesCount = 3;
        CountDownLatch latch = new CountDownLatch(subServicesCount);
        ExecutorService pool = Executors.newFixedThreadPool(subServicesCount);

        // Warm up cache, DB pool, and third-party payment gateway concurrently
        pool.execute(() -> { initializeService("DatabaseConnection"); latch.countDown(); });
        pool.execute(() -> { initializeService("RedisCache"); latch.countDown(); });
        pool.execute(() -> { initializeService("PaymentGatewayClient"); latch.countDown(); });

        // Wait until all 3 services countdown to zero
        latch.await(3, TimeUnit.SECONDS);
        System.out.println("All dependencies initialized. Starting HTTP listener...");

        pool.shutdown();
    }

    private static void initializeService(String name) {
        System.out.println("Initializing: " + name + " on " + Thread.currentThread().getName());
    }
}
```

---

## 7. Key Takeaways for Magento / PHP Developers

1. **Shared Memory Architecture:** In PHP-FPM, worker processes execute independently with isolated memory pools. In Java, all threads within the same JVM instance share heap memory. Shared fields on Spring beans (`@Service`, `@Component`, `@RestController`) must be **stateless** or thread-safe.
2. **Thread Contention & Race Conditions:** Two web requests in Spring Boot can hit the same method on a singleton bean simultaneously. Never store request-specific context in instance variables; pass request data as local variables on the method stack.
