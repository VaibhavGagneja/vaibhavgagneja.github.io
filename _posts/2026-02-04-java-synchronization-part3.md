---
title: "Java Synchronization Part 3: Deadlocks, ReentrantLock and Modern Concurrency"
description: Master deadlock prevention, ReentrantLock, and java.util.concurrent utilities for robust multithreaded applications
author: Vaibhav Gagneja
date: 2026-02-04 12:00:00 +0530
categories: [Development, Java]
tags: [java, synchronization, deadlock, reentrantlock, concurrency]
toc: true
image:
  path: /assets/photos/synchros.png
---

In **Part 1**, we covered synchronized methods and blocks. In **Part 2**, we explored inter-thread communication with `wait()` and `notify()`. Now let's tackle **deadlocks** and **modern locking mechanisms**.

---

## 1. Deadlock in Multithreading

### What is Deadlock?

**Deadlock** occurs when two or more threads are **waiting for each other forever**, and none can proceed.

### Real-Life Analogy: Two Cars at an Intersection 🚗

```
              ┌───────────────────┐
              │                   │
        Car A │ ──► Waiting for   │
              │     Car B to move │
              │         ▲         │
              │         │         │
              │    ┌────┴────┐    │
              │    │ DEADLOCK│    │
              │    └────┬────┘    │
              │         │         │
              │         ▼         │
              │     Waiting for   │ Car B
              │     Car A to move │ ◄──
              │                   │
              └───────────────────┘
              
        Both cars wait forever! Neither moves!
```

### Another Example: Pen and Paper 📝

- **Rahul** holds a **Pen** and needs **Paper** to write
- **Amit** holds **Paper** and needs a **Pen** to write
- Both wait forever for what the other has → **Deadlock!**

---

### Deadlock Conditions (All 4 Must Exist)

| Condition | Description |
|-----------|-------------|
| **1. Mutual Exclusion** | Resources cannot be shared (only one thread can use at a time) |
| **2. Hold and Wait** | Thread holds one resource while waiting for another |
| **3. No Preemption** | Resources cannot be forcibly taken from a thread |
| **4. Circular Wait** | Thread A waits for B, B waits for A (circular chain) |

---

### Deadlock Example

```java
class Resource {
    // Represents a shared resource
}

class MyThread1 extends Thread {
    Resource res1, res2;
    
    MyThread1(Resource res1, Resource res2) {
        this.res1 = res1;
        this.res2 = res2;
    }
    
    public void run() {
        synchronized (res1) {  // Step 1: Lock res1 FIRST
            System.out.println("Thread 1: Locked Resource 1");
            
            // Small delay to ensure Thread 2 locks res2
            try { Thread.sleep(1000); } catch (Exception e) { }
            
            System.out.println("Thread 1: Waiting for Resource 2...");
            synchronized (res2) {  // Step 2: Try to lock res2
                System.out.println("Thread 1: Locked Resource 2");
            }
        }
    }
}

class MyThread2 extends Thread {
    Resource res1, res2;
    
    MyThread2(Resource res1, Resource res2) {
        this.res1 = res1;
        this.res2 = res2;
    }
    
    public void run() {
        synchronized (res2) {  // Step 1: Lock res2 FIRST (opposite order!)
            System.out.println("Thread 2: Locked Resource 2");
            
            // Small delay to ensure Thread 1 locks res1
            try { Thread.sleep(1000); } catch (Exception e) { }
            
            System.out.println("Thread 2: Waiting for Resource 1...");
            synchronized (res1) {  // Step 2: Try to lock res1
                System.out.println("Thread 2: Locked Resource 1");
            }
        }
    }
}

public class DeadlockDemo {
    public static void main(String[] args) {
        Resource res1 = new Resource();
        Resource res2 = new Resource();
        
        new MyThread1(res1, res2).start();
        new MyThread2(res1, res2).start();
    }
}
```

### Explanation of Deadlock

```
┌──────────────────────────────────────────────────────────────────┐
│                         DEADLOCK SCENARIO                         │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  T=0    Thread 1: synchronized(res1) → ACQUIRES res1 lock ✓      │
│  T=0    Thread 2: synchronized(res2) → ACQUIRES res2 lock ✓      │
│                                                                   │
│  T=1    Thread 1: "Locked Resource 1"                             │
│         Thread 2: "Locked Resource 2"                             │
│                                                                   │
│  T=2    Thread 1: Thread.sleep(1000)                              │
│         Thread 2: Thread.sleep(1000)                              │
│                                                                   │
│  T=3    Thread 1: "Waiting for Resource 2..."                     │
│         Thread 1: synchronized(res2) → BLOCKED! (res2 held by T2) │
│                                                                   │
│         Thread 2: "Waiting for Resource 1..."                     │
│         Thread 2: synchronized(res1) → BLOCKED! (res1 held by T1) │
│                                                                   │
│         ┌─────────────────────────────────────────────────────┐   │
│         │                                                     │   │
│         │    Thread 1                    Thread 2             │   │
│         │       │                           │                 │   │
│         │       ▼                           ▼                 │   │
│         │   HOLDS res1 ────────────────► WANTS res1          │   │
│         │   WANTS res2 ◄──────────────── HOLDS res2          │   │
│         │       │                           │                 │   │
│         │       └─────── DEADLOCK! ─────────┘                 │   │
│         │                                                     │   │
│         └─────────────────────────────────────────────────────┘   │
│                                                                   │
│  T=∞    Both threads wait FOREVER. Program hangs! ❌              │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### Output (Program Hangs):
```
Thread 1: Locked Resource 1
Thread 2: Locked Resource 2
Thread 1: Waiting for Resource 2...
Thread 2: Waiting for Resource 1...
|
← Program freezes here! Press Ctrl+C to stop.
```

---

## 2. Preventing Deadlocks

### Strategy 1: Avoid Nested Locks

The simplest way to avoid deadlocks is to **not acquire multiple locks**.

```java
// ❌ BAD: Nested locks - deadlock risk
synchronized (res1) {
    synchronized (res2) {
        // Critical section
    }
}

// ✅ BETTER: Avoid nested locks if possible
synchronized (res1) {
    // Work with res1
}
synchronized (res2) {
    // Work with res2
}
```

---

### Strategy 2: Lock in Fixed Order

If you **must** use multiple locks, always acquire them in the **same order** everywhere.

```java
// Thread 1 - locks res1 FIRST, then res2
synchronized (res1) {
    synchronized (res2) {
        // Critical section
    }
}

// Thread 2 - SAME ORDER: res1 first, then res2
synchronized (res1) {   // ✅ Same order as Thread 1
    synchronized (res2) {
        // Critical section
    }
}
```

### Fixed Deadlock Example

```java
class MyThread2 extends Thread {
    Resource res1, res2;
    
    MyThread2(Resource res1, Resource res2) {
        this.res1 = res1;
        this.res2 = res2;
    }
    
    public void run() {
        // ✅ FIX: Lock in SAME order as Thread 1 (res1 first!)
        synchronized (res1) {   // Changed from res2 to res1
            System.out.println("Thread 2: Locked Resource 1");
            
            try { Thread.sleep(1000); } catch (Exception e) { }
            
            synchronized (res2) {   // Changed from res1 to res2
                System.out.println("Thread 2: Locked Resource 2");
            }
        }
    }
}
```

### Why This Works

```
┌──────────────────────────────────────────────────────────────────┐
│                    FIXED: NO DEADLOCK                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  T=0    Thread 1: synchronized(res1) → ACQUIRES res1 lock ✓      │
│         Thread 2: synchronized(res1) → BLOCKED (res1 held by T1) │
│                                                                   │
│  T=1    Thread 1: "Locked Resource 1"                             │
│         Thread 2: Waiting for res1...                             │
│                                                                   │
│  T=2    Thread 1: synchronized(res2) → ACQUIRES res2 lock ✓      │
│         Thread 2: Still waiting for res1...                       │
│                                                                   │
│  T=3    Thread 1: "Locked Resource 2"                             │
│         Thread 1: Exits synchronized blocks                       │
│         Thread 1: RELEASES res2 and res1 locks                    │
│                                                                   │
│  T=4    Thread 2: ACQUIRES res1 lock ✓                            │
│         Thread 2: "Locked Resource 1"                             │
│                                                                   │
│  T=5    Thread 2: synchronized(res2) → ACQUIRES res2 lock ✓      │
│         Thread 2: "Locked Resource 2"                             │
│         Thread 2: Exits synchronized blocks                       │
│                                                                   │
│  ✅ Both threads complete successfully! No deadlock!              │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### Output (No Deadlock):
```
Thread 1: Locked Resource 1
Thread 1: Locked Resource 2
Thread 2: Locked Resource 1
Thread 2: Locked Resource 2
```

---

### Strategy 3: Use Timeout with tryLock()

Using `ReentrantLock.tryLock()` with a timeout prevents indefinite waiting.

```java
if (lock.tryLock(1, TimeUnit.SECONDS)) {
    try {
        // Critical section
    } finally {
        lock.unlock();
    }
} else {
    System.out.println("Could not acquire lock, avoiding deadlock!");
    // Handle the situation gracefully
}
```

---

## 3. ReentrantLock: Modern Synchronization

The `java.util.concurrent.locks` package provides **advanced locking mechanisms** with more control than `synchronized`.

### Why Use ReentrantLock?

| Feature | synchronized | ReentrantLock |
|---------|--------------|---------------|
| **Lock/Unlock** | Automatic | Manual (`lock()` / `unlock()`) |
| **tryLock()** | ❌ No | ✅ Yes (avoid blocking) |
| **Timeout** | ❌ No | ✅ Yes (wait for limited time) |
| **Fairness** | ❌ No | ✅ Yes (FIFO order option) |
| **Interruptible** | ❌ No | ✅ Yes (can interrupt waiting thread) |
| **Multiple Conditions** | ❌ No | ✅ Yes (multiple `Condition` objects) |

---

### Important Methods

| Method | Description |
|--------|-------------|
| `lock()` | Acquires lock; waits indefinitely if unavailable |
| `unlock()` | Releases lock (**always call in finally!**) |
| `tryLock()` | Tries to acquire lock immediately; returns `true`/`false` |
| `tryLock(timeout, unit)` | Tries to acquire lock within timeout period |
| `lockInterruptibly()` | Acquires lock but can be interrupted while waiting |
| `isLocked()` | Returns `true` if lock is held by any thread |
| `isHeldByCurrentThread()` | Returns `true` if current thread holds the lock |
| `getHoldCount()` | Returns number of times current thread has acquired lock |

---

### Example 1: Basic ReentrantLock Usage

```java
import java.util.concurrent.locks.ReentrantLock;

class BankAccount {
    private int balance = 1000;
    private final ReentrantLock lock = new ReentrantLock();

    public void withdraw(int amount, String name) {
        System.out.println(name + ": Attempting to withdraw Rs. " + amount);
        
        lock.lock();  // Acquire the lock
        try {
            System.out.println(name + ": Lock acquired!");
            System.out.println(name + ": Checking balance...");
            
            if (balance >= amount) {
                // Simulate processing time
                try { Thread.sleep(1000); } catch (InterruptedException e) { }
                
                balance -= amount;
                System.out.println(name + ": Withdrawal successful!");
                System.out.println(name + ": New balance: Rs. " + balance);
            } else {
                System.out.println(name + ": Insufficient balance!");
            }
        } finally {
            lock.unlock();  // ALWAYS release in finally block!
            System.out.println(name + ": Lock released.");
        }
    }
}

class Customer extends Thread {
    BankAccount account;
    int amount;
    
    Customer(BankAccount account, int amount, String name) {
        super(name);
        this.account = account;
        this.amount = amount;
    }
    
    public void run() {
        account.withdraw(amount, getName());
    }
}

public class BankApp {
    public static void main(String[] args) {
        BankAccount account = new BankAccount();
        
        Customer rahul = new Customer(account, 800, "Rahul");
        Customer amit = new Customer(account, 500, "Amit");
        
        rahul.start();
        amit.start();
    }
}
```

### Explanation

**1. Creating the Lock:**
```java
private final ReentrantLock lock = new ReentrantLock();
```
- Create a single lock instance for the shared resource
- `final` ensures the lock object itself isn't changed

**2. Acquiring the Lock:**
```java
lock.lock();  // Blocks until lock is available
try {
    // Critical section
```
- If lock is available, thread acquires it immediately
- If lock is held by another thread, this thread **waits**

**3. Releasing the Lock (CRITICAL!):**
```java
} finally {
    lock.unlock();  // MUST be in finally block!
}
```
- **ALWAYS** release in a `finally` block
- Without `finally`, if an exception occurs, the lock is never released → **other threads wait forever!**

### Output:
```
Rahul: Attempting to withdraw Rs. 800
Amit: Attempting to withdraw Rs. 500
Rahul: Lock acquired!
Rahul: Checking balance...
Rahul: Withdrawal successful!
Rahul: New balance: Rs. 200
Rahul: Lock released.
Amit: Lock acquired!
Amit: Checking balance...
Amit: Insufficient balance!
Amit: Lock released.
```

---

### Example 2: Avoiding Deadlock with tryLock()

```java
import java.util.concurrent.locks.ReentrantLock;
import java.util.concurrent.TimeUnit;

class MyThread1 extends Thread {
    private final ReentrantLock lock1, lock2;

    MyThread1(ReentrantLock lock1, ReentrantLock lock2) {
        super("Thread-1");
        this.lock1 = lock1;
        this.lock2 = lock2;
    }

    @Override
    public void run() {
        try {
            // Try to acquire lock1 with 1 second timeout
            if (lock1.tryLock(1, TimeUnit.SECONDS)) {
                try {
                    System.out.println(getName() + ": Locked Resource 1");
                    Thread.sleep(500);  // Simulate work

                    // Try to acquire lock2 with 1 second timeout
                    if (lock2.tryLock(1, TimeUnit.SECONDS)) {
                        try {
                            System.out.println(getName() + ": Locked Resource 2");
                            System.out.println(getName() + ": Doing work with both resources...");
                        } finally {
                            lock2.unlock();
                            System.out.println(getName() + ": Released Resource 2");
                        }
                    } else {
                        // ✅ Could not get lock2, back off gracefully
                        System.out.println(getName() + ": Could not get Resource 2, backing off!");
                    }
                } finally {
                    lock1.unlock();
                    System.out.println(getName() + ": Released Resource 1");
                }
            } else {
                System.out.println(getName() + ": Could not get Resource 1, skipping!");
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

class MyThread2 extends Thread {
    private final ReentrantLock lock1, lock2;

    MyThread2(ReentrantLock lock1, ReentrantLock lock2) {
        super("Thread-2");
        this.lock1 = lock1;
        this.lock2 = lock2;
    }

    @Override
    public void run() {
        try {
            // Trying opposite order - would normally cause deadlock!
            if (lock2.tryLock(1, TimeUnit.SECONDS)) {
                try {
                    System.out.println(getName() + ": Locked Resource 2");
                    Thread.sleep(500);

                    if (lock1.tryLock(1, TimeUnit.SECONDS)) {
                        try {
                            System.out.println(getName() + ": Locked Resource 1");
                            System.out.println(getName() + ": Doing work with both resources...");
                        } finally {
                            lock1.unlock();
                            System.out.println(getName() + ": Released Resource 1");
                        }
                    } else {
                        // ✅ Could not get lock1, back off gracefully
                        System.out.println(getName() + ": Could not get Resource 1, backing off!");
                    }
                } finally {
                    lock2.unlock();
                    System.out.println(getName() + ": Released Resource 2");
                }
            } else {
                System.out.println(getName() + ": Could not get Resource 2, skipping!");
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

public class DeadlockPrevention {
    public static void main(String[] args) {
        ReentrantLock lock1 = new ReentrantLock();
        ReentrantLock lock2 = new ReentrantLock();

        new MyThread1(lock1, lock2).start();
        new MyThread2(lock1, lock2).start();
    }
}
```

### Explanation

**1. tryLock() with Timeout:**
```java
if (lock1.tryLock(1, TimeUnit.SECONDS)) {
    // Got the lock!
} else {
    // Couldn't get lock in 1 second, do something else
}
```
- Thread tries to acquire lock for **maximum 1 second**
- If successful, returns `true` and thread proceeds
- If timeout expires, returns `false` and thread can **back off gracefully**

**2. Why This Prevents Deadlock:**
- With `synchronized`, threads wait **forever** for a lock
- With `tryLock(timeout)`, threads **give up** after waiting too long
- The thread that backs off can retry later or do alternative work

### Output:
```
Thread-1: Locked Resource 1
Thread-2: Locked Resource 2
Thread-1: Could not get Resource 2, backing off!
Thread-1: Released Resource 1
Thread-2: Could not get Resource 1, backing off!
Thread-2: Released Resource 2
```

**Notice:** Both threads complete! No deadlock! They simply backed off when they couldn't get the second resource.

---

## 4. java.util.concurrent Package Overview

Java provides a comprehensive set of **high-level concurrency utilities**:

### Locks

| Class | Description | Use Case |
|-------|-------------|----------|
| `ReentrantLock` | Explicit lock with tryLock, timeout | Fine-grained control |
| `ReentrantReadWriteLock` | Separate read and write locks | Read-heavy workloads |
| `StampedLock` | Optimistic locking | High-performance reads |

### Synchronizers

| Class | Description | Use Case |
|-------|-------------|----------|
| `Semaphore` | Controls N threads accessing resource | Connection pool |
| `CountDownLatch` | Waits for N operations to complete | Startup initialization |
| `CyclicBarrier` | Threads wait for each other at barrier | Parallel processing |

### Executors (Thread Pools)

| Interface/Class | Description | Use Case |
|----------------|-------------|----------|
| `ExecutorService` | Manages thread pools | Async task execution |
| `ScheduledExecutorService` | Delayed/periodic tasks | Scheduled jobs |
| `CompletableFuture` | Async computation with chaining | Modern async code |

### Concurrent Collections

| Class | Description | Use Case |
|-------|-------------|----------|
| `ConcurrentHashMap` | Thread-safe map | Shared cache |
| `CopyOnWriteArrayList` | Thread-safe list (read-heavy) | Config lists |
| `BlockingQueue` | Producer-consumer queues | Task queues |

### Atomic Classes

| Class | Description | Use Case |
|-------|-------------|----------|
| `AtomicInteger` | Lock-free int operations | Counters |
| `AtomicLong` | Lock-free long operations | High-performance counters |
| `AtomicReference` | Lock-free object references | Non-blocking algorithms |

---

## Summary

### Quick Reference: When to Use What?

| Scenario | Solution |
|----------|----------|
| Avoid deadlock with synchronized | Lock resources in consistent order |
| Need tryLock or timeout | `ReentrantLock` |
| Back off if lock unavailable | `tryLock(timeout, unit)` |
| High-performance counters | `AtomicInteger` / `AtomicLong` |
| Thread-safe collections | `ConcurrentHashMap`, etc. |

### Key Takeaways

1. ✅ **Deadlock** occurs when threads wait for each other's locks forever
2. ✅ **Prevent deadlocks** by locking resources in consistent order
3. ✅ **ReentrantLock** provides more control than `synchronized`
4. ✅ **tryLock() with timeout** prevents indefinite waiting
5. ✅ **Always unlock in finally block** to prevent resource leaks
6. ⚠️ **java.util.concurrent** provides high-level utilities for complex scenarios

---

### Best Practices Summary

```
┌────────────────────────────────────────────────────────────────┐
│                 SYNCHRONIZATION BEST PRACTICES                  │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ DO:                                                         │
│  • Always lock resources in a consistent order                  │
│  • Use tryLock() with timeout for deadlock prevention          │
│  • Release locks in finally blocks                              │
│  • Consider java.util.concurrent utilities first               │
│                                                                 │
│  ❌ DON'T:                                                       │
│  • Don't hold locks longer than necessary                       │
│  • Don't call unknown code while holding a lock                │
│  • Don't ignore InterruptedException                           │
│  • Don't use String or Integer as lock objects                 │
│  • Don't assume thread scheduling order                        │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

---

### What's Next?

Continue your concurrency journey:
- 👉 [ExecutorService & CompletableFuture](/posts/java-executorservice-completablefuture/) — Thread Pools, Callable, Future, async programming
- 👉 [volatile, Atomics & Java Memory Model](/posts/java-volatile-atomics/) — Visibility, CAS, lock-free programming

---

*Happy Thread-Safe Coding! 🔒🧵*
