# 4. Advanced & Modern Java

[← Back](03-collections-generics-io.md) | [Index](README.md)

This final section covers the features that make modern Java expressive and powerful: functional programming with lambdas and streams, null-safety with `Optional`, concurrency, and the newest language features (records, sealed classes, pattern matching).

---

## 4.1 Functional interfaces and lambdas

A **functional interface** has exactly one abstract method. A **lambda** is a compact way to implement it.

```java
// Before lambdas — anonymous class
Runnable r1 = new Runnable() {
    @Override public void run() { System.out.println("Running"); }
};

// With a lambda
Runnable r2 = () -> System.out.println("Running");
```

### Built-in functional interfaces (`java.util.function`)

| Interface | Signature | Use case |
|-----------|-----------|----------|
| `Supplier<T>` | `() -> T` | Lazily produce a value |
| `Consumer<T>` | `T -> void` | Perform an action with a value |
| `Function<T,R>` | `T -> R` | Transform a value |
| `Predicate<T>` | `T -> boolean` | Test a condition |
| `BiFunction<T,U,R>` | `(T,U) -> R` | Combine two inputs |

```java
import java.util.function.*;

Supplier<String> greet = () -> "Hello";
Consumer<String> print = s -> System.out.println(s);
Function<Integer, Integer> square = x -> x * x;
Predicate<Integer> isEven = n -> n % 2 == 0;

print.accept(greet.get());              // Hello
System.out.println(square.apply(5));    // 25
System.out.println(isEven.test(4));     // true
```

### Method references

Shorthand for lambdas that just call an existing method.

```java
List<String> names = List.of("charlie", "alice", "bob");
names.stream().map(String::toUpperCase).forEach(System.out::println);
// String::toUpperCase   is   s -> s.toUpperCase()
// System.out::println   is   s -> System.out.println(s)
```

---

## 4.2 The Streams API

Streams process collections declaratively — you describe *what* you want, not *how* to loop. A stream pipeline has a **source**, zero or more **intermediate operations** (lazy), and one **terminal operation**.

**Use case: from a list of orders, get the total revenue of shipped orders over $50.**

```java
import java.util.List;

record Order(String product, double amount, boolean shipped) {}

List<Order> orders = List.of(
    new Order("Keyboard", 49.99, true),
    new Order("Monitor", 199.99, true),
    new Order("Mouse", 19.99, false),
    new Order("Desk", 149.99, true)
);

double revenue = orders.stream()
    .filter(Order::shipped)          // keep shipped
    .filter(o -> o.amount() > 50)    // over $50
    .mapToDouble(Order::amount)      // extract amounts
    .sum();                          // terminal

System.out.println("Revenue: $" + revenue);   // 349.98
```

### Common stream operations

```java
import java.util.List;
import java.util.stream.Collectors;

List<Integer> nums = List.of(1, 2, 3, 4, 5, 6);

// map + filter + collect
List<Integer> squaresOfEvens = nums.stream()
    .filter(n -> n % 2 == 0)
    .map(n -> n * n)
    .collect(Collectors.toList());   // [4, 16, 36]

// reduce — combine into one value
int product = nums.stream().reduce(1, (a, b) -> a * b);   // 720

// count, min, max, average
long count = nums.stream().filter(n -> n > 3).count();    // 3
double avg = nums.stream().mapToInt(Integer::intValue).average().orElse(0);  // 3.5
```

### Grouping and collecting

**Use case: group employees by department.**

```java
import java.util.*;
import java.util.stream.Collectors;

record Employee(String name, String dept, double salary) {}

List<Employee> staff = List.of(
    new Employee("Ada", "Eng", 120000),
    new Employee("Bob", "Eng", 95000),
    new Employee("Cara", "Sales", 80000)
);

// Map<String, List<Employee>>
Map<String, List<Employee>> byDept = staff.stream()
    .collect(Collectors.groupingBy(Employee::dept));

// Average salary per department
Map<String, Double> avgByDept = staff.stream()
    .collect(Collectors.groupingBy(Employee::dept,
             Collectors.averagingDouble(Employee::salary)));

// Join names into a string
String names = staff.stream()
    .map(Employee::name)
    .collect(Collectors.joining(", "));   // "Ada, Bob, Cara"
```

> **Note:** Streams are single-use. Once a terminal operation runs, you can't reuse that stream. Create a new one.

---

## 4.3 Optional — taming null

`Optional<T>` represents "a value that may or may not be present," replacing error-prone null checks.

```java
import java.util.Optional;

Optional<String> maybeName = findUserName(42);

// Provide a default
String name = maybeName.orElse("Guest");

// Act only if present
maybeName.ifPresent(n -> System.out.println("Found: " + n));

// Transform if present
int length = maybeName.map(String::length).orElse(0);

// Throw if absent
String required = maybeName.orElseThrow(() -> new IllegalStateException("No user"));
```

**Use case: safely navigate nested optional lookups.**

```java
Optional<Order> order = findOrder(id);
double amount = order
    .map(Order::amount)
    .filter(a -> a > 0)
    .orElse(0.0);
```

> Return `Optional` from methods that may not find a result. Don't use it for fields or method parameters.

---

## 4.4 Records (Java 16+)

A concise way to declare immutable data carriers. The compiler generates the constructor, getters, `equals`, `hashCode`, and `toString`.

```java
record Point(int x, int y) {}

Point p = new Point(3, 4);
System.out.println(p.x());        // 3
System.out.println(p);            // Point[x=3, y=4]
System.out.println(p.equals(new Point(3, 4)));  // true
```

Records can have validation and extra methods:

```java
record Range(int low, int high) {
    Range {   // compact constructor — validation
        if (low > high) throw new IllegalArgumentException("low > high");
    }
    int size() { return high - low; }
}
```

---

## 4.5 Sealed classes (Java 17+)

Restrict which classes may extend or implement a type — great for modeling a closed set of variants.

```java
sealed interface Shape permits Circle, Square {}
record Circle(double radius) implements Shape {}
record Square(double side) implements Shape {}
```

Combined with **pattern matching for switch** (Java 21), the compiler knows all cases are covered:

```java
static double area(Shape shape) {
    return switch (shape) {
        case Circle c -> Math.PI * c.radius() * c.radius();
        case Square s -> s.side() * s.side();
    };   // no default needed — the set is sealed and exhaustive
}
```

---

## 4.6 Concurrency

Java runs work in parallel using **threads**. Modern code favors the `java.util.concurrent` utilities over raw threads.

### Basic thread

```java
Thread t = new Thread(() -> System.out.println("Work on another thread"));
t.start();
t.join();   // wait for it to finish
```

### ExecutorService — a managed thread pool

**Use case: run several tasks concurrently and collect results.**

```java
import java.util.concurrent.*;
import java.util.List;

try (ExecutorService pool = Executors.newFixedThreadPool(4)) {   // Java 19+ auto-close
    List<Callable<Integer>> tasks = List.of(
        () -> compute(1),
        () -> compute(2),
        () -> compute(3)
    );
    List<Future<Integer>> results = pool.invokeAll(tasks);
    for (Future<Integer> f : results) {
        System.out.println(f.get());   // blocks until each result is ready
    }
}
```

### CompletableFuture — async pipelines

```java
import java.util.concurrent.CompletableFuture;

CompletableFuture
    .supplyAsync(() -> fetchPrice("AAPL"))     // run async
    .thenApply(price -> price * 1.1)           // transform
    .thenAccept(finalPrice -> System.out.println("Adjusted: " + finalPrice));
```

### Thread safety notes

- Prefer immutable objects (like records) — they're inherently thread-safe.
- For shared mutable counters, use `AtomicInteger` / `AtomicLong`.
- For shared maps, use `ConcurrentHashMap` instead of `HashMap`.
- Use `synchronized` blocks only when necessary and keep them small.

```java
import java.util.concurrent.atomic.AtomicInteger;

AtomicInteger hits = new AtomicInteger();
hits.incrementAndGet();   // thread-safe increment
```

> **Virtual threads (Java 21):** `Executors.newVirtualThreadPerTaskExecutor()` gives you lightweight threads ideal for high-concurrency I/O work — you can run millions of them cheaply.

---

## 4.7 A capstone example

Combines records, streams, grouping, and `Optional` to analyze transactions.

```java
import java.util.*;
import java.util.stream.Collectors;

public class TransactionAnalyzer {

    record Transaction(String user, String category, double amount) {}

    public static void main(String[] args) {
        List<Transaction> txns = List.of(
            new Transaction("ada", "food", 12.50),
            new Transaction("ada", "transport", 30.00),
            new Transaction("bob", "food", 8.75),
            new Transaction("ada", "food", 22.00),
            new Transaction("bob", "transport", 15.00)
        );

        // Total spend per user
        Map<String, Double> spendByUser = txns.stream()
            .collect(Collectors.groupingBy(
                Transaction::user,
                Collectors.summingDouble(Transaction::amount)));
        System.out.println("Spend by user: " + spendByUser);

        // Biggest single transaction
        Optional<Transaction> biggest = txns.stream()
            .max(Comparator.comparingDouble(Transaction::amount));
        biggest.ifPresent(t ->
            System.out.printf("Biggest: %s spent $%.2f on %s%n",
                t.user(), t.amount(), t.category()));

        // Categories sorted by total spend, descending
        txns.stream()
            .collect(Collectors.groupingBy(
                Transaction::category,
                Collectors.summingDouble(Transaction::amount)))
            .entrySet().stream()
            .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
            .forEach(e -> System.out.printf("%-10s $%.2f%n", e.getKey(), e.getValue()));
    }
}
```

---

## 4.8 Best practices recap

- **Prefer immutability.** Use `final`, records, and unmodifiable collections. Immutable objects are simpler and thread-safe.
- **Favor composition over inheritance.** Deep inheritance trees are brittle; interfaces + composition are flexible.
- **Program to interfaces.** Declare variables as `List`, not `ArrayList`, so implementations can change.
- **Use streams for data transformation,** loops for side-effect-heavy logic. Don't force everything into a stream.
- **Return `Optional`, never `null`,** for "maybe absent" results.
- **Handle exceptions meaningfully.** Don't swallow them with an empty catch block; log or rethrow with context.
- **Keep methods small and named for intent.** A method should do one thing.
- **Write tests.** Use JUnit 5 to lock in behavior before refactoring.
- **Use the latest LTS JDK** when you can — you get performance, security, and language improvements for free.

## Where to go next

- **Build tools:** Maven or Gradle for dependency management and builds.
- **Testing:** JUnit 5, Mockito, AssertJ.
- **Frameworks:** Spring Boot for web services and APIs (a natural next step from this tutorial).
- **Databases:** JDBC, then JPA/Hibernate.
- **Official docs:** [docs.oracle.com/javase](https://docs.oracle.com/en/java/javase/) and [dev.java](https://dev.java/).

## Practice exercises

1. Given a list of `Product` records, use streams to find the average price per category.
2. Rewrite a nested-null-check block using `Optional` chaining.
3. Model a `PaymentResult` as a sealed interface with `Success` and `Failure` variants; handle it with a switch.
4. Use an `ExecutorService` to download (simulate with `Thread.sleep`) several URLs concurrently and print total elapsed time.

[← Back to index](README.md)
