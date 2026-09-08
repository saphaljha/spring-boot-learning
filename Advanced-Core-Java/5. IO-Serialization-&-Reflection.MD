# Java File I/O, Serialization, and Reflection

A comprehensive deep dive into standard and non-blocking I/O (`java.nio.file`), object serialization protocols, and the Reflection API powering modern Java frameworks.

---

## 1. Java I/O vs. Java NIO (Non-blocking I/O)

Java provides two distinct subsystems for handling file and stream communication:

| Dimension | Standard I/O (`java.io`) | Modern NIO (`java.nio`) |
| :--- | :--- | :--- |
| **Model** | Stream-oriented (byte/character streams) | Buffer & Channel-oriented |
| **Blocking Behavior** | Blocking (threads sleep on read/write) | Non-blocking / Asynchronous options |
| **Core Abstractions** | `InputStream`, `OutputStream`, `Reader`, `Writer` | `Channel`, `Buffer`, `Selector`, `Path`, `Files` |
| **Filesystem Operations** | `java.io.File` (limited error handling/metadata) | `java.nio.file.Files`, `Paths` (rich metadata, symlinks) |

### Modern File Operations with `java.nio.file.Files`

```java
package com.roadmap.io;

import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.nio.file.*;
import java.util.List;

public class NioFileOperationsDemo {

    public static void main(String[] args) throws IOException {
        Path configDir = Paths.get("data", "config");
        Path configFile = configDir.resolve("application-override.json");

        // 1. Create directories recursively
        if (Files.notExists(configDir)) {
            Files.createDirectories(configDir);
        }

        // 2. Writing content atomically with StandardOpenOption
        String payload = "{"environment":"staging","debug":false}";
        Files.writeString(
            configFile,
            payload,
            StandardCharsets.UTF_8,
            StandardOpenOption.CREATE,
            StandardOpenOption.TRUNCATE_EXISTING
        );

        // 3. Memory-efficient reading of lines
        List<String> lines = Files.readAllLines(configFile, StandardCharsets.UTF_8);
        lines.forEach(System.out::println);

        // 4. Checking metadata
        System.out.println("File size (bytes): " + Files.size(configFile));
        System.out.println("Is readable: " + Files.isReadable(configFile));
    }
}
```

---

## 2. Java Object Serialization & `serialVersionUID`

Serialization converts an in-memory JVM object graph into a binary byte stream for disk storage, caching, or network transport. Deserialization reconstructs the live object.

### Essential Rules

1. The class must implement the marker interface `java.io.Serializable`.
2. Any field that should **not** be persisted must be declared `transient` (e.g., passwords, runtime buffers, open socket handles).
3. **`serialVersionUID`:** A unique 64-bit version identifier. If the sender and receiver have differing `serialVersionUID` values during deserialization, an `InvalidClassException` is thrown.

```java
package com.roadmap.io;

import java.io.*;

public class UserSessionToken implements Serializable {

    // Explicit versioning protects against binary incompatibility across deployments
    @Serial
    private static final long serialVersionUID = 1L;

    private final String userId;
    private final String sessionToken;

    // transient fields are skipped during serialization
    private transient String unencryptedPrivateKey;

    public UserSessionToken(String userId, String sessionToken, String unencryptedPrivateKey) {
        this.userId = userId;
        this.sessionToken = sessionToken;
        this.unencryptedPrivateKey = unencryptedPrivateKey;
    }

    public String getUserId() { return userId; }
    public String getSessionToken() { return sessionToken; }
    public String getUnencryptedPrivateKey() { return unencryptedPrivateKey; }
}
```

```java
package com.roadmap.io;

import java.io.*;

public class SerializationDemo {

    public static void main(String[] args) throws IOException, ClassNotFoundException {
        UserSessionToken session = new UserSessionToken("USR-101", "tok_live_99812", "secret_key");
        File targetFile = new File("session.ser");

        // 1. Serialization (Object -> Bytes)
        try (ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream(targetFile))) {
            out.writeObject(session);
        }

        // 2. Deserialization (Bytes -> Object)
        try (ObjectInputStream in = new ObjectInputStream(new FileInputStream(targetFile))) {
            UserSessionToken restored = (UserSessionToken) in.readObject();
            System.out.println("Restored User: " + restored.getUserId());
            System.out.println("Restored Token: " + restored.getSessionToken());
            System.out.println("Restored Transient Key: " + restored.getUnencryptedPrivateKey()); // null
        } finally {
            targetFile.delete();
        }
    }
}
```

> **Enterprise Backend Note:** Native Java serialization is generally avoided in modern REST/microservice communication due to security vulnerabilities (RCE gadget chains). Modern stacks favor **JSON (Jackson)** or **Protobuf**. However, knowing serialization is critical because distributed HTTP session clustering and older cache layers still use it.

---

## 3. Java Reflection API (`java.lang.reflect`)

Reflection gives the JVM inspecting and mutating capabilities over classes, constructors, methods, and fields at runtime—even private ones.

### What Reflection Enables

* Inspecting class definitions dynamically (`Class.forName(...)`, `instance.getClass()`).
* Instantiating objects without calling `new` directly.
* Reading or writing private fields (`field.setAccessible(true)`).
* Invoking methods by string name dynamically.

```java
package com.roadmap.reflection;

import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.Method;

public class ReflectionDemo {

    static class OrderService {
        private String gatewayApiKey = "internal_secret_999";

        private void printInternalDiagnostics(String label) {
            System.out.println("[Diagnostics: " + label + "] Gateway key length: " + gatewayApiKey.length());
        }
    }

    public static void main(String[] args) throws Exception {
        Class<?> clazz = OrderService.class;

        // 1. Instantiate via Reflection
        Constructor<?> constructor = clazz.getDeclaredConstructor();
        Object instance = constructor.newInstance();

        // 2. Inspect and read private field
        Field apiKeyField = clazz.getDeclaredField("gatewayApiKey");
        apiKeyField.setAccessible(true); // Bypass Java access control checks
        String originalValue = (String) apiKeyField.get(instance);
        System.out.println("Read private field value: " + originalValue);

        // 3. Mutate private field dynamically
        apiKeyField.set(instance, "updated_secret_123");

        // 4. Invoke private method
        Method method = clazz.getDeclaredMethod("printInternalDiagnostics", String.class);
        method.setAccessible(true);
        method.invoke(instance, "Manual Reflection Run");
    }
}
```

---

## 4. Custom Annotations: The Backbone of Spring Boot

Annotations are metadata tags evaluated either by the compiler (`SOURCE`), the classloader (`CLASS`), or the JVM at runtime via reflection (`RUNTIME`).

### Key Meta-Annotations

* `@Target`: Where the annotation can be placed (`ElementType.TYPE`, `METHOD`, `FIELD`, `PARAMETER`).
* `@Retention`: How long the annotation is retained:
  * `RetentionPolicy.SOURCE`: Discarded during compilation (e.g., `@Override`).
  * `RetentionPolicy.CLASS`: Stored in `.class` file, but discarded by the JVM (default).
  * `RetentionPolicy.RUNTIME`: Retained by the JVM at runtime and readable via Reflection (used by Spring).

### Implementing a Custom Runtime Annotation Engine

```java
package com.roadmap.reflection;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

// 1. Declare Custom Annotation
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface TransactionalAudit {
    String action();
    int retryLimit() default 3;
}
```

```java
package com.roadmap.reflection;

import java.lang.reflect.Method;

// 2. Target Component using the Annotation
public class PaymentPipeline {

    @TransactionalAudit(action = "CAPTURE_CHARGE", retryLimit = 5)
    public void capturePayment(String transactionId) {
        System.out.println("Payment captured: " + transactionId);
    }
}
```

```java
package com.roadmap.reflection;

import java.lang.reflect.Method;

// 3. The Runtime Engine (How Spring reads annotations internally)
public class AnnotationProcessorEngine {

    public static void main(String[] args) throws Exception {
        Class<?> targetClass = PaymentPipeline.class;

        for (Method method : targetClass.getDeclaredMethods()) {
            if (method.isAnnotationPresent(TransactionalAudit.class)) {
                TransactionalAudit audit = method.getAnnotation(TransactionalAudit.class);

                System.out.println("Discovered annotated method: " + method.getName());
                System.out.println(" -> Configured Action: " + audit.action());
                System.out.println(" -> Configured Retries: " + audit.retryLimit());

                // Invoke method dynamically
                Object instance = targetClass.getDeclaredConstructor().newInstance();
                method.invoke(instance, "TXN-77712");
            }
        }
    }
}
```

---

## 5. Key Takeaways for Magento / PHP Developers

1. **XML / Interceptors vs. Reflection & Annotations:** In Magento 2, metadata and plugins are wired via `di.xml` and class naming interceptor conventions (`aroundSave`, `beforeSave`) [cite: 1, 2]. In Spring, bytecode proxies inspect `@Aspect`, `@Transactional`, and custom annotations at startup using **Reflection** [cite: 1, 2, 3].
2. **File Paths & Cross-Platform Reliability:** In PHP, path resolution often uses strings (`/`, `DIRECTORY_SEPARATOR`). In Java, always use `java.nio.file.Path` and `Paths.get(...)` to ensure platform-independent path operations across Windows, macOS, and Linux containers.
