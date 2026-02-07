---
title: "Java Synchronization Part 1: Thread Safety, Synchronized Methods and Blocks"
description: Master Java synchronization fundamentals - understand race conditions, synchronized methods, synchronized blocks, and static synchronization with detailed examples
author: Vaibhav Gagneja
date: 2026-02-07 17:00:00 +0530
categories: [Development, Java]
tags: [java, synchronization, multithreading, concurrency, thread-safety]
toc: true
image:
  path: /assets/photos/synchros.png
---

In the previous article, we explored threads, their lifecycle, and various thread methods. Now it's time to tackle one of the most critical aspects of multithreading: **Synchronization**.

When multiple threads access shared resources simultaneously, things can go terribly wrong. Let's understand why and how to fix it.

---

## 1. The Problem: Why Do We Need Synchronization?

### Real-World Scenario: Movie Ticket Booking

Imagine a booking application like **BookMyShow** or **IRCTC**. Every user's booking request is handled in a separate thread.

**Scenario:**
- Total seats available: **10**
- **Rahul (Thread-1)** wants to book **6 seats**
- **Amit (Thread-2)** wants to book **5 seats**
- Both click "Book" at the **same time**

**What happens without synchronization?**

```
┌─────────────────────────────────────────────────────────────────┐
│                     WITHOUT SYNCHRONIZATION                      │
├──────────────────────────────┬──────────────────────────────────┤
│        Rahul (Thread-1)       │        Amit (Thread-2)           │
├──────────────────────────────┼──────────────────────────────────┤
│ Check: seats >= 6? YES (10)  │                                  │
│                              │ Check: seats >= 5? YES (10)      │
│ Book 6 seats ✓               │                                  │
│                              │ Book 5 seats ✓                   │
│ seats = 10 - 6 = 4           │                                  │
│                              │ seats = 4 - 5 = -1 ❌            │
└──────────────────────────────┴──────────────────────────────────┘
                    RESULT: 11 seats booked, but only 10 exist!
```

---

### Code Without Synchronization

```java
class BookMovieSeat {
    int total_seats = 10;

    void bookSeat(int seats, String name) {
        if (total_seats >= seats) {
            System.out.println(name + " booked " + seats + " seats successfully");
            total_seats = total_seats - seats;
            System.out.println("Total seats left: " + total_seats);
        } else {
            System.err.println(name + " sorry! Seats not available");
            System.err.println("Total seats left: " + total_seats);
        }
    }
}

class MyThread extends Thread {
    BookMovieSeat bts;
    int seats;

    public MyThread(BookMovieSeat bts, int seats) {
        this.bts = bts;
        this.seats = seats;
    }

    public void run() {
        bts.bookSeat(seats, Thread.currentThread().getName());
    }
}

public class MovieBookingApp {
    public static void main(String[] args) {
        BookMovieSeat bts = new BookMovieSeat();

        MyThread rahul = new MyThread(bts, 6);
        rahul.setName("Rahul");
        rahul.start();

        MyThread amit = new MyThread(bts, 5);
        amit.setName("Amit");
        amit.start();
    }
}
```

### Code Explanation

Let's break down each part:

**1. BookMovieSeat Class:**
```java
class BookMovieSeat {
    int total_seats = 10;  // Shared resource - this is the problem!
```
- The `total_seats` variable is **shared** between all threads
- When multiple threads modify it simultaneously, data corruption occurs

**2. bookSeat() Method - The Critical Section:**
```java
void bookSeat(int seats, String name) {
    if (total_seats >= seats) {           // Step 1: Check availability
        System.out.println(name + " booked " + seats + " seats");
        total_seats = total_seats - seats; // Step 2: Update seats
```
- **Problem:** Between Step 1 (checking) and Step 2 (updating), another thread can also check and see seats available
- This is called a **Race Condition** - threads "race" to access shared data

**3. MyThread Class:**
```java
class MyThread extends Thread {
    BookMovieSeat bts;  // Reference to shared object
    int seats;
    
    public void run() {
        bts.bookSeat(seats, Thread.currentThread().getName());
    }
}
```
- Both threads share the **same** `bts` object
- Each thread calls `bookSeat()` on the same object simultaneously

**4. Main Method:**
```java
BookMovieSeat bts = new BookMovieSeat();  // ONE shared object

MyThread rahul = new MyThread(bts, 6);    // Thread 1 with shared object
MyThread amit = new MyThread(bts, 5);     // Thread 2 with SAME object

rahul.start();  // Both start nearly simultaneously
amit.start();
```

### Output (Problematic):
```
Rahul booked 6 seats successfully
Total seats left: 4
Amit booked 5 seats successfully
Total seats left: -1    ← NEGATIVE SEATS! 🚨
```

---

### Problems Identified

| Problem | Description | Impact |
|---------|-------------|--------|
| **Data Inconsistency** | Multiple threads modify `total_seats` simultaneously | Wrong seat count (-1 seats) |
| **Race Condition** | Both threads check availability at the same time | Both get "seats available" |
| **Overbooking** | Check and update operations are not atomic | More tickets sold than available |
| **Unpredictable Output** | Results vary on every execution | Cannot trust the application |

---

## 2. What is Synchronization?

**Synchronization** is a mechanism that allows **only one thread at a time** to access a shared resource (critical section).

### Analogy: Bathroom with a Lock 🚪

Think of synchronization like a **bathroom with a lock**:
- Only **one person** can use it at a time
- Others must **wait outside** until the bathroom is free
- When done, the person **unlocks** the door
- The next person **enters and locks** the door

```
┌─────────────────────────────────────────────────────────────────┐
│                    SYNCHRONIZATION ANALOGY                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│     👤 Person 1                         👤 Person 2              │
│         │                                    │                   │
│         ▼                                    │                   │
│    ┌─────────┐                               │                   │
│    │ 🔒 LOCK │ ◄── Acquires lock             │                   │
│    │         │                               │                   │
│    │  🚿     │ ◄── Using bathroom            ▼                   │
│    │         │                         ┌─────────┐               │
│    │         │                         │ WAITING │               │
│    │ 🔓 UNLOCK│ ◄── Releases lock      │   ...   │               │
│    └─────────┘                         └────┬────┘               │
│         │                                   │                    │
│         │◄──────── Door opens ──────────────┘                    │
│         │                                   │                    │
│         │                              ┌────▼────┐               │
│         │                              │ 🔒 LOCK │               │
│         │                              │  🚿     │               │
│         │                              │ 🔓 UNLOCK│               │
│         │                              └─────────┘               │
└─────────────────────────────────────────────────────────────────┘
```

---

### Advantages of Synchronization

| Advantage | Description | Example |
|-----------|-------------|---------|
| **Thread Safety** | Prevents threads from interfering | No double booking |
| **Data Consistency** | Ensures correct results | Seat count never goes negative |
| **Race Condition Prevention** | Only one thread in critical section | Check-and-update is atomic |
| **Controlled Access** | Manages critical resources | Database connections |
| **Reliability** | Same behavior every time | Predictable output |

---

## 3. Types of Synchronization

```
                    SYNCHRONIZATION
                          │
          ┌───────────────┴───────────────┐
          │                               │
    PROCESS SYNC                    THREAD SYNC
    (OS Level)                      (Java Level)
                                          │
                          ┌───────────────┴───────────────┐
                          │                               │
                   MUTUAL EXCLUSION              COOPERATION
                      (Mutex)                      (ITC)
                          │                               │
              ┌───────────┼───────────┐           wait()
              │           │           │           notify()
         synchronized  synchronized  ReentrantLock  notifyAll()
           Method       Block
```

### Mutual Exclusion (Mutex)
- Allows **only one thread at a time** to access the critical section
- Implemented using: `synchronized` keyword or `ReentrantLock`

### Cooperation (Inter-Thread Communication)
- Allows threads to **communicate and coordinate**
- Covered in **Part 2** of this series

---

## 4. How Synchronization Works Internally

Every object in Java has an **intrinsic lock** (also called a **monitor**).

### Lock Acquisition Flow

```
┌──────────────────────────────────────────────────────────────┐
│                    OBJECT LOCK MECHANISM                      │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  Step 1: Thread-1 calls synchronized method                  │
│          Thread-1 ──► ACQUIRES LOCK ✓                         │
│                        │                                      │
│                        ▼                                      │
│  Step 2: Thread-1 enters critical section                    │
│                 [Executing bookSeat()]                        │
│                        │                                      │
│  Step 3: Thread-2 tries to call synchronized method          │
│          Thread-2 ──► BLOCKED (waiting for lock)             │
│                                                               │
│  Step 4: Thread-1 exits synchronized method                  │
│          Thread-1 ──► RELEASES LOCK                           │
│                        │                                      │
│                        ▼                                      │
│  Step 5: Thread-2 gets the lock                              │
│          Thread-2 ──► ACQUIRES LOCK ✓                         │
│                 [Now executing bookSeat()]                    │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### Key Points:

| Point | Description |
|-------|-------------|
| **Object Lock** | Every Java object has exactly one lock |
| **Lock Acquisition** | Thread must acquire lock before entering synchronized code |
| **Blocking** | Other threads wait if lock is not available |
| **Lock Release** | Happens automatically when method/block exits |
| **Reentrant** | Same thread can acquire the lock multiple times |

---

## 5. Synchronized Method

A **synchronized method** ensures only one thread can execute it at a time for a particular object.

### Syntax

```java
// Just add the 'synchronized' keyword before return type
synchronized returnType methodName(parameters) {
    // This entire method body is now protected
    // Only one thread at a time can execute this
}
```

### Solution: Synchronized Booking Method

```java
class BookMovieSeat {
    int total_seats = 10;

    // ✅ Adding synchronized keyword - THIS IS THE FIX!
    synchronized void bookSeat(int seats, String name) {
        if (total_seats >= seats) {
            System.out.println(name + " booked " + seats + " seats successfully");
            total_seats = total_seats - seats;
            System.out.println("Total seats left: " + total_seats);
        } else {
            System.err.println(name + " sorry! Seats not available");
            System.err.println("Total seats left: " + total_seats);
        }
    }
}

class MyThread extends Thread {
    BookMovieSeat bts;
    int seats;

    public MyThread(BookMovieSeat bts, int seats) {
        this.bts = bts;
        this.seats = seats;
    }

    public void run() {
        bts.bookSeat(seats, Thread.currentThread().getName());
    }
}

public class MovieBookingApp {
    public static void main(String[] args) {
        BookMovieSeat bts = new BookMovieSeat();

        MyThread rahul = new MyThread(bts, 6);
        rahul.setName("Rahul");
        rahul.start();

        MyThread amit = new MyThread(bts, 5);
        amit.setName("Amit");
        amit.start();
    }
}
```

### Detailed Explanation

**1. The synchronized Keyword:**
```java
synchronized void bookSeat(int seats, String name) {
```
- The `synchronized` keyword tells JVM: **"Only one thread at a time can execute this method on this object"**
- Before entering, thread must acquire the lock of the `bts` object
- If another thread holds the lock, this thread **waits**

**2. What Happens Step-by-Step:**

```
Timeline:
─────────────────────────────────────────────────────────────────

T=0ms   Rahul.start() called
T=1ms   Amit.start() called
T=2ms   Rahul calls bts.bookSeat(6, "Rahul")
        └──► Rahul tries to acquire lock on 'bts' object
        └──► Lock is FREE → Rahul ACQUIRES the lock ✓
        └──► Rahul enters bookSeat() method

T=3ms   Amit calls bts.bookSeat(5, "Amit")
        └──► Amit tries to acquire lock on 'bts' object
        └──► Lock is HELD by Rahul → Amit is BLOCKED ⏳
        └──► Amit waits in queue...

T=4ms   Rahul: if(10 >= 6) → TRUE
T=5ms   Rahul: Prints "Rahul booked 6 seats successfully"
T=6ms   Rahul: total_seats = 10 - 6 = 4
T=7ms   Rahul: Prints "Total seats left: 4"
T=8ms   Rahul exits bookSeat() method
        └──► Rahul RELEASES the lock 🔓
        └──► Amit is UNBLOCKED!

T=9ms   Amit ACQUIRES the lock ✓
        └──► Amit enters bookSeat() method
T=10ms  Amit: if(4 >= 5) → FALSE (seats updated!)
T=11ms  Amit: Prints "Amit sorry! Seats not available"
T=12ms  Amit: Prints "Total seats left: 4"
T=13ms  Amit exits bookSeat() method
        └──► Amit RELEASES the lock 🔓
```

**3. Why This Works:**
- The `synchronized` keyword makes the **check-and-update** operation **atomic**
- Amit can only check availability **after** Rahul has finished updating
- Total seats is **4** when Amit checks, so his booking correctly fails

### Output (Correct):
```
Rahul booked 6 seats successfully
Total seats left: 4
Amit sorry! Seats not available
Total seats left: 4
```

---

### Execution Flow Comparison

| Without Synchronization | With Synchronization |
|------------------------|---------------------|
| Both threads check simultaneously | Only one thread checks at a time |
| Both see 10 seats available | Second thread sees updated count |
| Both bookings succeed | Only valid booking succeeds |
| `total_seats = -1` ❌ | `total_seats = 4` ✅ |

---

## 6. Synchronized Block

A **synchronized block** locks only a **specific portion of code** instead of the entire method.

### Why Use Synchronized Blocks?

Sometimes your method has **100 lines of code**, but only **5 lines** actually access shared data. Making the entire method synchronized would:
- Slow down performance unnecessarily
- Block other threads for longer than needed

**Solution:** Synchronize only the critical section!

### Comparison

| Aspect | Synchronized Method | Synchronized Block |
|--------|--------------------|--------------------|
| **Scope** | Entire method locked | Only specific lines locked |
| **Performance** | Slower (unnecessary blocking) | Faster (minimal blocking) |
| **Flexibility** | Lock is always `this` | Can lock any object |
| **Use Case** | All code is critical | Only some code is critical |

### Syntax

```java
// Syntax 1: Lock on current object (this)
synchronized (this) {
    // Critical section - only one thread at a time
}

// Syntax 2: Lock on specific object
synchronized (someObject) {
    // Critical section - locked on someObject
}
```

### Example: Synchronized Block

```java
class BookMovieSeat {
    int total_seats = 10;

    void bookSeat(int seats, String name) {
        // ═══════════════════════════════════════════════════════
        // NON-CRITICAL SECTION - Can run concurrently
        // ═══════════════════════════════════════════════════════
        System.out.println(name + " is checking availability...");
        System.out.println(name + " is validating payment method...");
        
        // Simulate some non-critical processing
        try { Thread.sleep(100); } catch (InterruptedException e) { }

        // ═══════════════════════════════════════════════════════
        // CRITICAL SECTION - Must be synchronized
        // ═══════════════════════════════════════════════════════
        synchronized (this) {
            if (total_seats >= seats) {
                System.out.println(name + " booked " + seats + " seats successfully");
                total_seats = total_seats - seats;
                System.out.println("Total seats left: " + total_seats);
            } else {
                System.err.println(name + " sorry! Seats not available");
                System.err.println("Total seats left: " + total_seats);
            }
        }

        // ═══════════════════════════════════════════════════════
        // NON-CRITICAL SECTION - Can run concurrently
        // ═══════════════════════════════════════════════════════
        System.out.println(name + " booking process completed.");
        System.out.println(name + " sending confirmation email...");
    }
}

class MyThread extends Thread {
    BookMovieSeat bts;
    int seats;

    public MyThread(BookMovieSeat bts, int seats) {
        this.bts = bts;
        this.seats = seats;
    }

    public void run() {
        bts.bookSeat(seats, Thread.currentThread().getName());
    }
}

public class MovieBookingApp {
    public static void main(String[] args) {
        BookMovieSeat bts = new BookMovieSeat();

        MyThread rahul = new MyThread(bts, 6);
        rahul.setName("Rahul");
        rahul.start();

        MyThread amit = new MyThread(bts, 5);
        amit.setName("Amit");
        amit.start();
    }
}
```

### Detailed Explanation

**1. Non-Critical Section (Before synchronized block):**
```java
System.out.println(name + " is checking availability...");
System.out.println(name + " is validating payment method...");
```
- These print statements don't access `total_seats`
- Both threads can execute these **simultaneously**
- No data corruption risk here

**2. The Synchronized Block:**
```java
synchronized (this) {
    // Only ONE thread can be inside this block at a time
    if (total_seats >= seats) {
        total_seats = total_seats - seats;  // Modifying shared data
    }
}
```
- `synchronized (this)` means: **"Lock the current object before entering"**
- Even though `bookSeat()` is not synchronized, this specific block is protected
- The `this` refers to the `bts` object

**3. Non-Critical Section (After synchronized block):**
```java
System.out.println(name + " booking process completed.");
System.out.println(name + " sending confirmation email...");
```
- After exiting the synchronized block, the lock is released
- Both threads can again run concurrently
- Confirmation emails can be sent in parallel!

### Output:
```
Rahul is checking availability...
Amit is checking availability...
Rahul is validating payment method...
Amit is validating payment method...
Rahul booked 6 seats successfully
Total seats left: 4
Rahul booking process completed.
Amit sorry! Seats not available
Total seats left: 4
Rahul sending confirmation email...
Amit booking process completed.
Amit sending confirmation email...
```

### Performance Benefit

```
┌──────────────────────────────────────────────────────────────────┐
│              SYNCHRONIZED METHOD vs SYNCHRONIZED BLOCK            │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  SYNCHRONIZED METHOD:                                             │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ 🔒 LOCKED ─────────────────────────────────────────────────│ │
│  │ Line 1: Check availability          (blocks other threads) │ │
│  │ Line 2: Validate payment             (blocks other threads) │ │
│  │ Line 3: Process booking              (blocks other threads) │ │
│  │ Line 4: Update seats                 (blocks other threads) │ │
│  │ Line 5: Send confirmation            (blocks other threads) │ │
│  │ 🔓 UNLOCKED ───────────────────────────────────────────────│ │
│  └─────────────────────────────────────────────────────────────┘ │
│  Total blocked time: 100% of method                              │
│                                                                   │
│  SYNCHRONIZED BLOCK:                                              │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ Line 1: Check availability           (runs concurrently) ✓  │ │
│  │ Line 2: Validate payment             (runs concurrently) ✓  │ │
│  │ 🔒 LOCKED ─────────────────────────────────────────────────│ │
│  │ Line 3: Process booking              (blocks other threads) │ │
│  │ Line 4: Update seats                 (blocks other threads) │ │
│  │ 🔓 UNLOCKED ───────────────────────────────────────────────│ │
│  │ Line 5: Send confirmation            (runs concurrently) ✓  │ │
│  └─────────────────────────────────────────────────────────────┘ │
│  Total blocked time: Only 40% of method!                         │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## 7. Static Synchronization

**Static synchronization** locks at the **class level** instead of the instance level.

### The Problem: Multiple Objects

What if we have **two different** `BookMovieSeat` objects?

```java
BookMovieSeat bms1 = new BookMovieSeat();  // Object 1
BookMovieSeat bms2 = new BookMovieSeat();  // Object 2

// Thread 1 uses bms1
// Thread 2 uses bms2
```

With regular `synchronized`, each object has its **own lock**:
- Thread 1 locks `bms1` → enters synchronized method
- Thread 2 locks `bms2` → enters synchronized method **simultaneously!**

If they share **static data**, we still have a race condition!

### Solution: Static Synchronized Method

```java
class BookMovieSeat {
    // STATIC variable - shared across ALL instances
    static int total_seats = 10;

    // STATIC SYNCHRONIZED method - locks the CLASS, not the instance
    static synchronized void bookSeat(int seats, String name) {
        if (total_seats >= seats) {
            System.out.println(name + " booked " + seats + " seats successfully");
            total_seats = total_seats - seats;
            System.out.println("Total seats left: " + total_seats);
        } else {
            System.err.println(name + " sorry! Seats not available");
            System.err.println("Total seats left: " + total_seats);
        }
    }
}

class MyThread extends Thread {
    int seats;

    public MyThread(int seats) {
        this.seats = seats;
    }

    public void run() {
        // Calling static method - doesn't matter which object
        BookMovieSeat.bookSeat(seats, Thread.currentThread().getName());
    }
}

public class MovieBookingApp {
    public static void main(String[] args) {
        // Two DIFFERENT objects
        BookMovieSeat bms1 = new BookMovieSeat();
        BookMovieSeat bms2 = new BookMovieSeat();

        MyThread rahul = new MyThread(6);
        rahul.setName("Rahul");
        rahul.start();

        MyThread amit = new MyThread(5);
        amit.setName("Amit");
        amit.start();
    }
}
```

### Explanation

**1. Static Variable:**
```java
static int total_seats = 10;  // Belongs to CLASS, not instance
```
- All objects share the **same** `total_seats`
- Exists at class level, not object level

**2. Static Synchronized Method:**
```java
static synchronized void bookSeat(...)
```
- Locks the **Class object** (`BookMovieSeat.class`)
- All threads, regardless of which instance they use, compete for the **same lock**

**3. Lock Visualization:**

```
┌──────────────────────────────────────────────────────────────┐
│              INSTANCE vs STATIC SYNCHRONIZATION               │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  INSTANCE SYNCHRONIZED (separate locks):                      │
│                                                               │
│    bms1 Object          bms2 Object                          │
│    ┌──────────┐         ┌──────────┐                         │
│    │ Lock: 🔒 │         │ Lock: 🔒 │                         │
│    │ Thread-1 │         │ Thread-2 │                         │
│    │ (inside) │         │ (inside) │  ← Both run at once!    │
│    └──────────┘         └──────────┘                         │
│                                                               │
│  STATIC SYNCHRONIZED (one lock for all):                      │
│                                                               │
│    BookMovieSeat.class                                        │
│    ┌─────────────────────────────────┐                        │
│    │         Lock: 🔒                │                        │
│    │                                 │                        │
│    │   Thread-1 (inside)             │                        │
│    │   Thread-2 (WAITING...)         │  ← Only one at a time │
│    │                                 │                        │
│    └─────────────────────────────────┘                        │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### Output:
```
Rahul booked 6 seats successfully
Total seats left: 4
Amit sorry! Seats not available
Total seats left: 4
```

---

### Comparison: Instance vs Static Synchronization

| Aspect | Instance Synchronized | Static Synchronized |
|--------|----------------------|---------------------|
| **Lock Type** | Object lock (`this`) | Class lock (`ClassName.class`) |
| **Scope** | Per object | Per class (all objects) |
| **Use Case** | Instance variables | Static variables |
| **Multiple Objects** | Each object has own lock | All objects share one lock |
| **Syntax** | `synchronized void method()` | `static synchronized void method()` |

---

## 8. Disadvantages of Synchronization

While powerful, synchronization has drawbacks:

| Disadvantage | Description | Example |
|--------------|-------------|---------|
| **Performance Overhead** | Acquiring/releasing locks takes time | Method runs 10% slower |
| **Reduced Concurrency** | Only one thread runs critical section | Other threads wait idle |
| **Deadlock Risk** | Threads can wait for each other forever | Thread A waits for B, B waits for A |
| **Complex Debugging** | Timing issues make bugs hard to find | Bug appears only under load |
| **Starvation** | Some threads may wait indefinitely | Low priority thread never gets lock |

### When to Use Synchronization

| Scenario | Use Synchronization? |
|----------|---------------------|
| Multiple threads access shared mutable data | ✅ Yes |
| Read-only data | ❌ No |
| Each thread has its own data | ❌ No |
| Single-threaded application | ❌ No |

---

## Summary

### Quick Reference

| Synchronization Type | Lock On | Use When |
|---------------------|---------|----------|
| `synchronized method` | Current object (`this`) | All method code is critical |
| `synchronized (this) {}` | Current object | Only some code is critical |
| `synchronized (obj) {}` | Specific object | Need to lock a different object |
| `static synchronized` | Class object | Protecting static variables |

### Key Takeaways

1. ✅ **Race conditions** occur when multiple threads access shared data simultaneously
2. ✅ **synchronized** keyword ensures only one thread enters critical section
3. ✅ **synchronized blocks** are more performant than synchronized methods
4. ✅ **Static synchronization** is needed for static shared variables
5. ⚠️ **Don't over-synchronize** - it hurts performance

---

### What's Next?

In **Part 2**, we'll cover:
- **Inter-Thread Communication** (wait/notify/notifyAll)
- **Deadlocks** and how to prevent them
- **ReentrantLock** and modern Java concurrency
- **java.util.concurrent** package overview

---

*Happy Thread-Safe Coding! 🔒🧵*
