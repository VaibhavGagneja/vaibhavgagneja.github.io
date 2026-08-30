---
title: "Java I/O Streams: A Complete Guide from Byte Streams to Object Serialization"
description: Master Java I/O with byte streams, character streams, buffered I/O, Scanner, formatting, data streams, and object serialization with practical examples
author: Vaibhav Gagneja
date: 2026-02-11 12:00:00 +0530
categories: [Development, Java]
tags: [java, io, streams, serialization, file-handling, scanner]
toc: true
image:
  path: https://images.unsplash.com/photo-1637335088701-d204113650f3
---

Every Java program needs to read input or write output at some point — whether it's reading a file, accepting user input from the keyboard, or saving objects to disk. Java's **I/O (Input/Output) Streams** framework is how you do all of this.

In this guide, we'll go from the basics (byte streams) all the way to advanced topics (object serialization), with clear examples and explanations at every step.

---

## 1. What Is a Stream?

Think of a **stream** as a **pipeline** through which data flows:

```
┌─────────────────────────────────────────────────────────────────┐
│                     WHAT IS A STREAM?                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│   A stream is simply a SEQUENCE OF DATA flowing from             │
│   a SOURCE to a DESTINATION                                      │
│                                                                   │
│   INPUT STREAM (Reading):                                        │
│                                                                   │
│   ┌──────────┐     ┌──────────────┐     ┌──────────────┐        │
│   │  Source   │ ──► │ Input Stream │ ──► │ Your Program │        │
│   │ (File,   │     │  (Pipeline)  │     │              │        │
│   │ Network) │     └──────────────┘     └──────────────┘        │
│   └──────────┘                                                    │
│                                                                   │
│   OUTPUT STREAM (Writing):                                       │
│                                                                   │
│   ┌──────────────┐     ┌───────────────┐     ┌────────────┐     │
│   │ Your Program │ ──► │ Output Stream │ ──► │ Destination│     │
│   │              │     │  (Pipeline)   │     │ (File,     │     │
│   └──────────────┘     └───────────────┘     │ Screen)    │     │
│                                               └────────────┘     │
│                                                                   │
│   Sources/Destinations: Files, Network, Keyboard, Memory, etc.   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Key Points About Streams

| Property | Description |
|----------|-------------|
| **Direction** | Input (reading) or Output (writing) |
| **Data type** | Bytes (raw 8-bit) or Characters (16-bit Unicode) |
| **Behavior** | Some just pass data through; others transform it |
| **Universal model** | All streams work the same way: read/write one item at a time |

> **💡 Simple Rule:** No matter what you're reading from or writing to, the stream API gives you a consistent, simple interface.

---

## 2. Stream Hierarchy at a Glance

Before diving into code, let's see how Java organizes its stream classes:

```
┌─────────────────────────────────────────────────────────────────┐
│                    JAVA I/O STREAM FAMILY TREE                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│   BYTE STREAMS (raw 8-bit data)                                  │
│   ═════════════════════════════                                  │
│   InputStream (abstract)          OutputStream (abstract)        │
│   ├── FileInputStream             ├── FileOutputStream           │
│   ├── BufferedInputStream         ├── BufferedOutputStream       │
│   ├── DataInputStream             ├── DataOutputStream           │
│   ├── ObjectInputStream           ├── ObjectOutputStream         │
│   └── ByteArrayInputStream       └── ByteArrayOutputStream      │
│                                                                   │
│   CHARACTER STREAMS (16-bit Unicode)                             │
│   ══════════════════════════════════                             │
│   Reader (abstract)               Writer (abstract)              │
│   ├── FileReader                  ├── FileWriter                 │
│   ├── BufferedReader              ├── BufferedWriter             │
│   ├── InputStreamReader           ├── OutputStreamWriter         │
│   └── StringReader                ├── PrintWriter                │
│                                   └── StringWriter               │
│                                                                   │
│   🔑 Rule of Thumb:                                              │
│   • Text data → Use Character Streams (Reader/Writer)            │
│   • Binary data (images, audio) → Use Byte Streams               │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. Byte Streams

Byte streams handle **raw binary data** — 8 bits at a time. All byte stream classes descend from `InputStream` and `OutputStream`.

### Example: Copy a File Byte by Byte

```java
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.IOException;

public class CopyBytes {
    public static void main(String[] args) throws IOException {
        FileInputStream in = null;
        FileOutputStream out = null;

        try {
            in = new FileInputStream("xanadu.txt");
            out = new FileOutputStream("outagain.txt");
            int c;

            while ((c = in.read()) != -1) {   // Read one byte
                out.write(c);                   // Write one byte
            }
        } finally {
            if (in != null)  in.close();
            if (out != null) out.close();
        }
    }
}
```

### How It Works

```
┌──────────────────────────────────────────────────────────────┐
│                  BYTE-BY-BYTE COPY FLOW                       │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  xanadu.txt ──► FileInputStream ──► int c ──► FileOutputStream│
│                                                                │
│  Loop: read() returns next byte as int (0-255)                │
│        returns -1 when file ends                               │
│        write(c) writes that byte to output file               │
│                                                                │
│  "In Xanadu did Kubla Khan..."                                │
│   ↓  ↓  ↓  ↓ ↓ ↓ ↓ ↓ ↓ ↓                                   │
│  73 110 32 88 97 110 97 100 117 32 ...  (ASCII byte values)  │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Why `int` Instead of `byte`?

You might wonder: if we're reading bytes, why does `read()` return an `int`?

| Return Value | Meaning |
|-------------|---------|
| `0` to `255` | The byte value that was read |
| `-1` | End of stream (no more data) |

Since `byte` in Java is signed (-128 to 127), it can't represent all byte values *and* an end-of-stream signal. The `int` type gives us room for both!

### ⚠️ Always Close Streams!

```java
// ✅ Use finally block to guarantee closing
finally {
    if (in != null)  in.close();
    if (out != null) out.close();
}

// ✅✅ Even better — try-with-resources (Java 7+)
try (FileInputStream in = new FileInputStream("xanadu.txt");
     FileOutputStream out = new FileOutputStream("outagain.txt")) {

    int c;
    while ((c = in.read()) != -1) {
        out.write(c);
    }
}  // Streams auto-closed here! No finally block needed!
```

> **⚠️ Important:** Not closing streams causes **resource leaks** — your program holds onto file handles, memory, and even network connections that it no longer needs. This is one of the most common Java bugs!

### When NOT to Use Byte Streams

Byte streams are **too low-level** for text data. If you're working with text files, character data, or strings — use **Character Streams** instead.

> **So why learn byte streams?** Because **all other stream types are built on top of byte streams!** They're the foundation.

---

## 4. Character Streams

The Java platform stores characters using **Unicode** (16-bit). Character streams automatically translate between Unicode and the local character encoding (like UTF-8 or ASCII).

### Why Character Streams Matter

| Byte Streams | Character Streams |
|-------------|------------------|
| Read/write raw bytes (8-bit) | Read/write characters (16-bit Unicode) |
| No encoding awareness | Handles encoding automatically |
| `FileInputStream` / `FileOutputStream` | `FileReader` / `FileWriter` |
| Good for images, audio, binary | Good for text files, logs, config |

### Example: Copy a File Character by Character

```java
import java.io.FileReader;
import java.io.FileWriter;
import java.io.IOException;

public class CopyCharacters {
    public static void main(String[] args) throws IOException {
        FileReader inputStream = null;
        FileWriter outputStream = null;

        try {
            inputStream = new FileReader("xanadu.txt");
            outputStream = new FileWriter("characteroutput.txt");

            int c;
            while ((c = inputStream.read()) != -1) {
                outputStream.write(c);
            }
        } finally {
            if (inputStream != null)  inputStream.close();
            if (outputStream != null) outputStream.close();
        }
    }
}
```

### Byte vs Character: What's the Difference in the `int`?

The code looks almost identical to the byte version! But there's a subtle difference:

| | Byte Stream | Character Stream |
|-|-------------|-----------------|
| `read()` returns | `int` with **last 8 bits** = byte value | `int` with **last 16 bits** = character value |
| Range | 0–255 | 0–65535 (full Unicode) |
| `'A'` is stored as | `65` | `65` |
| `'₹'` (Rupee sign) | Can't represent it in one byte! | `8377` — works perfectly |

### Character Streams Use Byte Streams Internally

Here's a key insight: **character streams are wrappers around byte streams!**

```
┌─────────────────────────────────────────────────────┐
│          CHARACTER STREAM ARCHITECTURE                │
├─────────────────────────────────────────────────────┤
│                                                       │
│  Your Code                                           │
│      │                                               │
│      ▼                                               │
│  ┌──────────┐   Translates chars ↔ bytes             │
│  │FileReader│ ────────────────────────────┐           │
│  └──────────┘                             │           │
│      │                                    ▼           │
│      │ (internally uses)          ┌───────────────┐  │
│      └───────────────────────────►│FileInputStream│  │
│                                   └───────────────┘  │
│                                         │             │
│                                         ▼             │
│                                    Disk / File        │
│                                                       │
│  FileReader = FileInputStream + character decoding    │
│  FileWriter = FileOutputStream + character encoding   │
│                                                       │
└─────────────────────────────────────────────────────┘
```

### Bridge Classes: `InputStreamReader` and `OutputStreamWriter`

When you need to convert between byte and character streams manually (for example, reading from `System.in` which is a byte stream):

```java
// System.in is a byte stream — wrap it to get character stream
InputStreamReader cin = new InputStreamReader(System.in);

// Now you can read characters instead of bytes!
int c = cin.read();  // Returns a character, not a byte

// You can also specify encoding:
InputStreamReader reader = new InputStreamReader(
    new FileInputStream("data.txt"), "UTF-8"
);
```

---

## 5. Line-Oriented I/O

Reading character-by-character is tedious. Most text processing works with **entire lines** at a time.

### Example: Copy a File Line by Line

```java
import java.io.FileReader;
import java.io.FileWriter;
import java.io.BufferedReader;
import java.io.PrintWriter;
import java.io.IOException;

public class CopyLines {
    public static void main(String[] args) throws IOException {
        BufferedReader inputStream = null;
        PrintWriter outputStream = null;

        try {
            inputStream = new BufferedReader(new FileReader("xanadu.txt"));
            outputStream = new PrintWriter(new FileWriter("characteroutput.txt"));

            String l;
            while ((l = inputStream.readLine()) != null) {  // Read whole line
                outputStream.println(l);                     // Write whole line
            }
        } finally {
            if (inputStream != null)  inputStream.close();
            if (outputStream != null) outputStream.close();
        }
    }
}
```

### What's a "Line Terminator"?

Different operating systems use different characters to mark the end of a line:

| OS | Line Terminator | Symbol |
|----|----------------|--------|
| **Windows** | Carriage Return + Line Feed | `\r\n` |
| **Linux/Mac** | Line Feed only | `\n` |
| **Old Mac (pre-OS X)** | Carriage Return only | `\r` |

`BufferedReader.readLine()` handles **all three** formats automatically! And `PrintWriter.println()` writes the line terminator appropriate for **your** operating system.

---

## 6. Buffered Streams

### The Problem with Unbuffered I/O

Without buffering, **every `read()` or `write()` call hits the disk or network directly**. This is incredibly slow because disk access is thousands of times slower than memory access.

```
┌──────────────────────────────────────────────────────────────┐
│              UNBUFFERED vs BUFFERED I/O                        │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  UNBUFFERED (SLOW 🐢):                                       │
│  ───────────────────────                                      │
│  read()  → Disk Access!     (every single byte = disk hit)   │
│  read()  → Disk Access!                                       │
│  read()  → Disk Access!     (1000 reads = 1000 disk hits 😱) │
│  ...                                                           │
│                                                                │
│  BUFFERED (FAST ⚡):                                          │
│  ─────────────────────                                        │
│  read()  → From Buffer ✓    (memory access, FAST!)            │
│  read()  → From Buffer ✓                                      │
│  read()  → From Buffer ✓                                      │
│  ...                                                           │
│  read()  → Buffer empty! ──► Disk Access (refill buffer)     │
│  read()  → From Buffer ✓    (1000 reads ≈ 1-2 disk hits ✨) │
│  ...                                                           │
│                                                                │
│  Buffer = chunk of memory that stores data temporarily        │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### How to Add Buffering

Just **wrap** an unbuffered stream in a buffered one:

```java
// Byte streams — add buffering
BufferedInputStream  bis = new BufferedInputStream(new FileInputStream("data.bin"));
BufferedOutputStream bos = new BufferedOutputStream(new FileOutputStream("data.bin"));

// Character streams — add buffering
BufferedReader br = new BufferedReader(new FileReader("xanadu.txt"));
BufferedWriter bw = new BufferedWriter(new FileWriter("output.txt"));
```

### The Four Buffered Stream Classes

| Unbuffered | Buffered Wrapper | Type |
|-----------|-----------------|------|
| `FileInputStream` | `BufferedInputStream` | Byte input |
| `FileOutputStream` | `BufferedOutputStream` | Byte output |
| `FileReader` | `BufferedReader` | Character input |
| `FileWriter` | `BufferedWriter` | Character output |

### Flushing the Buffer

Sometimes you want to **force** the buffer to write its contents immediately, without waiting for it to fill up:

```java
BufferedWriter writer = new BufferedWriter(new FileWriter("log.txt"));
writer.write("Critical error occurred!");
writer.flush();  // Write to disk NOW, don't wait for buffer to fill!
```

> **When to flush:**
> - After writing critical log messages
> - Before waiting for user input (so prompts appear)
> - When you need data visible to other programs immediately
>
> **Auto-flush:** `PrintWriter` with autoflush enabled flushes on every `println()` or `format()` call.

---

## 7. Scanning — Parsing Formatted Input

The `Scanner` class makes it easy to **break input into tokens** and parse them into Java types.

### Reading Words from a File

```java
import java.io.*;
import java.util.Scanner;

public class ScanXan {
    public static void main(String[] args) throws IOException {
        Scanner s = null;

        try {
            s = new Scanner(new BufferedReader(new FileReader("xanadu.txt")));

            while (s.hasNext()) {
                System.out.println(s.next());  // Print each word
            }
        } finally {
            if (s != null) s.close();
        }
    }
}
```

**Input file (xanadu.txt):**
```
In Xanadu did Kubla Khan
A stately pleasure-dome decree:
```

**Output:**
```
In
Xanadu
did
Kubla
Khan
A
stately
pleasure-dome
decree:
```

### How Scanner Works

```
┌──────────────────────────────────────────────────────────────┐
│                    SCANNER TOKENIZATION                        │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Input: "In Xanadu did Kubla Khan"                            │
│          ↓  ↓       ↓    ↓     ↓                              │
│          │  │       │    │     │                               │
│  Token:  1  2       3    4     5                               │
│                                                                │
│  Default delimiter: whitespace (spaces, tabs, newlines)       │
│                                                                │
│  Custom delimiter:                                             │
│  s.useDelimiter(",\\s*");  // Split by comma + optional space │
│  "apple, banana, cherry" → ["apple", "banana", "cherry"]     │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Reading Different Data Types

```java
import java.io.*;
import java.util.Scanner;
import java.util.Locale;

public class ScanSum {
    public static void main(String[] args) throws IOException {
        Scanner s = null;
        double sum = 0;

        try {
            s = new Scanner(new BufferedReader(new FileReader("usnumbers.txt")));
            s.useLocale(Locale.US);  // Important for number formatting!

            while (s.hasNext()) {
                if (s.hasNextDouble()) {
                    sum += s.nextDouble();   // Parse as double
                } else {
                    s.next();                // Skip non-numeric tokens
                }
            }
        } finally {
            if (s != null) s.close();
        }

        System.out.println(sum);
    }
}
```

**Input file (usnumbers.txt):**
```
8.5
32,767
3.14159
1,000,000.1
```

**Output:** `1032778.74159`

### Scanner Cheat Sheet

| Method | What It Does |
|--------|-------------|
| `next()` | Returns next token as `String` |
| `nextInt()` | Returns next token as `int` |
| `nextDouble()` | Returns next token as `double` |
| `nextLine()` | Returns the rest of the current line |
| `hasNext()` | Is there another token? |
| `hasNextInt()` | Is the next token a valid `int`? |
| `useDelimiter(regex)` | Change the token separator |
| `useLocale(locale)` | Set locale for number parsing |

> **⚠️ Remember:** Even though `Scanner` is not a stream, you should still **close it** when done — this also closes the underlying stream!

---

## 8. Formatting Output

Java provides two main ways to format output: `print/println` for simple output, and `format` for precise control.

### `print` and `println` — Simple Output

```java
public class Root {
    public static void main(String[] args) {
        int i = 2;
        double r = Math.sqrt(i);

        System.out.print("The square root of ");
        System.out.print(i);
        System.out.print(" is ");
        System.out.print(r);
        System.out.println(".");

        // Or combine with string concatenation:
        System.out.println("The square root of " + i + " is " + r + ".");
    }
}
```

**Output:**
```
The square root of 2 is 1.4142135623730951.
The square root of 2 is 1.4142135623730951.
```

### `format` — Precise Control

```java
public class Root2 {
    public static void main(String[] args) {
        int i = 2;
        double r = Math.sqrt(i);

        System.out.format("The square root of %d is %f.%n", i, r);
    }
}
```

**Output:** `The square root of 2 is 1.414214.`

### Common Format Specifiers

| Specifier | Type | Example | Output |
|-----------|------|---------|--------|
| `%d` | Integer (decimal) | `format("%d", 42)` | `42` |
| `%f` | Float/Double | `format("%f", 3.14)` | `3.140000` |
| `%s` | String (any object) | `format("%s", "Hi")` | `Hi` |
| `%n` | Platform line separator | `format("line1%nline2")` | `line1↵line2` |
| `%x` | Integer (hexadecimal) | `format("%x", 255)` | `ff` |
| `%e` | Scientific notation | `format("%e", 0.001)` | `1.000000e-03` |
| `%b` | Boolean | `format("%b", true)` | `true` |
| `%%` | Literal `%` sign | `format("100%%")` | `100%` |

### Anatomy of a Format Specifier

```
┌──────────────────────────────────────────────────────────────┐
│              FORMAT SPECIFIER BREAKDOWN                        │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  System.out.format("%1$+020.10f", Math.PI);                   │
│                                                                │
│  Output: +00000003.1415926536                                 │
│                                                                │
│   %  1$  +  0  20  .10  f                                    │
│   │   │  │  │   │    │   │                                    │
│   │   │  │  │   │    │   └── Conversion: float                │
│   │   │  │  │   │    └────── Precision: 10 decimal places     │
│   │   │  │  │   └─────────── Width: minimum 20 characters     │
│   │   │  │  └──────────────── Flag: pad with zeros            │
│   │   │  └───────────────────  Flag: always show sign (+/-)   │
│   │   └──────────────────────── Argument index: use 1st arg   │
│   └─────────────────────────────  Start of specifier          │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### `%n` vs `\n` — Important Difference!

| | `%n` | `\n` |
|-|------|------|
| **What it produces** | Platform-specific line separator | Always `\u000A` (line feed) |
| **On Windows** | `\r\n` (CR+LF) | `\n` (just LF) |
| **On Linux/Mac** | `\n` (LF) | `\n` (LF) |
| **Use when** | You want **correct** line endings | You specifically want a linefeed only |

> **Best Practice:** Always use `%n` in `format()` strings for platform-portable code.

---

## 9. I/O from the Command Line

### Standard Streams

Java provides three pre-defined streams:

| Stream | Access Via | Direction | Type | Purpose |
|--------|-----------|-----------|------|---------|
| **Standard Input** | `System.in` | Input | `InputStream` (byte) | Keyboard input |
| **Standard Output** | `System.out` | Output | `PrintStream` (byte) | Normal output |
| **Standard Error** | `System.err` | Output | `PrintStream` (byte) | Error messages |

> **Why is `System.in` a byte stream?** Historical reasons! Wrap it for character support:
> ```java
> InputStreamReader cin = new InputStreamReader(System.in);
> BufferedReader reader = new BufferedReader(cin);
> String line = reader.readLine();
> ```

> **Why separate `System.out` and `System.err`?** So users can redirect normal output to a file while still seeing errors on screen: `java MyApp > output.txt` (errors still appear on screen!)

### The Console Class

`Console` is a more advanced alternative for command-line I/O, especially useful for **password input**:

```java
import java.io.Console;
import java.util.Arrays;

public class Password {
    public static void main(String[] args) {
        Console c = System.console();
        if (c == null) {
            System.err.println("No console available.");
            System.exit(1);
        }

        String login = c.readLine("Enter your login: ");
        char[] oldPassword = c.readPassword("Enter your password: ");

        // Process the password...

        // IMPORTANT: Overwrite password in memory when done!
        Arrays.fill(oldPassword, ' ');
    }
}
```

### Why `char[]` Instead of `String` for Passwords?

| `String` | `char[]` |
|----------|----------|
| **Immutable** — stays in memory until GC collects it | **Mutable** — you can overwrite it immediately |
| Can appear in heap dumps and logs | You fill it with blanks after use |
| Less secure 🔓 | More secure 🔒 |

---

## 10. Data Streams — Binary I/O of Primitives

Data streams let you read/write **primitive types** (`int`, `double`, `boolean`, etc.) and `String` values in binary format.

### Writing Data

```java
import java.io.*;

public class DataStreamWriter {
    public static void main(String[] args) throws IOException {
        String dataFile = "invoicedata";

        double[] prices = {19.99, 9.99, 15.99, 3.99, 4.99};
        int[] units = {12, 8, 13, 29, 50};
        String[] descs = {
            "Java T-shirt", "Java Mug", "Duke Juggling Dolls",
            "Java Pin", "Java Key Chain"
        };

        try (DataOutputStream out = new DataOutputStream(
                new BufferedOutputStream(
                    new FileOutputStream(dataFile)))) {

            for (int i = 0; i < prices.length; i++) {
                out.writeDouble(prices[i]);
                out.writeInt(units[i]);
                out.writeUTF(descs[i]);       // Modified UTF-8 encoding
            }
        }
        System.out.println("Data written successfully!");
    }
}
```

### Reading Data

```java
import java.io.*;

public class DataStreamReader {
    public static void main(String[] args) throws IOException {
        String dataFile = "invoicedata";

        try (DataInputStream in = new DataInputStream(
                new BufferedInputStream(
                    new FileInputStream(dataFile)))) {

            double total = 0.0;

            try {
                while (true) {
                    double price = in.readDouble();     // Must match writeDouble
                    int unit = in.readInt();             // Must match writeInt
                    String desc = in.readUTF();          // Must match writeUTF

                    System.out.format("You ordered %d units of %s at $%.2f%n",
                        unit, desc, price);
                    total += unit * price;
                }
            } catch (EOFException e) {
                // End of file reached — this is NORMAL for DataInputStream!
            }

            System.out.format("Total: $%.2f%n", total);
        }
    }
}
```

**Output:**
```
You ordered 12 units of Java T-shirt at $19.99
You ordered 8 units of Java Mug at $9.99
You ordered 13 units of Duke Juggling Dolls at $15.99
You ordered 29 units of Java Pin at $3.99
You ordered 50 units of Java Key Chain at $4.99
Total: $892.88
```

### Important Rules for Data Streams

```
┌──────────────────────────────────────────────────────────────┐
│                DATA STREAM GOLDEN RULES ⚠️                    │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  1. WRITE ORDER = READ ORDER                                  │
│     writeDouble → readDouble                                  │
│     writeInt    → readInt                                     │
│     writeUTF    → readUTF                                     │
│     ❌ If you mismatch, you'll read garbage!                  │
│                                                                │
│  2. END-OF-FILE = EOFException                                │
│     DataInput uses exceptions, NOT return values (-1)         │
│     Catch EOFException to detect end of data                  │
│                                                                │
│  3. DON'T USE float/double FOR MONEY!                         │
│     ❌ 0.1 + 0.2 = 0.30000000000000004                       │
│     ✅ Use BigDecimal + Object Streams instead                │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## 11. Object Streams — Serialization

Object streams allow you to **read and write entire Java objects** to/from a stream. This process is called **serialization** (writing) and **deserialization** (reading).

### What is Serialization?

```
┌──────────────────────────────────────────────────────────────┐
│                  SERIALIZATION EXPLAINED                       │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  SERIALIZATION (Writing):                                     │
│  Java Object ──► Byte Sequence ──► File / Network             │
│                                                                │
│  ┌────────────────┐       ┌──────────────┐       ┌─────────┐ │
│  │ Student Object │  ──►  │  01 4A 7B... │  ──►  │ File    │ │
│  │ name="Vaibhav" │       │  (bytes)     │       │ student │ │
│  │ age=25         │       └──────────────┘       │  .dat   │ │
│  └────────────────┘                               └─────────┘ │
│                                                                │
│  DESERIALIZATION (Reading):                                    │
│  File / Network ──► Byte Sequence ──► Java Object             │
│                                                                │
│  ┌─────────┐       ┌──────────────┐       ┌────────────────┐ │
│  │ File    │  ──►  │  01 4A 7B... │  ──►  │ Student Object │ │
│  │ student │       │  (bytes)     │       │ name="Vaibhav" │ │
│  │  .dat   │       └──────────────┘       │ age=25         │ │
│  └─────────┘                               └────────────────┘ │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Making a Class Serializable

Your class **must implement** the `Serializable` marker interface:

```java
import java.io.Serializable;

public class Student implements Serializable {
    private static final long serialVersionUID = 1L;  // Version control

    private String name;
    private int age;
    private transient String password;  // transient = NOT serialized!

    public Student(String name, int age, String password) {
        this.name = name;
        this.age = age;
        this.password = password;
    }

    @Override
    public String toString() {
        return "Student{name='" + name + "', age=" + age +
               ", password='" + password + "'}";
    }
}
```

### Writing Objects

```java
import java.io.*;

public class SerializeDemo {
    public static void main(String[] args) throws IOException {
        Student student = new Student("Vaibhav", 25, "secret123");

        try (ObjectOutputStream out = new ObjectOutputStream(
                new FileOutputStream("student.dat"))) {

            out.writeObject(student);
            System.out.println("Object serialized: " + student);
        }
    }
}
```

### Reading Objects

```java
import java.io.*;

public class DeserializeDemo {
    public static void main(String[] args)
            throws IOException, ClassNotFoundException {

        try (ObjectInputStream in = new ObjectInputStream(
                new FileInputStream("student.dat"))) {

            Student student = (Student) in.readObject();
            System.out.println("Object deserialized: " + student);
        }
    }
}
```

**Output:**
```
Object serialized:   Student{name='Vaibhav', age=25, password='secret123'}
Object deserialized: Student{name='Vaibhav', age=25, password='null'}
```

Notice that `password` is `null` after deserialization because it was marked `transient`!

### Important Serialization Concepts

| Concept | Explanation |
|---------|-------------|
| `Serializable` | Marker interface — no methods to implement |
| `serialVersionUID` | Version number — helps detect incompatible changes |
| `transient` | Fields marked `transient` are **skipped** during serialization |
| Object graph | If your object references other objects, **they get serialized too!** |
| Same reference | If two objects reference the same child, only one copy is stored |

### Object Reference Graph

When you serialize an object, Java **automatically serializes everything it references**:

```
┌──────────────────────────────────────────────────────────────┐
│              OBJECT GRAPH SERIALIZATION                        │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  writeObject(a)  → Writes ALL of these automatically!         │
│                                                                │
│       ┌───┐                                                    │
│       │ a │──────────────┐                                     │
│       └───┘              │                                     │
│         │                ▼                                     │
│         │              ┌───┐                                   │
│         │              │ c │                                   │
│         ▼              └───┘                                   │
│       ┌───┐                                                    │
│       │ b │──────────────┐                                     │
│       └───┘              │                                     │
│         │                ▼                                     │
│         ▼              ┌───┐                                   │
│       ┌───┐            │ e │                                   │
│       │ d │            └───┘                                   │
│       └───┘                                                    │
│                                                                │
│  writeObject(a) → writes a, b, c, d, e (5 objects total!)     │
│  readObject()   → reconstructs a, b, c, d, e with all links  │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Duplicate Reference Handling

```java
Object ob = new Object();
out.writeObject(ob);   // Writes the object
out.writeObject(ob);   // Writes only a REFERENCE (not a second copy!)

// Reading back:
Object ob1 = in.readObject();
Object ob2 = in.readObject();
System.out.println(ob1 == ob2);  // true — same object!
```

> **Key insight:** A stream stores only **one copy** of each object but can contain **multiple references** to it.

---

## 12. try-with-resources — The Modern Way

Since Java 7, you should use **try-with-resources** for all I/O operations. It automatically closes streams — even if exceptions occur!

```java
// ❌ OLD WAY (verbose, error-prone)
BufferedReader reader = null;
try {
    reader = new BufferedReader(new FileReader("file.txt"));
    String line = reader.readLine();
} finally {
    if (reader != null) reader.close();  // What if close() throws?!
}

// ✅ MODERN WAY (clean, safe)
try (BufferedReader reader = new BufferedReader(new FileReader("file.txt"))) {
    String line = reader.readLine();
}  // Auto-closed! Even if exception occurs!

// ✅ Multiple resources
try (FileInputStream fis = new FileInputStream("input.txt");
     FileOutputStream fos = new FileOutputStream("output.txt");
     BufferedInputStream bis = new BufferedInputStream(fis);
     BufferedOutputStream bos = new BufferedOutputStream(fos)) {

    int b;
    while ((b = bis.read()) != -1) {
        bos.write(b);
    }
}  // All four streams closed automatically in reverse order!
```

> Any class that implements `AutoCloseable` (or its subinterface `Closeable`) can be used with try-with-resources.

---

## 13. Complete I/O Cheat Sheet

### Which Stream to Use?

```
┌──────────────────────────────────────────────────────────────┐
│                  CHOOSING THE RIGHT STREAM                     │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  What are you doing?                                          │
│  │                                                             │
│  ├── Reading/writing TEXT?                                     │
│  │   ├── Whole lines? → BufferedReader / PrintWriter           │
│  │   ├── Parsing tokens? → Scanner                             │
│  │   └── Raw characters? → FileReader / FileWriter             │
│  │                                                             │
│  ├── Reading/writing BINARY data?                              │
│  │   ├── Primitive types (int, double)? → DataInputStream/Out  │
│  │   ├── Java objects? → ObjectInputStream/OutputStream        │
│  │   └── Raw bytes? → BufferedInputStream/OutputStream         │
│  │                                                             │
│  ├── Reading from KEYBOARD?                                    │
│  │   ├── Simple input? → Scanner(System.in)                    │
│  │   └── Password input? → System.console().readPassword()     │
│  │                                                             │
│  └── Writing to SCREEN?                                        │
│      ├── Simple text? → System.out.println()                   │
│      └── Formatted? → System.out.format()                      │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Complete Comparison Table

| Need | Class | Buffered? | Use Case |
|------|-------|-----------|----------|
| Read bytes | `FileInputStream` | ❌ | Low-level binary reading |
| Write bytes | `FileOutputStream` | ❌ | Low-level binary writing |
| Read bytes (fast) | `BufferedInputStream` | ✅ | Binary file reading |
| Write bytes (fast) | `BufferedOutputStream` | ✅ | Binary file writing |
| Read characters | `FileReader` | ❌ | Simple text reading |
| Write characters | `FileWriter` | ❌ | Simple text writing |
| Read lines | `BufferedReader` | ✅ | Text file, line-by-line |
| Write lines | `PrintWriter` | ✅ | Text output with formatting |
| Read primitives | `DataInputStream` | ❌* | Binary data (int, double) |
| Write primitives | `DataOutputStream` | ❌* | Binary data (int, double) |
| Read objects | `ObjectInputStream` | ❌* | Deserialization |
| Write objects | `ObjectOutputStream` | ❌* | Serialization |
| Parse input | `Scanner` | N/A | Tokenized input parsing |

\* *Wrap in `Buffered*Stream` for better performance!*

### The Wrapping Pattern

```java
// Streams are designed to be LAYERED (decorator pattern):

// Layer 1: Raw byte stream (talks to the file)
FileOutputStream fos = new FileOutputStream("data.bin");

// Layer 2: Buffered (adds performance)
BufferedOutputStream bos = new BufferedOutputStream(fos);

// Layer 3: Data stream (adds writeInt, writeDouble, etc.)
DataOutputStream dos = new DataOutputStream(bos);

// Now you can do:
dos.writeInt(42);         // Goes through all 3 layers!
dos.writeDouble(3.14);
dos.writeUTF("Hello");

// Or as a one-liner:
DataOutputStream dos = new DataOutputStream(
    new BufferedOutputStream(
        new FileOutputStream("data.bin")));
```

---

### Best Practices Summary

```
┌──────────────────────────────────────────────────────────────┐
│                   I/O BEST PRACTICES                          │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  ✅ DO:                                                       │
│  • Always close streams (use try-with-resources!)             │
│  • Use BufferedReader/Writer for text files                   │
│  • Use character streams for text, byte streams for binary   │
│  • Use Scanner for parsing user input                         │
│  • Use %n instead of \n in format strings                     │
│  • Add serialVersionUID to Serializable classes               │
│  • Mark sensitive fields as transient                         │
│                                                                │
│  ❌ DON'T:                                                     │
│  • Don't use byte streams for text data                       │
│  • Don't forget to flush buffered output                      │
│  • Don't use float/double for monetary values                 │
│  • Don't store passwords as String (use char[])               │
│  • Don't assume \n works on all platforms                      │
│  • Don't ignore IOExceptions                                  │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

*Happy I/O Coding! 📂🚀*
