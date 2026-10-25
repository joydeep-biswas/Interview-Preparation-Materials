# 03. Modern Java: Records, Sealed Classes, Streams & Functional Programming

A comprehensive guide covering Java functional programming paradigms, Stream API, Optional semantics, String internals and immutability, and robust exception-handling architectures.

---

## Architectural Overview: Modern Java Language & Functional Pipeline Architecture

![Modern Java Language & Functional Pipeline Architecture](assets/03_modern_java_streams_and_records/01_modern_java_type_system_and_streams.svg)

*D2 Source: [`01_modern_java_type_system_and_streams.d2`](assets/03_modern_java_streams_and_records/01_modern_java_type_system_and_streams.d2)*

---

## Table of Contents
1. [Modern Java Language Features (Java 14 – 21+)](#1-modern-java-language-features-java-14--21)
   - [1.1 Java Records & Compact Constructors](#11-java-records--compact-constructors)
   - [1.2 Sealed Classes & Sealed Interfaces](#12-sealed-classes--sealed-interfaces)
   - [1.3 Pattern Matching (Type Patterns, Switch & Record Deconstruction)](#13-pattern-matching-type-patterns-switch--record-deconstruction)
   - [1.4 Text Blocks (`"""`)](#14-text-blocks-)
   - [1.5 Advanced Enums (State Machines & Specialized Collections)](#15-advanced-enums-state-machines--specialized-collections)
   - [1.6 Interface Evolution: Default & Static Methods](#16-interface-evolution-default--static-methods)
2. [Lambda & Functional Programming](#2-lambda--functional-programming)
3. [Stream API](#3-stream-api)
4. [`Optional` Type](#4-optional-type)
5. [String & Immutability](#5-string--immutability)
6. [Exception Handling](#6-exception-handling)
7. [Functional Error Handling & Railway-Oriented Programming](#7-functional-error-handling--railway-oriented-programming)
8. [Modern Java Performance, Escape Analysis & JIT Inlining](#8-modern-java-performance-escape-analysis--jit-inlining)
   - [8.1 Escape Analysis & Scalar Replacement](#81-escape-analysis--scalar-replacement)
   - [8.2 Stream Inlining vs. Primitive For-Loops](#82-stream-inlining-vs-primitive-for-loops)
9. [Production Troubleshooting Playbook](#9-production-troubleshooting-playbook)
10. [Summary Comparison & Production Engineering Reference Matrix](#10-summary-comparison--production-engineering-reference-matrix)
11. [References & Further Reading](#11-references--further-reading)

> [!TIP]
> **Deep-Dive Interview Scenario:**
> - [Q2. Method References in Java](frequently_asked_questions/03_modern_java_streams_and_records_faq.md#q2-easy-method-references-in-java)

---

# 1. Modern Java Language Features (Java 14 – 21+)

---

### 1.1 Java Records & Compact Constructors

Records (`java.lang.Record`, Java 16+) are immutable data carriers that eliminate traditional POJO and Lombok boilerplate:
* Fields are implicitly `private final`.
* The class is implicitly `final` and cannot extend another class.
* The compiler automatically generates: canonical constructor, accessors (`name()`, not `getName()`), `equals()`, `hashCode()`, and `toString()`.

```java
// 1. Record definition with compact constructor validation
public record OrderItem(String sku, int quantity, BigDecimal unitPrice) {
    // Compact Constructor: No parameter list; runs before field assignment
    public OrderItem {
        Objects.requireNonNull(sku, "SKU must not be null");
        if (quantity <= 0) {
            throw new IllegalArgumentException("Quantity must be positive: " + quantity);
        }
        if (unitPrice.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Unit price cannot be negative");
        }
        // Normalize SKU
        sku = sku.trim().toUpperCase();
    }

    // Custom helper method
    public BigDecimal calculateTotal() {
        return unitPrice.multiply(BigDecimal.valueOf(quantity));
    }
}
```

---

### 1.2 Sealed Classes & Sealed Interfaces

Sealed classes (Java 17+) restrict which classes or interfaces may extend or implement them, enabling domain modeling with strict compiler-enforced exhaustiveness:

```java
// Top-level sealed interface
public sealed interface PaymentMethod permits CreditCard, Upi, BankTransfer {}

public final class CreditCard implements PaymentMethod {
    private final String cardNumber;
    public CreditCard(String cardNumber) { this.cardNumber = cardNumber; }
}

public final class Upi implements PaymentMethod {
    private final String vpaAddress;
    public Upi(String vpaAddress) { this.vpaAddress = vpaAddress; }
}

public non-sealed class BankTransfer implements PaymentMethod {
    // non-sealed allows further open extension if desired
}
```

* **Constraints on Permitted Subclasses**:
  1. Must reside in the same package (or module).
  2. Must directly extend/implement the sealed type.
  3. Must explicitly declare one modifier: `final` (cannot be extended), `sealed` (extends hierarchy), or `non-sealed` (opens hierarchy).

---

### 1.3 Pattern Matching (Type Patterns, Switch & Record Deconstruction)

Java 21 provides unified Pattern Matching that eliminates boilerplate casting and enables clean functional data decomposition:

#### 1. Exhaustive Pattern Matching in Switch Expressions:
When matching against a `sealed` hierarchy, the compiler guarantees **exhaustiveness**. If every permitted subtype is covered, no `default` clause is required. If a developer later introduces a new permitted subclass, the compiler immediately flags every unhandled `switch` site with a compile error!

```java
public class PatternMatchingDemo {

    // Java 21 Pattern Matching for switch with guarded patterns (when)
    public static String getPaymentStatusDescription(PaymentMethod payment) {
        return switch (payment) {
            case CreditCard cc when cc.isExpired() -> "Credit Card expired: " + cc.getMaskedNumber();
            case CreditCard cc when cc.isOverLimit() -> "Credit Card over limit: " + cc.getMaskedNumber();
            case CreditCard cc -> "Card payment valid: " + cc.getMaskedNumber();
            case Upi upi -> "UPI payment via " + upi.getVpaAddress();
            case BankTransfer bt -> "Wire transfer pending for routing: " + bt.getRoutingNumber();
            // No default needed! Compiler proves exhaustiveness from 'sealed interface PaymentMethod'
        };
    }

    // Java 21 Record Patterns with Nested Deconstruction
    public record CustomerName(String firstName, String lastName) {}
    public record Address(String city, String country) {}
    public record Customer(CustomerName name, Address address) {}
    public record Order(String orderId, Customer customer, BigDecimal total) {}

    public static String extractShippingCity(Object obj) {
        // Nested pattern matching: deconstructs Order -> Customer -> Address in a single expression!
        if (obj instanceof Order(String id, Customer(CustomerName name, Address(String city, String country)), BigDecimal total)) {
            return "Shipping order " + id + " for " + name.firstName() + " to " + city + ", " + country;
        }
        return "Unknown entity";
    }
}
```

---

### 1.4 Text Blocks (`"""`)

Text blocks (Java 15+) simplify multi-line strings (SQL queries, JSON payloads, HTML templates) without messy string concatenation or escape characters:

```java
public class QueryRepository {
    // Multi-line formatted SQL with proper indentation stripping
    public static final String FIND_CUSTOMER_ORDERS_SQL = """
            SELECT o.id, o.order_number, o.total_amount, o.status
            FROM orders o
            JOIN customers c ON o.customer_id = c.id
            WHERE c.status = 'ACTIVE'
              AND o.created_at >= :sinceDate
            ORDER BY o.created_at DESC
            LIMIT :limit;
            """;

    // JSON Payload template with String.formatted()
    public static String buildWebhookJson(String eventId, String status) {
        return """
            {
                "eventId": "%s",
                "timestamp": "%s",
                "status": "%s"
            }
            """.formatted(eventId, Instant.now(), status);
    }
}
```

---

### 1.5 Advanced Enums (State Machines & Specialized Collections)

Java `enum` types are specialized classes that can encapsulate behavior, abstract methods, and drive state machines:

```java
public enum OrderStatus {
    PENDING {
        @Override public boolean canTransitionTo(OrderStatus next) {
            return next == PAID || next == CANCELLED;
        }
    },
    PAID {
        @Override public boolean canTransitionTo(OrderStatus next) {
            return next == SHIPPED || next == REFUNDED;
        }
    },
    SHIPPED {
        @Override public boolean canTransitionTo(OrderStatus next) {
            return next == DELIVERED;
        }
    },
    DELIVERED {
        @Override public boolean canTransitionTo(OrderStatus next) {
            return false;
        }
    },
    CANCELLED {
        @Override public boolean canTransitionTo(OrderStatus next) {
            return false;
        }
    },
    REFUNDED {
        @Override public boolean canTransitionTo(OrderStatus next) {
            return false;
        }
    };

    public abstract boolean canTransitionTo(OrderStatus next);
}
```

* **High-Performance Collections**:
  * `EnumSet`: Implemented internally as a 64-bit bit-vector (`long`), offering faster than $O(1)$ operations with zero heap allocation overhead for elements.
  * `EnumMap`: Implemented internally as a compact Java array indexed by the enum's ordinal, dramatically outperforming `HashMap` with zero hashing overhead.

---

### 1.6 Interface Evolution: Default & Static Methods

Java 8+ introduced concrete method implementations in interfaces:
* **`default` methods**: Enable backward-compatible API evolution (adding new capabilities to interfaces without breaking existing implementing classes).
* **`static` methods**: Utility helper methods directly attached to the interface type.
* **The Diamond Problem Resolution**: If a class implements two interfaces declaring identical default method signatures, the compiler requires the implementing class to explicitly override the method and resolve the conflict using `InterfaceName.super.method()`:

```java
interface Auditor {
    default void log(String msg) { System.out.println("Audit: " + msg); }
}

interface Logger {
    default void log(String msg) { System.out.println("Log: " + msg); }
}

class TransactionService implements Auditor, Logger {
    @Override
    public void log(String msg) {
        // Explicitly disambiguate diamond conflict
        Auditor.super.log(msg);
    }
}
```

---

# 2. Lambda & Functional Programming

---

### Lambda Expressions

#### Definition
A Lambda Expression is an anonymous function (a method without a name, return type, or explicit access modifier) that can be passed around as if it were a first-class value. Introduced in Java 8, it provides a concise syntax for implementing single-method interfaces.

#### Syntax Breakdown
```
(parameters) -> { body }
(a, b) -> a + b           // Expression body: Implicit return
(x) -> { doWork(x); }     // Block body: Explicit statement
```

#### Working Code Example
```java
import java.util.Arrays;
import java.util.Collections;
import java.util.List;

public class LambdaDemo {
    public static void main(String[] args) {
        List<String> names = Arrays.asList("Sophia", "Liam", "Noah", "Olivia");

        // Anonymous inner class approach (Pre-Java 8)
        Collections.sort(names, (s1, s2) -> s1.compareTo(s2)); // Lambda expression

        // Method reference shorthand
        names.forEach(System.out::println);
    }
}
```

**Primary Source Reference**: [GeeksforGeeks - Lambda Expressions in Java 8](https://www.geeksforgeeks.org/lambda-expressions-java-8/)

---

### Functional Interfaces

#### Definition
A Functional Interface is an interface that contains **exactly one abstract method (Single Abstract Method - SAM)**. It can contain any number of `default` or `static` methods. The `@FunctionalInterface` annotation informs the compiler to enforce this rule.

#### Visual Diagram
![Single Abstract Method (SAM) Contract & Functional Interfaces](assets/03_modern_java_streams_and_records/02_functional_interface_sam_contract.svg)

*D2 Source: [`02_functional_interface_sam_contract.d2`](assets/03_modern_java_streams_and_records/02_functional_interface_sam_contract.d2)*

#### Working Code Example
```java
@FunctionalInterface
public interface CryptoTransformer<T, R> {
    // Single Abstract Method
    R transform(T rawInput);

    // Default methods do not violate the SAM rule
    default void logTransformation() {
        System.out.println("Executing cryptographic transformation...");
    }
}

class FunctionalInterfaceUsage {
    public static void main(String[] args) {
        CryptoTransformer<String, String> reverser = input -> new StringBuilder(input).reverse().toString();
        reverser.logTransformation();
        System.out.println("Result: " + reverser.transform("SecureData123"));
    }
}
```

**Primary Source Reference**: [GeeksforGeeks - Functional Interfaces in Java](https://www.geeksforgeeks.org/functional-interfaces-java/)

---

### Core Built-in Functional Interfaces

Java 8 provides standard functional interfaces in `java.util.function`:

| Interface | Method Signature | Purpose | Real-World Use Case |
| :--- | :--- | :--- | :--- |
| **`Predicate<T>`** | `boolean test(T t)` | Evaluates a boolean condition | Filtering lists, input validation |
| **`Consumer<T>`** | `void accept(T t)` | Accepts input, performs side effect | Printing, emitting events, audit logging |
| **`Supplier<T>`** | `T get()` | Takes no input, produces a result | Factory methods, lazy loading |
| **`Function<T, R>`** | `R apply(T t)` | Transforms input of type T to R | Entity to DTO mapping |
| **`BiFunction<T, U, R>`**| `R apply(T t, U u)` | Combines two arguments into a result | Merging objects, calculating tax |
| **`UnaryOperator<T>`** | `T apply(T t)` | Special `Function<T, T>` | String trimming, incrementing values |
| **`BinaryOperator<T>`**| `T apply(T t1, T t2)`| Special `BiFunction<T, T, T>` | Reductions, summing numbers |

#### Working Code Example
```java
import java.util.function.*;

public class BuiltInFunctionalInterfacesDemo {
    public static void main(String[] args) {
        // 1. Predicate: Check if balance is positive
        Predicate<Double> isSolvent = bal -> bal > 0.0;

        // 2. Consumer: Log message
        Consumer<String> logger = msg -> System.out.println("[AUDIT] " + msg);

        // 3. Supplier: Generate transaction ID
        Supplier<String> txIdGen = () -> "TX-" + System.currentTimeMillis();

        // 4. Function: Transform String to Length
        Function<String, Integer> lengthMapper = String::length;

        // 5. BinaryOperator: Sum two numbers
        BinaryOperator<Integer> adder = Integer::sum;

        if (isSolvent.test(150.0)) {
            logger.accept("Generated Tx: " + txIdGen.get() + " with length: " + lengthMapper.apply("Success"));
            System.out.println("Total: " + adder.apply(100, 250));
        }
    }
}
```

**Primary Source Reference**: [GeeksforGeeks - Java 8 Functional Interfaces](https://www.geeksforgeeks.org/functional-interfaces-java/)

> [!TIP]
> **Deep-Dive Interview Scenario:**
> - [Q1. Functional Interfaces, `@FunctionalInterface`, Built-ins & `Predicate`](frequently_asked_questions/03_modern_java_streams_and_records_faq.md#q1-easy-functional-interfaces-functionalinterface-built-ins--predicate)

---

# 3. Stream API

---

### Stream Architecture

#### Definition & Core Concepts
In Java 8+, the Streams API (`java.util.stream.Stream`) introduced a functional programming paradigm over sequences of data elements. Unlike a collection, a `Stream` does **not** store elements in physical heap memory structures; instead, it models an active computational pipeline that carries values from an underlying source (such as a `Collection`, an array, a generator function, or an I/O channel) through a declarative sequence of intermediate operations to a terminal operation.

#### First-Principles Breakdown: Stream vs. Collection
1. **Physical Storage vs. Computational Pipelines:**
   - **Collections (`List`, `Set`, `Map`):** In-memory data structures. Every element must be fully computed and physically stored in RAM before the collection can exist. Adding 10 million records allocates heap space for 10 million node or wrapper instances.
   - **Streams:** Fixed-memory computational pipelines. A stream holds zero data storage. Elements flow through operations on-demand, enabling infinite streams (`Stream.iterate()`) and constant-memory processing over massive datasets.
2. **Evaluation Timing: Eager vs. Lazy Computation:**
   - **Collections:** Eagerly evaluated. Constructing or mutating a collection executes immediately.
   - **Streams:** Strictly **lazily evaluated**. Chaining intermediate operations (`filter()`, `map()`, `sorted()`) performs zero work and creates only pipeline descriptors. Data processing begins only when a **terminal operation** (`collect()`, `count()`, `forEach()`) is invoked, allowing the JVM to fuse multiple operations into a single pass over the data.
3. **Traversal Lifecycle & Reusability:**
   - **Collections:** Can be traversed, iterated, and inspected repeatedly over their application lifecycle.
   - **Streams:** **Single-use consumable pipelines**. Like an I/O iterator or iterator over a network socket, once a stream's terminal operation executes, the stream is closed and consumed. Attempting to invoke another operation on a consumed stream throws `IllegalStateException: stream has already been operated upon or closed`.

#### Stream vs Collection
| Dimension | Collection | Stream |
| :--- | :--- | :--- |
| **Data Storage** | Stores data in physical memory structures | Holds no storage; operates on demand |
| **Evaluation** | Eagerly evaluated | **Lazily evaluated** (executes on terminal call) |
| **Reusability** | Can be traversed multiple times | **Single use only** (traversing twice throws exception) |
| **Internal vs External** | External iteration (`for-each`) | Internal iteration managed by JVM runtime |

#### Visual Pipeline Architecture
![Java Stream Pipeline Lazy Evaluation & Sink Chaining Architecture](assets/03_modern_java_streams_and_records/03_stream_pipeline_lazy_sink_chaining.svg)

*D2 Source: [`03_stream_pipeline_lazy_sink_chaining.d2`](assets/03_modern_java_streams_and_records/03_stream_pipeline_lazy_sink_chaining.d2)*

**Primary Source Reference**: [GeeksforGeeks - Stream in Java](https://www.geeksforgeeks.org/stream-in-java/)

---

### Intermediate Operations

Intermediate operations return a new `Stream` and are **lazy**—computation is deferred until a terminal operation is invoked.

- `filter(Predicate<T>)`: Selects elements satisfying a boolean condition.
- `map(Function<T, R>)`: Transforms elements one-to-one into another type.
- `flatMap(Function<T, Stream<R>>)`: Flattens nested streams (one-to-many).
- `sorted()`: Sorts elements naturally or via `Comparator`.
- `distinct()`: Eliminates duplicates based on `equals()`.
- `limit(n)`: Truncates the stream to at most $n$ elements.
- `skip(n)`: Discards the first $n$ elements.

#### Working Code Example
```java
import java.util.Arrays;
import java.util.List;

public class IntermediateOpsDemo {
    public static void main(String[] args) {
        List<List<String>> nestedSkills = Arrays.asList(
            Arrays.asList("Java", "Spring", "Docker"),
            Arrays.asList("Java", "Kubernetes", "AWS")
        );

        List<String> uniqueSkills = nestedSkills.stream()
            .flatMap(List::stream)             // 1. Flatten nested lists
            .map(String::toUpperCase)          // 2. Transform to uppercase
            .distinct()                        // 3. Remove duplicates
            .sorted()                          // 4. Sort alphabetically
            .skip(1)                           // 5. Skip first
            .limit(3)                          // 6. Take next 3
            .toList();                         // Terminal operation (Java 16+)

        System.out.println("Processed Skills: " + uniqueSkills);
    }
}
```

**Primary Source Reference**: [GeeksforGeeks - Java 8 Stream Intermediate Operations](https://www.geeksforgeeks.org/intermediate-and-terminal-operations-in-java-stream/)

---

### Terminal Operations

Terminal operations trigger the stream pipeline execution and produce a non-stream result (primitive, collection, void).

- `collect(Collector)`: Gathers elements into a container (`List`, `Set`, `Map`).
- `reduce(BinaryOperator)`: Combines elements into a single aggregate value.
- `forEach(Consumer)`: Executes side effects on each element.
- `count()`: Returns the total number of elements.
- `findFirst()` / `findAny()`: Returns an `Optional` element matching conditions.
- `anyMatch()` / `allMatch()` / `noneMatch()`: Short-circuiting boolean checks.

#### Working Code Example
```java
import java.util.List;
import java.util.Optional;

public class TerminalOpsDemo {
    public static void main(String[] args) {
        List<Integer> numbers = List.of(10, 25, 30, 45, 50);

        // reduce: sum of elements
        int totalSum = numbers.stream().reduce(0, Integer::sum);
        System.out.println("Total Sum: " + totalSum);

        // anyMatch: short-circuiting check
        boolean hasLargeNumber = numbers.stream().anyMatch(n -> n > 40);
        System.out.println("Contains > 40: " + hasLargeNumber);

        // findFirst: retrieves first match wrapped in Optional
        Optional<Integer> firstEven = numbers.stream().filter(n -> n % 2 == 0).findFirst();
        firstEven.ifPresent(n -> System.out.println("First even: " + n));
    }
}
```

**Primary Source Reference**: [GeeksforGeeks - Java 8 Stream Terminal Operations](https://www.geeksforgeeks.org/intermediate-and-terminal-operations-in-java-stream/)

---

### Collectors Advanced

`java.util.stream.Collectors` provides powerful accumulation strategies:
- `groupingBy`: Partitions elements into a `Map<K, List<V>>` based on a classification function.
- `partitioningBy`: Special grouping into a `Map<Boolean, List<V>>` based on a `Predicate`.
- `mapping`: Adapts a collector to accept elements through a transformation function.

#### Working Code Example
```java
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

public class CollectorsAdvancedDemo {
    record Employee(String name, String department, double salary) {}

    public static void main(String[] args) {
        List<Employee> employees = List.of(
            new Employee("Alice", "Engineering", 90000),
            new Employee("Bob", "Engineering", 80000),
            new Employee("Carol", "Marketing", 70000),
            new Employee("David", "Marketing", 65000),
            new Employee("Eve", "HR", 60000)
        );

        // 1. Group employees by Department
        Map<String, List<Employee>> byDept = employees.stream()
            .collect(Collectors.groupingBy(Employee::department));
        System.out.println("By Department: " + byDept.keySet());

        // 2. Average salary per department
        Map<String, Double> avgSalaryPerDept = employees.stream()
            .collect(Collectors.groupingBy(
                Employee::department,
                Collectors.averagingDouble(Employee::salary)
            ));
        System.out.println("Average Salaries: " + avgSalaryPerDept);

        // 3. Partition by high earners (> 75,000)
        Map<Boolean, List<String>> partitioned = employees.stream()
            .collect(Collectors.partitioningBy(
                e -> e.salary() > 75000,
                Collectors.mapping(Employee::name, Collectors.toList())
            ));
        System.out.println("High Earners: " + partitioned.get(true));
    }
}
```

**Primary Source Reference**: [GeeksforGeeks - Collectors in Java](https://www.geeksforgeeks.org/collectors-in-java/)

---

### Parallel Streams

#### Definition & Mechanism
Parallel streams decompose tasks across multiple CPU cores using the **ForkJoinPool.commonPool()**. Elements are recursively split into chunks (via `Spliterator`), processed independently, and merged.

![Parallel Stream Execution: Fork-Join Spliterator & Work-Stealing](assets/03_modern_java_streams_and_records/04_parallel_stream_fork_join_spliterator.svg)

*D2 Source: [`04_parallel_stream_fork_join_spliterator.d2`](assets/03_modern_java_streams_and_records/04_parallel_stream_fork_join_spliterator.d2)*

#### When to Use vs Avoid
- **Use When**: The dataset is massive ($N \times Q > 10,000$, where $N$ is element count and $Q$ is work per element), operations are CPU-bound and computationally intensive, and elements are independent.
- **Avoid When**:
  - Operations involve blocking I/O (database, HTTP calls)—this starves the shared `ForkJoinPool.commonPool()`.
  - Collections have poor splitting characteristics (e.g., `LinkedList` has $O(n)$ splitting; `ArrayList` has $O(1)$ splitting).
  - Preserving sequential encounter order matters.

**Primary Source Reference**: [GeeksforGeeks - Parallel Streams in Java](https://www.geeksforgeeks.org/parallel-streams-in-java/)

---

### Custom Spliterators & Custom Collectors Deep-Dive

#### 1. Implementing a Custom `Spliterator<T>` for Parallel Chunked Processing:
A `Spliterator` (Splitable Iterator) controls how streams partition data across CPU threads in `ForkJoinPool`:
* `tryAdvance(Consumer<? super T> action)`: Steps forward one element. Returns `true` if an element existed, `false` otherwise.
* `trySplit()`: Divides the remaining elements into two partitions. Returns a new `Spliterator` covering roughly half the workload, while `this` retains the rest. If the remaining size is below a threshold or cannot be partitioned, returns `null` to switch to sequential execution.
* `estimateSize()`: Estimated elements remaining.
* `characteristics()`: Bitmask flags (`ORDERED`, `DISTINCT`, `SORTED`, `SIZED`, `NONNULL`, `IMMUTABLE`, `CONCURRENT`, `SUBSIZED`).

```java
package com.enterprise.streams;

import java.util.Spliterator;
import java.util.function.Consumer;
import java.util.stream.Stream;
import java.util.stream.StreamSupport;

public class FixedBatchSpliterator<T> implements Spliterator<T> {

    private final T[] elements;
    private int origin;
    private final int fence;
    private static final int BATCH_THRESHOLD = 500;

    public FixedBatchSpliterator(T[] elements, int origin, int fence) {
        this.elements = elements;
        this.origin = origin;
        this.fence = fence;
    }

    @Override
    public boolean tryAdvance(Consumer<? super T> action) {
        if (origin < fence) {
            action.accept(elements[origin++]);
            return true;
        }
        return false;
    }

    @Override
    public Spliterator<T> trySplit() {
        int remaining = fence - origin;
        if (remaining <= BATCH_THRESHOLD) {
            return null; // Small chunk: do not split further, execute sequentially on current core!
        }
        int mid = origin + (remaining / 2);
        int oldOrigin = origin;
        origin = mid; // This spliterator now covers [mid, fence)
        return new FixedBatchSpliterator<>(elements, oldOrigin, mid); // Sibling covers [oldOrigin, mid)
    }

    @Override
    public long estimateSize() {
        return fence - origin;
    }

    @Override
    public int characteristics() {
        return ORDERED | SIZED | SUBSIZED | IMMUTABLE;
    }

    public static <E> Stream<E> toParallelStream(E[] array) {
        return StreamSupport.stream(new FixedBatchSpliterator<>(array, 0, array.length), true);
    }
}
```

#### 2. Building a Custom `Collector<T, A, R>`:
When standard collectors (`toList`, `groupingBy`) are insufficient, `Collector.of()` constructs custom accumulation pipelines:

```java
package com.enterprise.streams;

import java.util.*;
import java.util.stream.Collector;

public class CustomCollectors {

    // Custom Collector: Gathers top-N elements into a bounded PriorityQueue
    public static <T> Collector<T, ?, List<T>> topN(int n, Comparator<T> comparator) {
        return Collector.of(
            // Supplier: Accumulator container
            () -> new PriorityQueue<T>(n, comparator),
            // Accumulator: Add element and evict excess
            (queue, item) -> {
                queue.offer(item);
                if (queue.size() > n) {
                    queue.poll(); // Evict smallest element
                }
            },
            // Combiner: Merge two priority queues (for parallel streams)
            (q1, q2) -> {
                while (!q2.isEmpty()) {
                    q1.offer(q2.poll());
                    if (q1.size() > n) {
                        q1.poll();
                    }
                }
                return q1;
            },
            // Finisher: Convert PriorityQueue into sorted List
            queue -> {
                List<T> result = new ArrayList<>(queue);
                result.sort(comparator.reversed());
                return Collections.unmodifiableList(result);
            },
            Collector.Characteristics.UNORDERED
        );
    }
}
```

> [!TIP]
> **Deep-Dive Interview Scenarios:**
> - [Q4. Streams vs Collections & Internal Pipeline Mechanics](frequently_asked_questions/03_modern_java_streams_and_records_faq.md#q4-medium-streams-vs-collections--internal-pipeline-mechanics)
> - [Q5. `map()` vs `flatMap()`](frequently_asked_questions/03_modern_java_streams_and_records_faq.md#q5-medium-map-vs-flatmap)
> - [Q6. Parallel Streams & `ForkJoinPool` Concurrency](frequently_asked_questions/03_modern_java_streams_and_records_faq.md#q6-hard-parallel-streams--forkjoinpool-concurrency)
> - [Q7. String Filtering & Sorting with Stream API](frequently_asked_questions/03_modern_java_streams_and_records_faq.md#q7-medium-string-filtering--sorting-with-stream-api)
> - [Q8. Counting Vowels in a String Using Streams](frequently_asked_questions/03_modern_java_streams_and_records_faq.md#q8-medium-counting-vowels-in-a-string-using-streams)
> - [Q9. Finding Second Highest Integer in an Unsorted List](frequently_asked_questions/03_modern_java_streams_and_records_faq.md#q9-medium-finding-second-highest-integer-in-an-unsorted-list)
> - [Q10. Finding the Highest / Maximum Salary in Each Department Using Java Streams](frequently_asked_questions/03_modern_java_streams_and_records_faq.md#q10-medium-finding-the-highest--maximum-salary-in-each-department-using-java-streams)
> - [Q11. Group Employees into Dynamic Age Ranges Generically Using Streams](frequently_asked_questions/03_modern_java_streams_and_records_faq.md#q11-medium-group-employees-into-dynamic-age-ranges-generically-using-streams)
> - [Q12. Find the First Non-Repeating (Unique) Character in a String Using Stream API](frequently_asked_questions/03_modern_java_streams_and_records_faq.md#q12-medium-find-the-first-non-repeating-unique-character-in-a-string-using-stream-api)
> - [Q13. Character Frequencies, Duplicate Extraction & Special Character Counting via Stream API](frequently_asked_questions/03_modern_java_streams_and_records_faq.md#q13-medium-character-frequencies-duplicate-extraction--special-character-counting-via-stream-api)
> - [Q14. Stream Reduction vs Mutable Container Accumulation: `reduce()` vs `collect()` Architectural Dynamics](frequently_asked_questions/03_modern_java_streams_and_records_faq.md#q14-medium-stream-reduction-vs-mutable-container-accumulation-reduce-vs-collect-architectural-dynamics)

---

# 4. `Optional` Type

---

### `Optional` Fundamentals

#### Definition
`Optional<T>` is a container object introduced in Java 8 that may or may not contain a non-null value. It was designed primarily as a **method return type** to explicitly communicate the possibility of "no result" without resorting to returning `null` (preventing `NullPointerException`).

#### Core Methods
- `Optional.of(val)`: Creates Optional; throws NPE if `val` is null.
- `Optional.ofNullable(val)`: Safe creation; returns `Optional.empty()` if null.
- `map(Function)`: Transforms contained value if present.
- `flatMap(Function)`: Flattens an operation returning another `Optional`.
- `orElse(defaultValue)`: Returns value or fallback immediately.
- `orElseGet(Supplier)`: **Lazy evaluation**: Only computes default if absent.
- `orElseThrow(Supplier)`: Throws exception if empty.

#### Working Code Example
```java
import java.util.Optional;

public class OptionalDemo {
    public static Optional<String> findUserEmail(String userId) {
        if ("USR-101".equals(userId)) {
            return Optional.of("john.doe@company.com");
        }
        return Optional.empty();
    }

    public static void main(String[] args) {
        String email = findUserEmail("USR-101")
            .map(String::toUpperCase)
            .orElseGet(() -> "default@company.com"); // Lazy fallback evaluation
        System.out.println("Resolved email: " + email);

        // Safe exception throwing
        try {
            findUserEmail("NON-EXISTENT")
                .orElseThrow(() -> new IllegalArgumentException("User not found in registry"));
        } catch (IllegalArgumentException e) {
            System.out.println("Caught expected error: " + e.getMessage());
        }
    }
}
```

**Primary Source Reference**: [GeeksforGeeks - Java 8 Optional Class](https://www.geeksforgeeks.org/java-8-optional-class/)

---

### Optional Best Practices

#### Anti-Patterns vs Proper Usage
| Anti-Pattern | Correct Modern Idiom |
| :--- | :--- |
| `if (opt.isPresent()) { opt.get(); }` | Use `opt.ifPresent(v -> ...)` or `opt.map(...).orElse(...)` |
| Using `Optional` for class fields / entity attributes | Use standard nullable fields; Optional is not `Serializable` |
| Passing `Optional` as a method argument | Use standard method overloading or null checks |
| Using `orElse(computeExpensive())` | Use `orElseGet(() -> computeExpensive())` for lazy evaluation |
| Wrapping Collections: `Optional<List<T>>` | Return an empty `List<T>` (`Collections.emptyList()`) directly |

**Primary Source Reference**: [GeeksforGeeks - Best Practices for Java Optional](https://www.geeksforgeeks.org/best-practices-for-using-optional-in-java/)

> [!TIP]
> **Deep-Dive Interview Scenario:**
> - [Q3. `Optional` Architecture and Best Practices for NPE Prevention](frequently_asked_questions/03_modern_java_streams_and_records_faq.md#q3-easy-optional-architecture-and-best-practices-for-npe-prevention)

---

# 5. String & Immutability

---

### String Pool & Immutability

#### Definition
`java.lang.String` objects in Java are **immutable**—once created, their character sequence cannot be changed. The JVM optimizes string storage using the **String Constant Pool (SCP)**, a specialized memory area located inside the Heap.

#### Visual Memory Layout: Literal vs `new String()`
![Java String Constant Pool & Memory Layout Architecture](assets/03_modern_java_streams_and_records/05_string_pool_interning_memory_layout.svg)

*D2 Source: [`05_string_pool_interning_memory_layout.d2`](assets/03_modern_java_streams_and_records/05_string_pool_interning_memory_layout.d2)*

#### Why Strings are Immutable
1. **Security**: Strings store database credentials, file paths, and network URLs. Immutability prevents malicious tampering.
2. **Thread Safety**: Immutable objects are inherently thread-safe without synchronization.
3. **Caching & HashCode**: Hash code is cached upon creation (`hash` field); essential for fast `HashMap` lookups.
4. **String Pooling**: Allows multiple references to point to the same memory instance, saving heap space.

Immutability makes sharing a `String` between threads safe; mutable shared state still needs coordination. See [Topic 04: Java Concurrency & Multithreading](./04_java_concurrency_and_multithreading.md#2-synchronization--thread-safety).

**Primary Source Reference**: [GeeksforGeeks - String Constant Pool in Java](https://www.geeksforgeeks.org/string-constant-pool-in-java/)

---

### `StringBuilder` vs `StringBuffer`

#### Memory Mechanics of Mutable String Sequences
Strings in Java are immutable character sequences backed by contiguous byte arrays (compact Latin-1/UTF-16 encoding since Java 9). Performing repeated string concatenation inside loops (`s += "chunk"`) triggers catastrophic performance degradation: each iteration allocates a brand-new `String` heap object, copies all existing bytes, and discards the predecessor, resulting in quadratic $O(N^2)$ memory copying overhead and severe garbage collection pressure.

To eliminate repeated heap allocation during string assembly, Java provides mutable character buffers:

1. **`StringBuilder` (Introduced in Java 5):**
   - **Internal Mechanics:** Maintains a dynamically resizable contiguous buffer (`byte[] value` in Java 9+). As characters are appended, the buffer expands via over-allocation (`newCapacity = (oldCapacity << 1) + 2`), achieving amortized $O(1)$ appends without creating intermediate objects.
   - **Thread Model:** **Unsynchronized / Non-thread-safe**. Eliminates all synchronization locks, CPU memory barriers, and monitor contention. It is the absolute highest-performance choice for local method-scope string formatting and serialization.

2. **`StringBuffer` (Legacy Java 1.0):**
   - **Internal Mechanics:** Uses an identical expandable array buffer design to `StringBuilder`, but every public method (`synchronized append()`, `synchronized toString()`) acquires the object's intrinsic monitor lock.
   - **Production Reality:** Thread-safe, but rarely appropriate in modern architectures. High-throughput server applications rarely share a single mutable string buffer across concurrent threads. If a thread-local buffer is used, `StringBuffer` incurs wasteful locking overhead, making `StringBuilder` the universal modern default.

| Feature | `String` | `StringBuilder` (Java 5) | `StringBuffer` (Java 1.0) |
| :--- | :--- | :--- | :--- |
| **Immutability** | Immutable | Mutable | Mutable |
| **Thread Safety** | Thread-safe | Not thread-safe | Thread-safe (`synchronized` methods) |
| **Performance** | Slow for concatenation loops | **Fastest** (No locking overhead) | Slower due to lock contention |
| **Memory** | Allocates new object per edit | Modifies internal char/byte buffer | Modifies internal char/byte buffer |

**Primary Source Reference**: [GeeksforGeeks - StringBuilder vs StringBuffer](https://www.geeksforgeeks.org/stringbuffer-vs-stringbuilder-in-java/)

---

### String Intern & Concatenation

#### `String.intern()`
When invoked, `intern()` checks the String Pool. If the pool contains an equal string, the pool instance is returned; otherwise, the string is added to the pool and returned.

#### Concatenation Internals
- Pre-Java 9: `s1 + s2` was compiled into `new StringBuilder().append(s1).append(s2).toString()`.
- Java 9+: Uses `invokedynamic` with `StringConcatFactory.makeConcatWithConstants()`, enabling the JVM to optimize string assembly dynamically at runtime without allocating intermediate `StringBuilder` instances.

**Primary Source Reference**: [GeeksforGeeks - String intern() in Java](https://www.geeksforgeeks.org/string-intern-method-in-java/)

---

### `char[]` vs `String` for Passwords

#### Why Security Guidelines Mandate `char[]`
1. **Garbage Collection Delay**: `String` remains in heap memory indefinitely until GC runs. An attacker obtaining a memory dump can extract plaintext passwords.
2. **Deterministic Zeroing**: With a `char[]`, the application can immediately overwrite the memory with zeroes (`Arrays.fill(pwd, '0')`) as soon as authentication finishes.
3. **Log Prevention**: Strings are easily logged accidentally via `toString()`.

```java
import java.util.Arrays;

public class PasswordSecurityDemo {
    public static void main(String[] args) {
        char[] password = new char[]{'S', 'e', 'c', 'u', 'r', 'e', '!', '9'};

        try {
            // Perform authentication with char[]
            System.out.println("Authenticated successfully.");
        } finally {
            // Immediate zeroing of sensitive memory
            Arrays.fill(password, '0');
        }
    }
}
```

**Primary Source Reference**: [GeeksforGeeks - Why char array is preferred over String for passwords in Java](https://www.geeksforgeeks.org/why-char-array-is-preferred-over-string-for-storing-passwords-in-java/)

---

### Immutable Classes & Defensive Copying

#### Rules to Create a Truly Immutable Class
1. Declare the class `final` (prevents subclassing).
2. Make all fields `private` and `final`.
3. Do not provide any setter methods.
4. Perform **Defensive Copying** on mutable constructor parameters and mutable getter returns.

#### Working Code Example
```java
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public final class ImmutableUserProfile {
    private final String username;
    private final List<String> roles; // Mutable internal object

    public ImmutableUserProfile(String username, List<String> roles) {
        this.username = username;
        // Defensive Copy on input: prevents caller from mutating internal state
        this.roles = new ArrayList<>(roles);
    }

    public String getUsername() { return username; }

    public List<String> getRoles() {
        // Defensive Copy or Unmodifiable wrapper on output
        return Collections.unmodifiableList(roles);
    }
}
```

**Primary Source Reference**: [GeeksforGeeks - How to create an Immutable Class in Java](https://www.geeksforgeeks.org/create-immutable-class-java/)

---

# 6. Exception Handling

---

### Exception Hierarchy

#### Visual Hierarchy
![Java Throwable & Exception Class Hierarchy](assets/03_modern_java_streams_and_records/06_throwable_exception_hierarchy.svg)

*D2 Source: [`06_throwable_exception_hierarchy.d2`](assets/03_modern_java_streams_and_records/06_throwable_exception_hierarchy.d2)*

**Primary Source Reference**: [GeeksforGeeks - Exceptions in Java](https://www.geeksforgeeks.org/exceptions-in-java/)

---

### Checked vs Unchecked Exceptions

#### Exception Handling Architecture & Compiler Contracts
In Java, exceptions inherit from `java.lang.Throwable` and split into three architectural categories representing different severity levels and fault boundaries:

1. **Checked Exceptions (`Exception` Subclasses excluding `RuntimeException`):**
   - **Philosophical Intent:** Represent anticipated, recoverable environmental failures outside application control (e.g., file not found on disk, network socket closed by peer, database query deadlock).
   - **Compiler Enforcement:** The Java compiler (`javac`) strictly enforces the **Catch or Declare rule**. Any method calling code that throws a checked exception must either handle it within a `try-catch` block or declare it explicitly in its method signature via `throws`.
   - **Production Nuance:** While designed for fault tolerance, excessive checked exceptions clutter codebases with boilerplate and violate abstraction boundaries in modern functional and reactive frameworks (e.g., Lambdas and Streams cannot throw checked exceptions without cumbersome wrapping).

2. **Unchecked Exceptions (`RuntimeException` and Subclasses):**
   - **Philosophical Intent:** Represent programmer defects, logical bugs, and precondition contract violations (e.g., dereferencing a null pointer, array index out of bounds, invalid method argument).
   - **Compiler Enforcement:** Not checked at compile-time. Methods are not required to catch or declare them. Modern enterprise frameworks (Spring Boot, Hibernate, Quarkus) convert low-level checked exceptions into unchecked domain exceptions (e.g., `DataAccessException`) to keep business logic clean.

3. **System Errors (`Error` and Subclasses):**
   - **Philosophical Intent:** Represent catastrophic system-level hardware or JVM failures (e.g., `OutOfMemoryError`, `StackOverflowError`, `InternalError`). Application code should **never** attempt to catch or recover from `Error`—the JVM environment itself is in an undefined or unrecoverable state.

| Feature | Checked Exceptions | Unchecked Exceptions (`RuntimeException`) | `Error` |
| :--- | :--- | :--- | :--- |
| **Subclass Of** | `Exception` (excluding `RuntimeException`) | `RuntimeException` | `Error` |
| **Compiler Enforced**| **Yes**: Must declare (`throws`) or catch | **No**: Optional to catch | **No**: Should not attempt to catch |
| **Nature** | Foreseeable environmental failures | Programmer bugs, contract violations | Serious JVM system crashes |
| **Examples** | `IOException`, `SQLException` | `NullPointerException`, `IndexOutOfBoundsException` | `OutOfMemoryError`, `StackOverflowError` |

**Primary Source Reference**: [GeeksforGeeks - Checked vs Unchecked Exceptions in Java](https://www.geeksforgeeks.org/checked-vs-unchecked-exceptions-in-java/)

---

### Try-With-Resources

#### Definition
Introduced in Java 7, try-with-resources automatically closes any resource that implements `java.lang.AutoCloseable` at block exit, eliminating resource leaks and verbose `finally` blocks.

#### Working Code Example
```java
import java.io.BufferedReader;
import java.io.FileReader;
import java.io.IOException;

public class TryWithResourcesDemo {
    public static void readFirstLine(String path) {
        // Multiple AutoCloseable resources automatically closed in reverse order of declaration
        try (BufferedReader br = new BufferedReader(new FileReader(path))) {
            System.out.println("First line: " + br.readLine());
        } catch (IOException e) {
            System.err.println("File read failed: " + e.getMessage());
        }
        // No explicit finally required!
    }
}
```

**Primary Source Reference**: [GeeksforGeeks - Try with resources in Java](https://www.geeksforgeeks.org/try-with-resources-feature-in-java/)

---

### Custom Exceptions & Chaining

#### Exception Chaining
Wrapping an underlying low-level technical exception (`SQLException`) inside a high-level business exception (`PaymentProcessingException`) while preserving the original root cause stack trace via `super(message, cause)`.

#### Working Code Example
```java
// Custom Unchecked Business Exception
class InsufficientFundsException extends RuntimeException {
    private final double deficit;

    public InsufficientFundsException(String message, double deficit, Throwable cause) {
        super(message, cause); // Preserving root cause chain
        this.deficit = deficit;
    }

    public double getDeficit() { return deficit; }
}

public class ExceptionChainingDemo {
    public static void withdraw(double balance, double amount) {
        if (amount > balance) {
            throw new InsufficientFundsException(
                "Withdrawal exceeds balance",
                amount - balance,
                new IllegalStateException("Account balance below floor threshold")
            );
        }
    }

    public static void main(String[] args) {
        try {
            withdraw(50.0, 150.0);
        } catch (InsufficientFundsException e) {
            System.err.println("Business Error: " + e.getMessage() + " | Deficit: $" + e.getDeficit());
            System.err.println("Underlying Root Cause: " + e.getCause());
        }
    }
}
```

**Primary Source Reference**: [GeeksforGeeks - Chained Exceptions in Java](https://www.geeksforgeeks.org/chained-exception-in-java/)

---

### `throw` vs `throws`

- **`throw`**: Keyword used inside a method body to explicitly trigger/raise an exception instance (`throw new IllegalArgumentException();`).
- **`throws`**: Keyword in a method signature indicating that the method may propagate specified checked exceptions to its caller (`public void read() throws IOException`).

**Primary Source Reference**: [GeeksforGeeks - Difference between throw and throws in Java](https://www.geeksforgeeks.org/difference-between-throw-and-throws-in-java/)

---

### Multiple Catch Blocks & Multi-Catch Syntax

#### Definition
Java permits catching multiple distinct exception types either in separate `catch` blocks or grouped together in a single `catch` block using the **Multi-Catch pipe operator (`|`)** introduced in Java 7.

#### Working Code Example
```java
public class MultiCatchDemo {
    public static void executeOperation(String input) {
        try {
            int num = Integer.parseInt(input);
            int result = 100 / num;
            System.out.println("Result: " + result);
        } catch (NumberFormatException | ArithmeticException e) {
            // Multi-catch: 'e' is implicitly final
            System.err.println("Input computation error: " + e.getClass().getSimpleName() + " - " + e.getMessage());
        } catch (Exception e) {
            // Generic fallback catch block placed last
            System.err.println("Unexpected failure: " + e.getMessage());
        }
    }

    public static void main(String[] args) {
        executeOperation("invalid"); // Caught by multi-catch (NumberFormatException)
        executeOperation("0");       // Caught by multi-catch (ArithmeticException)
    }
}
```

**Primary Source Reference**: [GeeksforGeeks - Multiple Catch Blocks in Java](https://www.geeksforgeeks.org/multicatch-in-java/)

---

### Exception Propagation

#### Definition
When an exception is thrown inside a method, if not caught locally, it drops down the method call stack to the previous calling method. This unwinding continues until an appropriate `catch` handler is encountered or it reaches the JVM `main()` thread entry point, terminating the thread.

#### Visual Call Stack Unwinding
![JVM Call Stack Unwinding & Exception Propagation Mechanics](assets/03_modern_java_streams_and_records/07_call_stack_unwinding_exception_flow.svg)

*D2 Source: [`07_call_stack_unwinding_exception_flow.d2`](assets/03_modern_java_streams_and_records/07_call_stack_unwinding_exception_flow.d2)*

**Primary Source Reference**: [GeeksforGeeks - Exception Propagation in Java](https://www.geeksforgeeks.org/exception-propagation-in-java/)

---

### Business vs Technical Exceptions

| Dimension | Technical Exception | Business / Domain Exception |
| :--- | :--- | :--- |
| **Origin** | System infrastructure, networks, databases | Domain invariant or business rule violation |
| **Examples** | `SQLException`, `IOException`, `SocketTimeoutException` | `InsufficientFundsException`, `ProductOutOfStockException` |
| **Target Audience** | System administrators & DevOps engineers | End users (mapped to user-friendly UI messages) |
| **Recovery** | Retry, failover, circuit breaking | User correction (e.g. enter valid card or lower quantity) |
| **HTTP Status Code** | `500 Internal Server Error`, `502 Bad Gateway` | `400 Bad Request`, `404 Not Found`, `422 Unprocessable` |

#### Working Code Example: Handling Business vs Technical Failures
```java
public class OrderCheckoutWorkflow {
    public void checkout(String cartId) {
        try {
            // Domain validation
            validateInventory(cartId);
            // Low-level network call
            callPaymentGateway();
        } catch (ProductOutOfStockException e) {
            // Handled as Business Error -> Return HTTP 422 to user
            System.out.println("[USER ALERT] Item out of stock. Please adjust cart.");
        } catch (NetworkGatewayTimeoutException e) {
            // Handled as Technical Error -> Trigger retry or return HTTP 503
            System.err.println("[DEVOPS ALERT] Downstream payment gateway unreachable. Initiating retry.");
        }
    }

    private void validateInventory(String cartId) {}
    private void callPaymentGateway() {}
}

class ProductOutOfStockException extends RuntimeException {}
class NetworkGatewayTimeoutException extends RuntimeException {}
```

---

### Exception Best Practices

1. **Never swallow exceptions**: Avoid empty catch blocks (`catch (Exception e) {}`). Always log with context or rethrow.
2. **Favor Unchecked Exceptions for Business Logic**: Modern enterprise architectures (Spring) use unchecked runtime exceptions to avoid polluting API signatures.
3. **Catch Specific Exceptions First**: Order catch blocks from most specific subclass to most general (`catch (FileNotFoundException e)` before `catch (IOException e)`).
4. **Preserve the Cause**: Always pass the caught exception as the `cause` parameter when wrapping exceptions (`new BusinessException("msg", e)`).
5. **Use Standard Exceptions First**: Prefer `IllegalArgumentException`, `IllegalStateException`, and `NullPointerException` before creating custom classes.

> [!TIP]
> In enterprise web APIs, domain and technical exceptions are centralized and mapped to standard HTTP response bodies (RFC 9457 Problem Details) using `@RestControllerAdvice`. See [05. Spring Framework, Spring Boot & REST APIs - Section 15: API Error Handling & Resilience](05_spring_boot_and_rest_apis.md#15-api-error-handling--resilience).

---

# 7. Functional Error Handling & Railway-Oriented Programming

Traditional Java error handling relies on throwing exceptions. However, throwing exceptions carries heavy performance and architectural penalties:
1. **Stack Trace Cost**: `Throwable.fillInStackTrace()` traverses the native execution stack, consuming hundreds of CPU cycles per throw.
2. **Control Flow Hijacking**: Exceptions break local control flow like an arbitrary `goto`, obscuring the caller's reasoning.
3. **Hidden Invariants**: Unchecked exceptions are invisible in the method signature, leading to uncaught runtime failures.

**Railway-Oriented Programming (ROP)** treats execution as a two-track railway: a **Success Track** and a **Failure Track**. Operations transform the success value or automatically bypass subsequent steps if a failure occurs.

![Functional Error Handling & Railway-Oriented Programming (ROP)](assets/03_modern_java_streams_and_records/08_railway_oriented_programming_functional_error.svg)

*D2 Source: [`08_railway_oriented_programming_functional_error.d2`](assets/03_modern_java_streams_and_records/08_railway_oriented_programming_functional_error.d2)*

#### Production Code: The `Result<T, E>` Monad via Sealed Interfaces
```java
import java.util.Objects;
import java.util.function.Consumer;
import java.util.function.Function;

// Sealed Monadic Result Carrier
public sealed interface Result<T, E> permits Result.Success, Result.Failure {
    record Success<T, E>(T value) implements Result<T, E> {
        public Success { Objects.requireNonNull(value); }
    }
    record Failure<T, E>(E error) implements Result<T, E> {
        public Failure { Objects.requireNonNull(error); }
    }

    // Static factories
    static <T, E> Result<T, E> ok(T value) { return new Success<>(value); }
    static <T, E> Result<T, E> err(E error) { return new Failure<>(error); }

    // Monadic bind (flatMap)
    default <R> Result<R, E> flatMap(Function<? super T, Result<R, E>> mapper) {
        return switch (this) {
            case Success<T, E> s -> mapper.apply(s.value());
            case Failure<T, E> f -> Result.err(f.error());
        };
    }

    // Functor map
    default <R> Result<R, E> map(Function<? super T, ? extends R> mapper) {
        return flatMap(val -> Result.ok(mapper.apply(val)));
    }

    // Pattern matching fold / match
    default <R> R fold(Function<? super T, ? extends R> onSuccess, Function<? super E, ? extends R> onFailure) {
        return switch (this) {
            case Success<T, E> s -> onSuccess.apply(s.value());
            case Failure<T, E> f -> onFailure.apply(f.error());
        };
    }
}
```

---

# 8. Modern Java Performance, Escape Analysis & JIT Inlining

Java's HotSpot JVM employs sophisticated Just-In-Time (JIT) compiler optimizations that make modern functional idioms (Records, Lambdas, Streams) achieve near-native execution speeds.

### 8.1 Escape Analysis & Scalar Replacement
When an object is instantiated, the C2 JIT compiler analyzes its lifetime to determine if its reference "escapes" outside the allocating method or thread:
1. **GlobalEscape**: Object escapes the method and thread (e.g. stored in a static field, returned to caller). Must be allocated on the heap.
2. **ArgEscape**: Object is passed as an argument to another method but does not outlive the calling method.
3. **NoEscape**: Object is strictly local to the method.
   - **Scalar Replacement**: If an object does not escape, the JIT deconstructs the object into its constituent primitive and pointer fields, allocating them directly into **CPU Registers** or on the **Thread Stack**.
   - **Zero GC Allocation**: Small records or closure objects created in streams produce zero Garbage Collection pressure when scalar-replaced.

### 8.2 Stream Inlining vs. Primitive For-Loops

While Stream code is expressive, understanding its mechanical sympathy is crucial for sub-microsecond low-latency services:

| Dimension | Classic Primitive `for` Loop | Java Stream Pipeline |
| :--- | :--- | :--- |
| **Object Allocation** | Zero heap allocation (operates directly on stack frames) | Intermediate pipeline stages allocate `Stream`, `Spliterator`, and functional interface instances (unless scalar-replaced by C2 JIT). |
| **Primitive Boxing** | Keeps primitives unboxed (`long`, `double`) | `Stream<Integer>` incurs $16\text{B}$ `Integer` wrapper boxing. Always prefer `IntStream`, `LongStream`, or `DoubleStream`. |
| **JIT Optimization** | Trivial loop unrolling, SIMD auto-vectorization | Requires C2 monomorphic call-site inlining through nested lambda chains. |
| **Optimal Use Case** | Ultra-hot computational loops ($>10^7$ iterations/sec), hardware packet processing | Business logic, complex queries, concurrent collections, declarative ETL. |

---

# 9. Production Troubleshooting Playbook

### Incident 1: Infinite Streams & Unbounded Heap Exhaustion (`OutOfMemoryError`)
* **Symptom**: Microservice heap grows rapidly within minutes; GC thrashing occurs until the container dies with `java.lang.OutOfMemoryError: Java heap space`.
* **Root Cause**: Creating an infinite stream using `Stream.iterate()` or `Stream.generate()` without an explicit short-circuiting limit, or attempting to sort an infinite stream:
  ```java
  // LETHAL BUG: .sorted() is a stateful intermediate operation that must consume ALL elements!
  // It buffers infinitely in memory before emitting a single element!
  Stream.iterate(0, n -> n + 1)
      .sorted() // NEVER TERMINATES! OOM CRASH!
      .limit(10)
      .toList();
  ```
* **Production Remedy**:
  1. Always place `.limit()` **before** any stateful intermediate operation (`sorted()`, `distinct()`).
  2. Prefer the three-argument bounded `Stream.iterate()` available since Java 9:
     ```java
     // Explicit termination predicate prevents runaway stream generation
     List<Integer> numbers = Stream.iterate(0, n -> n < 1000, n -> n + 1)
         .toList();
     ```

---

### Incident 2: Parallel Stream Shared `ForkJoinPool` Starvation
* **Symptom**: Completely unrelated background services, CompletableFutures, and web request handlers freeze; application thread dumps show `ForkJoinPool.commonPool-worker-*` threads blocked in `WAITING` state on socket reads.
* **Root Cause**: A developer used `.parallelStream()` or `.stream().parallel()` to execute slow external REST calls or database queries. By default, **all** parallel streams across the entire JVM share a single static thread pool: `ForkJoinPool.commonPool()`, whose parallelism is capped at `Runtime.getRuntime().availableProcessors() - 1`. A few long-running I/O operations in a parallel stream consume all carrier threads, starving the entire application!
* **Production Remedy**:
  1. **Strictly Ban Blocking I/O inside Parallel Streams**: Parallel streams are designed solely for CPU-bound computations in memory.
  2. **Isolate Parallel Execution in Dedicated ForkJoinPool**:
     ```java
     ForkJoinPool customPool = new ForkJoinPool(8);
     try {
         List<Result> results = customPool.submit(() ->
             dataList.parallelStream()
                 .map(this::computeHeavyCpuTask)
                 .toList()
         ).get();
     } finally {
         customPool.shutdown();
     }
     ```

---

### Incident 3: Stream Reuse Exception (`IllegalStateException`)
* **Symptom**: Code throws `java.lang.IllegalStateException: stream has already been operated upon or closed`.
* **Root Cause**: Unlike collections, Java Streams are single-use traversals over an underlying data source. Once a terminal operation (`findFirst()`, `collect()`, `count()`) executes, the stream pipeline is permanently consumed and closed:
  ```java
  Stream<Order> orderStream = orderList.stream().filter(Order::isVip);
  long count = orderStream.count(); // Terminal operation 1: Consumes stream!
  List<Order> vipOrders = orderStream.toList(); // FAILS: IllegalStateException!
  ```
* **Production Remedy**:
  Re-instantiate the stream from the source collection, or chain operations in a single terminal collector using `teeing()`:
  ```java
  // Java 12+ teeing collector computes two aggregations in a single traversal!
  record OrderStats(long count, BigDecimal totalRevenue) {}

  OrderStats stats = orderList.stream()
      .filter(Order::isVip)
      .collect(Collectors.teeing(
          Collectors.counting(),
          Collectors.reducing(BigDecimal.ZERO, Order::amount, BigDecimal::add),
          OrderStats::new
      ));
  ```

---

### Incident 4: Non-Exhaustive Pattern Matching & Sealed Hierarchy Desynchronization
* **Symptom**: Upgrading an internal shared library causes microservice compile errors or runtime `MatchException` under Java 21 pattern-matching `switch`.
* **Root Cause**: A core shared library updates a `sealed interface PaymentMethod` by permitting a new record `record CryptoPayment(String walletId) implements PaymentMethod`. Client microservices utilizing `switch (payment)` without a `default` case fail to compile because the compiler exhaustiveness check detects an unhandled permitted variant.
* **Production Remedy**:
  1. **Leverage Compile-Time Enforcement**: This is an intended architectural feature, not a bug! The compiler prevents runtime gaps by forcing developers to handle the newly introduced domain type explicitly:
     ```java
     return switch (payment) {
         case CreditCard cc -> processCard(cc);
         case BankTransfer bt -> processTransfer(bt);
         case CryptoPayment cp -> processCrypto(cp); // Newly added branch!
     };
     ```
  2. **Provide Versioned Deprecation Notices**: When adding variants to public sealed hierarchies in shared enterprise libraries, announce breaking domain additions across release notes.

---

### Incident 5: State Mutation Inside Stream Operations (Race Conditions & Lost Updates)
* **Symptom**: Intermittent data corruption, missing elements, or `ArrayIndexOutOfBoundsException` when processing streams under high traffic.
* **Root Cause**: Mutating non-thread-safe external collections inside stream intermediate operations (`.peek()`, `.map()`):
  ```java
  // CRITICAL ANTI-PATTERN: Mutating shared state inside parallel stream!
  List<ProcessedOrder> outputList = new ArrayList<>(); // NOT THREAD SAFE!
  orders.parallelStream()
      .filter(Order::isValid)
      .forEach(outputList::add); // LETHAL RACE CONDITION on ArrayList internal elementData array!
  ```
* **Production Remedy**:
  Streams must be pure, side-effect-free functional transformations. Always collect elements via standard collectors:
  ```java
  // Thread-safe, lock-free reduction managed by the Stream collector engine
  List<ProcessedOrder> outputList = orders.parallelStream()
      .filter(Order::isValid)
      .map(this::transform)
      .toList();
  ```

---

### Incident 6: Accidental Autoboxing Overhead in High-Throughput Stream ETL
* **Symptom**: High GC Young Gen allocation rate; CPU profiling shows 40% of CPU time spent in `java.lang.Long.valueOf(long)`.
* **Root Cause**: Using `Stream<Long>` instead of primitive `LongStream` when aggregating millions of numeric metrics or financial transactions:
  ```java
  // Incurs 16-byte Long wrapper object allocation per element!
  Stream<Long> boxedStream = records.stream().map(Record::amountInCents);
  long total = boxedStream.reduce(0L, Long::sum); // Allocates millions of wrappers!
  ```
* **Production Remedy**:
  Always leverage specialized primitive streams (`IntStream`, `LongStream`, `DoubleStream`):
  ```java
  // Zero-allocation primitive traversal over CPU registers
  long total = records.stream()
      .mapToLong(Record::amountInCents)
      .sum();
  ```

---

# 10. Summary Comparison & Production Engineering Reference Matrix

| Feature | Introduced JDK | Low-Level Bytecode / JVM Mechanism | Memory & Performance Impact | Production Engineering Rule |
| :--- | :---: | :--- | :--- | :--- |
| **Records** | JDK 16 | Extends `java.lang.Record`, generated canonical constructor & accessors | Value-based identity; candidate for Scalar Replacement | Ideal for DTOs, Kafka payloads, and immutable domain events. |
| **Sealed Classes** | JDK 17 | `PermittedSubclasses` attribute in classfile header | Zero runtime overhead; compile-time exhaustiveness checking | Model closed algebraic domain types; eliminates runtime defensive `default:` blocks. |
| **Pattern Matching** | JDK 17–21 | `instanceof` with type binding + switch dynamic dispatch | Translates to synthetic tableswitch or chained bytecode comparisons | Replaces fragile `if-else` casting chains with clean deconstruction. |
| **Lambdas** | JDK 8 | `invokedynamic` with `LambdaMetafactory` generates anonymous class | Single call-site bootstrap; non-capturing lambdas are cached flyweights | Prefer non-capturing method references (`String::trim`) over capturing closures. |
| **Streams** | JDK 8 | Pull-based pipeline over `Spliterator` with chained terminal reducer | Lazy evaluation; pipeline overhead amortized over $>1,000$ elements | Never mutate shared state inside `.map()` or `.peek()`; use primitive streams. |
| **Optional** | JDK 8 | Value wrapper reference (`value` field) | Incurs $16\text{B}$ heap overhead if not scalar-replaced | Use strictly as return type for absent values; never as method params or entity fields. |
| **Text Blocks** | JDK 15 | Compile-time string literal constant pool entry | Zero runtime overhead; compiler strips incidental indentation | Standard for SQL queries, JSON schemas, and multiline documentation. |

---

# 11. References & Further Reading

- [Oracle OpenJDK - JDK 21 Features and Specifications](https://openjdk.org/projects/jdk/21/)
- [JEP 395: Records](https://openjdk.org/jeps/395) — OpenJDK specification for immutable data carriers.
- [JEP 409: Sealed Classes](https://openjdk.org/jeps/409) — OpenJDK specification for restricted class hierarchies.
- [JEP 440: Record Patterns](https://openjdk.org/jeps/440) — Deconstruction patterns for record values in Java 21.
- [Baeldung - Java Functional Programming, Records, and Streams](https://www.baeldung.com/category/java/java-core/)
- [Brian Goetz: Data-Oriented Programming in Java](https://www.infoq.com/articles/data-oriented-programming-java/) — Architectural paradigm for modeling state with records and sealed hierarchies.
