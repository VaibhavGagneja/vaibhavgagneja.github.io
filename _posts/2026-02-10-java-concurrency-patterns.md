---
title: "Java Concurrency: Common Patterns, Interview Problems & Best Practices"
description: Thread-safe Singleton, Immutability, classic interview problems (Odd-Even, Producer-Consumer, Dining Philosophers) and a complete best practices cheat sheet
author: Vaibhav Gagneja
date: 2026-02-10 12:00:00 +0530
categories: [Development, Java]
tags: [java, concurrency, design-patterns, interview, best-practices]
toc: true
image:
  path: /assets/photos/synchros.png
---

Welcome to the final installment of our Java Concurrency series! We'll bring everything together by exploring **common concurrency patterns**, solving **classic interview problems**, and compiling a **complete best practices** cheat sheet.

---

## 1. Thread-Safe Singleton Pattern

The Singleton pattern is one of the most commonly asked concurrency questions. Let's explore the evolution from broken to bulletproof.

### ❌ Broken: Naïve Singleton (Not Thread-Safe)

```java
public class NaiveSingleton {
    private static NaiveSingleton instance;

    private NaiveSingleton() { }

    public static NaiveSingleton getInstance() {
        if (instance == null) {         // Thread A checks: null
            // Thread B also checks: null (race condition!)
            instance = new NaiveSingleton();  // Both create instances!
        }
        return instance;
    }
}
```

### ❌ Slow: Synchronized (Thread-Safe but Bad Performance)

```java
public class SyncSingleton {
    private static SyncSingleton instance;

    // ❌ EVERY call acquires the lock — even after instance exists!
    public static synchronized SyncSingleton getInstance() {
        if (instance == null) {
            instance = new SyncSingleton();
        }
        return instance;
    }
}
```

### ✅ Solution 1: Double-Checked Locking

```java
public class DCLSingleton {
    // volatile prevents instruction reordering!
    private static volatile DCLSingleton instance;

    private DCLSingleton() { }

    public static DCLSingleton getInstance() {
        if (instance == null) {                    // First check (no lock)
            synchronized (DCLSingleton.class) {    // Lock only when needed
                if (instance == null) {             // Second check (with lock)
                    instance = new DCLSingleton();
                }
            }
        }
        return instance;  // No lock needed for subsequent calls!
    }
}
```

**Why `volatile` is essential here:**

Without `volatile`, the JVM might reorder the steps of `instance = new DCLSingleton()`:
1. Allocate memory
2. Assign reference to `instance` ← Other threads see non-null!
3. Run constructor ← But object isn't fully constructed yet!

`volatile` prevents this reordering.

### ✅ Solution 2: Bill Pugh (Holder Pattern) — Recommended!

```java
public class BillPughSingleton {
    private BillPughSingleton() { }

    // Inner class is loaded ONLY when getInstance() is called
    private static class Holder {
        private static final BillPughSingleton INSTANCE = new BillPughSingleton();
    }

    public static BillPughSingleton getInstance() {
        return Holder.INSTANCE;  // Lazy + thread-safe by JVM classloading!
    }
}
```

**Why this works:** The JVM guarantees that class loading is thread-safe. The `Holder` class isn't loaded until `getInstance()` is first called, making it both lazy and thread-safe — without any locks!

### ✅ Solution 3: Enum Singleton — Simplest & Safest

```java
public enum EnumSingleton {
    INSTANCE;

    private int value;

    public int getValue() { return value; }
    public void setValue(int value) { this.value = value; }
}

// Usage:
EnumSingleton.INSTANCE.setValue(42);
```

**Why this is the best:**
- Thread-safe by default (JVM handles enum initialization)
- Serialization-safe (prevents creating new instances via deserialization)
- Reflection-safe (can't create enum instances via reflection)

### Singleton Comparison

| Approach | Thread-Safe | Lazy | Performance | Serialization-Safe |
|----------|-------------|------|-------------|-------------------|
| Naïve | ❌ | ✅ | ⚡ Fast | ❌ |
| synchronized | ✅ | ✅ | 🐢 Slow | ❌ |
| Double-Checked Locking | ✅ | ✅ | ⚡ Fast | ❌ |
| Bill Pugh (Holder) | ✅ | ✅ | ⚡ Fast | ❌ |
| **Enum** | ✅ | ❌ (eager) | ⚡ Fast | ✅ |

---

## 2. Immutability as a Concurrency Strategy

### Why Immutable Objects Are Inherently Thread-Safe

If an object **cannot be modified** after creation, there's **no race condition** possible!

```java
// ✅ IMMUTABLE — automatically thread-safe!
public final class Money {                  // final class — can't extend
    private final String currency;          // final fields — can't change
    private final double amount;

    public Money(String currency, double amount) {
        this.currency = currency;
        this.amount = amount;
    }

    public String getCurrency() { return currency; }
    public double getAmount() { return amount; }

    // No setters! To "modify", create a new instance
    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Currency mismatch!");
        }
        return new Money(this.currency, this.amount + other.amount);
    }

    @Override
    public String toString() {
        return currency + " " + amount;
    }
}
```

```java
// Usage — 100% thread-safe, no synchronization needed!
Money price = new Money("INR", 500.0);
Money tax = new Money("INR", 90.0);
Money total = price.add(tax);  // Creates NEW object
System.out.println(total);     // INR 590.0
// price is unchanged: INR 500.0
```

### Immutability Checklist

| Rule | Reason |
|------|--------|
| Make class `final` | Prevents subclasses from adding mutable state |
| Make all fields `final` | Fields assigned once in constructor |
| No setters | No way to change state |
| Defensive copy mutable fields | Prevent callers from modifying internals |
| Return copies from getters | Same reason as above |

### Java's Built-in Immutable Classes

`String`, `Integer`, `Long`, `Double`, `BigDecimal`, `LocalDate`, `LocalTime`, `LocalDateTime`, `Duration`, `Instant` — all are **immutable and thread-safe!**

---

## 3. Thread Confinement

### The Concept

**Thread confinement** means keeping data accessible to only **one thread**. If no sharing happens, no synchronization is needed.

### Types of Thread Confinement

| Type | Mechanism | Example |
|------|-----------|---------|
| **Stack Confinement** | Local variables | Method-local variables are on the thread's stack |
| **ThreadLocal** | Per-thread storage | `ThreadLocal<SimpleDateFormat>` |
| **Object Confinement** | Encapsulate in single-thread context | Swing's Event Dispatch Thread |

```java
// Stack Confinement — automatically thread-safe!
public void processOrder(Order order) {
    // These variables exist ONLY on this thread's stack
    double subtotal = order.getTotal();
    double tax = subtotal * 0.18;
    double total = subtotal + tax;
    // No other thread can see these variables
}
```

---

## 4. Classic Interview Problems

### Problem 1: Print Odd-Even Numbers Using Two Threads

```java
public class OddEvenPrinter {
    private static int number = 1;
    private static final int MAX = 20;
    private static final Object lock = new Object();

    public static void main(String[] args) {
        Thread oddThread = new Thread(() -> {
            synchronized (lock) {
                while (number <= MAX) {
                    if (number % 2 != 0) {  // Odd
                        System.out.println("Odd  Thread: " + number);
                        number++;
                        lock.notify();
                    } else {
                        try { lock.wait(); } catch (InterruptedException e) { }
                    }
                }
                lock.notify();  // Wake up the other thread to exit
            }
        }, "Odd");

        Thread evenThread = new Thread(() -> {
            synchronized (lock) {
                while (number <= MAX) {
                    if (number % 2 == 0) {  // Even
                        System.out.println("Even Thread: " + number);
                        number++;
                        lock.notify();
                    } else {
                        try { lock.wait(); } catch (InterruptedException e) { }
                    }
                }
                lock.notify();
            }
        }, "Even");

        oddThread.start();
        evenThread.start();
    }
}
```

**Output:**
```
Odd  Thread: 1
Even Thread: 2
Odd  Thread: 3
Even Thread: 4
...
Odd  Thread: 19
Even Thread: 20
```

---

### Problem 2: Producer-Consumer with BlockingQueue

```java
import java.util.concurrent.*;

public class ProducerConsumerDemo {
    public static void main(String[] args) throws InterruptedException {
        BlockingQueue<Integer> buffer = new ArrayBlockingQueue<>(5);

        // Producer
        Thread producer = new Thread(() -> {
            try {
                for (int i = 1; i <= 15; i++) {
                    System.out.println("📦 Producing: " + i);
                    buffer.put(i);  // Blocks if full
                    Thread.sleep(200);
                }
                buffer.put(-1);  // Poison pill — signal to stop
            } catch (InterruptedException e) { }
        }, "Producer");

        // Consumer
        Thread consumer = new Thread(() -> {
            try {
                while (true) {
                    int item = buffer.take();  // Blocks if empty
                    if (item == -1) {
                        System.out.println("🛑 Consumer received poison pill. Stopping.");
                        break;
                    }
                    System.out.println("  🔧 Consuming: " + item);
                    Thread.sleep(500);  // Processing takes longer
                }
            } catch (InterruptedException e) { }
        }, "Consumer");

        producer.start();
        consumer.start();
        producer.join();
        consumer.join();

        System.out.println("\n✅ Done!");
    }
}
```

---

### Problem 3: Dining Philosophers (Deadlock Prevention)

Five philosophers sit around a table. Each needs two forks to eat. If everyone picks up the left fork simultaneously — **deadlock!**

```java
import java.util.concurrent.locks.*;

public class DiningPhilosophers {
    private static final int COUNT = 5;
    private static final ReentrantLock[] forks = new ReentrantLock[COUNT];

    static {
        for (int i = 0; i < COUNT; i++) {
            forks[i] = new ReentrantLock();
        }
    }

    static void philosopher(int id) {
        String name = "Philosopher-" + id;
        int leftFork = id;
        int rightFork = (id + 1) % COUNT;

        // ✅ DEADLOCK PREVENTION: Always pick up lower-numbered fork first!
        int first = Math.min(leftFork, rightFork);
        int second = Math.max(leftFork, rightFork);

        for (int meal = 1; meal <= 3; meal++) {
            // Think
            System.out.println("💭 " + name + " is thinking...");
            sleep(500);

            // Pick up forks in ORDER
            forks[first].lock();
            forks[second].lock();
            try {
                System.out.println("🍝 " + name + " is eating meal #" + meal + 
                                 " (forks " + first + " & " + second + ")");
                sleep(300);
            } finally {
                forks[second].unlock();
                forks[first].unlock();
            }

            System.out.println("✓  " + name + " finished meal #" + meal);
        }
        System.out.println("🎉 " + name + " is done (ate 3 meals)!");
    }

    public static void main(String[] args) {
        Thread[] threads = new Thread[COUNT];
        for (int i = 0; i < COUNT; i++) {
            int id = i;
            threads[i] = new Thread(() -> philosopher(id));
            threads[i].start();
        }
    }

    static void sleep(long ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) { }
    }
}
```

**Why this avoids deadlock:** By always acquiring the lower-numbered fork first, we break the **circular wait** condition (one of the four conditions required for deadlock).

---

### Problem 4: Thread-Safe Bounded Buffer (Without BlockingQueue)

```java
import java.util.LinkedList;
import java.util.Queue;
import java.util.concurrent.locks.*;

class BoundedBuffer<T> {
    private final Queue<T> queue = new LinkedList<>();
    private final int capacity;
    private final Lock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();

    BoundedBuffer(int capacity) {
        this.capacity = capacity;
    }

    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == capacity) {
                notFull.await();  // Wait until space available
            }
            queue.add(item);
            notEmpty.signal();   // Signal waiting consumers
        } finally {
            lock.unlock();
        }
    }

    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty()) {
                notEmpty.await();  // Wait until item available
            }
            T item = queue.poll();
            notFull.signal();     // Signal waiting producers
            return item;
        } finally {
            lock.unlock();
        }
    }
}
```

---


## 6. Series Recap — Complete Learning Path

Congratulations! You've completed the entire Java Concurrency series. Here's a summary of everything we covered:

| # | Blog Post | Key Topics |
|---|-----------|------------|
| 1 | **Mastering Java Threads** | Processes vs Threads, lifecycle, creating threads, daemon threads |
| 2 | **Synchronization Basics** | Race conditions, synchronized methods/blocks, object locks |
| 3 | **Inter-Thread Communication** | wait(), notify(), notifyAll(), producer-consumer |
| 4 | **Deadlocks & ReentrantLock** | Deadlock detection/prevention, tryLock(), j.u.c overview |
| 5 | **ExecutorService & CompletableFuture** | Thread pools, Callable/Future, async chaining |
| 6 | **volatile, Atomics & JMM** | Visibility, CAS, AtomicInteger, LongAdder, happens-before |
| 7 | **Concurrent Collections** | ConcurrentHashMap, BlockingQueue, CopyOnWriteArrayList |
| 8 | **Synchronizers** | CountDownLatch, CyclicBarrier, Semaphore, ReadWriteLock, StampedLock |
| 9 | **Fork/Join & Virtual Threads** | Divide-and-conquer, ThreadLocal, Project Loom |
| 10 | **Patterns & Best Practices** | Singleton, immutability, interview problems, cheat sheet |


---

*Thank you for following this entire series! Happy Concurrent Coding! 🧵🚀🎉*
