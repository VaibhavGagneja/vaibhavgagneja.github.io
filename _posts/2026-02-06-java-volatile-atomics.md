---
title: "Java Concurrency: volatile, Atomic Variables & Java Memory Model"
description: Understand CPU caching pitfalls, the volatile keyword, atomic classes, CAS mechanism, and the happens-before relationship in Java
author: Vaibhav Gagneja
date: 2026-02-06 12:00:00 +0530
categories: [Development, Java]
tags: [java, volatile, atomic, concurrency, memory-model, cas]
toc: true
image:
  path: /assets/photos/synchros.png
---

In the previous articles, we covered synchronization, locks, and thread pools. But there's an entire class of concurrency bugs that **`synchronized` alone can't explain** — bugs caused by **CPU caching and memory visibility**. Let's dive in!

---

## 1. The Visibility Problem: Why Threads Don't See Each Other's Writes

### The Surprising Bug

Consider this seemingly simple code:

```java
public class VisibilityProblem {
    private static boolean running = true;

    public static void main(String[] args) throws InterruptedException {
        Thread worker = new Thread(() -> {
            int count = 0;
            while (running) {  // Reads 'running' from CPU cache
                count++;
            }
            System.out.println("Worker stopped after count: " + count);
        });

        worker.start();

        Thread.sleep(1000);  // Let worker run for 1 second

        running = false;  // Main thread updates 'running'
        System.out.println("Main: Set running to false");
    }
}
```

**Expected:** Worker thread stops after ~1 second.
**Actual:** Worker thread **may run forever!** 🚨

---

### Why Does This Happen? CPU Caching

Modern CPUs have **multiple levels of cache** for performance:

```
┌──────────────────────────────────────────────────────────────────┐
│                     MULTI-CORE CPU ARCHITECTURE                   │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│   Core 1 (Main Thread)           Core 2 (Worker Thread)          │
│   ┌─────────────────┐           ┌─────────────────┐              │
│   │  L1 Cache        │           │  L1 Cache        │              │
│   │  running = false │           │  running = true  │ ← STALE!    │
│   └────────┬────────┘           └────────┬────────┘              │
│            │                              │                       │
│   ┌────────┴────────┐           ┌────────┴────────┐              │
│   │  L2 Cache        │           │  L2 Cache        │              │
│   └────────┬────────┘           └────────┬────────┘              │
│            │                              │                       │
│            └──────────┬───────────────────┘                       │
│                       │                                           │
│              ┌────────┴────────┐                                  │
│              │   MAIN MEMORY    │                                  │
│              │  running = false │                                  │
│              └─────────────────┘                                  │
│                                                                   │
│  Problem: Core 2 reads from its LOCAL CACHE, not main memory!    │
│           It never sees the update made by Core 1!               │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

**What's happening:**
1. Both threads read `running = true` into their CPU caches
2. Main thread sets `running = false` in **its cache** and eventually main memory
3. Worker thread keeps reading `running = true` from **its own cache**
4. Worker never sees the update → **runs forever**

This is known as a **visibility problem** — one thread's writes are **not visible** to other threads.

---

## 2. The volatile Keyword

### What is volatile?

The `volatile` keyword tells the JVM: **"Always read/write this variable from/to main memory — never cache it."**

```java
private static volatile boolean running = true;  // ✅ FIX!
```

### How volatile Works

```
┌──────────────────────────────────────────────────────────────────┐
│                    WITH volatile KEYWORD                           │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│   Core 1 (Main Thread)           Core 2 (Worker Thread)          │
│   ┌─────────────────┐           ┌─────────────────┐              │
│   │  L1 Cache        │           │  L1 Cache        │              │
│   │  (bypassed!)     │           │  (bypassed!)     │              │
│   └────────┬────────┘           └────────┬────────┘              │
│            │                              │                       │
│            └──── ALWAYS READ/WRITE ───────┘                       │
│                       │                                           │
│              ┌────────┴────────┐                                  │
│              │   MAIN MEMORY    │                                  │
│              │  running = false │  ← Both threads use this!       │
│              └─────────────────┘                                  │
│                                                                   │
│  ✅ Core 2 reads directly from main memory every time!            │
│     It immediately sees the update made by Core 1!               │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### Fixed Code

```java
public class VisibilityFixed {
    private static volatile boolean running = true;  // ✅ volatile!

    public static void main(String[] args) throws InterruptedException {
        Thread worker = new Thread(() -> {
            int count = 0;
            while (running) {  // Reads from main memory every time
                count++;
            }
            System.out.println("Worker stopped after count: " + count);
        });

        worker.start();

        Thread.sleep(1000);

        running = false;  // Written to main memory immediately
        System.out.println("Main: Set running to false");
    }
}
```

**Output:**
```
Main: Set running to false
Worker stopped after count: 234567890
```

✅ Worker thread now sees the update and stops!

---

### volatile Guarantees

| Guarantee | Description |
|-----------|-------------|
| **Visibility** | Writes to a `volatile` variable are immediately visible to all threads |
| **Ordering** | Prevents compiler/CPU from reordering instructions around volatile reads/writes |
| **NOT Atomicity** | `volatile` does **NOT** make compound operations (like `count++`) thread-safe |

### What volatile Does NOT Do

```java
private static volatile int counter = 0;

// ❌ NOT THREAD-SAFE even with volatile!
counter++;  // This is actually 3 operations:
            // 1. Read counter (e.g., 5)
            // 2. Add 1       (e.g., 6)
            // 3. Write back  (e.g., 6)
            // Another thread can read between steps!
```

**The `counter++` problem:**

```
┌──────────────────────────────────────────────────────────────────┐
│              WHY volatile DOESN'T FIX counter++                   │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│   Thread 1                         Thread 2                       │
│   ────────                         ────────                       │
│   Read counter = 5                                                │
│                                    Read counter = 5               │
│   Compute 5 + 1 = 6                                              │
│                                    Compute 5 + 1 = 6             │
│   Write counter = 6                                              │
│                                    Write counter = 6  ← LOST!    │
│                                                                   │
│   Expected: counter = 7                                           │
│   Actual:   counter = 6  ❌  (One increment lost!)                │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

### volatile vs synchronized

| Aspect | volatile | synchronized |
|--------|----------|-------------|
| **Visibility** | ✅ Yes | ✅ Yes |
| **Atomicity** | ❌ No | ✅ Yes |
| **Blocking** | ❌ No (non-blocking) | ✅ Yes (blocks other threads) |
| **Performance** | ⚡ Faster | 🐢 Slower (lock overhead) |
| **Use Case** | Simple flags, status variables | Compound operations, critical sections |
| **Scope** | Single variable | Block of code |

### When to Use volatile

| Scenario | Use volatile? |
|----------|--------------|
| **Stop flag** (`running = false`) | ✅ Yes |
| **Status variable** (one writer, many readers) | ✅ Yes |
| **Counter** (`count++`) | ❌ No – use `AtomicInteger` |
| **Check-then-act** (`if (x > 0) x--`) | ❌ No – use `synchronized` |
| **Multiple variables updated together** | ❌ No – use `synchronized` |

---

## 3. The Happens-Before Relationship

The **Java Memory Model (JMM)** defines rules about when one thread is **guaranteed to see** another thread's writes. These rules are called **happens-before** relationships.

### Key Happens-Before Rules

| Rule | Description |
|------|-------------|
| **Program Order** | Within a single thread, each action happens-before every action that comes after it |
| **Monitor Lock** | An unlock on a monitor happens-before every subsequent lock on that monitor |
| **volatile Variable** | A write to a volatile field happens-before every subsequent read of that field |
| **Thread Start** | A call to `Thread.start()` happens-before any action in the started thread |
| **Thread Join** | All actions in a thread happen-before another thread returns from `join()` on that thread |
| **Transitivity** | If A happens-before B, and B happens-before C, then A happens-before C |

### Why This Matters

```java
// Thread 1
data = loadFromDatabase();    // Step A
volatile_flag = true;         // Step B (volatile write)

// Thread 2
if (volatile_flag) {          // Step C (volatile read)
    process(data);            // Step D - GUARANTEED to see updated 'data'!
}
```

Because of the happens-before relationship:
- **A happens-before B** (program order)
- **B happens-before C** (volatile write → read)
- **A happens-before C** (transitivity)
- So Thread 2 reading `data` in Step D is guaranteed to see what Thread 1 wrote in Step A

---

## 4. Atomic Variables: Lock-Free Thread Safety

The `java.util.concurrent.atomic` package provides classes that support **lock-free, thread-safe** operations on single variables.

### The Problem They Solve

```java
// ❌ Not thread-safe
private int counter = 0;
counter++;  // Race condition!

// ❌ Thread-safe but slow (lock overhead)
private int counter = 0;
synchronized void increment() { counter++; }

// ✅ Thread-safe AND fast (no locking!)
private AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();  // Atomic operation!
```

### Available Atomic Classes

| Class | Type | Description |
|-------|------|-------------|
| `AtomicInteger` | int | Atomic operations on integers |
| `AtomicLong` | long | Atomic operations on longs |
| `AtomicBoolean` | boolean | Atomic operations on booleans |
| `AtomicReference<V>` | Object | Atomic operations on object references |
| `AtomicIntegerArray` | int[] | Atomic operations on integer arrays |
| `AtomicLongArray` | long[] | Atomic operations on long arrays |
| `AtomicReferenceArray<V>` | Object[] | Atomic operations on object arrays |

---

### Example 1: Thread-Safe Counter with AtomicInteger

```java
import java.util.concurrent.atomic.AtomicInteger;

class WebsiteVisitorCounter {
    private AtomicInteger visitorCount = new AtomicInteger(0);

    public void recordVisitor() {
        int newCount = visitorCount.incrementAndGet();  // Atomic!
        System.out.println(Thread.currentThread().getName() + 
                          ": Visitor #" + newCount);
    }

    public int getCount() {
        return visitorCount.get();
    }
}

public class AtomicCounterDemo {
    public static void main(String[] args) throws InterruptedException {
        WebsiteVisitorCounter counter = new WebsiteVisitorCounter();

        // Create 10 threads, each recording 100 visitors
        Thread[] threads = new Thread[10];
        for (int i = 0; i < 10; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < 100; j++) {
                    counter.recordVisitor();
                }
            }, "Thread-" + i);
            threads[i].start();
        }

        // Wait for all threads to complete
        for (Thread t : threads) {
            t.join();
        }

        System.out.println("\n═══════════════════════════════════════");
        System.out.println("Final visitor count: " + counter.getCount());
        System.out.println("Expected: 1000");
        System.out.println("═══════════════════════════════════════");
    }
}
```

**Output:**
```
Thread-0: Visitor #1
Thread-3: Visitor #2
Thread-1: Visitor #3
...
═══════════════════════════════════════
Final visitor count: 1000
Expected: 1000
═══════════════════════════════════════
```

✅ **Exactly 1000 every time** — no lost updates, no locking!

---

### Important AtomicInteger Methods

| Method | Description | Example |
|--------|-------------|---------|
| `get()` | Read current value | `int v = counter.get();` |
| `set(int)` | Set value | `counter.set(10);` |
| `incrementAndGet()` | ++counter (returns new value) | `int v = counter.incrementAndGet();` |
| `getAndIncrement()` | counter++ (returns old value) | `int v = counter.getAndIncrement();` |
| `decrementAndGet()` | --counter | `int v = counter.decrementAndGet();` |
| `addAndGet(int)` | counter += delta | `int v = counter.addAndGet(5);` |
| `compareAndSet(expect, update)` | CAS operation | `boolean ok = counter.compareAndSet(5, 10);` |
| `getAndUpdate(func)` | Apply function atomically | `counter.getAndUpdate(x -> x * 2);` |
| `updateAndGet(func)` | Apply function, return new | `counter.updateAndGet(x -> x * 2);` |

---

### Example 2: AtomicBoolean as One-Time Flag

```java
import java.util.concurrent.atomic.AtomicBoolean;

class DatabaseInitializer {
    private AtomicBoolean initialized = new AtomicBoolean(false);

    public void initialize() {
        // Only the FIRST thread to call this will actually initialize
        if (initialized.compareAndSet(false, true)) {
            System.out.println(Thread.currentThread().getName() + 
                             ": Initializing database... (first call wins!)");
            try { Thread.sleep(1000); } catch (InterruptedException e) { }
            System.out.println("Database initialized! ✓");
        } else {
            System.out.println(Thread.currentThread().getName() + 
                             ": Already initialized, skipping.");
        }
    }
}

public class AtomicBooleanDemo {
    public static void main(String[] args) {
        DatabaseInitializer db = new DatabaseInitializer();

        // 5 threads all try to initialize at once
        for (int i = 0; i < 5; i++) {
            new Thread(db::initialize, "Thread-" + i).start();
        }
    }
}
```

**Output:**
```
Thread-0: Initializing database... (first call wins!)
Thread-1: Already initialized, skipping.
Thread-2: Already initialized, skipping.
Thread-3: Already initialized, skipping.
Thread-4: Already initialized, skipping.
Database initialized! ✓
```

---

### Example 3: AtomicReference for Lock-Free Object Updates

```java
import java.util.concurrent.atomic.AtomicReference;

class UserProfile {
    final String name;
    final String email;
    final int loginCount;

    UserProfile(String name, String email, int loginCount) {
        this.name = name;
        this.email = email;
        this.loginCount = loginCount;
    }

    @Override
    public String toString() {
        return name + " (" + email + ") - Logins: " + loginCount;
    }
}

public class AtomicReferenceDemo {
    private static AtomicReference<UserProfile> currentUser = 
        new AtomicReference<>(new UserProfile("Rahul", "rahul@gmail.com", 0));

    public static void recordLogin() {
        UserProfile oldProfile, newProfile;
        do {
            oldProfile = currentUser.get();
            newProfile = new UserProfile(
                oldProfile.name, 
                oldProfile.email, 
                oldProfile.loginCount + 1  // Increment login count
            );
        } while (!currentUser.compareAndSet(oldProfile, newProfile));
        
        System.out.println(Thread.currentThread().getName() + 
                          ": Login recorded → " + newProfile);
    }

    public static void main(String[] args) throws InterruptedException {
        Thread[] threads = new Thread[5];
        for (int i = 0; i < 5; i++) {
            threads[i] = new Thread(AtomicReferenceDemo::recordLogin, "Thread-" + i);
            threads[i].start();
        }
        for (Thread t : threads) t.join();
        
        System.out.println("\nFinal: " + currentUser.get());
    }
}
```

**Output:**
```
Thread-0: Login recorded → Rahul (rahul@gmail.com) - Logins: 1
Thread-2: Login recorded → Rahul (rahul@gmail.com) - Logins: 2
Thread-1: Login recorded → Rahul (rahul@gmail.com) - Logins: 3
Thread-3: Login recorded → Rahul (rahul@gmail.com) - Logins: 4
Thread-4: Login recorded → Rahul (rahul@gmail.com) - Logins: 5
Final: Rahul (rahul@gmail.com) - Logins: 5
```

---

## 5. CAS: Compare-And-Swap (How Atomics Work Internally)

### What is CAS?

**Compare-And-Swap** is a CPU-level instruction that performs an atomic operation:

> "If the value at memory location X is still **expected_value**, replace it with **new_value**. Otherwise, do nothing."

### How CAS Works

```
┌──────────────────────────────────────────────────────────────────┐
│                    CAS (Compare-And-Swap)                          │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  compareAndSet(expectedValue, newValue)                           │
│                                                                   │
│  Step 1: Read current value from memory                          │
│          currentValue = memory[address]                           │
│                                                                   │
│  Step 2: Compare                                                  │
│          if (currentValue == expectedValue)                       │
│              memory[address] = newValue  ← Swap! ✅               │
│              return true                                          │
│          else                                                     │
│              return false  ← Someone else changed it! ❌          │
│                                                                   │
│  ALL OF THIS HAPPENS AS ONE ATOMIC CPU INSTRUCTION!              │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### CAS in Action: How `incrementAndGet()` Works

```java
// Pseudocode of AtomicInteger.incrementAndGet()
public int incrementAndGet() {
    while (true) {                           // Keep trying!
        int current = get();                 // Read current value
        int next = current + 1;              // Calculate new value
        if (compareAndSet(current, next)) {  // Try to update
            return next;                     // Success!
        }
        // If CAS failed, another thread changed the value
        // Loop back and try again with the new current value
    }
}
```

### CAS vs Locking

```
┌──────────────────────────────────────────────────────────────────┐
│             LOCKING vs CAS APPROACH                               │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  LOCKING (synchronized/ReentrantLock):                            │
│  ┌──────────────────────────────────────────────┐                │
│  │ Thread 1: ACQUIRE lock                       │                │
│  │           read counter (5)                   │                │
│  │           counter = 6                        │                │
│  │           RELEASE lock                       │                │
│  │                                              │                │
│  │ Thread 2: WAITING... WAITING... WAITING...   │ ← BLOCKED!    │
│  │           ACQUIRE lock                       │                │
│  │           read counter (6)                   │                │
│  │           counter = 7                        │                │
│  │           RELEASE lock                       │                │
│  └──────────────────────────────────────────────┘                │
│                                                                   │
│  CAS (Lock-free):                                                 │
│  ┌──────────────────────────────────────────────┐                │
│  │ Thread 1: read 5, CAS(5→6) ✅ Success!       │                │
│  │ Thread 2: read 5, CAS(5→6) ❌ Retry!         │ ← NOT BLOCKED │
│  │ Thread 2: read 6, CAS(6→7) ✅ Success!       │                │
│  └──────────────────────────────────────────────┘                │
│                                                                   │
│  CAS: No blocking, no context switching, no deadlocks!           │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

| Aspect | Locking | CAS (Lock-Free) |
|--------|---------|-----------------|
| **Blocking** | Yes – threads wait for lock | No – threads retry on failure |
| **Deadlock Risk** | Yes | No |
| **Performance (low contention)** | Good | Excellent |
| **Performance (high contention)** | Good | May degrade (excessive retries) |
| **Use Case** | Complex critical sections | Simple atomic updates |

---

## 6. LongAdder & LongAccumulator (Java 8+)

### The Problem with AtomicLong Under High Contention

When many threads compete to update the **same** `AtomicLong`, CAS failures cause **excessive retries**:

```
Thread 1: CAS(100→101) ✅
Thread 2: CAS(100→101) ❌ retry → CAS(101→102) ✅
Thread 3: CAS(100→101) ❌ retry → CAS(102→103) ✅
Thread 4: CAS(100→101) ❌ retry → CAS(103→104) ❌ retry → ...
```

### The Solution: LongAdder

`LongAdder` maintains **multiple internal counters** (cells) that threads can update independently. The final value is computed by summing all cells.

```
┌──────────────────────────────────────────────────────────────────┐
│                    LongAdder ARCHITECTURE                          │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  AtomicLong:     All threads ──► [Single Value: 42]               │
│                  (High contention! Many CAS retries)             │
│                                                                   │
│  LongAdder:      Thread 1 ──► [Cell 0: 12]                       │
│                  Thread 2 ──► [Cell 1: 15]                       │
│                  Thread 3 ──► [Cell 2: 8]                        │
│                  Thread 4 ──► [Cell 3: 7]                        │
│                                                                   │
│                  sum() = 12 + 15 + 8 + 7 = 42                    │
│                  (Low contention! Each thread has its own cell)   │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### Example: LongAdder vs AtomicLong Performance

```java
import java.util.concurrent.atomic.AtomicLong;
import java.util.concurrent.atomic.LongAdder;

public class LongAdderDemo {
    static AtomicLong atomicCounter = new AtomicLong(0);
    static LongAdder adderCounter = new LongAdder();

    public static void main(String[] args) throws InterruptedException {
        int threadCount = 10;
        int incrementsPerThread = 1_000_000;

        // Test AtomicLong
        long start1 = System.currentTimeMillis();
        Thread[] threads1 = new Thread[threadCount];
        for (int i = 0; i < threadCount; i++) {
            threads1[i] = new Thread(() -> {
                for (int j = 0; j < incrementsPerThread; j++) {
                    atomicCounter.incrementAndGet();
                }
            });
            threads1[i].start();
        }
        for (Thread t : threads1) t.join();
        long time1 = System.currentTimeMillis() - start1;

        // Test LongAdder
        long start2 = System.currentTimeMillis();
        Thread[] threads2 = new Thread[threadCount];
        for (int i = 0; i < threadCount; i++) {
            threads2[i] = new Thread(() -> {
                for (int j = 0; j < incrementsPerThread; j++) {
                    adderCounter.increment();
                }
            });
            threads2[i].start();
        }
        for (Thread t : threads2) t.join();
        long time2 = System.currentTimeMillis() - start2;

        System.out.println("═══════════════════════════════════════════");
        System.out.println("         PERFORMANCE COMPARISON");
        System.out.println("═══════════════════════════════════════════");
        System.out.println("Threads: " + threadCount + 
                          " | Increments each: " + incrementsPerThread);
        System.out.println("───────────────────────────────────────────");
        System.out.printf("AtomicLong: %d ms (value: %d)%n", 
                          time1, atomicCounter.get());
        System.out.printf("LongAdder:  %d ms (value: %d)%n", 
                          time2, adderCounter.sum());
        System.out.println("═══════════════════════════════════════════");
    }
}
```

**Typical Output:**
```
═══════════════════════════════════════════
         PERFORMANCE COMPARISON
═══════════════════════════════════════════
Threads: 10 | Increments each: 1000000
───────────────────────────────────────────
AtomicLong: 450 ms (value: 10000000)
LongAdder:  120 ms (value: 10000000)
═══════════════════════════════════════════
```

`LongAdder` is **3-4x faster** under high contention! 🚀

### When to Use What

| Scenario | Best Choice |
|----------|-------------|
| **Simple counter, low contention** | `AtomicInteger` / `AtomicLong` |
| **High-throughput counter (many threads)** | `LongAdder` |
| **Need exact current value frequently** | `AtomicLong` (sum() on LongAdder is approximate during updates) |
| **Flexible accumulation** (sum, max, min) | `LongAccumulator` |
| **Single flag or reference** | `AtomicBoolean` / `AtomicReference` |

### LongAccumulator Example

```java
import java.util.concurrent.atomic.LongAccumulator;

// Find the maximum value across threads
LongAccumulator maxFinder = new LongAccumulator(Long::max, Long.MIN_VALUE);

// Thread 1
maxFinder.accumulate(42);

// Thread 2
maxFinder.accumulate(99);

// Thread 3
maxFinder.accumulate(73);

System.out.println("Max: " + maxFinder.get());  // 99
```

---

## Summary

### Quick Reference: When to Use What?

| Problem | Solution |
|---------|----------|
| Thread can't see another thread's write | `volatile` |
| Simple flag (one writer, many readers) | `volatile boolean` |
| Thread-safe counter | `AtomicInteger` / `AtomicLong` |
| One-time initialization flag | `AtomicBoolean.compareAndSet()` |
| High-throughput counter | `LongAdder` |
| Lock-free object swaps | `AtomicReference` |
| Compound operations (check-then-act) | `synchronized` or `ReentrantLock` |

### Key Takeaways

1. ✅ **volatile** guarantees visibility but **not** atomicity
2. ✅ **Atomic classes** provide lock-free thread-safe operations via CAS
3. ✅ **CAS** (Compare-And-Swap) is a CPU instruction — no locking needed
4. ✅ **LongAdder** outperforms `AtomicLong` under high contention
5. ✅ **Happens-before** rules define when writes become visible
6. ⚠️ Use `volatile` only for simple reads/writes, not compound operations

---

### Best Practices

```
┌────────────────────────────────────────────────────────────────┐
│           VOLATILE & ATOMICS BEST PRACTICES                     │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ DO:                                                         │
│  • Use volatile for simple flags (stop signals, status)        │
│  • Use AtomicInteger/AtomicLong for counters                   │
│  • Use LongAdder for high-contention counters                  │
│  • Use compareAndSet for one-time operations                   │
│  • Prefer atomic classes over synchronized for simple ops      │
│                                                                 │
│  ❌ DON'T:                                                       │
│  • Don't assume volatile makes count++ thread-safe             │
│  • Don't use atomics for compound operations                   │
│  • Don't use volatile when multiple fields must update together │
│  • Don't overuse atomics – synchronized is simpler for         │
│    complex logic                                                │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

---

### What's Next?

In the next article, we'll explore:
- **Concurrent Collections** — `ConcurrentHashMap`, `BlockingQueue`, and more
- **Why regular collections fail** in multithreaded code
- **Producer-Consumer** pattern using `BlockingQueue`

👉 [Continue to Concurrent Collections](/posts/java-concurrent-collections/)

---

*Happy Lock-Free Coding! ⚛️🧵*
