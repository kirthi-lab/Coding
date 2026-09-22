---
title: Java Beginner Tutorial
author: Kiruthika Kannan
date: 2026-09-22
keywords: [ Java, OOP ] 
---




# 1. Beginner: Getting Started with Java

[← Back to index](README.md) | Next → [2. Methods & OOP](02-intermediate-oop.md)

This section takes you from zero to writing and running real Java programs. By the end you'll understand the language basics: variables, types, operators, and control flow.

---

## 1.1 Installing the JDK

Java code needs the **JDK** (Java Development Kit) to compile and run.

1. Download a JDK. Good choices: [Eclipse Temurin (Adoptium)](https://adoptium.net/) or [Oracle JDK](https://www.oracle.com/java/technologies/downloads/). Pick an **LTS** version (e.g., 17 or 21).
2. Install it, then verify from a terminal:

```bash
java -version
javac -version
```

You should see output like `openjdk version "21.0.x"`. If the command isn't found, add the JDK's `bin` folder to your `PATH`.

> **JDK vs JRE vs JVM:** The JDK contains the compiler (`javac`) and the runtime. The runtime includes the JVM, which executes your compiled bytecode. You compile once and run anywhere a JVM exists — that's Java's portability promise.

---

## 1.2 Your first program

Create a file named `HelloWorld.java`. The file name **must** match the public class name.

```java
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello, World!");
    }
}
```

Compile and run it:

```bash
javac HelloWorld.java   # produces HelloWorld.class (bytecode)
java HelloWorld         # runs the bytecode on the JVM
```

**Breaking down the code:**

| Part | Meaning |
|------|---------|
| `public class HelloWorld` | Every Java program lives inside a class. |
| `public static void main(String[] args)` | The entry point. The JVM calls this method to start. |
| `System.out.println(...)` | Prints a line to the console. |
| `;` | Statements end with a semicolon. |

> Since Java 21 you can run a single file without compiling first: `java HelloWorld.java`.

---

## 1.3 Comments

```java
// Single-line comment

/* Multi-line
   comment */

/**
 * Javadoc comment — used to document classes and methods.
 * Tools generate HTML docs from these.
 */
```

---

## 1.4 Variables and data types

A variable is a named box that holds a value of a specific type. Java is **statically typed**: you declare the type, and it's checked at compile time.

### Primitive types

| Type | Size | Example | Use case |
|------|------|---------|----------|
| `byte` | 8-bit | `byte b = 100;` | Memory-tight data like raw file bytes |
| `short` | 16-bit | `short s = 20000;` | Rarely used; small integers |
| `int` | 32-bit | `int age = 30;` | Default choice for whole numbers |
| `long` | 64-bit | `long pop = 8_000_000_000L;` | Large counts, timestamps (note the `L`) |
| `float` | 32-bit | `float f = 3.14f;` | Low-precision decimals (note the `f`) |
| `double` | 64-bit | `double price = 19.99;` | Default choice for decimals |
| `char` | 16-bit | `char grade = 'A';` | A single character |
| `boolean` | 1-bit | `boolean active = true;` | true/false flags |

```java
int quantity = 5;
double unitPrice = 2.50;
double total = quantity * unitPrice;   // 12.5
boolean inStock = true;
char firstInitial = 'J';
long fileSizeBytes = 4_294_967_296L;   // underscores improve readability
```

### Reference types

Anything that isn't a primitive is a **reference type** (objects). The most common is `String`.

```java
String name = "Ada Lovelace";
System.out.println(name.length());       // 12
System.out.println(name.toUpperCase());  // ADA LOVELACE
```

### `var` — local type inference (Java 10+)

```java
var message = "Hello";   // inferred as String
var count = 42;          // inferred as int
```

Use `var` when the type is obvious from the right side. Avoid it when it hurts readability.

### Constants with `final`

```java
final double TAX_RATE = 0.08;   // cannot be reassigned
```

---

## 1.5 Operators

### Arithmetic

```java
int a = 10, b = 3;
System.out.println(a + b);   // 13
System.out.println(a - b);   // 7
System.out.println(a * b);   // 30
System.out.println(a / b);   // 3  (integer division truncates!)
System.out.println(a % b);   // 1  (remainder / modulo)
System.out.println(10.0 / 3); // 3.333... (floating-point division)
```

> **Gotcha:** `a / b` with two ints gives an int. Cast one to `double` for a decimal result: `(double) a / b`.

### Comparison and logical

```java
int score = 75;
boolean pass = score >= 60;                 // true
boolean bonus = score > 70 && score < 90;   // true (AND)
boolean edge = score < 60 || score > 90;    // false (OR)
boolean fail = !pass;                        // false (NOT)
```

### Assignment shortcuts

```java
int x = 10;
x += 5;   // x = x + 5  -> 15
x -= 2;   // 13
x *= 2;   // 26
x++;      // 27 (increment)
x--;      // 26 (decrement)
```

---

## 1.6 Reading input from the user

Use `Scanner` to read from the console.

```java
import java.util.Scanner;

public class Greeter {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);
        System.out.print("What's your name? ");
        String name = scanner.nextLine();
        System.out.print("What's your age? ");
        int age = scanner.nextInt();

        System.out.println("Hi " + name + ", next year you'll be " + (age + 1) + ".");
        scanner.close();
    }
}
```

---

## 1.7 Control flow

### `if` / `else if` / `else`

**Use case: grading a test score.**

```java
int score = 82;

if (score >= 90) {
    System.out.println("Grade: A");
} else if (score >= 80) {
    System.out.println("Grade: B");
} else if (score >= 70) {
    System.out.println("Grade: C");
} else {
    System.out.println("Grade: F");
}
```

### Ternary operator

A compact `if/else` that produces a value.

```java
int age = 20;
String status = (age >= 18) ? "adult" : "minor";
```

### `switch`

**Use case: mapping a day number to a name.**

```java
int day = 3;

// Modern switch expression (Java 14+) — cleaner, no fall-through bugs
String name = switch (day) {
    case 1 -> "Monday";
    case 2 -> "Tuesday";
    case 3 -> "Wednesday";
    default -> "Unknown";
};
System.out.println(name);   // Wednesday
```

### Loops

**`for` loop — when you know the count.**

```java
// Print a 5-row multiplication table for 7
for (int i = 1; i <= 5; i++) {
    System.out.println("7 x " + i + " = " + (7 * i));
}
```

**`while` loop — when you loop until a condition changes.**

```java
int balance = 100;
int day = 0;
while (balance > 0) {
    balance -= 30;   // spend 30 per day
    day++;
}
System.out.println("Ran out of money on day " + day);
```

**`do-while` — runs at least once.**

```java
int attempts = 0;
do {
    attempts++;
    System.out.println("Attempt " + attempts);
} while (attempts < 3);
```

**Enhanced `for` (for-each) — iterate a collection or array.**

```java
int[] temps = {68, 72, 75, 70};
int sum = 0;
for (int t : temps) {
    sum += t;
}
System.out.println("Average: " + (sum / temps.length));
```

### `break` and `continue`

```java
for (int i = 1; i <= 10; i++) {
    if (i == 5) continue;   // skip 5
    if (i == 8) break;      // stop at 8
    System.out.print(i + " ");   // 1 2 3 4 6 7
}
```

---

## 1.8 Mini project: a simple number-guessing game

Ties together variables, loops, conditionals, and input.

```java
import java.util.Random;
import java.util.Scanner;

public class GuessingGame {
    public static void main(String[] args) {
        int secret = new Random().nextInt(100) + 1;  // 1..100
        Scanner scanner = new Scanner(System.in);
        int guess;
        int tries = 0;

        System.out.println("I'm thinking of a number between 1 and 100.");
        do {
            System.out.print("Your guess: ");
            guess = scanner.nextInt();
            tries++;

            if (guess < secret) {
                System.out.println("Too low!");
            } else if (guess > secret) {
                System.out.println("Too high!");
            } else {
                System.out.println("Correct! You got it in " + tries + " tries.");
            }
        } while (guess != secret);

        scanner.close();
    }
}
```

---

## Key takeaways

- Java is statically typed and compiled to bytecode that runs on the JVM.
- Every program has a `main` method as its entry point.
- Prefer `int` and `double` for numbers, `String` for text.
- Use `switch` expressions and enhanced `for` loops for cleaner, safer code.

## Practice exercises

1. Write a program that converts Celsius to Fahrenheit (`F = C * 9/5 + 32`).
2. Print all even numbers from 1 to 50 using a loop.
3. Ask the user for a number and print whether it's prime.
4. Build a simple calculator that reads two numbers and an operator (`+ - * /`).

Next → [2. Methods & OOP](02-intermediate-oop.md)
