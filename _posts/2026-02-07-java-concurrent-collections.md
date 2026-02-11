---
title: "Java Concurrency: Concurrent Collections — Thread-Safe Data Structures"
description: Deep dive into ConcurrentHashMap, BlockingQueue, CopyOnWriteArrayList and other thread-safe collections in Java
author: Vaibhav Gagneja
date: 2026-02-07 12:00:00 +0530
categories: [Development, Java]
tags: [java, concurrency, concurrent-collections, blocking-queue, concurrenthashmap]
toc: true
image:
  path: /assets/photos/synchros.png
---

In the previous article, we learned about `volatile` and atomic variables for individual values. But what about **collections**? When multiple threads read and write to a `HashMap` or `ArrayList`, things can break in spectacular ways. Let's explore Java's purpose-built **concurrent collections**.

---

## 1. Why Regular Collections Fail in Multithreaded Code

### The HashMap Disaster

```java
import java.util.*;

public class UnsafeHashMapDemo {
    public static void main(String[] args) throws InterruptedException {
        Map<String, Integer> map = new HashMap<>();

        Thread writer1 = new Thread(() -> {
            for (int i = 0; i < 10000; i++) {
                map.put("key" + i, i);  // ❌ Not thread-safe!
            }
        });

        Thread writer2 = new Thread(() -> {
            for (int i = 10000; i < 20000; i++) {
                map.put("key" + i, i);  // ❌ Not thread-safe!
            }
        });

        writer1.start();
        writer2.start();
        writer1.join();
        writer2.join();

        System.out.println("Expected size: 20000");
        System.out.println("Actual size: " + map.size());  // Could be anything!
    }
}
```

**What can go wrong:**
- Data loss (entries silently disappear)
- `ConcurrentModificationException`
- **Infinite loop** (yes, really — corrupted internal linked list in Java 7)
- `NullPointerException` from corrupted internal state

### The Legacy "Fix" (Don't Use This!)

```java
// ❌ This works but is VERY slow — locks the ENTIRE map for every operation
Map<String, Integer> syncMap = Collections.synchronizedMap(new HashMap<>());
```

```
┌──────────────────────────────────────────────────────────────────┐
│           WHY synchronizedMap IS SLOW                              │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  synchronizedMap:                                                 │
│  ┌────────────────────────────────────────────────┐              │
│  │           ONE GIANT LOCK 🔒                     │              │
│  │                                                 │              │
│  │  Thread 1: put("A", 1)  ← LOCKED               │              │
│  │  Thread 2: get("B")     ← WAITING...           │              │
│  │  Thread 3: put("C", 3)  ← WAITING...           │              │
│  │  Thread 4: get("D")     ← WAITING...           │              │
│  │                                                 │              │
│  │  Even READS must wait! 🐢                       │              │
│  └────────────────────────────────────────────────┘              │
│                                                                   │
│  ConcurrentHashMap:                                               │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐                            │
│  │Seg 0 │ │Seg 1 │ │Seg 2 │ │Seg 3 │  Multiple segments!       │
│  │🔒    │ │🔒    │ │🔒    │ │🔒    │                            │
│  │T1:put│ │T2:get│ │T3:put│ │T4:get│  All working in parallel! │
│  └──────┘ └──────┘ └──────┘ └──────┘                            │
│                                                                   │
│  Reads are LOCK-FREE! Multiple writes in parallel! 🚀            │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## 2. ConcurrentHashMap — The Star of the Show

`ConcurrentHashMap` is the most widely used concurrent collection. It provides **thread-safe** operations without locking the entire map.

### Basic Usage

```java
import java.util.concurrent.ConcurrentHashMap;

public class ConcurrentHashMapDemo {
    public static void main(String[] args) throws InterruptedException {
        ConcurrentHashMap<String, Integer> scores = new ConcurrentHashMap<>();

        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 10000; i++) {
                scores.put("player" + i, i * 10);
            }
        });

        Thread t2 = new Thread(() -> {
            for (int i = 10000; i < 20000; i++) {
                scores.put("player" + i, i * 10);
            }
        });

        t1.start();
        t2.start();
        t1.join();
        t2.join();

        System.out.println("Size: " + scores.size());  // Always 20000! ✅
    }
}
```

### Atomic Operations on ConcurrentHashMap

The real power lies in its **atomic compound operations**:

```java
ConcurrentHashMap<String, Integer> wordCount = new ConcurrentHashMap<>();

// ❌ NOT atomic — race condition between get and put!
Integer count = wordCount.get("hello");
wordCount.put("hello", (count == null ? 0 : count) + 1);

// ✅ ATOMIC — single operation, no race condition
wordCount.merge("hello", 1, Integer::sum);
```

### Important Atomic Methods

| Method | Description | Example |
|--------|-------------|---------|
| `putIfAbsent(key, value)` | Put only if key doesn't exist | `map.putIfAbsent("A", 1);` |
| `remove(key, value)` | Remove only if key maps to given value | `map.remove("A", 1);` |
| `replace(key, old, new)` | Replace only if key maps to old value | `map.replace("A", 1, 2);` |
| `compute(key, func)` | Compute new value atomically | `map.compute("A", (k, v) -> v + 1);` |
| `computeIfAbsent(key, func)` | Compute value only if absent | `map.computeIfAbsent("A", k -> 1);` |
| `computeIfPresent(key, func)` | Compute new value only if present | `map.computeIfPresent("A", (k, v) -> v + 1);` |
| `merge(key, value, func)` | Merge value if key exists, put if not | `map.merge("A", 1, Integer::sum);` |

### Practical Example: Real-Time Word Counter

```java
import java.util.concurrent.*;

public class WordCounterDemo {
    public static void main(String[] args) throws InterruptedException {
        ConcurrentHashMap<String, Integer> wordCount = new ConcurrentHashMap<>();

        String[] documents = {
            "java is great java is powerful",
            "python is great python is easy",
            "java and python are both great"
        };

        Thread[] threads = new Thread[documents.length];
        for (int i = 0; i < documents.length; i++) {
            final String doc = documents[i];
            threads[i] = new Thread(() -> {
                for (String word : doc.split(" ")) {
                    // ✅ Atomic increment — no race conditions!
                    wordCount.merge(word, 1, Integer::sum);
                }
            }, "Parser-" + i);
            threads[i].start();
        }

        for (Thread t : threads) t.join();

        System.out.println("═══════════════════════════════════════");
        System.out.println("         WORD FREQUENCIES");
        System.out.println("═══════════════════════════════════════");
        wordCount.entrySet().stream()
            .sorted((a, b) -> b.getValue() - a.getValue())
            .forEach(e -> System.out.printf("  %-10s → %d%n", 
                                            e.getKey(), e.getValue()));
    }
}
```

**Output:**
```
═══════════════════════════════════════
         WORD FREQUENCIES
═══════════════════════════════════════
  is         → 4
  great      → 3
  java       → 3
  python     → 3
  are        → 1
  both       → 1
  and        → 1
  powerful   → 1
  easy       → 1
```

---

## 3. CopyOnWriteArrayList — For Read-Heavy Workloads

`CopyOnWriteArrayList` creates a **complete copy of the underlying array** on every write operation. This makes reads extremely fast (no locking needed) but writes are expensive.

### How It Works

```
┌──────────────────────────────────────────────────────────────────┐
│               CopyOnWriteArrayList MECHANISM                      │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Original:  [A, B, C, D]                                         │
│                                                                   │
│  add("E"):                                                        │
│    1. Create NEW array: [A, B, C, D, E]  ← COPY + ADD            │
│    2. Switch reference to new array                              │
│    3. Old array is still being read by other threads (safe!)     │
│                                                                   │
│  Reader threads:     Writer thread:                               │
│  ┌────────────┐     ┌───────────────────────┐                    │
│  │ No lock!   │     │ Creates new copy      │                    │
│  │ Read [A,B, │     │ Modifies copy         │                    │
│  │  C,D] fast │     │ Swaps reference       │                    │
│  └────────────┘     └───────────────────────┘                    │
│                                                                   │
│  ✅ Perfect when: Reads >> Writes (e.g., listener lists)         │
│  ❌ Bad when: Frequent writes (copies are expensive!)            │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### Example: Event Listener Registry

```java
import java.util.concurrent.CopyOnWriteArrayList;

interface EventListener {
    void onEvent(String event);
}

class EventBus {
    // CopyOnWriteArrayList is perfect here:
    //   - Listeners are added/removed rarely
    //   - Events are fired frequently (many reads)
    private CopyOnWriteArrayList<EventListener> listeners = 
        new CopyOnWriteArrayList<>();

    public void addListener(EventListener listener) {
        listeners.add(listener);  // Write (rare) — creates copy
        System.out.println("Listener added. Total: " + listeners.size());
    }

    public void removeListener(EventListener listener) {
        listeners.remove(listener);  // Write (rare)
    }

    public void fireEvent(String event) {
        // Read (frequent) — NO LOCK needed! 
        // Safe even if another thread adds/removes listeners concurrently
        for (EventListener listener : listeners) {
            listener.onEvent(event);
        }
    }
}

public class CopyOnWriteDemo {
    public static void main(String[] args) throws InterruptedException {
        EventBus bus = new EventBus();

        // Add some listeners
        bus.addListener(e -> System.out.println("  Logger:  " + e));
        bus.addListener(e -> System.out.println("  Metrics: " + e));
        bus.addListener(e -> System.out.println("  Alert:   " + e));

        // Fire events from multiple threads
        Thread t1 = new Thread(() -> {
            for (int i = 1; i <= 3; i++) {
                bus.fireEvent("User login #" + i);
            }
        });

        Thread t2 = new Thread(() -> {
            for (int i = 1; i <= 3; i++) {
                bus.fireEvent("Order placed #" + i);
            }
        });

        t1.start();
        t2.start();
        t1.join();
        t2.join();
    }
}
```

---

## 4. BlockingQueue — The Producer-Consumer Powerhouse

`BlockingQueue` is a thread-safe queue that **blocks** when you try to:
- **Take** from an empty queue (waits until element is available)
- **Put** into a full queue (waits until space is available)

### How It Works

```
┌──────────────────────────────────────────────────────────────────┐
│                    BlockingQueue FLOW                              │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  PRODUCER                                        CONSUMER         │
│  ┌──────────┐    ┌──────────────────────┐    ┌──────────┐        │
│  │ Thread 1 │──► │                      │───►│ Thread 3 │        │
│  └──────────┘    │   BlockingQueue      │    └──────────┘        │
│  ┌──────────┐    │   [A] [B] [C] [D]    │    ┌──────────┐        │
│  │ Thread 2 │──► │                      │───►│ Thread 4 │        │
│  └──────────┘    └──────────────────────┘    └──────────┘        │
│                                                                   │
│  put() → Blocks if queue is FULL  (waits for space)              │
│  take() → Blocks if queue is EMPTY (waits for element)           │
│                                                                   │
│  No explicit synchronization needed! Queue handles everything!   │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### BlockingQueue Methods

| Operation | Throws Exception | Blocks | Times Out | Returns Special |
|-----------|-----------------|--------|-----------|-----------------|
| **Insert** | `add(e)` | `put(e)` | `offer(e, time, unit)` | `offer(e)` |
| **Remove** | `remove()` | `take()` | `poll(time, unit)` | `poll()` |
| **Examine** | `element()` | — | — | `peek()` |

### BlockingQueue Implementations

| Implementation | Bounded? | Description |
|---------------|----------|-------------|
| `ArrayBlockingQueue` | ✅ Yes | Fixed-size, backed by array |
| `LinkedBlockingQueue` | Optional | Optionally bounded, backed by linked list |
| `PriorityBlockingQueue` | ❌ No | Elements sorted by priority |
| `SynchronousQueue` | ❌ (size 0) | Direct handoff — no buffering |
| `DelayQueue` | ❌ No | Elements available only after delay |

### Example: Food Order System (Producer-Consumer)

```java
import java.util.concurrent.*;

class Order {
    final int id;
    final String item;

    Order(int id, String item) {
        this.id = id;
        this.item = item;
    }

    @Override
    public String toString() {
        return "Order #" + id + " (" + item + ")";
    }
}

public class FoodOrderSystem {
    public static void main(String[] args) throws InterruptedException {
        // Kitchen can handle max 5 orders at a time
        BlockingQueue<Order> orderQueue = new ArrayBlockingQueue<>(5);

        String[] menuItems = {"Burger", "Pizza", "Pasta", "Salad", "Biryani"};

        // === PRODUCER: Customer placing orders ===
        Thread customer = new Thread(() -> {
            try {
                for (int i = 1; i <= 10; i++) {
                    String item = menuItems[i % menuItems.length];
                    Order order = new Order(i, item);
                    System.out.println("📝 Customer placing: " + order);
                    
                    orderQueue.put(order);  // Blocks if queue is full!
                    
                    System.out.println("  ✓ Order placed! Queue size: " + 
                                     orderQueue.size());
                    Thread.sleep(300);  // Time between orders
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }, "Customer");

        // === CONSUMER: Chef preparing orders ===
        Thread chef = new Thread(() -> {
            try {
                for (int i = 0; i < 10; i++) {
                    Order order = orderQueue.take();  // Blocks if queue is empty!
                    
                    System.out.println("👨‍🍳 Chef preparing: " + order);
                    Thread.sleep(800);  // Cooking time (slower than ordering)
                    System.out.println("  🍽️  " + order + " READY!");
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }, "Chef");

        customer.start();
        chef.start();
        customer.join();
        chef.join();

        System.out.println("\n═══════════════════════════════════════");
        System.out.println("All orders completed! 🎉");
    }
}
```

**Output:**
```
📝 Customer placing: Order #1 (Pizza)
  ✓ Order placed! Queue size: 1
👨‍🍳 Chef preparing: Order #1 (Pizza)
📝 Customer placing: Order #2 (Pasta)
  ✓ Order placed! Queue size: 1
📝 Customer placing: Order #3 (Salad)
  ✓ Order placed! Queue size: 2
  🍽️  Order #1 (Pizza) READY!
👨‍🍳 Chef preparing: Order #2 (Pasta)
📝 Customer placing: Order #4 (Biryani)
  ✓ Order placed! Queue size: 2
...
═══════════════════════════════════════
All orders completed! 🎉
```

Notice how the queue naturally **balances** the producer and consumer — the customer can keep ordering while the chef cooks!

### Comparison: BlockingQueue vs wait/notify

Remember using `wait()` and `notify()` for Producer-Consumer? Look how much simpler `BlockingQueue` is:

```java
// ❌ OLD WAY: Manual synchronization (error-prone!)
synchronized (buffer) {
    while (buffer.isFull()) {
        buffer.wait();         // Wait for space
    }
    buffer.add(item);
    buffer.notifyAll();        // Notify consumers
}

// ✅ NEW WAY: BlockingQueue (one line!)
queue.put(item);  // Automatically waits if full, notifies consumers
```

---

## 5. ConcurrentLinkedQueue — Non-Blocking Queue

Unlike `BlockingQueue`, `ConcurrentLinkedQueue` **never blocks**. It uses CAS (Compare-And-Swap) for thread safety and returns `null` when empty.

```java
import java.util.concurrent.ConcurrentLinkedQueue;

public class TaskQueueDemo {
    public static void main(String[] args) throws InterruptedException {
        ConcurrentLinkedQueue<String> taskQueue = new ConcurrentLinkedQueue<>();

        // Producers add tasks
        Thread producer = new Thread(() -> {
            String[] tasks = {"Send email", "Generate report", "Backup DB", 
                            "Clean cache", "Sync data"};
            for (String task : tasks) {
                taskQueue.offer(task);
                System.out.println("  + Added: " + task);
            }
        });

        producer.start();
        producer.join();

        // Consumers process tasks
        System.out.println("\n--- Processing ---");
        Thread[] workers = new Thread[2];
        for (int i = 0; i < 2; i++) {
            workers[i] = new Thread(() -> {
                String task;
                while ((task = taskQueue.poll()) != null) {
                    System.out.println(Thread.currentThread().getName() + 
                                     " → " + task);
                }
            }, "Worker-" + i);
            workers[i].start();
        }

        for (Thread w : workers) w.join();
        System.out.println("\nAll tasks processed! ✅");
    }
}
```

---

## 6. When to Use Which Collection

### Decision Table

| Need | Collection | Why |
|------|------------|-----|
| Thread-safe `HashMap` | `ConcurrentHashMap` | Segment-level locking, concurrent reads/writes |
| Thread-safe `ArrayList` (read-heavy) | `CopyOnWriteArrayList` | Lock-free reads, creates copy on write |
| Thread-safe `HashSet` (read-heavy) | `CopyOnWriteArraySet` | Same as CopyOnWriteArrayList internally |
| Thread-safe `TreeMap` | `ConcurrentSkipListMap` | Sorted, concurrent, lock-free |
| Thread-safe `TreeSet` | `ConcurrentSkipListSet` | Sorted set, concurrent, lock-free |
| Bounded producer-consumer | `ArrayBlockingQueue` | Fixed size, blocking put/take |
| Unbounded producer-consumer | `LinkedBlockingQueue` | Growing size, blocking take |
| Priority-based processing | `PriorityBlockingQueue` | Elements processed by priority |
| Direct handoff between threads | `SynchronousQueue` | Zero-capacity, direct transfer |
| Delayed processing | `DelayQueue` | Elements available after delay |
| Non-blocking queue | `ConcurrentLinkedQueue` | CAS-based, returns null when empty |

### Don't Use These (Legacy)

| Legacy | Modern Replacement |
|--------|--------------------|
| `Vector` | `CopyOnWriteArrayList` or `Collections.synchronizedList()` |
| `Hashtable` | `ConcurrentHashMap` |
| `Collections.synchronizedMap()` | `ConcurrentHashMap` |
| `Collections.synchronizedList()` | `CopyOnWriteArrayList` (if read-heavy) |

---

## Summary

### Key Takeaways

1. ✅ **Never use regular collections** from multiple threads — they will corrupt
2. ✅ **ConcurrentHashMap** is your go-to for thread-safe maps — it's fast and scalable
3. ✅ **BlockingQueue** eliminates manual wait/notify for Producer-Consumer
4. ✅ **CopyOnWriteArrayList** shines when reads vastly outnumber writes
5. ✅ Use **atomic compound operations** (`merge`, `compute`, `putIfAbsent`) on ConcurrentHashMap
6. ⚠️ **Avoid `synchronizedMap/List`** — they use a single global lock

---

### What's Next?

In the next article, we'll explore:
- **Synchronizers** — `CountDownLatch`, `CyclicBarrier`, `Semaphore`
- **ReadWriteLock** & **StampedLock** for fine-grained locking
- Coordinating complex multi-thread workflows

👉 [Continue to Java Synchronizers](/posts/java-synchronizers/)

---

*Happy Threading! 🧵📦*
