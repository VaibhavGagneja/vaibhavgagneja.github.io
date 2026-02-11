---
title: "Java Concurrency: Synchronizers — CountDownLatch, Semaphore, CyclicBarrier & Locks"
description: Master Java synchronizers including CountDownLatch, CyclicBarrier, Semaphore, Phaser, ReadWriteLock and StampedLock
author: Vaibhav Gagneja
date: 2026-02-08 12:00:00 +0530
categories: [Development, Java]
tags: [java, concurrency, synchronizers, countdownlatch, semaphore, cyclicbarrier, locks]
toc: true
image:
  path: /assets/photos/synchros.png
---

So far in this series, we've covered thread synchronization, concurrent collections, and atomic variables. But real-world applications often require more sophisticated **coordination** — waiting for multiple tasks to finish, limiting concurrent access, or synchronizing threads at checkpoints. Java provides a rich set of **synchronizers** for exactly these purposes.

---

## 1. CountDownLatch — "Wait for N Tasks to Complete"

### The Concept

A `CountDownLatch` starts with a **count**. Threads call `countDown()` to decrement it, and other threads call `await()` to **block until the count reaches zero**.

```
┌──────────────────────────────────────────────────────────────────┐
│                     CountDownLatch                                 │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Latch count = 3                                                  │
│                                                                   │
│  Thread-A: countDown() → count = 2                               │
│  Thread-B: countDown() → count = 1                               │
│  Thread-C: countDown() → count = 0                               │
│                                                                   │
│  Main Thread: await() ─────► UNBLOCKED! ✅                        │
│                 (was waiting for count to reach 0)               │
│                                                                   │
│  ⚠️ One-time use only! Cannot reset the count.                   │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### Example: App Startup — Wait for All Services

```java
import java.util.concurrent.CountDownLatch;

public class AppStartup {
    public static void main(String[] args) throws InterruptedException {
        int serviceCount = 3;
        CountDownLatch latch = new CountDownLatch(serviceCount);

        System.out.println("🚀 Starting application...\n");

        // Service 1: Database
        new Thread(() -> {
            try {
                System.out.println("[DB] Connecting to database...");
                Thread.sleep(2000);
                System.out.println("[DB] ✓ Database ready!");
                latch.countDown();  // count: 3 → 2
            } catch (InterruptedException e) { }
        }).start();

        // Service 2: Cache
        new Thread(() -> {
            try {
                System.out.println("[Cache] Warming up cache...");
                Thread.sleep(1500);
                System.out.println("[Cache] ✓ Cache ready!");
                latch.countDown();  // count: 2 → 1
            } catch (InterruptedException e) { }
        }).start();

        // Service 3: Message Queue
        new Thread(() -> {
            try {
                System.out.println("[MQ] Connecting to message queue...");
                Thread.sleep(1000);
                System.out.println("[MQ] ✓ Message queue ready!");
                latch.countDown();  // count: 1 → 0
            } catch (InterruptedException e) { }
        }).start();

        // Main thread waits for ALL services
        latch.await();  // Blocks until count = 0

        System.out.println("\n═══════════════════════════════════════");
        System.out.println("✅ All services ready! Application started!");
        System.out.println("═══════════════════════════════════════");
    }
}
```

**Output:**
```
🚀 Starting application...

[DB] Connecting to database...
[Cache] Warming up cache...
[MQ] Connecting to message queue...
[MQ] ✓ Message queue ready!
[Cache] ✓ Cache ready!
[DB] ✓ Database ready!

═══════════════════════════════════════
✅ All services ready! Application started!
═══════════════════════════════════════
```

### Key Points

| Method | Behavior |
|--------|----------|
| `new CountDownLatch(N)` | Create with initial count N |
| `countDown()` | Decrement count by 1 (if count > 0) |
| `await()` | Block until count reaches 0 |
| `await(timeout, unit)` | Block with timeout |
| `getCount()` | Get current count |

> **Important:** A `CountDownLatch` is **one-time use**. Once the count reaches zero, it stays zero. Use `CyclicBarrier` if you need to reset.

---

## 2. CyclicBarrier — "Threads Wait for Each Other"

### The Concept

A `CyclicBarrier` makes a group of threads **wait at a common point** (the barrier) until ALL threads have arrived. Then all are released simultaneously.

```
┌──────────────────────────────────────────────────────────────────┐
│                      CyclicBarrier                                │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Barrier parties = 3                                              │
│                                                                   │
│  Thread-A: ──────────► await() ───┐                              │
│                                    │ WAITING...                  │
│  Thread-B: ────► await() ─────────┤                              │
│                                    │ WAITING...                  │
│  Thread-C: ──► await() ──────────┤                              │
│                                    │                              │
│                               All 3 arrived!                     │
│                               ═══════════════                    │
│                               BARRIER TRIPPED!                   │
│                               ═══════════════                    │
│                                    │                              │
│  Thread-A: ◄───────────────────────┤                              │
│  Thread-B: ◄───────────────────────┤  All proceed together!     │
│  Thread-C: ◄───────────────────────┘                              │
│                                                                   │
│  ✅ Can be REUSED (cyclic!) for multiple rounds                  │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### CyclicBarrier vs CountDownLatch

| Feature | CountDownLatch | CyclicBarrier |
|---------|---------------|---------------|
| **Reusable?** | ❌ One-time only | ✅ Resets automatically |
| **Who waits?** | One or more threads wait | All participating threads wait |
| **Who counts down?** | Any thread can count down | Threads count down by awaiting |
| **Barrier action?** | ❌ No | ✅ Optional action when all arrive |

### Example: Multiplayer Game — Wait for All Players

```java
import java.util.concurrent.*;

public class MultiplayerGame {
    public static void main(String[] args) {
        int playerCount = 4;

        // Barrier action runs when all players are ready
        CyclicBarrier barrier = new CyclicBarrier(playerCount, () -> {
            System.out.println("\n🎮 ALL PLAYERS READY — ROUND STARTS! 🎮\n");
        });

        String[] players = {"Alice", "Bob", "Charlie", "Diana"};

        for (String player : players) {
            new Thread(() -> {
                try {
                    // Round 1
                    int loadTime = (int)(Math.random() * 2000) + 500;
                    System.out.println("  " + player + " loading... (" + 
                                     loadTime + "ms)");
                    Thread.sleep(loadTime);
                    System.out.println("  " + player + " ✓ READY!");
                    barrier.await();  // Wait for all players

                    // Round 2 (barrier resets automatically!)
                    int playTime = (int)(Math.random() * 1500) + 500;
                    System.out.println("  " + player + " playing round... (" + 
                                     playTime + "ms)");
                    Thread.sleep(playTime);
                    System.out.println("  " + player + " ✓ FINISHED ROUND!");
                    barrier.await();  // Wait again for all to finish

                    System.out.println("  " + player + " → Moving to next round!");

                } catch (InterruptedException | BrokenBarrierException e) {
                    e.printStackTrace();
                }
            }, player).start();
        }
    }
}
```

**Output:**
```
  Alice loading... (1200ms)
  Bob loading... (800ms)
  Charlie loading... (1800ms)
  Diana loading... (600ms)
  Diana ✓ READY!
  Bob ✓ READY!
  Alice ✓ READY!
  Charlie ✓ READY!

🎮 ALL PLAYERS READY — ROUND STARTS! 🎮

  Alice playing round... (900ms)
  Bob playing round... (1100ms)
  Charlie playing round... (700ms)
  Diana playing round... (1400ms)
  Charlie ✓ FINISHED ROUND!
  Alice ✓ FINISHED ROUND!
  Bob ✓ FINISHED ROUND!
  Diana ✓ FINISHED ROUND!

🎮 ALL PLAYERS READY — ROUND STARTS! 🎮

  Alice → Moving to next round!
  Charlie → Moving to next round!
  Bob → Moving to next round!
  Diana → Moving to next round!
```

---

## 3. Semaphore — "Limit Concurrent Access to N"

### The Concept

A `Semaphore` controls access to a shared resource by maintaining a set of **permits**. Threads must acquire a permit before accessing the resource and release it when done.

```
┌──────────────────────────────────────────────────────────────────┐
│                       Semaphore                                    │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Semaphore permits = 3                                            │
│                                                                   │
│  Thread-A: acquire() → permits = 2  ✅ IN                        │
│  Thread-B: acquire() → permits = 1  ✅ IN                        │
│  Thread-C: acquire() → permits = 0  ✅ IN                        │
│  Thread-D: acquire() → BLOCKED! (no permits left)               │
│  Thread-E: acquire() → BLOCKED!                                  │
│                                                                   │
│  Thread-A: release() → permits = 1                               │
│  Thread-D: acquire() → permits = 0  ✅ IN (unblocked!)           │
│                                                                   │
│  Key: acquire() = get permit, release() = return permit          │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### Example: Database Connection Pool

```java
import java.util.concurrent.Semaphore;

public class ConnectionPoolDemo {
    // Only 3 connections available
    private static final Semaphore connectionPool = new Semaphore(3);
    private static int activeConnections = 0;

    static void queryDatabase(String user) {
        try {
            System.out.println(user + " waiting for connection...");
            
            connectionPool.acquire();  // Get a permit (blocks if none available)
            
            activeConnections++;
            System.out.println(user + " ✓ connected! (Active: " + 
                             activeConnections + "/3)");
            
            // Simulate database query
            Thread.sleep((long)(Math.random() * 2000) + 1000);
            
            activeConnections--;
            System.out.println(user + " done. Releasing connection. (Active: " + 
                             activeConnections + "/3)");
            
            connectionPool.release();  // Return the permit
            
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    public static void main(String[] args) {
        String[] users = {"Alice", "Bob", "Charlie", "Diana", 
                         "Eve", "Frank", "Grace"};

        System.out.println("Connection pool size: 3");
        System.out.println("Users requesting: " + users.length);
        System.out.println("═══════════════════════════════════════\n");

        for (String user : users) {
            new Thread(() -> queryDatabase(user), user).start();
        }
    }
}
```

**Output:**
```
Connection pool size: 3
Users requesting: 7
═══════════════════════════════════════

Alice waiting for connection...
Bob waiting for connection...
Charlie waiting for connection...
Diana waiting for connection...
Eve waiting for connection...
Alice ✓ connected! (Active: 1/3)
Bob ✓ connected! (Active: 2/3)
Charlie ✓ connected! (Active: 3/3)
Frank waiting for connection...
Grace waiting for connection...
Bob done. Releasing connection. (Active: 2/3)
Diana ✓ connected! (Active: 3/3)
Alice done. Releasing connection. (Active: 2/3)
Eve ✓ connected! (Active: 3/3)
...
```

### Key Methods

| Method | Description |
|--------|-------------|
| `new Semaphore(permits)` | Create with N permits |
| `new Semaphore(permits, true)` | Create with **fairness** (FIFO order) |
| `acquire()` | Get 1 permit (blocks if unavailable) |
| `acquire(n)` | Get N permits |
| `tryAcquire()` | Try to get permit, return false if unavailable |
| `tryAcquire(timeout, unit)` | Try with timeout |
| `release()` | Return 1 permit |
| `availablePermits()` | Check available permits |

---

## 4. Phaser — Flexible Multi-Phase Synchronization

`Phaser` is a **reusable, flexible** synchronizer that can replace both `CountDownLatch` and `CyclicBarrier`. It supports **dynamic registration** — threads can join and leave at any point.

### Example: Multi-Phase Data Processing

```java
import java.util.concurrent.Phaser;

public class DataPipelineDemo {
    public static void main(String[] args) {
        // 3 worker threads will participate
        Phaser phaser = new Phaser(1);  // 1 = main thread registered

        String[] workers = {"Extractor", "Transformer", "Loader"};

        for (String worker : workers) {
            phaser.register();  // Register each worker
            new Thread(() -> {
                // Phase 0: Extract
                System.out.println("  [" + worker + "] Phase 0: Extracting data...");
                sleep(500 + (int)(Math.random() * 1000));
                System.out.println("  [" + worker + "] ✓ Extract done");
                phaser.arriveAndAwaitAdvance();  // Wait for all to finish phase 0

                // Phase 1: Transform
                System.out.println("  [" + worker + "] Phase 1: Transforming data...");
                sleep(500 + (int)(Math.random() * 1000));
                System.out.println("  [" + worker + "] ✓ Transform done");
                phaser.arriveAndAwaitAdvance();  // Wait for all to finish phase 1

                // Phase 2: Load
                System.out.println("  [" + worker + "] Phase 2: Loading data...");
                sleep(500 + (int)(Math.random() * 1000));
                System.out.println("  [" + worker + "] ✓ Load done");
                phaser.arriveAndDeregister();  // Done — deregister
            }, worker).start();
        }

        // Main thread coordinates phases
        for (int phase = 0; phase < 3; phase++) {
            phaser.arriveAndAwaitAdvance();
            System.out.println("\n══ Phase " + phase + " complete! ══\n");
        }
        phaser.arriveAndDeregister();
        System.out.println("Pipeline finished! 🎉");
    }

    static void sleep(long ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) { }
    }
}
```

### When to Use Phaser vs Others

| Scenario | Best Choice |
|----------|-------------|
| Wait for N tasks to finish (one-time) | `CountDownLatch` |
| Threads wait for each other (reusable) | `CyclicBarrier` |
| Dynamic number of participants | `Phaser` |
| Multiple phases with varying participants | `Phaser` |

---

## 5. Exchanger — Two Threads Swap Data

`Exchanger<V>` is a synchronization point at which **two threads can swap objects**. When one thread calls `exchange()`, it blocks until the other thread also calls `exchange()`.

### Example: Producer-Consumer Data Swap

```java
import java.util.concurrent.Exchanger;
import java.util.ArrayList;
import java.util.List;

public class ExchangerDemo {
    public static void main(String[] args) {
        Exchanger<List<String>> exchanger = new Exchanger<>();

        // Producer fills a buffer and exchanges it for an empty one
        new Thread(() -> {
            try {
                List<String> buffer = new ArrayList<>();
                for (int round = 1; round <= 3; round++) {
                    // Fill buffer
                    buffer.add("Data-" + round + "A");
                    buffer.add("Data-" + round + "B");
                    System.out.println("Producer filled: " + buffer);

                    // Swap full buffer for empty one
                    buffer = exchanger.exchange(buffer);
                    System.out.println("Producer got empty buffer: " + buffer);
                }
            } catch (InterruptedException e) { }
        }, "Producer").start();

        // Consumer takes the full buffer and returns an empty one
        new Thread(() -> {
            try {
                List<String> buffer = new ArrayList<>();
                for (int round = 1; round <= 3; round++) {
                    // Swap empty buffer for full one
                    buffer = exchanger.exchange(buffer);
                    System.out.println("Consumer received: " + buffer);

                    // Process data and clear
                    buffer.clear();
                }
            } catch (InterruptedException e) { }
        }, "Consumer").start();
    }
}
```

---

## 6. ReadWriteLock — Fine-Grained Read/Write Access

### The Problem

With `synchronized` or `ReentrantLock`, a thread reading data **blocks** all other readers — even though reads don't conflict with each other!

```
┌──────────────────────────────────────────────────────────────────┐
│                  THE PROBLEM WITH SINGLE LOCK                     │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ReentrantLock (single lock):                                     │
│  ┌──────────────────────────────────────────────┐                │
│  │ Reader-1: LOCKED (reading)                   │                │
│  │ Reader-2: WAITING... (could read safely!)    │ ← WASTED!     │
│  │ Reader-3: WAITING... (could read safely!)    │ ← WASTED!     │
│  │ Writer-1: WAITING... (must wait, OK)         │ ← CORRECT     │
│  └──────────────────────────────────────────────┘                │
│                                                                   │
│  ReadWriteLock:                                                   │
│  ┌──────────────────────────────────────────────┐                │
│  │ Reader-1: READ LOCK (reading)                │                │
│  │ Reader-2: READ LOCK (reading concurrently!)  │ ← FAST! ✅     │
│  │ Reader-3: READ LOCK (reading concurrently!)  │ ← FAST! ✅     │
│  │ Writer-1: WAITING for write lock...          │ ← CORRECT     │
│  └──────────────────────────────────────────────┘                │
│                                                                   │
│  Multiple readers in parallel! Writers still exclusive!          │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### Rules

| Lock State | New Read Request | New Write Request |
|------------|------------------|-------------------|
| **No locks held** | ✅ Read lock granted | ✅ Write lock granted |
| **Read lock(s) held** | ✅ Read lock granted | ❌ Blocked until readers done |
| **Write lock held** | ❌ Blocked | ❌ Blocked |

### Example: In-Memory Cache

```java
import java.util.concurrent.locks.*;
import java.util.*;

class ThreadSafeCache<K, V> {
    private final Map<K, V> cache = new HashMap<>();
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Lock readLock = rwLock.readLock();
    private final Lock writeLock = rwLock.writeLock();

    // Multiple threads can read simultaneously! ✅
    public V get(K key) {
        readLock.lock();
        try {
            System.out.println(Thread.currentThread().getName() + 
                             " reading key: " + key);
            return cache.get(key);
        } finally {
            readLock.unlock();
        }
    }

    // Only ONE thread can write at a time (exclusive) ✅
    public void put(K key, V value) {
        writeLock.lock();
        try {
            System.out.println(Thread.currentThread().getName() + 
                             " writing: " + key + " = " + value);
            cache.put(key, value);
        } finally {
            writeLock.unlock();
        }
    }

    public int size() {
        readLock.lock();
        try {
            return cache.size();
        } finally {
            readLock.unlock();
        }
    }
}

public class ReadWriteLockDemo {
    public static void main(String[] args) throws InterruptedException {
        ThreadSafeCache<String, String> cache = new ThreadSafeCache<>();

        // Writer thread
        Thread writer = new Thread(() -> {
            String[] cities = {"Mumbai", "Delhi", "Bangalore", "Chennai"};
            for (int i = 0; i < cities.length; i++) {
                cache.put("city" + i, cities[i]);
                try { Thread.sleep(500); } catch (InterruptedException e) { }
            }
        }, "Writer");

        // Multiple reader threads
        Thread[] readers = new Thread[3];
        for (int i = 0; i < 3; i++) {
            readers[i] = new Thread(() -> {
                for (int j = 0; j < 4; j++) {
                    String val = cache.get("city" + j);
                    System.out.println("  " + Thread.currentThread().getName() + 
                                     " got: city" + j + " = " + val);
                    try { Thread.sleep(200); } catch (InterruptedException e) { }
                }
            }, "Reader-" + i);
        }

        writer.start();
        Thread.sleep(100);  // Let writer add some data first
        for (Thread r : readers) r.start();

        writer.join();
        for (Thread r : readers) r.join();

        System.out.println("\nFinal cache size: " + cache.size());
    }
}
```

---

## 7. StampedLock — Optimistic Reading (Java 8+)

`StampedLock` is a more advanced version of `ReadWriteLock` that supports **optimistic reading** — a read that **doesn't acquire any lock at all**.

### How Optimistic Reading Works

```
┌──────────────────────────────────────────────────────────────────┐
│                    StampedLock MODES                               │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  1. WRITE LOCK (exclusive, same as ReentrantLock)                │
│     long stamp = lock.writeLock();                               │
│     // ... write ...                                              │
│     lock.unlockWrite(stamp);                                     │
│                                                                   │
│  2. READ LOCK (shared, same as ReadWriteLock)                    │
│     long stamp = lock.readLock();                                │
│     // ... read ...                                               │
│     lock.unlockRead(stamp);                                      │
│                                                                   │
│  3. OPTIMISTIC READ (no lock at all! 🚀)                        │
│     long stamp = lock.tryOptimisticRead();  // No lock acquired! │
│     // ... read values ...                                       │
│     if (lock.validate(stamp)) {                                  │
│         // No write happened → values are good ✅                 │
│     } else {                                                     │
│         // A write happened → fall back to read lock ⚠️           │
│     }                                                            │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### Example: Point Class with Optimistic Reading

```java
import java.util.concurrent.locks.StampedLock;

class Point {
    private double x, y;
    private final StampedLock sl = new StampedLock();

    // Write — exclusive access
    void move(double deltaX, double deltaY) {
        long stamp = sl.writeLock();
        try {
            x += deltaX;
            y += deltaY;
        } finally {
            sl.unlockWrite(stamp);
        }
    }

    // Read — optimistic first, fall back to read lock
    double distanceFromOrigin() {
        // Step 1: Try optimistic read (no lock!)
        long stamp = sl.tryOptimisticRead();
        double currentX = x;
        double currentY = y;

        // Step 2: Validate — did a write happen during our read?
        if (!sl.validate(stamp)) {
            // A write occurred! Fall back to read lock
            stamp = sl.readLock();
            try {
                currentX = x;
                currentY = y;
            } finally {
                sl.unlockRead(stamp);
            }
        }

        // Step 3: Use the values (guaranteed consistent)
        return Math.sqrt(currentX * currentX + currentY * currentY);
    }
}
```

### When to Use StampedLock

| Lock Type | Best For |
|-----------|----------|
| `synchronized` | Simple cases, low complexity |
| `ReentrantLock` | Need tryLock, conditions, or interruptibility |
| `ReadWriteLock` | Read-heavy workload with moderate writes |
| `StampedLock` | **Extreme** read-heavy workload, performance-critical |

> **⚠️ Warning:** `StampedLock` is **NOT reentrant**! If a thread holds a write lock and tries to acquire it again, it will **deadlock**.

---

## Comparison: All Synchronizers at a Glance

| Synchronizer | Purpose | Reusable? | Key Method |
|-------------|---------|-----------|------------|
| `CountDownLatch` | Wait for N events | ❌ One-time | `countDown()` / `await()` |
| `CyclicBarrier` | Threads meet at checkpoint | ✅ Automatic | `await()` |
| `Semaphore` | Limit concurrent access | ✅ Always | `acquire()` / `release()` |
| `Phaser` | Multi-phase coordination | ✅ Per-phase | `arriveAndAwaitAdvance()` |
| `Exchanger` | 2 threads swap data | ✅ Always | `exchange(V)` |
| `ReadWriteLock` | Many readers, few writers | N/A | `readLock()` / `writeLock()` |
| `StampedLock` | Optimistic reads | N/A | `tryOptimisticRead()` |

---

## Summary

### Key Takeaways

1. ✅ **CountDownLatch** — "Wait until N tasks complete" (startup gates, test harnesses)
2. ✅ **CyclicBarrier** — "Wait until all threads arrive, then release together" (game rounds, phased processing)
3. ✅ **Semaphore** — "Only N threads can access this at once" (connection pools, rate limiting)
4. ✅ **Phaser** — "Dynamic, multi-phase synchronization" (pipelines, iterative algorithms)
5. ✅ **ReadWriteLock** — "Multiple readers OR one writer" (caches, read-heavy structures)
6. ✅ **StampedLock** — "Even faster reads with optimistic locking" (performance-critical code)

---

### Quick Decision Guide

```
Do you need to...

  Wait for N tasks to finish?
    → CountDownLatch (one-time) or Phaser (reusable)

  Have threads synchronize at a checkpoint?
    → CyclicBarrier (fixed) or Phaser (dynamic)

  Limit concurrent access to a resource?
    → Semaphore

  Allow concurrent reads but exclusive writes?
    → ReadWriteLock or StampedLock

  Two threads exchange data?
    → Exchanger
```

---

### What's Next?

In the next article, we'll explore:
- **Fork/Join Framework** — divide-and-conquer parallelism
- **ThreadLocal** — thread-confined variables
- **Virtual Threads** — Java 21's revolutionary lightweight threads

👉 [Continue to Fork/Join & Virtual Threads](/posts/java-forkjoin-virtual-threads/)

---

*Happy Synchronizing! 🔄🧵*
