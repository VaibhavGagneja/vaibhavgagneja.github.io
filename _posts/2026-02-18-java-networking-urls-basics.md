---
title: "Java Networking Part 1: Networking Basics, URLs, and URLConnections"
description: Understand TCP vs UDP, ports, the URL class, reading from URLs, and URLConnections with clear examples and diagrams
author: Vaibhav Gagneja
date: 2026-02-17 12:00:00 +0530
categories: [Development, Java]
tags: [java, networking, tcp, udp, url, urlconnection, ports]
toc: true
image:
  path: https://images.unsplash.com/photo-1544197150-b99a580bb7a8
---

Every Java developer uses the network — whether it's loading a web page, calling a REST API, or fetching a file from a server. But how does it all actually work under the hood? In this three-part series, we'll go from the **fundamentals of networking** all the way to building **multi-client servers** and working with **datagrams**.

In Part 1, we cover the foundation: how computers talk to each other, what TCP and UDP are, how ports work, and how Java makes it all easy through the `URL` and `URLConnection` classes.

---

## 1. How Java Uses the Network — Without You Realizing

You've probably used networking in Java already without thinking twice:

| What You Did | Networking Happening Behind the Scenes |
|-------------|---------------------------------------|
| Ran an applet in a browser | Browser downloaded the applet's `.class` files over the network |
| Loaded an image from a URL | Java fetched the image bytes from a remote server |
| Called a REST API with `HttpURLConnection` | TCP socket opened, HTTP request sent, response received |
| Used JDBC to query a remote database | Socket connection to the database server on a specific port |

Java provides several **levels of network access**, from high-level (easy, less control) to low-level (more work, full control):

```
┌──────────────────────────────────────────────────────────────────┐
│              LEVELS OF NETWORK ACCESS IN JAVA                      │
├──────────────────────────────────────────────────────────────────┤
│                                                                      │
│  HIGH LEVEL (Easiest)                                               │
│  ═══════════════════                                                │
│  URL.openStream()          → Read content from any URL              │
│  URLConnection             → Read + write + set headers             │
│  HttpURLConnection         → HTTP-specific features                 │
│                                                                      │
│  MID LEVEL                                                          │
│  ═════════                                                          │
│  Socket / ServerSocket     → TCP connections (reliable)             │
│  DatagramSocket            → UDP packets (fast, unreliable)         │
│                                                                      │
│  LOW LEVEL (Most Control)                                           │
│  ════════════════════════                                           │
│  SocketChannel / Selector  → Non-blocking I/O (Java NIO)           │
│  NetworkInterface          → Query network adapters directly        │
│                                                                      │
│  This series covers ▲ all of these ▲                                │
│                                                                      │
└──────────────────────────────────────────────────────────────────┘
```

---

## 2. Networking Basics: How Computers Talk

When two computers communicate over the Internet, they use a **protocol** — a set of rules for how data is formatted and transmitted. The two most important protocols are **TCP** and **UDP**.

```
┌──────────────────────────────────────────────────────────────────┐
│                   THE NETWORK LAYER CAKE                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────────────────────────────────┐                    │
│  │         APPLICATION LAYER                    │  ← Your Java code │
│  │      (HTTP, FTP, SMTP, your app)             │                    │
│  ├─────────────────────────────────────────────┤                    │
│  │         TRANSPORT LAYER                      │  ← TCP or UDP     │
│  │      (TCP or UDP)                            │                    │
│  ├─────────────────────────────────────────────┤                    │
│  │         NETWORK LAYER                        │  ← IP addresses   │
│  │      (IP — Internet Protocol)                │                    │
│  ├─────────────────────────────────────────────┤                    │
│  │         LINK LAYER                           │  ← Physical wires │
│  │      (Ethernet, Wi-Fi)                       │                    │
│  └─────────────────────────────────────────────┘                    │
│                                                                      │
│  When you write Java programs, you work at the APPLICATION layer.   │
│  Java's java.net package handles TCP/UDP details for you!           │
│                                                                      │
└──────────────────────────────────────────────────────────────────┘
```

### TCP — The Reliable Telephone Call

**TCP (Transmission Control Protocol)** is like making a **phone call**:
- You first **establish a connection** (dial the number, the other person answers)
- You talk **back and forth** — every word is guaranteed to arrive, **in order**
- When you're done, you **hang up** (close the connection)

**Key properties of TCP:**

| Property | Description |
|----------|-------------|
| **Connection-based** | Must establish a connection before sending data |
| **Reliable** | Every byte is guaranteed to arrive |
| **Ordered** | Data arrives in the exact order it was sent |
| **Error-checked** | Built-in error detection and retransmission |
| **Slower** | Reliability comes at a speed cost |

**Real-world services using TCP:**

| Service | Why TCP? |
|---------|---------|
| **HTTP/HTTPS** (web browsing) | Web pages must load completely and correctly |
| **FTP** (file transfer) | Files must arrive intact — even one missing byte corrupts the file |
| **SMTP** (email) | Emails must be delivered completely |
| **Telnet / SSH** | Every keystroke must arrive in order |

> **💡 The Telephone Analogy:** If you want to speak to your aunt in another city, you dial her number, she answers, and you have a reliable two-way conversation. If a word gets garbled, you say "Could you repeat that?" — TCP does this automatically!

### UDP — The Fast Postcard

**UDP (User Datagram Protocol)** is like sending a **postcard**:
- No connection needed — just write the address and drop it in the mailbox
- No guarantee it will arrive
- No guarantee of order — postcards sent Monday might arrive after those sent Tuesday
- But it's **fast** because there's no setup overhead

**Key properties of UDP:**

| Property | Description |
|----------|-------------|
| **Connectionless** | No setup needed — just send |
| **Unreliable** | Packets might be lost, duplicated, or arrive out of order |
| **Fast** | Minimal overhead |
| **Independent** | Each packet (datagram) is self-contained |

**Real-world services using UDP:**

| Service | Why UDP? |
|---------|---------|
| **DNS** (name resolution) | Quick query-response; if lost, just ask again |
| **Video streaming** | A few dropped frames are fine; speed matters more |
| **Online gaming** | Player positions must update fast, old data is useless |
| **VoIP** (voice calls) | Slight glitches are tolerable; delay is not |
| **ping** | Testing connectivity — dropped packets ARE the data! |

> **⚠️ Firewall Note:** Many firewalls block UDP packets by default. If UDP communication fails, check your firewall settings first.

### TCP vs UDP — Side by Side

```
┌──────────────────────────────────────────────────────────────────┐
│                    TCP vs UDP COMPARISON                             │
├──────────────────────────────────────────────────────────────────┤
│                                                                      │
│  TCP (Telephone Call)              UDP (Postcard)                   │
│  ═════════════════════             ═══════════════                  │
│                                                                      │
│  📞 "Hello? Can you hear me?"    📮 *drops postcard in mailbox*    │
│  📞 "Yes! Go ahead."             📮 *hopes it arrives*             │
│  📞 "I'll send you the data."    📮 *sends another postcard*       │
│  📞 "Got it. Send more."         📮 *no idea if first one arrived* │
│  📞 "Goodbye!"                   📮 *keeps sending*                │
│                                                                      │
│  ✅ Reliable                     ⚡ Fast                            │
│  ✅ Ordered                      ⚡ Low overhead                    │
│  ❌ Slower                       ❌ May lose data                   │
│  ❌ More overhead                ❌ No order guarantee              │
│                                                                      │
└──────────────────────────────────────────────────────────────────┘
```

---

## 3. Understanding Ports

A computer has a single network connection (like one front door), but many applications need to use the network simultaneously. **Ports** solve this — they're like **apartment numbers** inside a building.

```
┌──────────────────────────────────────────────────────────────────┐
│                   HOW PORTS WORK                                    │
├──────────────────────────────────────────────────────────────────┤
│                                                                      │
│   Your Computer (IP: 192.168.1.10)                                  │
│   ┌─────────────────────────────────────────────┐                   │
│   │                                              │                   │
│   │  Port 80  ──► Web Server (HTTP)             │                   │
│   │  Port 443 ──► Web Server (HTTPS)            │                   │
│   │  Port 22  ──► SSH Server                    │                   │
│   │  Port 3306──► MySQL Database                │                   │
│   │  Port 5000──► Your Java App!                │                   │
│   │                                              │                   │
│   │  All share the SAME network cable, but      │                   │
│   │  each gets its own dedicated "mailbox"       │                   │
│   │                                              │                   │
│   └─────────────────────────────────────────────┘                   │
│                                                                      │
│   Data arriving on the network contains:                            │
│   [Destination IP: 192.168.1.10] + [Destination Port: 5000]        │
│   → OS routes it to YOUR Java application!                          │
│                                                                      │
└──────────────────────────────────────────────────────────────────┘
```

### Port Number Ranges

| Range | Name | Usage |
|-------|------|-------|
| **0 – 1023** | Well-known ports | Reserved for system services (HTTP=80, FTP=21, SSH=22). **Your apps should NOT use these.** |
| **1024 – 49151** | Registered ports | Used by specific applications (MySQL=3306, PostgreSQL=5432). You can use these. |
| **49152 – 65535** | Dynamic/ephemeral ports | Assigned automatically by the OS for temporary client connections |

> **💡 Key Insight:** Ports are 16-bit numbers, so they range from 0 to 65,535. That's 65,536 possible ports per machine. An **IP address** identifies the machine; a **port number** identifies the application on that machine.

### How TCP and UDP Use Ports Differently

**TCP (Connection-based):**
```
Client creates Socket ──► Server's ServerSocket listening on port
                          Server calls accept() → gets new Socket
                          Both sides communicate through their Sockets
```

**UDP (Connectionless):**
```
Client creates DatagramPacket with destination port
Client sends packet ──► DatagramSocket on server receives it
Server extracts sender's address/port from the packet to reply
```

---

## 4. Java Networking Classes at a Glance

Java's `java.net` package gives you everything you need:

| Class | Protocol | Purpose |
|-------|----------|---------|
| `URL` | TCP (HTTP) | Represents a web address; easy reading from URLs |
| `URLConnection` | TCP (HTTP) | Full control over URL connections (read + write + headers) |
| `HttpURLConnection` | TCP (HTTP) | HTTP-specific features (GET, POST, status codes) |
| `Socket` | TCP | Client-side TCP endpoint |
| `ServerSocket` | TCP | Server-side TCP listener |
| `DatagramSocket` | UDP | Send/receive UDP datagrams |
| `DatagramPacket` | UDP | A single UDP packet (data + address) |
| `MulticastSocket` | UDP | Receive UDP broadcasts to a group |
| `InetAddress` | — | Represents an IP address |
| `NetworkInterface` | — | Represents a network adapter (NIC) |

> We cover `URL` and `URLConnection` in this post. `Socket`/`ServerSocket` come in **Part 2**, and datagrams + network interfaces in **Part 3**.

---

## 5. What Is a URL?

A **URL (Uniform Resource Locator)** is the address of a resource on the Internet. You use URLs every day — every link you click in a browser is a URL.

```
┌──────────────────────────────────────────────────────────────────┐
│                   ANATOMY OF A URL                                  │
├──────────────────────────────────────────────────────────────────┤
│                                                                      │
│  https://www.example.com:8080/docs/tutorial/index.html?q=java#top  │
│  ─┬───   ─────────┬────── ─┬── ──────────┬──────────── ──┬──── ┬─  │
│   │                │        │              │                │     │   │
│   │                │        │              │                │     │   │
│   Protocol         Host     Port           Path           Query  Ref │
│   (http/https)     Name     (optional)     to resource    string     │
│                                                         (optional)   │
│                                                                      │
│  Protocol: HOW to fetch (http, https, ftp, file)                    │
│  Host:     WHERE the server is                                      │
│  Port:     WHICH door on the server (default: 80/443)               │
│  Path:     WHAT resource on the server                              │
│  Query:    PARAMETERS for the resource                              │
│  Reference: BOOKMARK within the resource                            │
│                                                                      │
└──────────────────────────────────────────────────────────────────┘
```

### Creating URL Objects in Java

**Method 1: From a complete URL string (most common)**

```java
URL myURL = new URL("https://www.example.com/docs/page1.html");
```

**Method 2: Relative to a base URL**

When you know two pages share the same base path:

```java
URL base = new URL("https://www.example.com/docs/");
URL page1 = new URL(base, "page1.html");
URL page2 = new URL(base, "page2.html");

// page1 = https://www.example.com/docs/page1.html
// page2 = https://www.example.com/docs/page2.html
```

This is exactly how browsers resolve relative links in HTML! When you write `<a href="page2.html">`, the browser constructs the full URL relative to the current page.

**Method 3: From individual components**

```java
// Without port (uses protocol's default)
URL url1 = new URL("https", "www.example.com", "/docs/page1.html");

// With port
URL url2 = new URL("https", "www.example.com", 8080, "/docs/page1.html");
```

**Method 4: Handling special characters with URI**

URLs can't contain spaces or other special characters directly. Use `URI` to handle encoding:

```java
// ❌ WRONG — spaces are not valid in URLs
URL bad = new URL("https://example.com/hello world/");

// ✅ CORRECT — encode special characters
URL encoded = new URL("https://example.com/hello%20world/");

// ✅ EVEN BETTER — let URI handle encoding for you
URI uri = new URI("https", "example.com", "/hello world/", null);
URL url = uri.toURL();
// Result: https://example.com/hello%20world/
```

### Handling MalformedURLException

Every URL constructor can throw `MalformedURLException` if the URL is invalid:

```java
try {
    URL myURL = new URL("htp://invalid-protocol.com");  // Wrong protocol!
} catch (MalformedURLException e) {
    System.err.println("Invalid URL: " + e.getMessage());
}
```

> **⚠️ Important:** URL objects are **immutable** — once created, you cannot change their protocol, host, port, or path. You must create a new URL object instead.

---

## 6. Parsing a URL — Extracting Its Components

The `URL` class provides getter methods to extract every component:

```java
import java.net.URL;

public class ParseURLDemo {
    public static void main(String[] args) throws Exception {
        URL url = new URL(
            "https://www.example.com:8080/docs/books/tutorial/index.html"
            + "?name=networking#DOWNLOADING"
        );

        System.out.println("Protocol:  " + url.getProtocol());
        System.out.println("Authority: " + url.getAuthority());
        System.out.println("Host:      " + url.getHost());
        System.out.println("Port:      " + url.getPort());
        System.out.println("Path:      " + url.getPath());
        System.out.println("Query:     " + url.getQuery());
        System.out.println("File:      " + url.getFile());
        System.out.println("Reference: " + url.getRef());
    }
}
```

**Output:**
```
Protocol:  https
Authority: www.example.com:8080
Host:      www.example.com
Port:      8080
Path:      /docs/books/tutorial/index.html
Query:     name=networking
File:      /docs/books/tutorial/index.html?name=networking
Reference: DOWNLOADING
```

### URL Getter Methods Explained

| Method | Returns | Example |
|--------|---------|---------|
| `getProtocol()` | Protocol name | `"https"` |
| `getAuthority()` | Host + port | `"www.example.com:8080"` |
| `getHost()` | Hostname only | `"www.example.com"` |
| `getPort()` | Port number (or `-1` if not set) | `8080` |
| `getPath()` | Path to the resource | `"/docs/books/tutorial/index.html"` |
| `getQuery()` | Query string after `?` | `"name=networking"` |
| `getFile()` | Path + query combined | `"/docs/books/.../index.html?name=networking"` |
| `getRef()` | Fragment/anchor after `#` | `"DOWNLOADING"` |

> **💡 Pro Tip:** With these methods, you never need to manually parse URL strings with `split()` or regex. Just create a `URL` object and call the getters!

---

## 7. Reading Directly from a URL

The simplest way to fetch content from the Internet is `URL.openStream()`. It returns an `InputStream` that you can read like any file:

```java
import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.net.URL;

public class URLReaderDemo {
    public static void main(String[] args) throws Exception {
        URL oracle = new URL("https://www.example.com/");

        // openStream() → returns InputStream → wrap in readers
        BufferedReader reader = new BufferedReader(
            new InputStreamReader(oracle.openStream())
        );

        String line;
        while ((line = reader.readLine()) != null) {
            System.out.println(line);
        }
        reader.close();
    }
}
```

This prints the raw HTML of the webpage to your console!

```
┌──────────────────────────────────────────────────────────────────┐
│                HOW openStream() WORKS                               │
├──────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Your Code                                                          │
│     │                                                                │
│     │ url.openStream()                                              │
│     ▼                                                                │
│  ┌────────────┐    TCP Connection    ┌────────────┐                 │
│  │InputStream │ ◄────────────────── │ Web Server │                 │
│  └────────────┘    (HTTP GET)        └────────────┘                 │
│     │                                                                │
│     │ Wrap in BufferedReader                                        │
│     ▼                                                                │
│  readLine() → "<html>..."                                           │
│  readLine() → "<head>..."                                           │
│  readLine() → null (done)                                           │
│                                                                      │
│  Internally, openStream() is equivalent to:                         │
│  url.openConnection().getInputStream()                              │
│                                                                      │
└──────────────────────────────────────────────────────────────────┘
```

> **⚠️ Note:** If you're behind a proxy/firewall, the program might hang. You'd need to configure Java's proxy settings.

---

## 8. URLConnection — More Control Over Connections

`URL.openStream()` is a shortcut, but `URLConnection` gives you **full control** over the connection — you can set headers, read response codes, and even **write data** (POST requests).

### Reading with URLConnection

```java
import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.net.URL;
import java.net.URLConnection;

public class URLConnectionReaderDemo {
    public static void main(String[] args) throws Exception {
        URL url = new URL("https://www.example.com/");

        // Step 1: Open a connection object (not yet connected!)
        URLConnection connection = url.openConnection();

        // Step 2: Get the input stream (this triggers the actual connection)
        BufferedReader reader = new BufferedReader(
            new InputStreamReader(connection.getInputStream())
        );

        String line;
        while ((line = reader.readLine()) != null) {
            System.out.println(line);
        }
        reader.close();
    }
}
```

### The Connection Lifecycle

```
┌──────────────────────────────────────────────────────────────────┐
│              URLConnection LIFECYCLE                                │
├──────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. url.openConnection()                                            │
│     └─► Creates URLConnection object                                │
│     └─► NOT yet connected! Just a configuration object.             │
│                                                                      │
│  2. Configure (optional, BEFORE connecting):                        │
│     └─► connection.setRequestProperty("User-Agent", "MyApp/1.0")   │
│     └─► connection.setDoOutput(true)  // enable writing             │
│     └─► connection.setConnectTimeout(5000)  // 5-sec timeout        │
│                                                                      │
│  3. connection.connect()  OR  connection.getInputStream()           │
│     └─► NOW the TCP connection is actually established              │
│     └─► HTTP request is sent                                        │
│     └─► Response starts arriving                                    │
│                                                                      │
│  4. Read from / write to the connection                             │
│                                                                      │
│  5. Close streams                                                   │
│                                                                      │
│  ⚠️ A new URLConnection is created each time you call              │
│     openConnection(). They are NOT reused!                          │
│                                                                      │
└──────────────────────────────────────────────────────────────────┘
```

> **💡 Key Point:** You don't always need to call `connect()` explicitly. Methods like `getInputStream()` and `getOutputStream()` will connect automatically if needed.

### Writing to a URL (HTTP POST)

Many web applications use **HTTP POST** to send data from client to server (form submissions, API calls, etc.). Here's how to do it in Java:

```java
import java.io.*;
import java.net.*;

public class PostDemo {
    public static void main(String[] args) throws Exception {
        // The server endpoint that accepts POST data
        URL url = new URL("https://httpbin.org/post");

        // Step 1: Open connection and enable output (writing)
        URLConnection connection = url.openConnection();
        connection.setDoOutput(true);  // ← This enables POST mode!

        // Step 2: Write data to the server
        OutputStreamWriter writer = new OutputStreamWriter(
            connection.getOutputStream()
        );
        String postData = URLEncoder.encode("message", "UTF-8")
                        + "="
                        + URLEncoder.encode("Hello from Java!", "UTF-8");
        writer.write(postData);  // Sends: message=Hello+from+Java%21
        writer.close();

        // Step 3: Read the server's response
        BufferedReader reader = new BufferedReader(
            new InputStreamReader(connection.getInputStream())
        );
        String line;
        while ((line = reader.readLine()) != null) {
            System.out.println(line);
        }
        reader.close();
    }
}
```

### The Six-Step Process for Writing to a URL

```
┌──────────────────────────────────────────────────────────────────┐
│              POSTING DATA TO A URL                                   │
├──────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Step 1 → Create a URL object                                      │
│  Step 2 → Call openConnection() to get URLConnection               │
│  Step 3 → Call setDoOutput(true) to enable writing                 │
│  Step 4 → Get the output stream: connection.getOutputStream()      │
│  Step 5 → Write your data to the output stream                     │
│  Step 6 → Close the output stream (triggers the request)           │
│                                                                      │
│  Then read the response via connection.getInputStream()            │
│                                                                      │
│  ⚠️ Always URL-encode user data with URLEncoder.encode()!         │
│  Spaces become +, special chars become %XX                          │
│                                                                      │
└──────────────────────────────────────────────────────────────────┘
```

### URLEncoder — Why and How

When sending data in a URL or POST body, special characters must be **encoded**:

| Character | Encoded As | Why |
|-----------|-----------|-----|
| Space | `+` or `%20` | Spaces aren't valid in URLs |
| `&` | `%26` | `&` separates parameters |
| `=` | `%3D` | `=` separates key from value |
| `?` | `%3F` | `?` starts the query string |
| `#` | `%23` | `#` starts a fragment |

```java
// Encoding
String encoded = URLEncoder.encode("Hello World! A&B=C", "UTF-8");
// Result: "Hello+World%21+A%26B%3DC"

// Decoding
String decoded = URLDecoder.decode("Hello+World%21", "UTF-8");
// Result: "Hello World!"
```

---

## 9. HTTP Cookies — Remembering State in a Stateless Protocol

HTTP is inherently **stateless** — each request/response pair is independent. The server treats every request like a stranger. But how does Amazon remember your shopping cart? How does a website keep you logged in? The answer is **cookies**.

```
┌──────────────────────────────────────────────────────────────────┐
│              HOW COOKIES WORK                                       │
├──────────────────────────────────────────────────────────────────┤
│                                                                      │
│  WITHOUT Cookies:                                                   │
│                                                                      │
│  Request 1: "Show me shoes"        Server: "Here are shoes"        │
│  Request 2: "Add shoe #42 to cart" Server: "What cart? Who are you?"│
│                                                                      │
│  ─────────────────────────────────────────────────────────         │
│                                                                      │
│  WITH Cookies:                                                      │
│                                                                      │
│  Request 1: "Show me shoes"                                        │
│  Response:  "Here are shoes" + Set-Cookie: session=abc123          │
│                                                                      │
│  Request 2: "Add shoe #42 to cart" + Cookie: session=abc123        │
│  Response:  "Added to YOUR cart!" (server recognizes abc123)       │
│                                                                      │
│  The cookie acts like a name tag — the server sticks it on you     │
│  during your first visit, and you wear it on every return visit!   │
│                                                                      │
└──────────────────────────────────────────────────────────────────┘
```

A **cookie** is a small piece of data sent by a server and stored in the client (browser/application). There are two types:

| Type | Lifetime | Use Case |
|------|----------|----------|
| **Session cookie** | Until browser/app is closed | Login sessions, temporary preferences |
| **Persistent cookie** | Weeks, months, or even years | "Remember me" login, user preferences |

### CookieHandler — The Callback Mechanism

Java implements HTTP state management through `java.net.CookieHandler`. This class provides a **callback mechanism** — when Java's HTTP protocol handler (used by `URL`, `URLConnection`) processes requests and responses, it calls back to the `CookieHandler` to manage cookies.

```
┌──────────────────────────────────────────────────────────────────┐
│              CookieHandler CALLBACK FLOW                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Your Code                                                          │
│     │                                                                │
│     │ url.openConnection()                                          │
│     ▼                                                                │
│  HTTP Protocol Handler                                              │
│     │                                                                │
│     ├──► SENDING request?                                           │
│     │    Call CookieHandler.get(uri, requestHeaders)               │
│     │    → Returns stored cookies to attach to the request         │
│     │                                                                │
│     ├──► RECEIVED response?                                         │
│     │    Call CookieHandler.put(uri, responseHeaders)              │
│     │    → Extracts Set-Cookie headers and stores them             │
│     │                                                                │
│     └──► All automatic — YOUR code doesn't need to do anything!    │
│                                                                      │
└──────────────────────────────────────────────────────────────────┘
```

`CookieHandler` is an **abstract class** with two key method pairs:

| Method | Purpose |
|--------|---------|
| `CookieHandler.getDefault()` | Gets the currently installed system-wide handler |
| `CookieHandler.setDefault(handler)` | Installs your own handler system-wide |
| `get(uri, requestHeaders)` | Called before sending a request — returns cookies to attach |
| `put(uri, responseHeaders)` | Called after receiving a response — stores cookies from headers |

> **⚠️ Important:** No default `CookieHandler` is installed in standalone Java applications! You must set one up manually. Java Web Start and Java Plug-in have their own default handler.

### CookieManager — The Ready-Made Implementation

For most users, `java.net.CookieManager` provides everything you need. It's a concrete implementation of `CookieHandler` that separates cookie **storage** from **acceptance policy**:

```
┌──────────────────────────────────────────────────────────────────┐
│              CookieManager ARCHITECTURE                             │
├──────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌────────────────────────────────────────────┐                     │
│  │              CookieManager                   │                     │
│  │                                              │                     │
│  │  ┌──────────────┐    ┌──────────────────┐   │                     │
│  │  │ CookiePolicy  │    │   CookieStore     │   │                     │
│  │  │               │    │                    │   │                     │
│  │  │ "Should I     │    │ "Where do I keep  │   │                     │
│  │  │  accept this  │    │  the cookies?"    │   │                     │
│  │  │  cookie?"     │    │                    │   │                     │
│  │  └──────────────┘    └──────────────────┘   │                     │
│  └────────────────────────────────────────────┘                     │
│                                                                      │
│  CookiePolicy decides: Accept or reject?                            │
│  CookieStore handles:  Store, retrieve, delete                      │
│                                                                      │
└──────────────────────────────────────────────────────────────────┘
```

**Setting up cookie management in one line:**

```java
// Create and install a default CookieManager
java.net.CookieManager cm = new java.net.CookieManager();
java.net.CookieHandler.setDefault(cm);

// That's it! From now on, ALL URL/URLConnection requests
// will automatically manage cookies!
```

The default `CookieManager` uses:
- **In-memory `CookieStore`** — cookies are lost when the JVM exits
- **`CookiePolicy.ACCEPT_ORIGINAL_SERVER`** — only accepts cookies from the server you connected to (not third-party cookies)

### CookiePolicy — Who Gets In?

`CookiePolicy` decides which cookies to accept. Java provides three built-in policies:

| Policy | Behavior |
|--------|----------|
| `CookiePolicy.ACCEPT_ORIGINAL_SERVER` | ✅ Accepts cookies only from the server you're connecting to (default — most secure) |
| `CookiePolicy.ACCEPT_ALL` | Accepts cookies from any server (including third-party trackers) |
| `CookiePolicy.ACCEPT_NONE` | ❌ Rejects all cookies (privacy mode) |

**Custom Cookie Policy — Blacklist Example:**

What if you want to accept cookies from most servers but block specific domains? Implement the `CookiePolicy` interface:

```java
import java.net.*;

public class BlacklistCookiePolicy implements CookiePolicy {
    private String[] blacklist;

    public BlacklistCookiePolicy(String[] blacklist) {
        this.blacklist = blacklist;
    }

    @Override
    public boolean shouldAccept(URI uri, HttpCookie cookie) {
        // Resolve the hostname
        String host;
        try {
            host = InetAddress.getByName(uri.getHost())
                              .getCanonicalHostName();
        } catch (UnknownHostException e) {
            host = uri.getHost();
        }

        // Check against blacklist
        for (String blocked : blacklist) {
            if (HttpCookie.domainMatches(blocked, host)) {
                return false;  // ❌ Blocked!
            }
        }

        // Fall back to original-server policy for everything else
        return CookiePolicy.ACCEPT_ORIGINAL_SERVER
                           .shouldAccept(uri, cookie);
    }
}
```

**Using the custom policy:**

```java
String[] blockedDomains = { ".ads-tracker.com", ".example.com" };

CookieManager cm = new CookieManager(
    null,  // use default CookieStore
    new BlacklistCookiePolicy(blockedDomains)
);
CookieHandler.setDefault(cm);

// Now cookies from *.ads-tracker.com and *.example.com are rejected
// but cookies from other servers are accepted normally
```

| Domain | Cookie Accepted? |
|--------|-----------------|
| `host.example.com` | ❌ No — matches `.example.com` |
| `deep.sub.example.com` | ❌ No — matches `.example.com` |
| `example.com` | ✅ Yes — doesn't match (no leading dot) |
| `example.org` | ✅ Yes — not in blacklist |
| `ads-tracker.com` | ✅ Yes — doesn't match `.ads-tracker.com` |

### CookieStore — Where Cookies Live

`CookieStore` is the interface that manages cookie **storage**. The default implementation keeps cookies **in memory** — they vanish when the JVM shuts down.

For applications that need cookies to survive restarts (like a desktop email client), you can build a **persistent CookieStore**:

```java
import java.net.*;
import java.util.*;

public class PersistentCookieStore implements CookieStore, Runnable {
    private CookieStore store;  // Delegate to the default in-memory store

    public PersistentCookieStore() {
        // Get the default in-memory cookie store
        store = new CookieManager().getCookieStore();

        // TODO: Read saved cookies from a file/database
        // and add them to 'store' using store.add(uri, cookie)

        // Register a shutdown hook to save cookies on exit
        Runtime.getRuntime().addShutdownHook(new Thread(this));
    }

    @Override
    public void run() {
        // Called on JVM shutdown
        // TODO: Write cookies from store to persistent storage
        // for (HttpCookie cookie : store.getCookies()) {
        //     saveToDisk(cookie);
        // }
    }

    // ═══════════════════════════════════════════
    // All CookieStore methods delegate to the default store
    // ═══════════════════════════════════════════

    @Override
    public void add(URI uri, HttpCookie cookie) {
        store.add(uri, cookie);
    }

    @Override
    public List<HttpCookie> get(URI uri) {
        return store.get(uri);
    }

    @Override
    public List<HttpCookie> getCookies() {
        return store.getCookies();
    }

    @Override
    public List<URI> getURIs() {
        return store.getURIs();
    }

    @Override
    public boolean remove(URI uri, HttpCookie cookie) {
        return store.remove(uri, cookie);
    }

    @Override
    public boolean removeAll() {
        return store.removeAll();
    }
}
```

**Using it:**

```java
CookieManager cm = new CookieManager(
    new PersistentCookieStore(),   // custom persistent store
    CookiePolicy.ACCEPT_ORIGINAL_SERVER
);
CookieHandler.setDefault(cm);
```

```
┌──────────────────────────────────────────────────────────────────┐
│              PERSISTENT CookieStore LIFECYCLE                       │
├──────────────────────────────────────────────────────────────────┤
│                                                                      │
│  JVM Starts                                                         │
│     │                                                                │
│     ▼                                                                │
│  PersistentCookieStore()                                            │
│     ├─► Load saved cookies from disk → add to in-memory store      │
│     └─► Register shutdown hook                                      │
│                                                                      │
│  During Runtime                                                     │
│     ├─► add() → stores cookies in memory (fast!)                   │
│     └─► get() → retrieves cookies from memory                      │
│                                                                      │
│  JVM Shuts Down                                                     │
│     └─► Shutdown hook fires → save all cookies to disk             │
│                                                                      │
│  Next JVM Start → Cookies are loaded back! ♻️                     │
│                                                                      │
└──────────────────────────────────────────────────────────────────┘
```

> **💡 Design Insight:** Notice the pattern — `PersistentCookieStore` **delegates** to the default in-memory store for all operations and only adds persistence on top. This leverages Java's existing implementation and keeps your code minimal!

---

## 10. openStream() vs openConnection() — When to Use Which

| Feature | `openStream()` | `openConnection()` |
|---------|---------------|-------------------|
| **Simplicity** | ✅ One-liner | Requires more setup |
| **Read data** | ✅ Yes | ✅ Yes |
| **Write data (POST)** | ❌ No | ✅ Yes |
| **Set headers** | ❌ No | ✅ Yes |
| **Set timeouts** | ❌ No | ✅ Yes |
| **Get response code** | ❌ No | ✅ Yes (via `HttpURLConnection`) |
| **Use case** | Quick reads | Full control |

**Rule of thumb:**
- Need to quickly **read** a URL? → `openStream()`
- Need to **POST data**, **set headers**, or **check status codes**? → `openConnection()`

---

## 11. Practical Example: Building a Simple Web Scraper

Let's combine everything we've learned into a practical example:

```java
import java.io.*;
import java.net.*;

public class SimpleWebScraper {
    public static void main(String[] args) {
        String targetUrl = "https://www.example.com/";

        try {
            // Create URL and open connection
            URL url = new URL(targetUrl);
            HttpURLConnection connection = (HttpURLConnection) url.openConnection();

            // Configure the request
            connection.setRequestMethod("GET");
            connection.setRequestProperty("User-Agent", "JavaWebScraper/1.0");
            connection.setConnectTimeout(5000);  // 5-second connect timeout
            connection.setReadTimeout(10000);     // 10-second read timeout

            // Check the response code
            int responseCode = connection.getResponseCode();
            System.out.println("Response Code: " + responseCode);
            System.out.println("Content Type:  " + connection.getContentType());
            System.out.println("Content Length: " + connection.getContentLength());
            System.out.println("─────────────────────────────────────");

            if (responseCode == HttpURLConnection.HTTP_OK) {  // 200
                // Read the content
                BufferedReader reader = new BufferedReader(
                    new InputStreamReader(connection.getInputStream())
                );

                StringBuilder content = new StringBuilder();
                String line;
                while ((line = reader.readLine()) != null) {
                    content.append(line).append("\n");
                }
                reader.close();

                System.out.println("Page content (" + content.length() + " chars):");
                // Print first 500 characters
                System.out.println(content.substring(0, Math.min(500, content.length())));
            } else {
                System.out.println("Request failed with code: " + responseCode);
            }

            connection.disconnect();

        } catch (MalformedURLException e) {
            System.err.println("Invalid URL: " + e.getMessage());
        } catch (SocketTimeoutException e) {
            System.err.println("Connection timed out: " + e.getMessage());
        } catch (IOException e) {
            System.err.println("I/O error: " + e.getMessage());
        }
    }
}
```

### Common HTTP Response Codes

| Code | Constant | Meaning |
|------|----------|---------|
| 200 | `HTTP_OK` | Success |
| 301 | `HTTP_MOVED_PERM` | Permanently redirected |
| 404 | `HTTP_NOT_FOUND` | Resource not found |
| 500 | `HTTP_INTERNAL_ERROR` | Server error |
| 403 | `HTTP_FORBIDDEN` | Access denied |

---

## 12. Summary and What's Next

Let's review what we covered in Part 1:

| Topic | Key Takeaway |
|-------|-------------|
| **TCP** | Reliable, ordered, connection-based — like a phone call |
| **UDP** | Fast, unreliable, connectionless — like a postcard |
| **Ports** | 16-bit numbers (0–65535) that identify which application receives data |
| **URL class** | Represents a web address; can parse, create, and read from URLs |
| **openStream()** | Simplest way to read content from a URL |
| **URLConnection** | Full control — read, write, set headers, timeouts |
| **URLEncoder** | Encodes special characters for safe URL transmission |
| **Cookies** | CookieManager + CookiePolicy + CookieStore for HTTP state management |

```
┌──────────────────────────────────────────────────────────────────┐
│                    SERIES ROADMAP                                    │
├──────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ✅ Part 1: Networking Basics, URLs, URLConnections (YOU ARE HERE)  │
│                                                                      │
│  📋 Part 2: Socket Programming                                     │
│     • What is a Socket?                                             │
│     • Reading from and Writing to Sockets                           │
│     • Building a Client-Server application                          │
│     • Supporting Multiple Clients with threads                      │
│                                                                      │
│  📋 Part 3: Datagrams and Network Interfaces                       │
│     • UDP with DatagramSocket and DatagramPacket                    │
│     • Broadcasting with MulticastSocket                             │
│     • Querying Network Interfaces                                   │
│                                                                      │
└──────────────────────────────────────────────────────────────────┘
```

> **💡 Up Next:** In Part 2, we dive into the heart of network programming — **Sockets**. You'll build a complete client-server application (including the famous Knock Knock joke server!) and learn how to handle multiple clients simultaneously. See you there! 🚀
