---
title: "Java Synchronization Part 2: Inter-Thread Communication"
description: Master wait(), notify() and notifyAll() for thread coordination with practical examples and deep-dive internals
author: Vaibhav Gagneja
date: 2026-02-07 18:00:00 +0530
categories: [Development, Java]
tags: [java, synchronization, deadlock, reentrantlock, concurrency, wait-notify]
toc: true
image:
  path: /assets/photos/synchros.png
---

In **Part 1**, we covered the fundamentals of synchronization: synchronized methods, synchronized blocks, and static synchronization. Now let's explore the **advanced topics**: inter-thread communication, deadlocks, and modern locking mechanisms.

---

## 1. Inter-Thread Communication (ITC)

### What is Inter-Thread Communication?

**Inter-Thread Communication** is a mechanism that allows threads to **cooperate** with each other instead of running independently. It's about one thread saying *"I'm done, you can proceed"* and another thread saying *"I'll wait until you tell me."*

### The Core Concept: Understanding wait() and notify()

Before diving into code, let's understand **what these methods actually do**:

```
┌─────────────────────────────────────────────────────────────────────┐
│                THE wait() AND notify() MENTAL MODEL                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   Think of it like a WAITING ROOM system:                           │
│                                                                      │
│   ┌─────────────────┐         ┌─────────────────┐                   │
│   │  SYNCHRONIZED   │         │   WAITING ROOM  │                   │
│   │  BLOCK (Office) │         │   (Sleep Area)  │                   │
│   │  🔒 Only 1 at   │         │   💤💤💤        │                   │
│   │    a time       │         │   Threads sleep │                   │
│   └────────┬────────┘         └────────┬────────┘                   │
│            │                           │                             │
│   wait() = │ "I'll step out to the    │                             │
│            │  waiting room, someone   │                             │
│            │  else can use the office"│                             │
│            └───────────────────────────►                             │
│                                                                      │
│            ◄───────────────────────────┐                             │
│  notify() = "Hey! One person from the │ Thread wakes up,            │
│              waiting room, come back! │ WAITS for lock,             │
│              (but I'm still in office │ then re-enters              │
│               until I leave)"         │                             │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### What EXACTLY Happens?

| When you call... | What happens IMMEDIATELY | What happens NEXT |
|------------------|-------------------------|-------------------|
| `wait()` | 1. Thread **releases the lock** <br> 2. Thread goes to **sleep** (waiting state) | Thread stays asleep until `notify()` is called |
| `notify()` | 1. One sleeping thread is **marked for wakeup** <br> 2. Notifying thread **keeps the lock** | Woken thread waits to **reacquire lock** before continuing |
| `notifyAll()` | All sleeping threads are **marked for wakeup** | All woken threads compete to reacquire the lock |

> **🔑 Key Insight:** `notify()` doesn't immediately give the lock to the waiting thread! It just says *"wake up and get ready."* The waiting thread still has to wait until the notifying thread **exits the synchronized block**.
>
> **Note:** These methods belong to the `Object` class (not `Thread`) and must be called from within a synchronized context.

---

### Real-World Analogy: Restaurant Kitchen 🍳

```
┌──────────────────────────────────────────────────────────────────┐
│                    RESTAURANT ANALOGY                             │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  CHEF (Producer Thread)                WAITER (Consumer Thread)  │
│  ═══════════════════════               ══════════════════════    │
│                                                                   │
│  1. Chef is cooking...                 Waiter: "I'll wait() -    │
│     (holds the kitchen)                I'll sit in waiting room  │
│                                        until food is ready"      │
│          │                                     │                 │
│          │                                     ▼                 │
│          │                               ┌──────────┐            │
│          │                               │ WAITING  │            │
│          │                               │  ROOM 💤 │            │
│          │                               └──────────┘            │
│          ▼                                                       │
│  2. "Order ready!"                                               │
│     Chef calls notify()                                          │
│     (still in kitchen)                                           │
│          │                                     │                 │
│          │    ──── "Hey, wake up!" ──────────►│                 │
│          │                                     │                 │
│          ▼                                     ▼                 │
│  3. Chef exits kitchen                 Waiter wakes up but       │
│     (releases lock)                    can't enter YET...        │
│          │                                     │                 │
│          │    ──── Lock available! ──────────►│                 │
│          │                                     │                 │
│          ▼                                     ▼                 │
│  4. Chef done                          Waiter enters kitchen,    │
│                                        picks up food, serves!    │
│                                                                   │
│  WITHOUT wait/notify: Waiter keeps asking "Ready? Ready? Ready?" │
│  🔴 This wastes CPU cycles (called "busy waiting" or "polling")  │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

### Example: Producer-Consumer Pattern

```java
class MovieEarning extends Thread {
    int total_earning = 0;
    int ticket_price = 250;
    int tickets_sold = 10;
    boolean isCalculated = false;

    @Override
    public void run() {
        // Simulating some time-consuming calculation
        try {
            System.out.println("MovieEarning thread: Calculating earnings...");
            Thread.sleep(2000);  // Simulate 2 seconds of calculation
        } catch (InterruptedException e) { }

        synchronized (this) {
            // Calculate earnings
            total_earning = ticket_price * tickets_sold;
            isCalculated = true;
            
            System.out.println("MovieEarning thread: Calculation done! Notifying...");
            
            // Notify main thread that calculation is complete
            this.notify();
        }
    }
}

public class MainApp {
    public static void main(String[] args) throws InterruptedException {
        MovieEarning me = new MovieEarning();
        me.start();

        synchronized (me) {
            System.out.println("Main thread: Waiting for earnings calculation...");
            
            // Wait until MovieEarning thread notifies
            while (!me.isCalculated) {
                me.wait();
            }
            
            System.out.println("Main thread: Received notification!");
            System.out.println("Total movie earnings: Rs. " + me.total_earning);
        }
    }
}
```

### Step-by-Step Breakdown

Let's trace through exactly what happens:

**Step 1: Main thread starts MovieEarning thread**
```java
MovieEarning me = new MovieEarning();
me.start();  // MovieEarning thread created and running
```

**Step 2: Main thread enters synchronized block and calls wait()**
```java
synchronized (me) {           // Main thread ACQUIRES lock on 'me'
    while (!me.isCalculated) {
        me.wait();            // Main thread RELEASES lock and goes to SLEEP
    }
```
- Main thread **had** the lock
- After `wait()`, main thread **no longer has** the lock
- Main thread is now sleeping in the "waiting room"

**Step 3: MovieEarning thread can now acquire the lock**
```java
synchronized (this) {         // MovieEarning ACQUIRES lock (main released it!)
    total_earning = ticket_price * tickets_sold;
    isCalculated = true;
    this.notify();            // Wakes up main thread (but doesn't give lock yet)
}                             // NOW lock is released when block ends
```
- `notify()` just signals main thread: *"Wake up and get ready!"*
- Main thread is awake but **waiting for the lock**
- Only when MovieEarning **exits** the synchronized block, the lock is released

**Step 4: Main thread reacquires lock and continues**
```java
        // Main thread wakes up here, reacquires lock
        me.wait();  // Execution resumes AFTER this line
    }
    // Now isCalculated is true, loop exits
    System.out.println("Total movie earnings: Rs. " + me.total_earning);
}
```

### Visual Execution Flow

```
┌──────────────────────────────────────────────────────────────────┐
│                    DETAILED EXECUTION TIMELINE                    │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  TIME   MAIN THREAD                MOVIEEARNING THREAD           │
│  ════   ═══════════                ══════════════════            │
│                                                                   │
│  T=0    me.start() ─────────────────► Thread created             │
│                                       │                          │
│  T=1    synchronized(me)              │ Running Thread.sleep()   │
│         ACQUIRES lock ✓               │ (outside sync block)     │
│                                       │                          │
│  T=2    Checks: isCalculated?         │                          │
│         FALSE → enters while loop     │                          │
│                                       │                          │
│  T=3    me.wait() ─────┐              │                          │
│         • RELEASES lock│              │                          │
│         • Goes to SLEEP│              │                          │
│                        │              │                          │
│  T=4    💤 SLEEPING    │              │ synchronized(this)       │
│         (no lock)      │              │ Tries to acquire lock... │
│                        │              │ ✓ ACQUIRES lock!         │
│                        │              │ (Main released it!)      │
│                        │              │                          │
│  T=5    💤 SLEEPING    │              │ Calculates earnings      │
│                        │              │ isCalculated = true      │
│                        │              │                          │
│  T=6    💤 SLEEPING    │              │ this.notify()            │
│         ◄──────────────│──────────────│ "Hey main, wake up!"     │
│         WOKEN UP! ⏰   │              │ (still holds lock)       │
│         But no lock... │              │                          │
│         WAITING...     │              │                          │
│                        │              │                          │
│  T=7    Still waiting  │              │ } ← Exits sync block     │
│         for lock...    │              │ RELEASES lock ✓          │
│                        │              │                          │
│  T=8    ACQUIRES lock ✓│              │ Thread ends              │
│         Continues after│              │                          │
│         wait() line    │              │                          │
│                        │              │                          │
│  T=9    Checks: isCalculated?                                    │
│         TRUE → exits while loop                                  │
│         Prints: "Rs. 2500"                                       │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### Output:
```
Main thread: Waiting for earnings calculation...
MovieEarning thread: Calculating earnings...
MovieEarning thread: Calculation done! Notifying...
Main thread: Received notification!
Total movie earnings: Rs. 2500
```

---

### The wait() Method Variants

The `wait()` method has three overloaded versions:

#### 1. wait() - Wait Indefinitely
```java
public final void wait() throws InterruptedException
```
- Causes the thread to wait **indefinitely** until `notify()` or `notifyAll()` is called
- Use when you don't know how long the wait will be

#### 2. wait(long timeout) - Wait with Timeout
```java
public final void wait(long timeout) throws InterruptedException
```
- Thread waits for the specified **milliseconds** OR until notified
- If timeout expires, thread wakes up automatically
- `wait(0)` is equivalent to `wait()` - waits indefinitely

```java
synchronized (lock) {
    while (!condition) {
        lock.wait(5000);  // Wait max 5 seconds
        // Check if we woke up due to timeout or notification
        if (!condition) {
            System.out.println("Timed out, but condition still false!");
        }
    }
}
```

#### 3. wait(long timeout, int nanos) - High Precision Wait
```java
public final void wait(long timeout, int nanos) throws InterruptedException
```
- Provides **nanosecond precision** for timeout
- Total timeout = `(timeout * 1_000_000) + nanos` nanoseconds

---

### How to Own an Object's Monitor?

To call `wait()`, `notify()`, or `notifyAll()`, the thread **must own the object's monitor**. There are three ways to acquire it:

| Method | Example |
|--------|---------|
| **Synchronized instance method** | `public synchronized void method() { this.wait(); }` |
| **Synchronized block on object** | `synchronized (obj) { obj.wait(); }` |
| **Synchronized static method** | `public static synchronized void method() { ClassName.class.wait(); }` |

> **Note:** Only ONE thread can own an object's monitor at a time!

---

### Practical Example: Sender-Receiver Synchronization

Let's implement a more robust example - a **data packet transmission** between a Sender and Receiver:

**Problem Statement:**
- Sender sends data packets one at a time
- Receiver cannot process until Sender finishes sending
- Sender cannot send next packet until Receiver has processed the previous one

```java
public class Data {
    private String packet;
    
    // true = receiver should wait (sender's turn)
    // false = sender should wait (receiver's turn)
    private boolean transfer = true;

    // Receiver calls this to receive data
    public synchronized String receive() {
        // Wait while sender is preparing data
        while (transfer) {
            try {
                wait();  // Release lock and wait for sender
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                System.err.println("Thread Interrupted");
            }
        }
        
        // Data received, now it's sender's turn
        transfer = true;
        
        String returnPacket = packet;
        notifyAll();  // Wake up sender
        return returnPacket;
    }

    // Sender calls this to send data
    public synchronized void send(String packet) {
        // Wait while receiver is processing previous data
        while (!transfer) {
            try {
                wait();  // Release lock and wait for receiver
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                System.err.println("Thread Interrupted");
            }
        }
        
        // Receiver finished, now send new data
        transfer = false;
        
        this.packet = packet;
        notifyAll();  // Wake up receiver
    }
}
```

**How the Synchronization Works:**

```
┌──────────────────────────────────────────────────────────────────┐
│                 SENDER-RECEIVER SYNCHRONIZATION                   │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  INITIAL STATE: transfer = true (sender's turn)                  │
│                                                                   │
│  ┌─────────────────┐                   ┌─────────────────┐       │
│  │     SENDER      │                   │    RECEIVER     │       │
│  └────────┬────────┘                   └────────┬────────┘       │
│           │                                     │                │
│  Step 1:  │ send("Packet 1")                    │ receive()      │
│           │ transfer=true? YES                  │ transfer=true? │
│           │ → Don't wait! ✅                    │ YES → wait() 💤│
│           │                                     │                │
│  Step 2:  │ Set packet = "Packet 1"             │ 💤 SLEEPING    │
│           │ transfer = false                    │                │
│           │ notifyAll() → wakes receiver        │                │
│           │                                     │ ◄── WOKEN UP!  │
│           │                                     │                │
│  Step 3:  │ ← Returns from send()               │ Reacquires lock│
│           │                                     │ transfer=false?│
│           │                                     │ YES → Proceed! │
│           │                                     │                │
│  Step 4:  │ send("Packet 2")                    │ Read packet    │
│           │ transfer=false? NO                  │ transfer = true│
│           │ → Must wait() 💤                    │ notifyAll()    │
│           │                                     │                │
│  Step 5:  │ ◄── WOKEN UP!                       │ Returns packet │
│           │ transfer=true? YES                  │                │
│           │ → Continue sending!                 │                │
│           │                                     │                │
│  ... and the cycle continues ...                                 │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

**Sender Implementation:**
```java
public class Sender implements Runnable {
    private Data data;

    public Sender(Data data) {
        this.data = data;
    }

    public void run() {
        String[] packets = {
            "First packet",
            "Second packet",
            "Third packet",
            "Fourth packet",
            "End"  // Special termination signal
        };

        for (String packet : packets) {
            data.send(packet);
            System.out.println("Sent: " + packet);

            // Simulate processing time
            try {
                Thread.sleep(ThreadLocalRandom.current().nextInt(1000, 3000));
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }
}
```

**Receiver Implementation:**
```java
public class Receiver implements Runnable {
    private Data data;

    public Receiver(Data data) {
        this.data = data;
    }

    public void run() {
        // Keep receiving until we get "End" signal
        for (String receivedMessage = data.receive();
             !"End".equals(receivedMessage);
             receivedMessage = data.receive()) {

            System.out.println("Received: " + receivedMessage);

            // Simulate processing time
            try {
                Thread.sleep(ThreadLocalRandom.current().nextInt(1000, 3000));
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
        System.out.println("Transmission complete!");
    }
}
```

**Main Application:**
```java
public class NetworkSimulation {
    public static void main(String[] args) {
        Data data = new Data();
        
        Thread sender = new Thread(new Sender(data));
        Thread receiver = new Thread(new Receiver(data));

        sender.start();
        receiver.start();
    }
}
```

**Output:**
```
Sent: First packet
Received: First packet
Sent: Second packet
Received: Second packet
Sent: Third packet
Received: Third packet
Sent: Fourth packet
Received: Fourth packet
Sent: End
Transmission complete!
```

### Why This Example Works

| Principle | Implementation |
|-----------|----------------|
| **Bidirectional synchronization** | `transfer` flag alternates between sender and receiver |
| **No busy waiting** | Both threads use `wait()` instead of polling |
| **Proper lock handling** | Methods are `synchronized`, ensuring mutual exclusion |
| **notifyAll() used** | Wakes up the other thread reliably |
| **while loop for wait()** | Protects against spurious wakeups |

> **💡 Pro Tip:** This is the classic **Producer-Consumer pattern** with a single-item buffer. For production systems, consider using `BlockingQueue` from `java.util.concurrent` which handles all this complexity for you!

### A Note on Modern Alternatives

While `wait()`, `notify()`, and `notifyAll()` are fundamental to understanding Java concurrency, they are **low-level APIs**. Modern Java provides higher-level mechanisms that are often simpler and less error-prone:

| Modern Alternative | Use Case |
|-------------------|----------|
| `BlockingQueue` | Producer-consumer patterns (handles wait/notify internally) |
| `Lock` + `Condition` | More flexible than synchronized + wait/notify |
| `Semaphore` | Controlling access to limited resources |
| `CountDownLatch` | Waiting for N operations to complete |
| `CompletableFuture` | Async programming with chaining |

> **Good Practice:** Learn `wait()` and `notify()` to understand how synchronization works internally, but prefer `java.util.concurrent` utilities for production code!

---

## 🔍 Deep Dive: The Internals of wait() and notify()

### 1. The Paradox of wait()

**Question:** If `wait()` pauses the thread, how does it release the lock?

**Answer:** `wait()` is not a simple "sleep" command. It is a complex, **atomic sequence** of three steps performed by the JVM:

1. **Release Lock:** The thread strictly gives up its ownership of the monitor (lock).
   - *Why?* If it kept the lock while sleeping, no other thread (like the Producer) could enter the synchronized block to do work or call `notify()`. This would cause an instant **Deadlock**.

2. **Enter Wait Set:** The thread is moved to a special internal queue called the "Wait Set."

3. **Sleep:** The thread pauses execution and stops competing for CPU time.

> **Note:** This is why `wait()` must be called inside a `synchronized` block. You cannot release a lock you do not own. If you try, Java throws `IllegalMonitorStateException`.

---

### 2. The Lifecycle of a Notification

**Question:** Does `notify()` immediately start the waiting thread?

**Answer:** No. "Waking up" is not the same as "Running."

1. **The Signal:** Thread A calls `notify()`.
2. **The Move:** The JVM picks a thread (Thread B) from the **Wait Set** and moves it to the **Entry Set** (the "Hallway").
3. **The Block:** Thread B is now awake but **BLOCKED**. It cannot run yet because Thread A **still holds the lock**.
4. **The Handover:**
   - Thread A exits the `synchronized` block (releases lock).
   - Thread B competes with other threads to **re-acquire** the lock.
5. **The Resume:** Once Thread B gets the lock, it resumes execution *exactly* at the line following `wait()`.

**Visual State Transition:**

```
[RUNNING] -> calls wait() -> [WAITING] (In Wait Set)
                                  ↓
                             calls notify()
                                  ↓
                             [BLOCKED] (In Entry Set, waiting for lock)
                                  ↓
                             Acquires Lock
                                  ↓
                             [RUNNING] (Resumes code)
```

---

### 3. Critical Edge Cases

#### A. The "Lost Notification" Problem

If a Producer calls `notify()` *before* the Consumer calls `wait()`, the signal is lost forever. The Consumer will wait indefinitely because it missed the "wake up" call.

**Fix:** Always use a boolean flag (e.g., `isDataReady`) to track the state. The Consumer should check this flag *before* waiting.

#### B. Spurious Wakeups (Why we use `while`)

Rarely, the OS might wake up a waiting thread *without* a notification.

- **The Danger:** If you use `if`, the thread wakes up, assumes data is ready, and crashes when it finds an empty buffer.
- **The Fix:** Use `while`. This forces the thread to **re-check the condition** after waking up. If it was a fake wakeup, the condition is still false, and it goes back to sleep.

```java
// ✅ THE GOLDEN PATTERN
synchronized (lock) {
    while (!condition) {  // Loop handles Spurious Wakeups
        lock.wait();      // Releases lock here, Re-acquires on wakeup
    }
    // Process data
}
```

---
## Summary

In this part, we covered everything about **Inter-Thread Communication** (ITC):

### What We Learned

| Topic | Key Takeaway |
|-------|--------------|
| **wait() and notify()** | Mechanism for threads to cooperate instead of running independently |
| **wait() releases lock** | Thread gives up the lock and enters the Wait Set |
| **notify() doesn't release** | Only signals the waiting thread; lock released when exiting synchronized block |
| **while loop, not if** | Protects against spurious wakeups |
| **wait() variants** | `wait()`, `wait(timeout)`, `wait(timeout, nanos)` |
| **Sender-Receiver pattern** | Practical bidirectional synchronization example |
| **Modern alternatives** | `BlockingQueue`, `Lock+Condition`, `Semaphore`, etc. |

### The Golden Pattern

```java
synchronized (lock) {
    while (!condition) {  // Loop handles Spurious Wakeups
        lock.wait();      // Releases lock here, Re-acquires on wakeup
    }
    // Process data
}
```

---

### What's Next?

In **Part 3**, we'll explore:
- **Deadlocks** - What they are and how to prevent them
- **ReentrantLock** - Modern synchronization with more control
- **java.util.concurrent** - High-level concurrency utilities

👉 [Continue to Part 3: Deadlocks, ReentrantLock and Modern Concurrency](/posts/java-synchronization-part3/)

---

*Happy Thread-Safe Coding! 🔒🧵*

