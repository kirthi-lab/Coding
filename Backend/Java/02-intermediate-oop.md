---
title: Java Intermediate Tutorial
author: Kiruthika Kannan
date: 2026-09-26
keywords: OOP
---





# 2. Methods, Arrays, Strings & Object-Oriented Programming

[← Back](01-beginner.md) | [Index](README.md) | Next → [3. Collections, Generics, I/O](03-collections-generics-io.md)

Now we move from writing statements to *structuring* code. Methods let you reuse logic. Arrays and strings let you handle data. OOP lets you model the real world.

---

## 2.1 Methods

A method is a named, reusable block of code that can take inputs (parameters) and return a value.

```java
public class MathUtils {

    // returns the larger of two numbers
    static int max(int a, int b) {
        return (a > b) ? a : b;
    }

    // void returns nothing
    static void printBanner(String text) {
        System.out.println("=== " + text + " ===");
    }

    public static void main(String[] args) {
        printBanner("Results");
        System.out.println(max(7, 12));   // 12
    }
}
```

### Method overloading

Same name, different parameter lists. The compiler picks the right one.

```java
static int add(int a, int b)          { return a + b; }
static double add(double a, double b) { return a + b; }
static int add(int a, int b, int c)   { return a + b + c; }
```

### Varargs — variable number of arguments

**Use case: sum any number of values.**

```java
static int sum(int... numbers) {
    int total = 0;
    for (int n : numbers) total += n;
    return total;
}
// sum(1, 2)  ->  3
// sum(1, 2, 3, 4, 5)  ->  15
```

---

## 2.2 Arrays

A fixed-size, ordered collection of the same type.

```java
int[] scores = new int[5];        // 5 zeros
scores[0] = 90;
scores[1] = 85;

int[] primes = {2, 3, 5, 7, 11};  // literal
System.out.println(primes.length); // 5
System.out.println(primes[2]);     // 5
```

**Use case: find the highest score.**

```java
int[] scores = {72, 88, 95, 63, 77};
int highest = scores[0];
for (int s : scores) {
    if (s > highest) highest = s;
}
System.out.println("Top score: " + highest);   // 95
```

### Multi-dimensional arrays

**Use case: a tic-tac-toe board.**

```java
char[][] board = {
    {'X', 'O', 'X'},
    {' ', 'X', 'O'},
    {'O', ' ', 'X'}
};
System.out.println(board[1][1]);   // X (center)
```

### Helpful array utilities

```java
import java.util.Arrays;

int[] nums = {5, 2, 8, 1};
Arrays.sort(nums);                       // [1, 2, 5, 8]
System.out.println(Arrays.toString(nums));
int idx = Arrays.binarySearch(nums, 5);  // 2 (must be sorted)
```

---

## 2.3 Strings

Strings are **immutable** — every "modification" creates a new String.

```java
String s = "Hello, World";
System.out.println(s.length());           // 12
System.out.println(s.charAt(0));           // H
System.out.println(s.substring(7));        // World
System.out.println(s.indexOf("World"));    // 7
System.out.println(s.replace("World", "Java")); // Hello, Java
System.out.println(s.toLowerCase());       // hello, world
System.out.println("  trim me  ".trim());  // "trim me"
System.out.println("a,b,c".split(",").length); // 3
```

### Comparing strings

```java
String a = "hello";
String b = "hello";
System.out.println(a.equals(b));            // true  (compare content)
System.out.println(a.equalsIgnoreCase("HELLO")); // true
// NEVER use == to compare String content — it compares references.
```

### Building strings efficiently

Concatenating in a loop is slow because each `+` creates a new String. Use `StringBuilder`.

```java
StringBuilder sb = new StringBuilder();
for (int i = 1; i <= 5; i++) {
    sb.append("Item ").append(i).append("\n");
}
System.out.println(sb.toString());
```

### Text blocks (Java 15+)

**Use case: embedding JSON or SQL.**

```java
String json = """
    {
        "name": "Ada",
        "role": "Engineer"
    }
    """;
```

### Formatting

```java
String out = String.format("Total: $%.2f for %d items", 12.5, 3);
// "Total: $12.50 for 3 items"
```

---

## 2.4 Object-Oriented Programming (OOP)

OOP models your program as interacting **objects**. A **class** is a blueprint; an **object** is an instance of it. Java's OOP rests on four pillars: **encapsulation, inheritance, polymorphism, abstraction**.

### Classes and objects

**Use case: model a bank account.**

```java
public class BankAccount {
    // fields (state)
    private String owner;
    private double balance;

    // constructor — runs when you create an object
    public BankAccount(String owner, double openingBalance) {
        this.owner = owner;
        this.balance = openingBalance;
    }

    // methods (behavior)
    public void deposit(double amount) {
        if (amount > 0) balance += amount;
    }

    public boolean withdraw(double amount) {
        if (amount > 0 && amount <= balance) {
            balance -= amount;
            return true;
        }
        return false;
    }

    public double getBalance() {
        return balance;
    }
}
```

Using it:

```java
BankAccount acct = new BankAccount("Ada", 100.0);
acct.deposit(50);
acct.withdraw(30);
System.out.println(acct.getBalance());   // 120.0
```

### Pillar 1: Encapsulation

Keep fields `private` and expose controlled access through methods (getters/setters). This protects the object's internal state — notice `withdraw` above refuses to overdraw. That rule can never be bypassed because `balance` is private.

```java
public class Person {
    private int age;

    public int getAge() { return age; }

    public void setAge(int age) {
        if (age >= 0) this.age = age;   // validation guards invalid state
    }
}
```

### Pillar 2: Inheritance

A subclass reuses and extends a superclass with `extends`.

**Use case: different account types share common behavior.**

```java
public class SavingsAccount extends BankAccount {
    private double interestRate;

    public SavingsAccount(String owner, double opening, double rate) {
        super(owner, opening);   // call parent constructor
        this.interestRate = rate;
    }

    public void addInterest() {
        deposit(getBalance() * interestRate);
    }
}
```

### Pillar 3: Polymorphism

One interface, many implementations. Method **overriding** lets a subclass redefine behavior.

**Use case: draw different shapes uniformly.**

```java
class Shape {
    double area() { return 0; }
}

class Circle extends Shape {
    private double r;
    Circle(double r) { this.r = r; }
    @Override double area() { return Math.PI * r * r; }
}

class Rectangle extends Shape {
    private double w, h;
    Rectangle(double w, double h) { this.w = w; this.h = h; }
    @Override double area() { return w * h; }
}

// Treat them all as Shape:
Shape[] shapes = { new Circle(2), new Rectangle(3, 4) };
for (Shape s : shapes) {
    System.out.printf("Area: %.2f%n", s.area());   // calls the right override
}
```

### Pillar 4: Abstraction

Hide implementation details behind a contract. Use `abstract` classes or `interface`s.

**Abstract class** — a partial blueprint that can't be instantiated.

```java
abstract class Animal {
    abstract String sound();          // no body — subclasses must implement

    void describe() {                 // shared concrete method
        System.out.println("This animal says " + sound());
    }
}

class Dog extends Animal {
    @Override String sound() { return "Woof"; }
}
```

**Interface** — a pure contract. A class can implement many.

```java
interface Payable {
    double calculatePay();
}

interface Reportable {
    String report();
}

class Employee implements Payable, Reportable {
    private double hours, rate;
    Employee(double hours, double rate) { this.hours = hours; this.rate = rate; }

    @Override public double calculatePay() { return hours * rate; }
    @Override public String report() { return "Pay: $" + calculatePay(); }
}
```

> **Abstract class vs interface:** Use an **abstract class** when types share state and common code. Use an **interface** to define a capability that unrelated classes can implement. Since Java 8, interfaces can also have `default` methods with bodies.

---

## 2.5 Static vs instance members

```java
public class Counter {
    static int totalCreated = 0;   // shared across all instances
    int id;                        // unique per instance

    Counter() {
        totalCreated++;
        id = totalCreated;
    }
}
```

`static` members belong to the class itself; instance members belong to each object.

---

## 2.6 The `Object` class and `toString`, `equals`, `hashCode`

Every class implicitly extends `Object`. Override these for meaningful behavior.

```java
public class Point {
    int x, y;
    Point(int x, int y) { this.x = x; this.y = y; }

    @Override public String toString() { return "(" + x + ", " + y + ")"; }

    @Override public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Point p)) return false;   // pattern matching (Java 16+)
        return x == p.x && y == p.y;
    }

    @Override public int hashCode() {
        return java.util.Objects.hash(x, y);
    }
}
```

> **Rule:** if you override `equals`, always override `hashCode` too, or hash-based collections (`HashMap`, `HashSet`) will misbehave.

---

## 2.7 Enums

A fixed set of named constants — safer than raw strings or ints.

```java
enum Status { ACTIVE, SUSPENDED, CLOSED }

Status s = Status.ACTIVE;
switch (s) {
    case ACTIVE -> System.out.println("Good to go");
    case SUSPENDED -> System.out.println("On hold");
    case CLOSED -> System.out.println("Gone");
}
```

Enums can also carry data and methods:

```java
enum Planet {
    EARTH(9.81), MARS(3.71);
    private final double gravity;
    Planet(double gravity) { this.gravity = gravity; }
    double weight(double mass) { return mass * gravity; }
}
```

---

## Key takeaways

- Methods encapsulate reusable logic; overloading and varargs add flexibility.
- Arrays are fixed-size; strings are immutable — use `StringBuilder` for heavy concatenation.
- OOP's four pillars: encapsulation (hide state), inheritance (reuse), polymorphism (one interface, many forms), abstraction (contracts).
- Override `equals`/`hashCode` together; prefer enums over magic constants.

## Practice exercises

1. Model a `Library` with `Book` objects; add methods to check books in and out.
2. Create a `Shape` hierarchy with `Triangle` and compute total area of a mixed array.
3. Define a `PaymentMethod` interface implemented by `CreditCard` and `PayPal`.
4. Write an `enum` for the days of the week with a method returning whether it's a weekend.

Next → [3. Collections, Generics, Exceptions & I/O](03-collections-generics-io.md)
