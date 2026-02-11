---
title: "Java Concurrency: Fork/Join Framework, ThreadLocal & Virtual Threads"
description: Master the Fork/Join framework for parallel divide-and-conquer, ThreadLocal for thread-confined data, and Java 21's revolutionary Virtual Threads
author: Vaibhav Gagneja
date: 2026-02-09 12:00:00 +0530
categories: [Development, Java]
tags: [java, concurrency, forkjoin, virtual-threads, threadlocal, project-loom]
toc: true
image:
  path: /assets/photos/synchros.png
---

We've covered thread safety, synchronizers, and concurrent collections. Now let's explore three powerful mechanisms: **Fork/Join** for divide-and-conquer parallelism, **ThreadLocal** for thread-confined data, and the game-changing **Virtual Threads** from Java 21.

---

## 1. Fork/Join Framework — Divide and Conquer

### The Concept

The Fork/Join framework is designed for **recursive, parallelizable** tasks — problems that can be split into smaller sub-problems, solved independently, and then combined.

```
┌──────────────────────────────────────────────────────────────────┐
│              FORK/JOIN: Divide and Conquer                        │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  PROBLEM: Sum an array of 1,000,000 numbers                     │
│                                                                   │
│             [Full Array: 1M elements]                             │
│                    │                                              │
│              FORK  │  (split into halves)                        │
│            ┌───────┴───────┐                                     │
│            │               │                                     │
│       [Left 500K]    [Right 500K]                                │
│            │               │                                     │
│      FORK  │         FORK  │                                     │
│      ┌─────┴─┐       ┌────┴──┐                                  │
│   [250K]  [250K]  [250K]  [250K]                                │
│      │       │       │       │                                   │
│    sum=X   sum=Y   sum=Z   sum=W    ← Compute in parallel!     │
│      │       │       │       │                                   │
│      └───┬───┘       └───┬───┘                                  │
│        X + Y           Z + W         ← JOIN (combine results)   │
│          │               │                                       │
│          └───────┬───────┘                                       │
│              TOTAL SUM                                           │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### Key Components

| Component | Purpose |
|-----------|---------|
| `ForkJoinPool` | Thread pool optimized for fork/join tasks |
| `RecursiveTask<V>` | A task that returns a result |
| `RecursiveAction` | A task that doesn't return a result |
| `fork()` | Submit sub-task for asynchronous execution |
| `join()` | Wait for sub-task to complete and get result |

### Work-Stealing Algorithm

What makes `ForkJoinPool` special is its **work-stealing** strategy:

```
┌──────────────────────────────────────────────────────────────────┐
│                   WORK-STEALING ALGORITHM                         │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Each thread has its own DEQUE (double-ended queue):             │
│                                                                   │
│  Thread-0: [Task-A] [Task-B] [Task-C]  ← Pushes to head        │
│  Thread-1: [Task-D]                     ← Almost done!           │
│  Thread-2: [Task-E] [Task-F], [Task-G]                           │
│  Thread-3: [ EMPTY ]                    ← IDLE!                  │
│                                                                   │
│  Thread-3 is idle → STEALS from Thread-0's TAIL!                │
│                                                                   │
│  Thread-0: [Task-A] [Task-B]            ← Task-C was stolen!    │
│  Thread-3: [Task-C]                     ← Working now!           │
│                                                                   │
│  Result: All CPUs stay busy! Maximum parallelism! 🚀             │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### Example 1: Parallel Sum with RecursiveTask

```java
import java.util.concurrent.*;

class ParallelSum extends RecursiveTask<Long> {
    private static final int THRESHOLD = 10_000;  // When to stop splitting
    private final long[] array;
    private final int start, end;

    ParallelSum(long[] array, int start, int end) {
        this.array = array;
        this.start = start;
        this.end = end;
    }

    @Override
    protected Long compute() {
        int length = end - start;

        // Base case: small enough to compute directly
        if (length <= THRESHOLD) {
            long sum = 0;
            for (int i = start; i < end; i++) {
                sum += array[i];
            }
            return sum;
        }

        // Split the task into two halves
        int mid = start + length / 2;

        ParallelSum leftTask = new ParallelSum(array, start, mid);
        ParallelSum rightTask = new ParallelSum(array, mid, end);

        leftTask.fork();   // Submit left half to pool (async)
        
        long rightResult = rightTask.compute();  // Compute right half in current thread
        long leftResult = leftTask.join();        // Wait for left half result

        return leftResult + rightResult;  // Combine!
    }
}

public class ForkJoinSumDemo {
    public static void main(String[] args) {
        int size = 10_000_000;
        long[] array = new long[size];
        for (int i = 0; i < size; i++) {
            array[i] = i + 1;  // 1 to 10,000,000
        }

        // Sequential sum
        long start1 = System.currentTimeMillis();
        long seqSum = 0;
        for (long val : array) seqSum += val;
        long seqTime = System.currentTimeMillis() - start1;

        // Fork/Join parallel sum
        ForkJoinPool pool = new ForkJoinPool();
        long start2 = System.currentTimeMillis();
        long parSum = pool.invoke(new ParallelSum(array, 0, size));
        long parTime = System.currentTimeMillis() - start2;

        System.out.println("═══════════════════════════════════════════");
        System.out.println("        SEQUENTIAL vs FORK/JOIN");
        System.out.println("═══════════════════════════════════════════");
        System.out.println("Array size: " + size);
        System.out.println("───────────────────────────────────────────");
        System.out.printf("Sequential: sum=%d, time=%dms%n", seqSum, seqTime);
        System.out.printf("Fork/Join:  sum=%d, time=%dms%n", parSum, parTime);
        System.out.printf("Speedup:    %.1fx%n", (double) seqTime / parTime);
        System.out.println("═══════════════════════════════════════════");
    }
}
```

**Typical Output (4-core CPU):**
```
═══════════════════════════════════════════
        SEQUENTIAL vs FORK/JOIN
═══════════════════════════════════════════
Array size: 10000000
───────────────────────────────────────────
Sequential: sum=50000005000000, time=25ms
Fork/Join:  sum=50000005000000, time=8ms
Speedup:    3.1x
═══════════════════════════════════════════
```

### Example 2: RecursiveAction — Parallel Array Increment

```java
import java.util.concurrent.*;

class ParallelIncrement extends RecursiveAction {
    private static final int THRESHOLD = 50_000;
    private final int[] array;
    private final int start, end;

    ParallelIncrement(int[] array, int start, int end) {
        this.array = array;
        this.start = start;
        this.end = end;
    }

    @Override
    protected void compute() {
        if (end - start <= THRESHOLD) {
            // Base case: increment directly
            for (int i = start; i < end; i++) {
                array[i]++;
            }
        } else {
            int mid = start + (end - start) / 2;
            ParallelIncrement left = new ParallelIncrement(array, start, mid);
            ParallelIncrement right = new ParallelIncrement(array, mid, end);
            invokeAll(left, right);  // Fork both and wait for completion
        }
    }
}

public class RecursiveActionDemo {
    public static void main(String[] args) {
        int[] data = new int[1_000_000];
        // All zeros initially

        ForkJoinPool pool = new ForkJoinPool();
        pool.invoke(new ParallelIncrement(data, 0, data.length));

        // Verify: all elements should be 1
        boolean allOnes = true;
        for (int val : data) {
            if (val != 1) { allOnes = false; break; }
        }
        System.out.println("All elements incremented: " + allOnes);  // true
    }
}
```

---

## 2. ThreadLocal — Thread-Confined Variables

### The Concept

`ThreadLocal<T>` provides a **separate copy** of a variable for each thread. Each thread can read and write its own copy without affecting other threads.

```
┌──────────────────────────────────────────────────────────────────┐
│                       ThreadLocal                                  │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ThreadLocal<SimpleDateFormat> dateFormat = ...                  │
│                                                                   │
│  Thread-A: ┌───────────────────────────┐                         │
│            │ Own copy of dateFormat    │ ← Isolated!             │
│            └───────────────────────────┘                         │
│                                                                   │
│  Thread-B: ┌───────────────────────────┐                         │
│            │ Own copy of dateFormat    │ ← Different instance!   │
│            └───────────────────────────┘                         │
│                                                                   │
│  Thread-C: ┌───────────────────────────┐                         │
│            │ Own copy of dateFormat    │ ← Each gets their own!  │
│            └───────────────────────────┘                         │
│                                                                   │
│  No synchronization needed — each thread has its OWN copy!      │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### Why ThreadLocal?

`SimpleDateFormat` is **NOT thread-safe**. Without `ThreadLocal`, you'd need to either:
- Create a new instance every time (wasteful) 
- Synchronize access (slow)

`ThreadLocal` gives each thread its own instance — both fast AND safe!

### Example 1: Thread-Safe Date Formatting

```java
import java.text.SimpleDateFormat;
import java.util.Date;

public class ThreadLocalDemo {
    // Each thread gets its OWN SimpleDateFormat instance
    private static final ThreadLocal<SimpleDateFormat> dateFormat = 
        ThreadLocal.withInitial(() -> new SimpleDateFormat("dd-MMM-yyyy HH:mm:ss"));

    public static void main(String[] args) throws InterruptedException {
        Runnable task = () -> {
            String formatted = dateFormat.get().format(new Date());
            System.out.println(Thread.currentThread().getName() + 
                             ": " + formatted);
        };

        Thread[] threads = new Thread[5];
        for (int i = 0; i < 5; i++) {
            threads[i] = new Thread(task, "Thread-" + i);
            threads[i].start();
        }

        for (Thread t : threads) t.join();
    }
}
```

### Example 2: Request Context in Web Applications

```java
public class UserContext {
    private static final ThreadLocal<String> currentUser = new ThreadLocal<>();
    private static final ThreadLocal<String> requestId = new ThreadLocal<>();

    public static void setUser(String user) { currentUser.set(user); }
    public static String getUser() { return currentUser.get(); }

    public static void setRequestId(String id) { requestId.set(id); }
    public static String getRequestId() { return requestId.get(); }

    // ⚠️ CRITICAL: Always clean up in thread pools!
    public static void clear() {
        currentUser.remove();
        requestId.remove();
    }
}

// Usage in a web server:
class RequestHandler implements Runnable {
    private final String user;
    private final String reqId;

    RequestHandler(String user, String reqId) {
        this.user = user;
        this.reqId = reqId;
    }

    @Override
    public void run() {
        try {
            UserContext.setUser(user);
            UserContext.setRequestId(reqId);

            // Service layer can access context without passing parameters!
            processOrder();
            sendNotification();
        } finally {
            UserContext.clear();  // ⚠️ MUST clean up in thread pools!
        }
    }

    void processOrder() {
        System.out.println("[" + UserContext.getRequestId() + "] " + 
                         "Processing order for: " + UserContext.getUser());
    }

    void sendNotification() {
        System.out.println("[" + UserContext.getRequestId() + "] " + 
                         "Sending notification to: " + UserContext.getUser());
    }
}
```

### InheritableThreadLocal

When a **parent thread** creates a **child thread**, `ThreadLocal` values are **NOT** inherited. Use `InheritableThreadLocal` to pass values from parent to child:

```java
public class InheritableDemo {
    private static final InheritableThreadLocal<String> context = 
        new InheritableThreadLocal<>();

    public static void main(String[] args) throws InterruptedException {
        context.set("ParentValue");

        Thread child = new Thread(() -> {
            // Child thread gets parent's value automatically!
            System.out.println("Child sees: " + context.get());  // "ParentValue"

            context.set("ChildValue");
            System.out.println("Child changed to: " + context.get());  // "ChildValue"
        });

        child.start();
        child.join();

        // Parent is unaffected
        System.out.println("Parent still has: " + context.get());  // "ParentValue"
    }
}
```

### ThreadLocal Best Practices

| ✅ Do | ❌ Don't |
|-------|---------|
| Use `withInitial()` for lazy initialization | Don't forget to `remove()` in thread pools |
| Always call `remove()` in `finally` blocks | Don't store large objects in ThreadLocal |
| Use for per-thread caching (DateFormat, Random) | Don't use as a substitute for proper parameter passing |
| Use InheritableThreadLocal for parent-child data | Don't rely on InheritableThreadLocal with ExecutorService |

---

## 3. Virtual Threads — Java 21's Game Changer

### The Problem with Platform Threads

Traditional Java threads (now called **platform threads**) are expensive:

```
┌──────────────────────────────────────────────────────────────────┐
│               PLATFORM THREAD LIMITATIONS                         │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Each Platform Thread:                                            │
│  • Wraps an OS-level thread                                      │
│  • Costs ~1 MB of stack memory                                   │
│  • OS context switch is expensive                                │
│  • Typically limited to ~2,000-10,000 threads per app            │
│                                                                   │
│  For I/O-heavy apps (web servers, microservices):                │
│  • 10,000 concurrent requests = 10,000 threads                  │
│  • 10,000 × 1 MB = 10 GB of memory just for stacks! 💥          │
│  • Thread creation/destruction overhead                          │
│  • OS scheduler overwhelmed                                      │
│                                                                   │
│  That's why we use thread pools — but they limit concurrency!    │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### Virtual Threads: The Solution

Java 21 introduced **Virtual Threads** (Project Loom) — lightweight threads managed by the JVM, not the OS.

```
┌──────────────────────────────────────────────────────────────────┐
│            PLATFORM vs VIRTUAL THREADS                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  PLATFORM THREADS:                                                │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐            │
│  │ Java T1 │  │ Java T2 │  │ Java T3 │  │ Java T4 │            │
│  │  ~1 MB  │  │  ~1 MB  │  │  ~1 MB  │  │  ~1 MB  │            │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘            │
│       │            │            │            │                   │
│  ┌────┴────┐  ┌────┴────┐  ┌────┴────┐  ┌────┴────┐            │
│  │  OS T1  │  │  OS T2  │  │  OS T3  │  │  OS T4  │            │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘            │
│  1:1 mapping — EXPENSIVE! Limited to ~10K threads               │
│                                                                   │
│  ─────────────────────────────────────────────────────────       │
│                                                                   │
│  VIRTUAL THREADS:                                                 │
│  ┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐         │
│  │VT1││VT2││VT3││VT4││VT5││VT6││VT7││VT8││VT9││...│         │
│  └─┬─┘└─┬─┘└─┬─┘└─┬─┘└─┬─┘└─┬─┘└─┬─┘└─┬─┘└─┬─┘└───┘         │
│    └──┬──┘    └──┬──┘   └──┬──┘   └──┬──┘                       │
│  ┌────┴────┐┌────┴────┐┌────┴────┐┌────┴────┐                   │
│  │Carrier 1││Carrier 2││Carrier 3││Carrier 4│ (Platform threads) │
│  └─────────┘└─────────┘└─────────┘└─────────┘                   │
│  M:N mapping — Millions of VTs on few carriers! 🚀              │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### Comparison

| Aspect | Platform Thread | Virtual Thread |
|--------|----------------|----------------|
| **Managed by** | OS | JVM |
| **Stack size** | ~1 MB (fixed) | ~few KB (grows as needed) |
| **Creation cost** | Expensive | Cheap (like an object) |
| **Max count** | ~10,000 | **Millions** |
| **Blocking** | Wastes OS thread | JVM unmounts from carrier |
| **Best for** | CPU-heavy tasks | I/O-heavy tasks |
| **Thread pool needed?** | Yes (essential) | No (create per task) |

---

### Creating Virtual Threads

```java
// Method 1: Thread.ofVirtual()
Thread vt = Thread.ofVirtual()
    .name("my-virtual-thread")
    .start(() -> {
        System.out.println("Hello from virtual thread!");
        System.out.println("Is virtual? " + Thread.currentThread().isVirtual());
    });

// Method 2: Thread.startVirtualThread()
Thread vt2 = Thread.startVirtualThread(() -> {
    System.out.println("Another virtual thread!");
});

// Method 3: ExecutorService with virtual threads (recommended!)
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> System.out.println("Task 1"));
    executor.submit(() -> System.out.println("Task 2"));
    executor.submit(() -> System.out.println("Task 3"));
}
```

### Example 1: Handling 100,000 Concurrent Requests

```java
import java.util.concurrent.*;
import java.time.*;

public class VirtualThreadScaleDemo {
    public static void main(String[] args) throws InterruptedException {
        int taskCount = 100_000;

        System.out.println("═══════════════════════════════════════════");
        System.out.println("    LAUNCHING " + taskCount + " CONCURRENT TASKS");
        System.out.println("═══════════════════════════════════════════\n");

        // VIRTUAL THREADS — one per task!
        Instant start = Instant.now();
        
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            for (int i = 0; i < taskCount; i++) {
                executor.submit(() -> {
                    // Simulate I/O operation (API call, DB query, etc.)
                    try { Thread.sleep(Duration.ofSeconds(1)); } 
                    catch (InterruptedException e) { }
                });
            }
        }  // Auto-waits for all tasks to complete
        
        Duration duration = Duration.between(start, Instant.now());

        System.out.println("✅ All " + taskCount + " tasks completed!");
        System.out.println("⏱️  Total time: " + duration.toMillis() + "ms");
        System.out.println("   (~" + duration.toSeconds() + " seconds)");

        // With platform threads: Would need 100K threads = 100GB RAM!
        // With virtual threads: Uses ~4 carrier threads, a few MB total!
    }
}
```

**Output:**
```
═══════════════════════════════════════════
    LAUNCHING 100000 CONCURRENT TASKS
═══════════════════════════════════════════

✅ All 100000 tasks completed!
⏱️  Total time: 1850ms
   (~1 seconds)
```

100,000 tasks that each sleep for 1 second — completed in **~1.8 seconds** total! With platform threads, this would require 100 GB of RAM or take minutes with a limited thread pool.

### Example 2: Virtual Threads vs Platform Threads

```java
import java.util.concurrent.*;
import java.time.*;

public class PlatformVsVirtualDemo {
    static void runWithExecutor(String name, ExecutorService executor, int tasks) 
            throws InterruptedException {
        Instant start = Instant.now();
        
        CountDownLatch latch = new CountDownLatch(tasks);
        for (int i = 0; i < tasks; i++) {
            executor.submit(() -> {
                try {
                    Thread.sleep(100);  // Simulate I/O
                } catch (InterruptedException e) { }
                latch.countDown();
            });
        }
        latch.await();
        
        long ms = Duration.between(start, Instant.now()).toMillis();
        System.out.printf("  %-25s: %5d ms%n", name, ms);
    }

    public static void main(String[] args) throws InterruptedException {
        int tasks = 10_000;
        
        System.out.println("═══════════════════════════════════════════");
        System.out.println("  " + tasks + " tasks, each sleeping 100ms");
        System.out.println("═══════════════════════════════════════════");

        // Platform threads — limited pool
        try (var pool = Executors.newFixedThreadPool(200)) {
            runWithExecutor("Platform (200 threads)", pool, tasks);
        }

        // Virtual threads — one per task
        try (var pool = Executors.newVirtualThreadPerTaskExecutor()) {
            runWithExecutor("Virtual (per-task)", pool, tasks);
        }

        System.out.println("═══════════════════════════════════════════");
    }
}
```

**Typical Output:**
```
═══════════════════════════════════════════
  10000 tasks, each sleeping 100ms
═══════════════════════════════════════════
  Platform (200 threads)   :  5200 ms
  Virtual (per-task)       :   180 ms
═══════════════════════════════════════════
```

Virtual threads are **~29x faster** for I/O-bound workloads! 🚀

---

### How Virtual Threads Handle Blocking

When a virtual thread performs a **blocking operation** (sleep, I/O, lock), the JVM **unmounts** it from the carrier thread, allowing other virtual threads to use that carrier:

```
┌──────────────────────────────────────────────────────────────────┐
│           HOW VIRTUAL THREADS HANDLE BLOCKING                     │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  VT-1 is running on Carrier-1:                                   │
│  ┌─────────┐                                                     │
│  │  VT-1   │ ──── running  ──── │ Thread.sleep(1000) │           │
│  │(mounted)│                    │     BLOCKING!      │           │
│  └────┬────┘                    └────────┬───────────┘           │
│       │                                   │                       │
│  Carrier-1                          JVM UNMOUNTS VT-1!           │
│                                           │                       │
│                                   ┌───────┴───────┐              │
│                                   │ VT-1 parked   │              │
│                                   │ (in heap, ~KB)│              │
│                                   └───────────────┘              │
│                                                                   │
│  Carrier-1 is now FREE for another Virtual Thread!               │
│  ┌─────────┐                                                     │
│  │  VT-42  │ ──── mounted on Carrier-1, running!                │
│  │(mounted)│                                                     │
│  └─────────┘                                                     │
│                                                                   │
│  After 1 second, VT-1 is remounted on ANY available carrier.    │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

### When to Use Virtual Threads

| Scenario | Use Virtual Threads? |
|----------|---------------------|
| Web server handling many requests | ✅ **Yes** — I/O-bound, high concurrency |
| Microservice making API calls | ✅ **Yes** — mostly waiting for responses |
| Database-heavy application | ✅ **Yes** — waiting for queries |
| CPU-intensive computation | ❌ **No** — use platform threads/Fork-Join |
| Real-time, low-latency | ⚠️ **Careful** — unmounting adds slight overhead |
| Tasks using `synchronized` heavily | ⚠️ **Careful** — can pin virtual thread to carrier |

### Virtual Thread Pitfalls

```java
// ⚠️ PINNING: synchronized blocks PIN the virtual thread to carrier!
synchronized (lock) {      // ❌ Pins virtual thread
    blockingOperation();   // Carrier thread is blocked too!
}

// ✅ FIX: Use ReentrantLock instead
ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    blockingOperation();   // Virtual thread unmounts normally
} finally {
    lock.unlock();
}
```

| ✅ Do with Virtual Threads | ❌ Don't with Virtual Threads |
|---------------------------|-------------------------------|
| Use `ReentrantLock` instead of `synchronized` | Use `synchronized` for long-running blocks |
| Create one per task (they're cheap!) | Pool virtual threads (defeats the purpose) |
| Use for I/O-bound work | Use for CPU-bound work |
| Let them block naturally | Use complex reactive/async code |

---

## Summary

### Quick Decision Guide

```
What kind of parallelism do you need?

  Divide-and-conquer (recursive splitting)?
    → Fork/Join Framework (RecursiveTask / RecursiveAction)

  Per-thread isolated data (no sharing)?
    → ThreadLocal

  Handling many concurrent I/O operations?
    → Virtual Threads (Java 21+)

  CPU-intensive parallel streams?
    → Fork/Join Pool (used internally by parallel streams)
```

### Key Takeaways

1. ✅ **Fork/Join** splits problems recursively and uses work-stealing for maximum CPU utilization
2. ✅ **ThreadLocal** gives each thread its own isolated copy of a variable
3. ✅ **Virtual Threads** enable millions of concurrent tasks with minimal resources
4. ✅ Virtual threads excel at **I/O-bound** work; Fork/Join excels at **CPU-bound** work
5. ⚠️ Always `remove()` ThreadLocal values in thread pools to prevent memory leaks
6. ⚠️ Avoid `synchronized` with virtual threads — use `ReentrantLock` instead

---

### What's Next?

In the final article of this series, we'll cover:
- **Common Concurrency Patterns** — Singleton, Immutability, Thread Confinement
- **Concurrency Interview Problems** — Odd/Even, Producer-Consumer, Dining Philosophers
- **Best Practices & Anti-Patterns** cheat sheet

👉 [Continue to Concurrency Patterns & Best Practices](/posts/java-concurrency-patterns/)

---

*Happy Parallel Programming! 🚀🧵*
