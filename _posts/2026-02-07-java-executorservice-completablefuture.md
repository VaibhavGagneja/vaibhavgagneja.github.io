---
title: "Java Concurrency: ExecutorService, Callable, Future and CompletableFuture"
description: Master modern Java concurrency with Thread Pools, ExecutorService, Callable, Future, and CompletableFuture for async programming
author: Vaibhav Gagneja
date: 2026-02-07 19:00:00 +0530
categories: [Development, Java]
tags: [java, concurrency, executorservice, completablefuture, async, thread-pool]
toc: true
image:
  path: https://lh3.googleusercontent.com/rd-gg-dl/AOI_d__Zr-q_kG7bLyq1O8VA1SOIeexWRkpUFt3Jrx4UY5qf1v81gZUw9-jj_slXYFw6_0hDz3VxuWbY55UMzR5PYEnrCHUI-kmCh6XPgpj5EbAMHt8dQk29X3pUeAdDIRUhGb8g2UqR2AXcJpWjkz-20tx90c7Kc-ng0RaxGommAU2mIV3iDXcfbK-ULuKEnHOYfJqhl0lGt08PLKZcYKheE0RPoHxgbynEm3SuBma4QfBaXx87jpDclDJ--e05eb_Rpqr67Yat4g5rb5yZHBjPJISCwmcWMIbliKjK3qBtXz_w_2SKXkjeoJNiOljef9TzOmHUw_O-ii-lSKFxCzrIlwgj-Hs56Q5hlbUTtV_Jyfsp6m_0Sz91I5TToVMBFKaVkIvbqilcPb_JqQhHhKqZTA3BUOv6QwlL8hJ9MxWgCspv8nxGkRRvucnmxiI4Yyeejqf_mJz6TSu_WRA7CySV-3z-CBhjDiGKaPa4qJRTMFM97ihsv44i_Cup6BhKc_dZEBgrPyMCitPJGuFD242fmI2_c7ac5JIXcwt0WDz9j3iOadjq8_KFwA2GScycgA_V8Ot0IBwfhNAWO4FZ4v-4A5Vl33CxFzesOwpALh06GTVYcB-XoYORBEwEL9fQNk8YxmEW64Dr5vuBbYDRakCqcSCOCMHEDg6KpIaYo4ccOwcyHj_OfPyfPqaJDLO2jis9y3A8ZFQ_79dQC8noqWsMOJ7YRbQj3tfdhVLnM3c7xDOCD-m65p_dYbXWWQQzdATea-h2Cz8Ly3Da-zaU7DZpbefhQOTRfUaiWv1kuj5EC9TkyQ47CYbiiHQ1BkXpcjH4MfvP8n9153kvvc8BPW0BElAD440fZOEgRfw1U__82yjLm7KxktwPUx9Y6QIEMtFYxcHSLDmPSdZVufPJqwJovepyKSmRG1ec7EC_lAWceFmQE64eBnJzzWjLlg1aVf3J2PTxPzRHKz_2mKz-l9BhWfWmHdcNWQ2SfKpw0D4zdZvH7mLlmr9W3ulnXSzZ3VrcRBzGAF3qNTOfHiyVhRxI5mfwYeKLzw8cj0hEaPkzkhb-4lEnnQc948ZwF8lSu65Ukgc80gLp9MBLmQIj_qRnkNYa54LZotiqxUvcDNbihFTmxZjyIXf1ZJ20PgKVFMiu6PTJZFhg1MDUQXlCYwn1NGhpk60d_HM430iK1xk2z14V9cKb9n2WDDs=s1024-rj
---

In the previous articles, we covered threads, synchronization, and deadlock prevention. Now let's explore **modern Java concurrency** - how to manage threads efficiently and write clean asynchronous code.

Creating threads manually with `new Thread()` is like hiring a new employee for every small task - **expensive and inefficient**. Let's learn the better way!

---

## 1. Why Thread Pools?

### The Problem with Manual Thread Creation

```java
// ❌ BAD: Creating new thread for each task
for (int i = 0; i < 1000; i++) {
    new Thread(() -> processTask()).start();  // 1000 threads! 😱
}
```

**Problems:**
- Creating 1000 threads is **expensive** (memory, CPU overhead)
- OS has a **limit** on number of threads
- No **control** over how many threads run simultaneously
- Threads are **not reused** - created and destroyed repeatedly

### The Solution: Thread Pools

A **Thread Pool** is a collection of reusable worker threads that execute tasks.

```
┌─────────────────────────────────────────────────────────────────┐
│                        THREAD POOL                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│    Task Queue                    Worker Threads                  │
│    ┌────────┐                   ┌──────────────┐                │
│    │ Task 1 │ ────────────────► │  Thread 1    │ ──► Execute    │
│    │ Task 2 │                   │  (busy)      │                │
│    │ Task 3 │ ────────────────► │  Thread 2    │ ──► Execute    │
│    │ Task 4 │                   │  (busy)      │                │
│    │ Task 5 │ ──── waiting ──── │  Thread 3    │ ──► Execute    │
│    │ ...    │                   │  (busy)      │                │
│    └────────┘                   └──────────────┘                │
│                                                                  │
│    When Thread 1 finishes, it picks up the next waiting task!   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Benefits of Thread Pools

| Benefit | Description |
|---------|-------------|
| **Resource Management** | Limited, fixed number of threads |
| **Performance** | Threads are reused, no creation overhead |
| **Control** | Configure max threads, queue size, etc. |
| **Stability** | Prevents system overload |
| **Task Management** | Queue tasks when all threads are busy |

---

## 2. ExecutorService: The Thread Pool Manager

`ExecutorService` is an interface that represents an asynchronous execution service for managing thread pools.

### Creating Thread Pools

Java provides factory methods in the `Executors` class:

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

// Fixed Thread Pool - exact number of threads
ExecutorService fixedPool = Executors.newFixedThreadPool(5);

// Cached Thread Pool - creates threads as needed, reuses idle ones
ExecutorService cachedPool = Executors.newCachedThreadPool();

// Single Thread Executor - only one thread
ExecutorService singleThread = Executors.newSingleThreadExecutor();

// Scheduled Thread Pool - for delayed/periodic tasks
ScheduledExecutorService scheduledPool = Executors.newScheduledThreadPool(3);
```

### Types of Thread Pools

| Pool Type | Description | Best For |
|-----------|-------------|----------|
| `newFixedThreadPool(n)` | Exactly n threads | Known, consistent workload |
| `newCachedThreadPool()` | Creates threads as needed | Short-lived async tasks |
| `newSingleThreadExecutor()` | Single worker thread | Sequential task execution |
| `newScheduledThreadPool(n)` | For scheduled tasks | Delayed/periodic execution |

---

### Example: Fixed Thread Pool

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

class EmailTask implements Runnable {
    private String recipient;
    
    public EmailTask(String recipient) {
        this.recipient = recipient;
    }
    
    @Override
    public void run() {
        String threadName = Thread.currentThread().getName();
        System.out.println(threadName + ": Sending email to " + recipient);
        
        // Simulate email sending (2 seconds)
        try { Thread.sleep(2000); } catch (InterruptedException e) { }
        
        System.out.println(threadName + ": ✓ Email sent to " + recipient);
    }
}

public class ThreadPoolDemo {
    public static void main(String[] args) {
        // Create a pool with 3 worker threads
        ExecutorService executor = Executors.newFixedThreadPool(3);
        
        // Submit 6 email tasks
        String[] recipients = {"rahul@gmail.com", "amit@gmail.com", "priya@gmail.com",
                              "neha@gmail.com", "vikram@gmail.com", "anita@gmail.com"};
        
        System.out.println("Submitting 6 email tasks to pool of 3 threads...\n");
        
        for (String recipient : recipients) {
            executor.execute(new EmailTask(recipient));  // Submit task
        }
        
        // Shutdown the executor (no new tasks accepted)
        executor.shutdown();
        
        System.out.println("\nAll tasks submitted. Waiting for completion...");
    }
}
```

### Detailed Explanation

**1. Creating the Pool:**
```java
ExecutorService executor = Executors.newFixedThreadPool(3);
```
- Creates a pool with **exactly 3 threads**
- These 3 threads will process all tasks
- Threads are reused after completing each task

**2. Submitting Tasks:**
```java
executor.execute(new EmailTask(recipient));
```
- `execute()` accepts a `Runnable`
- Task is added to the queue
- A free thread picks it up and executes

**3. Execution Flow:**
```
┌──────────────────────────────────────────────────────────────────┐
│                    THREAD POOL EXECUTION                          │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Time    Thread-1           Thread-2           Thread-3           │
│  ────    ────────           ────────           ────────           │
│                                                                   │
│  T=0     Email: rahul       Email: amit        Email: priya       │
│  T=2     ✓ Done (rahul)     ✓ Done (amit)      ✓ Done (priya)    │
│          Picks: neha        Picks: vikram      Picks: anita       │
│  T=4     ✓ Done (neha)      ✓ Done (vikram)    ✓ Done (anita)    │
│                                                                   │
│  Total time: 4 seconds (instead of 12 with 1 thread!)            │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

**4. Shutting Down:**
```java
executor.shutdown();
```
- Stops accepting new tasks
- Allows currently running and queued tasks to complete
- For immediate shutdown, use `shutdownNow()`

### Output:
```
Submitting 6 email tasks to pool of 3 threads...

pool-1-thread-1: Sending email to rahul@gmail.com
pool-1-thread-2: Sending email to amit@gmail.com
pool-1-thread-3: Sending email to priya@gmail.com

All tasks submitted. Waiting for completion...

pool-1-thread-1: ✓ Email sent to rahul@gmail.com
pool-1-thread-1: Sending email to neha@gmail.com
pool-1-thread-2: ✓ Email sent to amit@gmail.com
pool-1-thread-2: Sending email to vikram@gmail.com
pool-1-thread-3: ✓ Email sent to priya@gmail.com
pool-1-thread-3: Sending email to anita@gmail.com
pool-1-thread-1: ✓ Email sent to neha@gmail.com
pool-1-thread-2: ✓ Email sent to vikram@gmail.com
pool-1-thread-3: ✓ Email sent to anita@gmail.com
```

**Notice:** Thread-1 processed rahul AND neha (thread reuse)! 🔄

---

## 3. Callable and Future: Getting Results from Threads

### The Problem with Runnable

```java
public interface Runnable {
    void run();  // Returns nothing! 😢
}
```

- `Runnable.run()` returns `void`
- No way to get a **result** from the task
- No way to throw **checked exceptions**

### The Solution: Callable<V>

```java
public interface Callable<V> {
    V call() throws Exception;  // Returns a value! 🎉
}
```

- `Callable.call()` returns a **result** of type `V`
- Can throw **checked exceptions**
- Used with `ExecutorService.submit()` which returns a `Future`

### What is Future<V>?

A `Future` represents the **result of an asynchronous computation** - it's a placeholder for a value that will be available later.

```
┌─────────────────────────────────────────────────────────────────┐
│                    FUTURE: A PROMISE OF VALUE                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Main Thread              Worker Thread                         │
│   ───────────              ─────────────                         │
│                                                                  │
│   submit(task) ──────────► Start execution                       │
│        │                         │                               │
│        ▼                         │                               │
│   Returns Future                 │ (working...)                  │
│   immediately!                   │                               │
│        │                         │                               │
│   (do other work)                │                               │
│        │                         │                               │
│   future.get() ◄────── result ──┘ (completed)                   │
│        │                                                         │
│   Got the result!                                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Important Future Methods

| Method | Description |
|--------|-------------|
| `get()` | Waits for result and returns it (blocks if not ready) |
| `get(timeout, unit)` | Waits for result with timeout |
| `isDone()` | Returns `true` if task is completed |
| `isCancelled()` | Returns `true` if task was cancelled |
| `cancel(boolean)` | Attempts to cancel the task |

---

### Example: Callable and Future

```java
import java.util.concurrent.*;

class PriceCalculator implements Callable<Double> {
    private String product;
    private int quantity;
    
    public PriceCalculator(String product, int quantity) {
        this.product = product;
        this.quantity = quantity;
    }
    
    @Override
    public Double call() throws Exception {
        String threadName = Thread.currentThread().getName();
        System.out.println(threadName + ": Calculating price for " + product + "...");
        
        // Simulate price calculation (database lookup, API call, etc.)
        Thread.sleep(2000);
        
        // Return the calculated price
        double pricePerUnit = Math.random() * 100 + 50;  // Rs. 50-150
        double totalPrice = pricePerUnit * quantity;
        
        System.out.println(threadName + ": ✓ " + product + " price calculated: Rs. " + 
                          String.format("%.2f", totalPrice));
        return totalPrice;
    }
}

public class CallableFutureDemo {
    public static void main(String[] args) throws Exception {
        ExecutorService executor = Executors.newFixedThreadPool(3);
        
        System.out.println("Submitting price calculation tasks...\n");
        
        // Submit Callable tasks - get Future objects back
        Future<Double> laptopPrice = executor.submit(new PriceCalculator("Laptop", 2));
        Future<Double> phonePrice = executor.submit(new PriceCalculator("Phone", 5));
        Future<Double> tabletPrice = executor.submit(new PriceCalculator("Tablet", 3));
        
        System.out.println("Tasks submitted! Main thread doing other work...\n");
        
        // Main thread can do other work while tasks execute
        System.out.println("Main: Preparing invoice header...");
        Thread.sleep(500);
        System.out.println("Main: Fetching customer details...\n");
        
        // Now get the results (blocks if not ready)
        System.out.println("Main: Fetching calculated prices...\n");
        
        Double laptop = laptopPrice.get();   // Waits if not done
        Double phone = phonePrice.get();
        Double tablet = tabletPrice.get();
        
        double grandTotal = laptop + phone + tablet;
        
        System.out.println("\n════════════════════════════════════════");
        System.out.println("                 INVOICE                 ");
        System.out.println("════════════════════════════════════════");
        System.out.printf("Laptop (x2):   Rs. %.2f%n", laptop);
        System.out.printf("Phone (x5):    Rs. %.2f%n", phone);
        System.out.printf("Tablet (x3):   Rs. %.2f%n", tablet);
        System.out.println("────────────────────────────────────────");
        System.out.printf("GRAND TOTAL:   Rs. %.2f%n", grandTotal);
        System.out.println("════════════════════════════════════════");
        
        executor.shutdown();
    }
}
```

### Explanation

**1. Implementing Callable:**
```java
class PriceCalculator implements Callable<Double> {
    @Override
    public Double call() throws Exception {
        // Calculate and RETURN the result
        return totalPrice;
    }
}
```
- Callable is parameterized with the return type (`Double`)
- The `call()` method returns a value and can throw exceptions

**2. Submitting Callable Tasks:**
```java
Future<Double> laptopPrice = executor.submit(new PriceCalculator("Laptop", 2));
```
- `submit()` returns a `Future<Double>` immediately
- The actual calculation happens in the background
- Main thread doesn't block here

**3. Getting Results:**
```java
Double laptop = laptopPrice.get();  // Blocks if not done yet
```
- `get()` waits for the task to complete and returns the result
- If task is already done, returns immediately
- If exception occurred in task, throws `ExecutionException`

### Output:
```
Submitting price calculation tasks...

pool-1-thread-1: Calculating price for Laptop...
pool-1-thread-2: Calculating price for Phone...
pool-1-thread-3: Calculating price for Tablet...
Tasks submitted! Main thread doing other work...

Main: Preparing invoice header...
Main: Fetching customer details...

Main: Fetching calculated prices...

pool-1-thread-1: ✓ Laptop price calculated: Rs. 234.56
pool-1-thread-2: ✓ Phone price calculated: Rs. 567.89
pool-1-thread-3: ✓ Tablet price calculated: Rs. 345.67

════════════════════════════════════════
                 INVOICE                 
════════════════════════════════════════
Laptop (x2):   Rs. 234.56
Phone (x5):    Rs. 567.89
Tablet (x3):   Rs. 345.67
────────────────────────────────────────
GRAND TOTAL:   Rs. 1148.12
════════════════════════════════════════
```

---

### Using get() with Timeout

```java
try {
    Double result = future.get(5, TimeUnit.SECONDS);  // Wait max 5 seconds
} catch (TimeoutException e) {
    System.out.println("Task took too long! Cancelling...");
    future.cancel(true);  // Cancel the task
}
```

---

## 4. CompletableFuture: Modern Async Programming

### The Problem with Future

While `Future` is useful, it has limitations:

| Limitation | Description |
|------------|-------------|
| **Blocking get()** | Must call `get()` to get result, which blocks |
| **No Chaining** | Cannot chain tasks together |
| **No Combining** | Cannot easily combine multiple futures |
| **No Exception Handling** | No elegant way to handle errors |

### The Solution: CompletableFuture

`CompletableFuture` (Java 8+) provides a **non-blocking, functional, chainable** way to handle async operations.

```java
// Fluent, readable, non-blocking code!
CompletableFuture
    .supplyAsync(() -> fetchUserFromDB(userId))
    .thenApply(user -> enrichUserData(user))
    .thenAccept(user -> sendWelcomeEmail(user))
    .exceptionally(ex -> handleError(ex));
```

---

### Creating CompletableFuture

**1. supplyAsync - With Return Value:**
```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    // This runs in ForkJoinPool.commonPool()
    return "Hello from async task!";
});
```

**2. runAsync - No Return Value:**
```java
CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
    System.out.println("Running async task...");
});
```

**3. With Custom Executor:**
```java
ExecutorService executor = Executors.newFixedThreadPool(5);
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    return "Using custom pool!";
}, executor);
```

---

### Chaining Operations

| Method | Input | Output | Use Case |
|--------|-------|--------|----------|
| `thenApply(Function)` | T | R | Transform result |
| `thenAccept(Consumer)` | T | void | Consume result |
| `thenRun(Runnable)` | - | void | Run action after completion |
| `thenCompose(Function)` | T | CompletableFuture<R> | Chain another async operation |

### Example: Chaining with thenApply

```java
import java.util.concurrent.CompletableFuture;

public class ChainDemo {
    public static void main(String[] args) throws Exception {
        System.out.println("Main thread: " + Thread.currentThread().getName());
        
        CompletableFuture<String> future = CompletableFuture
            // Step 1: Fetch user ID
            .supplyAsync(() -> {
                System.out.println("Step 1 [" + Thread.currentThread().getName() + 
                                  "]: Fetching user ID...");
                sleep(1000);
                return 12345;
            })
            // Step 2: Fetch user name using ID
            .thenApply(userId -> {
                System.out.println("Step 2 [" + Thread.currentThread().getName() + 
                                  "]: Fetching user name for ID: " + userId);
                sleep(1000);
                return "Rahul Sharma";
            })
            // Step 3: Format greeting
            .thenApply(userName -> {
                System.out.println("Step 3 [" + Thread.currentThread().getName() + 
                                  "]: Creating greeting for: " + userName);
                return "Welcome, " + userName + "!";
            });
        
        System.out.println("Main: Tasks chained, doing other work...\n");
        
        // Get final result
        String greeting = future.get();
        System.out.println("\nFinal Result: " + greeting);
    }
    
    static void sleep(long ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) { }
    }
}
```

### Execution Flow

```
┌──────────────────────────────────────────────────────────────────┐
│                    COMPLETABLEFUTURE CHAIN                        │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│   supplyAsync(() -> 12345)                                        │
│        │                                                          │
│        ▼ (returns Integer)                                        │
│   thenApply(userId -> "Rahul Sharma")                            │
│        │                                                          │
│        ▼ (returns String)                                         │
│   thenApply(userName -> "Welcome, " + userName + "!")            │
│        │                                                          │
│        ▼ (returns String)                                         │
│   "Welcome, Rahul Sharma!"                                        │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### Output:
```
Main thread: main
Main: Tasks chained, doing other work...

Step 1 [ForkJoinPool.commonPool-worker-1]: Fetching user ID...
Step 2 [ForkJoinPool.commonPool-worker-1]: Fetching user name for ID: 12345
Step 3 [ForkJoinPool.commonPool-worker-1]: Creating greeting for: Rahul Sharma

Final Result: Welcome, Rahul Sharma!
```

---

### Combining Multiple Futures

**1. thenCombine - Combine Two Futures:**
```java
CompletableFuture<Double> priceFuture = CompletableFuture.supplyAsync(() -> 100.0);
CompletableFuture<Double> discountFuture = CompletableFuture.supplyAsync(() -> 0.1);

CompletableFuture<Double> finalPrice = priceFuture.thenCombine(
    discountFuture,
    (price, discount) -> price - (price * discount)
);

System.out.println("Final Price: " + finalPrice.get());  // 90.0
```

**2. allOf - Wait for All:**
```java
CompletableFuture<Void> allDone = CompletableFuture.allOf(
    future1, future2, future3
);
allDone.get();  // Waits until ALL complete
```

**3. anyOf - Wait for First:**
```java
CompletableFuture<Object> firstDone = CompletableFuture.anyOf(
    future1, future2, future3
);
Object result = firstDone.get();  // First one to complete wins!
```

---

### Example: Combining Futures (E-commerce Checkout)

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.TimeUnit;

public class ECommerceCheckout {
    public static void main(String[] args) throws Exception {
        long startTime = System.currentTimeMillis();
        
        System.out.println("Starting checkout process...\n");
        
        // These three operations can run in PARALLEL
        CompletableFuture<Double> cartTotal = CompletableFuture.supplyAsync(() -> {
            System.out.println("[Cart] Calculating cart total...");
            sleep(2000);
            System.out.println("[Cart] ✓ Done");
            return 5000.0;
        });
        
        CompletableFuture<Double> discount = CompletableFuture.supplyAsync(() -> {
            System.out.println("[Discount] Fetching discount...");
            sleep(1500);
            System.out.println("[Discount] ✓ Done");
            return 500.0;
        });
        
        CompletableFuture<Double> shipping = CompletableFuture.supplyAsync(() -> {
            System.out.println("[Shipping] Calculating shipping...");
            sleep(1000);
            System.out.println("[Shipping] ✓ Done");
            return 99.0;
        });
        
        // Combine all three results
        CompletableFuture<Double> finalAmount = cartTotal
            .thenCombine(discount, (cart, disc) -> {
                System.out.println("\n[Combine] Cart - Discount = " + (cart - disc));
                return cart - disc;
            })
            .thenCombine(shipping, (subtotal, ship) -> {
                System.out.println("[Combine] Subtotal + Shipping = " + (subtotal + ship));
                return subtotal + ship;
            });
        
        // Get final result
        Double total = finalAmount.get();
        
        long endTime = System.currentTimeMillis();
        
        System.out.println("\n════════════════════════════════════════");
        System.out.println("          CHECKOUT SUMMARY               ");
        System.out.println("════════════════════════════════════════");
        System.out.printf("Cart Total:     Rs. %.2f%n", 5000.0);
        System.out.printf("Discount:       Rs. -%.2f%n", 500.0);
        System.out.printf("Shipping:       Rs. %.2f%n", 99.0);
        System.out.println("────────────────────────────────────────");
        System.out.printf("FINAL AMOUNT:   Rs. %.2f%n", total);
        System.out.println("════════════════════════════════════════");
        System.out.println("\nTotal time: " + (endTime - startTime) + " ms");
        System.out.println("(Sequential would take: 4500+ ms)");
    }
    
    static void sleep(long ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) { }
    }
}
```

### Output:
```
Starting checkout process...

[Cart] Calculating cart total...
[Discount] Fetching discount...
[Shipping] Calculating shipping...
[Shipping] ✓ Done
[Discount] ✓ Done
[Cart] ✓ Done

[Combine] Cart - Discount = 4500.0
[Combine] Subtotal + Shipping = 4599.0

════════════════════════════════════════
          CHECKOUT SUMMARY               
════════════════════════════════════════
Cart Total:     Rs. 5000.00
Discount:       Rs. -500.00
Shipping:       Rs. 99.00
────────────────────────────────────────
FINAL AMOUNT:   Rs. 4599.00
════════════════════════════════════════

Total time: 2015 ms
(Sequential would take: 4500+ ms)
```

**Notice:** All three operations ran in parallel! Total time ~2 seconds instead of 4.5 seconds! 🚀

---

### Exception Handling

**1. exceptionally - Handle Errors:**
```java
CompletableFuture<String> future = CompletableFuture
    .supplyAsync(() -> {
        if (true) throw new RuntimeException("Database error!");
        return "Success";
    })
    .exceptionally(ex -> {
        System.out.println("Error: " + ex.getMessage());
        return "Default Value";  // Fallback value
    });

System.out.println(future.get());  // "Default Value"
```

**2. handle - Handle Success OR Failure:**
```java
CompletableFuture<String> future = CompletableFuture
    .supplyAsync(() -> {
        // May succeed or fail
        return riskyOperation();
    })
    .handle((result, exception) -> {
        if (exception != null) {
            return "Error: " + exception.getMessage();
        }
        return "Success: " + result;
    });
```

---

### Complete Example: Order Processing System

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class OrderProcessingSystem {
    
    private static ExecutorService executor = Executors.newFixedThreadPool(4);
    
    public static void main(String[] args) throws Exception {
        String orderId = "ORD-2024-001";
        
        System.out.println("╔════════════════════════════════════════════╗");
        System.out.println("║      ORDER PROCESSING SYSTEM                ║");
        System.out.println("╚════════════════════════════════════════════╝\n");
        
        CompletableFuture<String> orderProcess = CompletableFuture
            // Step 1: Validate Order
            .supplyAsync(() -> validateOrder(orderId), executor)
            
            // Step 2: Reserve Inventory
            .thenApply(validOrder -> {
                System.out.println("📦 Reserving inventory for: " + validOrder);
                sleep(1000);
                System.out.println("   ✓ Inventory reserved!\n");
                return validOrder;
            })
            
            // Step 3: Process Payment (async - returns CompletableFuture)
            .thenCompose(order -> processPayment(order))
            
            // Step 4: Send Confirmation
            .thenApply(paymentStatus -> {
                System.out.println("📧 Sending confirmation email...");
                sleep(500);
                System.out.println("   ✓ Email sent!\n");
                return "Order " + orderId + " - " + paymentStatus + " - Confirmed!";
            })
            
            // Handle any errors
            .exceptionally(ex -> {
                System.out.println("❌ Error: " + ex.getMessage());
                return "Order " + orderId + " - FAILED";
            });
        
        // Wait for completion
        String result = orderProcess.get();
        System.out.println("═══════════════════════════════════════════");
        System.out.println("FINAL STATUS: " + result);
        System.out.println("═══════════════════════════════════════════");
        
        executor.shutdown();
    }
    
    static String validateOrder(String orderId) {
        System.out.println("🔍 Validating order: " + orderId);
        sleep(800);
        System.out.println("   ✓ Order validated!\n");
        return orderId;
    }
    
    static CompletableFuture<String> processPayment(String orderId) {
        return CompletableFuture.supplyAsync(() -> {
            System.out.println("💳 Processing payment for: " + orderId);
            sleep(1500);
            System.out.println("   ✓ Payment successful!\n");
            return "PAID";
        }, executor);
    }
    
    static void sleep(long ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) { }
    }
}
```

### Output:
```
╔════════════════════════════════════════════╗
║      ORDER PROCESSING SYSTEM                ║
╚════════════════════════════════════════════╝

🔍 Validating order: ORD-2024-001
   ✓ Order validated!

📦 Reserving inventory for: ORD-2024-001
   ✓ Inventory reserved!

💳 Processing payment for: ORD-2024-001
   ✓ Payment successful!

📧 Sending confirmation email...
   ✓ Email sent!

═══════════════════════════════════════════
FINAL STATUS: Order ORD-2024-001 - PAID - Confirmed!
═══════════════════════════════════════════
```

---

## 5. Quick Reference

### ExecutorService Methods

| Method | Description |
|--------|-------------|
| `execute(Runnable)` | Execute task, no return value |
| `submit(Callable)` | Submit task, returns Future |
| `shutdown()` | Stop accepting new tasks |
| `shutdownNow()` | Stop all tasks immediately |
| `awaitTermination(timeout, unit)` | Wait for all tasks to complete |

### Future Methods

| Method | Description |
|--------|-------------|
| `get()` | Block and get result |
| `get(timeout, unit)` | Get result with timeout |
| `isDone()` | Check if completed |
| `cancel(boolean)` | Cancel the task |

### CompletableFuture Methods

| Method | Description |
|--------|-------------|
| `supplyAsync(Supplier)` | Create with return value |
| `runAsync(Runnable)` | Create without return value |
| `thenApply(Function)` | Transform result |
| `thenAccept(Consumer)` | Consume result |
| `thenCompose(Function)` | Chain another CompletableFuture |
| `thenCombine(Future, BiFunction)` | Combine two futures |
| `allOf(CompletableFuture...)` | Wait for all |
| `anyOf(CompletableFuture...)` | Wait for first |
| `exceptionally(Function)` | Handle exceptions |
| `handle(BiFunction)` | Handle success or failure |

---

## Summary

### When to Use What?

| Scenario | Solution |
|----------|----------|
| Execute tasks, no result needed | `executor.execute(Runnable)` |
| Execute tasks, need result | `executor.submit(Callable)` + `Future` |
| Chain operations | `CompletableFuture.thenApply()` |
| Parallel independent operations | `CompletableFuture.allOf()` |
| Get first result | `CompletableFuture.anyOf()` |
| Complex async pipelines | `CompletableFuture` chains |

### Key Takeaways

1. ✅ **Thread Pools** reuse threads efficiently - don't create threads manually
2. ✅ **ExecutorService** manages thread pools and task execution
3. ✅ **Callable** returns values, unlike Runnable
4. ✅ **Future** is a placeholder for an async result
5. ✅ **CompletableFuture** enables non-blocking, chainable async code
6. ✅ Always **shutdown()** ExecutorService when done
7. ✅ Always **unlock in finally** block (from previous articles!)
8. ⚠️ **get()** blocks - prefer CompletableFuture for non-blocking code

---

### Best Practices

```
┌────────────────────────────────────────────────────────────────┐
│              MODERN CONCURRENCY BEST PRACTICES                  │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ DO:                                                         │
│  • Use thread pools instead of manual thread creation          │
│  • Use CompletableFuture for async operations                  │
│  • Chain operations with thenApply/thenCompose                 │
│  • Handle exceptions with exceptionally/handle                 │
│  • Use timeouts with get() to avoid indefinite waiting         │
│  • Shutdown executor services when done                        │
│                                                                 │
│  ❌ DON'T:                                                       │
│  • Don't create unlimited threads                              │
│  • Don't block on get() unnecessarily                          │
│  • Don't ignore exceptions from async tasks                    │
│  • Don't forget to shutdown executors (resource leak!)         │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

---

*Happy Async Coding! 🚀⚡*
