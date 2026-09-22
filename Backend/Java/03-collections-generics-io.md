# 3. Collections, Generics, Exceptions & File I/O

[← Back](02-intermediate-oop.md) | [Index](README.md) | Next → [4. Advanced / Modern Java](04-advanced-modern-java.md)

This section covers the tools you'll reach for daily in real applications: dynamic data structures, type-safe reusable code, robust error handling, and reading/writing files.

---

## 3.1 The Collections Framework

Arrays are fixed size. **Collections** grow and shrink, and offer rich operations. The main interfaces are `List`, `Set`, `Map`, and `Queue`.

| Interface | Common impl | Ordered? | Duplicates? | Use case |
|-----------|-------------|----------|-------------|----------|
| `List` | `ArrayList` | Yes (by index) | Yes | Ordered sequence, frequent reads |
| `List` | `LinkedList` | Yes | Yes | Frequent inserts/removes at ends |
| `Set` | `HashSet` | No | No | Unique items, fast lookup |
| `Set` | `LinkedHashSet` | Insertion order | No | Unique + predictable order |
| `Set` | `TreeSet` | Sorted | No | Unique + auto-sorted |
| `Map` | `HashMap` | No | Keys unique | Key→value lookup |
| `Map` | `TreeMap` | Sorted by key | Keys unique | Sorted key→value |
| `Queue` | `ArrayDeque` | FIFO/LIFO | Yes | Stacks, queues, buffers |

### List

**Use case: a to-do list.**

```java
import java.util.ArrayList;
import java.util.List;

List<String> todos = new ArrayList<>();
todos.add("Write report");
todos.add("Email client");
todos.add("Review PR");

todos.remove("Email client");
System.out.println(todos.get(0));       // Write report
System.out.println(todos.size());        // 2
System.out.println(todos.contains("Review PR")); // true

for (String task : todos) {
    System.out.println("- " + task);
}
```

### Set

**Use case: track unique visitors.**

```java
import java.util.HashSet;
import java.util.Set;

Set<String> visitors = new HashSet<>();
visitors.add("alice");
visitors.add("bob");
visitors.add("alice");            // ignored — already present
System.out.println(visitors.size());  // 2
```

### Map

**Use case: count word frequencies.**

```java
import java.util.HashMap;
import java.util.Map;

String text = "the cat the dog the bird";
Map<String, Integer> counts = new HashMap<>();

for (String word : text.split(" ")) {
    counts.merge(word, 1, Integer::sum);   // add 1, or start at 1
}
System.out.println(counts);   // {the=3, cat=1, dog=1, bird=1}

// Iterate entries
for (Map.Entry<String, Integer> e : counts.entrySet()) {
    System.out.println(e.getKey() + " -> " + e.getValue());
}
```

### Queue / Deque

**Use case: a print queue (FIFO) and an undo stack (LIFO).**

```java
import java.util.ArrayDeque;
import java.util.Deque;

Deque<String> printQueue = new ArrayDeque<>();
printQueue.offer("doc1");     // enqueue
printQueue.offer("doc2");
System.out.println(printQueue.poll());  // doc1 (FIFO)

Deque<String> undo = new ArrayDeque<>();
undo.push("type A");          // stack
undo.push("type B");
System.out.println(undo.pop());  // type B (LIFO)
```

---

## 3.2 Generics

Generics make classes and methods **type-safe and reusable** without casting. The `<String>` in `List<String>` is a generic type argument.

### Generic method

```java
// works for any type T
static <T> T firstOrNull(List<T> list) {
    return list.isEmpty() ? null : list.get(0);
}
```

### Generic class

**Use case: a type-safe container pair.**

```java
public class Pair<K, V> {
    private final K key;
    private final V value;

    public Pair(K key, V value) { this.key = key; this.value = value; }
    public K getKey()   { return key; }
    public V getValue() { return value; }
}

Pair<String, Integer> age = new Pair<>("Ada", 36);
System.out.println(age.getKey() + " is " + age.getValue());
```

### Bounded types and wildcards

```java
// T must be a Number (or subclass)
static double sum(List<? extends Number> nums) {
    double total = 0;
    for (Number n : nums) total += n.doubleValue();
    return total;
}
```

- `<? extends T>` — read-only "producer" (accepts T and its subtypes).
- `<? super T>` — write "consumer" (accepts T and its supertypes).
- Mnemonic: **PECS** — Producer Extends, Consumer Super.

---

## 3.3 Exception handling

Exceptions signal that something went wrong. Handle them so your program fails gracefully instead of crashing.

### try / catch / finally

```java
try {
    int[] arr = {1, 2, 3};
    System.out.println(arr[5]);          // throws ArrayIndexOutOfBoundsException
} catch (ArrayIndexOutOfBoundsException e) {
    System.out.println("Index out of range: " + e.getMessage());
} finally {
    System.out.println("This always runs (cleanup).");
}
```

### Checked vs unchecked exceptions

- **Checked** (e.g., `IOException`): the compiler forces you to handle or declare them. Used for recoverable, expected failures like a missing file.
- **Unchecked** (`RuntimeException` subclasses like `NullPointerException`): usually programming bugs. Not required to catch.

```java
// Declaring a checked exception with 'throws'
static String readFirstLine(String path) throws java.io.IOException {
    return java.nio.file.Files.readAllLines(java.nio.file.Path.of(path)).get(0);
}
```

### Multi-catch and custom exceptions

```java
try {
    process();
} catch (IllegalArgumentException | IllegalStateException e) {
    System.out.println("Bad input or state: " + e.getMessage());
}
```

**Use case: a domain-specific exception.**

```java
class InsufficientFundsException extends Exception {
    public InsufficientFundsException(String msg) { super(msg); }
}

void withdraw(double amount, double balance) throws InsufficientFundsException {
    if (amount > balance) {
        throw new InsufficientFundsException("Tried to withdraw " + amount + " but balance is " + balance);
    }
}
```

### try-with-resources

Automatically closes resources (files, sockets, DB connections) — no manual `finally` needed. Any resource implementing `AutoCloseable` works.

```java
import java.io.BufferedReader;
import java.io.FileReader;
import java.io.IOException;

try (BufferedReader br = new BufferedReader(new FileReader("data.txt"))) {
    System.out.println(br.readLine());
} catch (IOException e) {
    System.out.println("Could not read file: " + e.getMessage());
}   // br is closed automatically, even if an exception is thrown
```

---

## 3.4 File I/O

Modern Java uses the `java.nio.file` API (`Path`, `Files`) — cleaner than the old `java.io.File`.

### Reading a whole file

```java
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.List;

Path path = Path.of("notes.txt");
List<String> lines = Files.readAllLines(path);
for (String line : lines) {
    System.out.println(line);
}

String whole = Files.readString(path);   // entire file as one String (Java 11+)
```

### Writing a file

```java
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.List;

Path out = Path.of("output.txt");
Files.writeString(out, "Hello file!\n");          // overwrites
Files.write(out, List.of("line 1", "line 2"));    // write lines
```

### Appending

```java
import java.nio.file.StandardOpenOption;

Files.writeString(out, "appended line\n", StandardOpenOption.APPEND);
```

### Practical use case: read a CSV and summarize

Suppose `sales.csv` contains `product,amount` rows:

```
Keyboard,49.99
Mouse,19.99
Keyboard,49.99
Monitor,199.99
```

```java
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.HashMap;
import java.util.Map;

public class SalesReport {
    public static void main(String[] args) throws Exception {
        Map<String, Double> totals = new HashMap<>();

        for (String line : Files.readAllLines(Path.of("sales.csv"))) {
            String[] parts = line.split(",");
            String product = parts[0];
            double amount = Double.parseDouble(parts[1]);
            totals.merge(product, amount, Double::sum);
        }

        totals.forEach((product, total) ->
            System.out.printf("%-10s $%.2f%n", product, total));
    }
}
```

Output:

```
Keyboard   $99.98
Mouse      $19.99
Monitor    $199.99
```

---

## Key takeaways

- Pick the right collection: `ArrayList` for ordered data, `HashSet` for uniqueness, `HashMap` for key→value lookups.
- Generics give compile-time type safety; remember PECS for wildcards.
- Handle checked exceptions; use custom exceptions for domain errors; prefer try-with-resources for anything closeable.
- Use `java.nio.file.Files` and `Path` for concise, modern file I/O.

## Practice exercises

1. Read a text file and print the 5 most common words (use a `Map` and sort).
2. Build a `Stack<T>` generic class backed by an `ArrayList`.
3. Write a program that validates user input and throws a custom `InvalidAgeException`.
4. Merge two CSV files by a shared key column and write the result to a new file.

Next → [4. Advanced / Modern Java](04-advanced-modern-java.md)
