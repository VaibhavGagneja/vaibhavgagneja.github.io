---
title: "Java Networking Part 2: Socket Programming — Building Client-Server Applications"
description: Master Java Socket and ServerSocket classes, build an Echo server, a Knock Knock joke server, and learn multi-client handling with threads
author: Vaibhav Gagneja
date: 2026-02-17 12:00:00 +0530
categories: [Development, Java]
tags: [java, networking, sockets, serversocket, tcp, multithreading, client-server]
toc: true
image:
  path: https://images.unsplash.com/photo-1558494949-ef010cbdcc31
---

In [Part 1](/posts/java-networking-urls-basics/) we covered the fundamentals — TCP vs UDP, ports, and Java's URL classes. Now it's time to get our hands dirty with **Socket programming** — the heart of network communication in Java.

By the end of this post, you'll build a complete **client-server application** from scratch, understand how servers handle **multiple clients** using threads, and see a real working example (the classic Knock Knock joke server).

---

## 1. What Is a Socket?

A **socket** is one end of a two-way communication link between two programs running on the network. Think of it like a telephone handset — you need one at each end to have a conversation.

```
┌──────────────────────────────────────────────────────────────────┐
│                    WHAT IS A SOCKET?                                 │
├──────────────────────────────────────────────────────────────────┤
│                                                                      │
│  A socket = IP Address + Port Number                                │
│                                                                      │
│  Client Machine                     Server Machine                   │
│  ┌─────────────────┐                ┌─────────────────┐             │
│  │ IP: 192.168.1.5 │                │ IP: 10.0.0.1    │             │
│  │                  │                │                  │             │
│  │  Socket:         │   TCP Link     │  Socket:         │             │
│  │  192.168.1.5     │◄═════════════►│  10.0.0.1        │             │
│  │  :49152          │                │  :5000           │             │
│  │  (random port)   │                │  (fixed port)    │             │
│  │                  │                │                  │             │
│  └─────────────────┘                └─────────────────┘             │
│                                                                      │
│  Together, the TWO sockets uniquely identify this connection:       │
│  (192.168.1.5:49152) ←→ (10.0.0.1:5000)                           │
│                                                                      │
│  This is why a server can handle MANY clients on the SAME port —   │
│  each connection has a unique pair of endpoints!                    │
│                                                                      │
└──────────────────────────────────────────────────────────────────┘
```

### How a Connection Is Established

```
┌──────────────────────────────────────────────────────────────────┐
│            TCP CONNECTION HANDSHAKE                                  │
├──────────────────────────────────────────────────────────────────┤
│                                                                      │
│  SERVER                                     CLIENT                  │
│  ══════                                     ══════                  │
│                                                                      │
│  1. ServerSocket ss =                                               │
│     new ServerSocket(5000);                                         │
│     // Starts listening on port 5000                                │
│                                                                      │
│  2. Socket client = ss.accept();            Socket s =              │
│     // Blocks, waiting...         ◄──────  new Socket("10.0.0.1",  │
│                                              5000);                  │
│                                             // Sends connect request │
│                                                                      │
│  3. accept() returns a NEW Socket  ──────►  Socket created!         │
│     bound to server:5000                    Connected!               │
│     with remote = client:49152                                      │
│                                                                      │
│  4. ServerSocket STILL listening            Client uses Socket      │
│     for MORE connections!                   to read/write data      │
│                                                                      │
└──────────────────────────────────────────────────────────────────┘
```

### Key Points to Remember

| Concept | Detail |
|---------|--------|
| **Server** uses `ServerSocket` | Listens on a fixed port, calls `accept()` to wait for clients |
| **Client** uses `Socket` | Connects to the server's host + port |
| `accept()` returns a **new** `Socket` | The original `ServerSocket` keeps listening for more clients |
| **Both sides** use `Socket` to communicate | Once connected, client and server are peers |
| Connection is **uniquely identified** | By the combination of both endpoints (IP:port pairs) |

---

## 2. Java Classes for Socket Programming

| Class | Side | Purpose |
|-------|------|---------|
| `ServerSocket` | Server | Listens on a port and accepts incoming connections |
| `Socket` | Both | Represents one end of a TCP connection; used to read/write data |
| `InetAddress` | Both | Represents an IP address |

Both `ServerSocket` and `Socket` live in the `java.net` package.

### ServerSocket — The Receptionist

```java
// Create a server that listens on port 5000
ServerSocket serverSocket = new ServerSocket(5000);

// Wait for a client to connect (BLOCKS until someone connects)
Socket clientSocket = serverSocket.accept();

// Now use clientSocket to talk to the client
// serverSocket is still running — ready for more clients!
```

### Socket — The Communication Channel

```java
// Client connects to server at 192.168.1.10, port 5000
Socket socket = new Socket("192.168.1.10", 5000);

// Get streams for reading and writing
InputStream  in  = socket.getInputStream();   // Read FROM server
OutputStream out = socket.getOutputStream();  // Write TO server
```

---

## 3. Reading from and Writing to a Socket — The Echo Client

The simplest possible client-server interaction: the **Echo** pattern. Client sends text, server sends it right back. Like shouting into a canyon and hearing your own voice.

### The Echo Server

```java
import java.io.*;
import java.net.*;

public class EchoServer {
    public static void main(String[] args) throws IOException {
        int port = 7;  // Traditional echo port

        // Try-with-resources auto-closes everything
        try (
            ServerSocket serverSocket = new ServerSocket(port);
            Socket clientSocket = serverSocket.accept();
            PrintWriter out = new PrintWriter(
                clientSocket.getOutputStream(), true);
            BufferedReader in = new BufferedReader(
                new InputStreamReader(clientSocket.getInputStream()))
        ) {
            System.out.println("Client connected!");

            String inputLine;
            while ((inputLine = in.readLine()) != null) {
                System.out.println("Received: " + inputLine);
                out.println(inputLine);  // Echo it back!
            }
        }
    }
}
```

### The Echo Client

```java
import java.io.*;
import java.net.*;

public class EchoClient {
    public static void main(String[] args) throws IOException {
        String hostName = args[0];     // Server hostname
        int portNumber = Integer.parseInt(args[1]);  // Server port

        try (
            // 1. Create socket connection to server
            Socket echoSocket = new Socket(hostName, portNumber);

            // 2. Open writer — sends data TO server
            PrintWriter out = new PrintWriter(
                echoSocket.getOutputStream(), true);

            // 3. Open reader — reads data FROM server
            BufferedReader in = new BufferedReader(
                new InputStreamReader(echoSocket.getInputStream()));

            // 4. Reader for keyboard input
            BufferedReader stdIn = new BufferedReader(
                new InputStreamReader(System.in))
        ) {
            String userInput;
            while ((userInput = stdIn.readLine()) != null) {
                out.println(userInput);                         // Send to server
                System.out.println("echo: " + in.readLine());  // Print server reply
            }

        } catch (UnknownHostException e) {
            System.err.println("Unknown host: " + hostName);
        } catch (IOException e) {
            System.err.println("Connection error: " + e.getMessage());
        }
    }
}
```

### How the Echo Client Works — Step by Step

```
┌──────────────────────────────────────────────────────────────────┐
│                HOW THE ECHO WORKS                                   │
├──────────────────────────────────────────────────────────────────┤
│                                                                      │
│  EchoClient                               EchoServer                │
│  ══════════                               ══════════                │
│                                                                      │
│  User types: "Hello"                                                │
│       │                                                              │
│       ▼                                                              │
│  out.println("Hello")  ──────────────►  in.readLine() → "Hello"    │
│                                              │                       │
│                                              ▼                       │
│  in.readLine() → "Hello" ◄──────────── out.println("Hello")       │
│       │                                                              │
│       ▼                                                              │
│  Prints: "echo: Hello"                                              │
│                                                                      │
│  The user then types another line... cycle repeats!                 │
│                                                                      │
│  ⚠️ readLine() BLOCKS until the other side sends data.             │
│  That's why this simple approach works — it's strictly turn-based.  │
│                                                                      │
└──────────────────────────────────────────────────────────────────┘
```

### The Five-Step Socket Pattern

Every client program follows the same pattern:

| Step | Code | Purpose |
|------|------|---------|
| 1 | `new Socket(host, port)` | Open a connection |
| 2 | `socket.getOutputStream()` | Get the output stream (to send) |
| 3 | `socket.getInputStream()` | Get the input stream (to receive) |
| 4 | Read/write according to protocol | Communicate |
| 5 | Close streams and socket | Clean up |

> **💡 Pro Tip:** Use `try-with-resources` (as shown above) and Java handles Step 5 automatically! Resources are closed **in reverse order** of creation — streams first, then the socket itself.

---

## 4. Building a Real Application: The Knock Knock Joke Server

The Echo server is simple but boring. Let's build something with a **real protocol** — a server that tells Knock Knock jokes! This demonstrates how client and server agree on a **conversation format**.

### The Protocol

```
┌──────────────────────────────────────────────────────────────────┐
│              KNOCK KNOCK PROTOCOL                                    │
├──────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Server                          Client                             │
│  ══════                          ══════                             │
│                                                                      │
│  "Knock! Knock!"  ────────────►                                    │
│                    ◄────────────  "Who's there?"                    │
│  "Turnip"          ────────────►                                    │
│                    ◄────────────  "Turnip who?"                     │
│  "Turnip the heat, ────────────►                                    │
│   it's cold in here!                                                │
│   Want another? (y/n)"                                              │
│                    ◄────────────  "y" or "n"                        │
│                                                                      │
│  If "y" → start over with new joke                                 │
│  If "n" → "Bye." → connection closed                               │
│                                                                      │
│  Wrong response? → "You're supposed to say 'Who's there?'!"       │
│                     "Try again. Knock! Knock!"                      │
│                                                                      │
└──────────────────────────────────────────────────────────────────┘
```

### The Protocol Class — The "Brain" of the Server

This class tracks the **state** of the conversation and generates the correct response:

```java
import java.net.*;
import java.io.*;

public class KnockKnockProtocol {
    private static final int WAITING = 0;
    private static final int SENT_KNOCK = 1;
    private static final int SENT_CLUE = 2;
    private static final int ANOTHER = 3;

    private static final int NUM_JOKES = 5;

    private int state = WAITING;
    private int currentJoke = 0;

    // Joke clues and answers
    private String[] clues = {
        "Turnip", "Little Old Lady", "Atch", "Who", "Who"
    };
    private String[] answers = {
        "Turnip the heat, it's cold in here!",
        "I didn't know you could yodel!",
        "Bless you!",
        "Is there an owl in here?",
        "Is there an echo in here?"
    };

    public String processInput(String input) {
        String output = null;

        if (state == WAITING) {
            output = "Knock! Knock!";
            state = SENT_KNOCK;

        } else if (state == SENT_KNOCK) {
            if (input.equalsIgnoreCase("Who's there?")) {
                output = clues[currentJoke];
                state = SENT_CLUE;
            } else {
                output = "You're supposed to say \"Who's there?\"! "
                       + "Try again. Knock! Knock!";
            }

        } else if (state == SENT_CLUE) {
            if (input.equalsIgnoreCase(clues[currentJoke] + " who?")) {
                output = answers[currentJoke] + " Want another? (y/n)";
                state = ANOTHER;
            } else {
                output = "You're supposed to say \""
                       + clues[currentJoke] + " who?\""
                       + "! Try again. Knock! Knock!";
                state = SENT_KNOCK;
            }

        } else if (state == ANOTHER) {
            if (input.equalsIgnoreCase("y")) {
                output = "Knock! Knock!";
                currentJoke = (currentJoke + 1) % NUM_JOKES;
                state = SENT_KNOCK;
            } else {
                output = "Bye.";
                state = WAITING;
            }
        }
        return output;
    }
}
```

### The Knock Knock Server

```java
import java.net.*;
import java.io.*;

public class KnockKnockServer {
    public static void main(String[] args) throws IOException {
        int portNumber = Integer.parseInt(args[0]);

        try (
            ServerSocket serverSocket = new ServerSocket(portNumber);
            Socket clientSocket = serverSocket.accept();
            PrintWriter out = new PrintWriter(
                clientSocket.getOutputStream(), true);
            BufferedReader in = new BufferedReader(
                new InputStreamReader(clientSocket.getInputStream()))
        ) {
            String inputLine, outputLine;

            // Create the protocol handler
            KnockKnockProtocol protocol = new KnockKnockProtocol();

            // Server speaks FIRST — initiate with null input
            outputLine = protocol.processInput(null);
            out.println(outputLine);

            // Then read client responses and reply
            while ((inputLine = in.readLine()) != null) {
                outputLine = protocol.processInput(inputLine);
                out.println(outputLine);
                if (outputLine.equals("Bye."))
                    break;
            }
        }
    }
}
```

### The Knock Knock Client

```java
import java.io.*;
import java.net.*;

public class KnockKnockClient {
    public static void main(String[] args) throws IOException {
        String hostName = args[0];
        int portNumber = Integer.parseInt(args[1]);

        try (
            Socket socket = new Socket(hostName, portNumber);
            PrintWriter out = new PrintWriter(
                socket.getOutputStream(), true);
            BufferedReader in = new BufferedReader(
                new InputStreamReader(socket.getInputStream()));
            BufferedReader stdIn = new BufferedReader(
                new InputStreamReader(System.in))
        ) {
            String fromServer;
            String fromUser;

            while ((fromServer = in.readLine()) != null) {
                System.out.println("Server: " + fromServer);

                if (fromServer.equals("Bye."))
                    break;

                fromUser = stdIn.readLine();
                if (fromUser != null) {
                    System.out.println("Client: " + fromUser);
                    out.println(fromUser);
                }
            }

        } catch (UnknownHostException e) {
            System.err.println("Unknown host: " + hostName);
        } catch (IOException e) {
            System.err.println("I/O Error: " + e.getMessage());
        }
    }
}
```

### Running It!

```bash
# Terminal 1 — Start the server
java KnockKnockServer 4444

# Terminal 2 — Start the client
java KnockKnockClient localhost 4444
```

**Sample session:**
```
Server: Knock! Knock!
Who's there?
Client: Who's there?
Server: Turnip
Turnip who?
Client: Turnip who?
Server: Turnip the heat, it's cold in here! Want another? (y/n)
n
Client: n
Server: Bye.
```

> **💡 Design Insight:** Notice how the **protocol logic** is completely separated from the **networking code**? The `KnockKnockProtocol` class knows nothing about sockets — it just processes strings. This separation of concerns makes the code clean and testable.

---

## 5. Understanding the Code — Key Concepts Explained

### PrintWriter vs DataOutputStream

In the Echo and Knock Knock examples, we used `PrintWriter` and `BufferedReader`. But you can also use `DataOutputStream`/`DataInputStream` (as we saw in Part 1 with `readUTF`/`writeUTF`):

| Approach | Classes | Read/Write | Best For |
|----------|---------|------------|----------|
| **Character streams** | `PrintWriter` + `BufferedReader` | `println()` / `readLine()` | Text-based protocols (HTTP, chat) |
| **Data streams** | `DataOutputStream` + `DataInputStream` | `writeUTF()` / `readUTF()` | Structured data (length-prefixed strings) |
| **Byte streams** | `OutputStream` + `InputStream` | `write(byte[])` / `read(byte[])` | Binary data (files, images) |

### try-with-resources — Automatic Cleanup

```java
try (
    ServerSocket server = new ServerSocket(5000);    // Created first
    Socket client = server.accept();                  // Created second
    PrintWriter out = new PrintWriter(...);            // Created third
    BufferedReader in = new BufferedReader(...)        // Created last
) {
    // Use the resources...
}
// ALL resources are AUTOMATICALLY closed here!
// Closed in REVERSE order: in → out → client → server
// This is correct because streams should close before their socket.
```

> **⚠️ Without try-with-resources**, you'd need a `finally` block with manual closing. This is error-prone because `close()` itself can throw exceptions!

### Blocking I/O

Both `readLine()` and `accept()` are **blocking** calls:

```
┌──────────────────────────────────────────────────────────────────┐
│              BLOCKING CALLS IN SOCKET PROGRAMMING                   │
├──────────────────────────────────────────────────────────────────┤
│                                                                      │
│  serverSocket.accept()   → Blocks until a client connects           │
│  in.readLine()           → Blocks until the other side sends a line │
│  socket.getInputStream() → Blocks until data is available           │
│                                                                      │
│  While blocked, the thread does NOTHING — it just waits.            │
│  This is why multi-client servers need THREADS!                     │
│                                                                      │
│  ⚠️ Problem: If both sides are calling readLine() at the same     │
│  time, DEADLOCK! Both are waiting for the other to speak first.    │
│                                                                      │
│  Solution: Clear protocol rules (who speaks first, who responds)   │
│            OR separate reading thread (covered in threading post)   │
│                                                                      │
└──────────────────────────────────────────────────────────────────┘
```

---

## 6. The Big Problem: Only One Client at a Time

Our Knock Knock server has a serious limitation — it can only serve **one client**. If you try to connect a second client while the first is still chatting, the second client just **hangs**. It won't even connect until the first client disconnects.

Why? Because `accept()` is only called **once**, and then the server is busy in the `while` loop talking to that single client.

```
┌──────────────────────────────────────────────────────────────────┐
│              THE SINGLE-CLIENT PROBLEM                              │
├──────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Server Timeline:                                                    │
│  ┌──────────┐  ┌──────────────────────────────────┐                │
│  │ accept() │→ │ Talking to Client 1 (while loop) │                │
│  └──────────┘  └──────────────────────────────────┘                │
│                                                                      │
│  Client 2 tries to connect... but nobody is calling accept()!       │
│  Client 2 waits... and waits... and waits...                        │
│  Connection request sits in a QUEUE until someone accepts it.       │
│                                                                      │
│  Only after Client 1 disconnects → server loop ends →              │
│  program exits → Client 2 gets "Connection refused"                 │
│                                                                      │
│  ❌ This is unacceptable for real applications!                     │
│                                                                      │
└──────────────────────────────────────────────────────────────────┘
```

---

## 7. Supporting Multiple Clients with Threads

The solution is **multithreading**: for each client that connects, we create a **separate thread** to handle the conversation. The main thread goes right back to listening for more clients.

```
┌──────────────────────────────────────────────────────────────────┐
│              MULTI-CLIENT ARCHITECTURE                              │
├──────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Main Thread (Listener)                                             │
│  ═══════════════════════                                            │
│  while (true) {                                                     │
│      Socket client = accept();  ──┐                                 │
│      // immediately goes back  ◄──┘                                 │
│      // to accept() for more       └──► New Thread                  │
│  }                                      handles this client         │
│                                                                      │
│  ┌──────────────────────────────────────────────┐                   │
│  │               RUNNING THREADS                  │                   │
│  │                                                │                   │
│  │  Main Thread ──► accept(), accept(), accept()  │                   │
│  │  Thread-1    ──► Talking to Client 1           │                   │
│  │  Thread-2    ──► Talking to Client 2           │                   │
│  │  Thread-3    ──► Talking to Client 3           │                   │
│  │                                                │                   │
│  └──────────────────────────────────────────────┘                   │
│                                                                      │
└──────────────────────────────────────────────────────────────────┘
```

### Multi-Client Knock Knock Server

**The server thread class** (handles one client):

```java
import java.net.*;
import java.io.*;

public class KKMultiServerThread extends Thread {
    private Socket socket;

    public KKMultiServerThread(Socket socket) {
        super("KKMultiServerThread");
        this.socket = socket;
    }

    @Override
    public void run() {
        try (
            PrintWriter out = new PrintWriter(
                socket.getOutputStream(), true);
            BufferedReader in = new BufferedReader(
                new InputStreamReader(socket.getInputStream()))
        ) {
            String inputLine, outputLine;
            KnockKnockProtocol protocol = new KnockKnockProtocol();

            // Initiate conversation
            outputLine = protocol.processInput(null);
            out.println(outputLine);

            // Handle conversation
            while ((inputLine = in.readLine()) != null) {
                outputLine = protocol.processInput(inputLine);
                out.println(outputLine);
                if (outputLine.equals("Bye."))
                    break;
            }

            socket.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

**The multi-client server** (accepts connections in a loop):

```java
import java.net.*;
import java.io.*;

public class KKMultiServer {
    public static void main(String[] args) throws IOException {
        int portNumber = Integer.parseInt(args[0]);
        boolean listening = true;

        try (ServerSocket serverSocket = new ServerSocket(portNumber)) {
            System.out.println("Server started on port " + portNumber);

            while (listening) {
                // accept() blocks until a client connects
                Socket clientSocket = serverSocket.accept();
                System.out.println("New client connected: "
                    + clientSocket.getInetAddress());

                // Create a new thread for this client and start it
                new KKMultiServerThread(clientSocket).start();

                // Loop immediately goes back to accept()!
            }
        }
    }
}
```

### How It All Works Together

```
┌──────────────────────────────────────────────────────────────────┐
│              MULTI-CLIENT FLOW                                      │
├──────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Time ──►                                                           │
│                                                                      │
│  Main:  accept()·····accept()·····accept()·····accept()·····       │
│                │              │              │                        │
│                ▼              ▼              ▼                        │
│  Thread1: ▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇ (Client 1 — Knock Knock jokes)         │
│  Thread2:           ▇▇▇▇▇▇▇▇▇▇▇ (Client 2 — Knock Knock jokes)   │
│  Thread3:                     ▇▇▇▇ (Client 3 — quick session)      │
│                                                                      │
│  Each client gets its OWN:                                          │
│  • Thread                                                           │
│  • KnockKnockProtocol instance (own state, own joke position)      │
│  • Input/Output streams                                             │
│  • Socket                                                           │
│                                                                      │
│  Clients are COMPLETELY independent of each other!                  │
│                                                                      │
└──────────────────────────────────────────────────────────────────┘
```

> **💡 Key Pattern:** The multi-client architecture is one of the most important patterns in network programming. Almost every real server (web servers, game servers, chat servers) uses this approach — or a more advanced version of it (thread pools, NIO, async I/O).

---

## 8. Socket Timeouts and Error Handling

### Setting Timeouts

Without timeouts, your server thread blocks **forever** waiting for a client that might have disappeared:

```java
// Connection timeout — how long to wait when CONNECTING
Socket socket = new Socket();
socket.connect(new InetSocketAddress("server.com", 5000), 5000);  // 5 seconds

// Read timeout — how long to wait on readLine() / read()
socket.setSoTimeout(30000);  // 30 seconds

// ServerSocket timeout — how long accept() waits
ServerSocket server = new ServerSocket(5000);
server.setSoTimeout(60000);  // 60 seconds, then SocketTimeoutException
```

### Common Exceptions

| Exception | When It Happens | What To Do |
|-----------|----------------|------------|
| `UnknownHostException` | DNS can't resolve the hostname | Check the hostname spelling |
| `ConnectException` | Server is not running or port is wrong | Verify server is up and port matches |
| `SocketTimeoutException` | Connect/read exceeded timeout | Retry or inform user |
| `SocketException: Connection reset` | Other side abruptly closed | Handle gracefully — client disconnected |
| `BindException: Address already in use` | Port is already taken | Use a different port or wait |

### Robust Error Handling Example

```java
public class RobustClient {
    public static void main(String[] args) {
        String host = "localhost";
        int port = 5000;

        try {
            // Connection with timeout
            Socket socket = new Socket();
            socket.connect(new InetSocketAddress(host, port), 5000);
            socket.setSoTimeout(30000);

            BufferedReader in = new BufferedReader(
                new InputStreamReader(socket.getInputStream()));
            PrintWriter out = new PrintWriter(
                socket.getOutputStream(), true);

            out.println("Hello Server!");

            String response = in.readLine();
            if (response != null) {
                System.out.println("Server says: " + response);
            } else {
                System.out.println("Server closed the connection.");
            }

            in.close();
            out.close();
            socket.close();

        } catch (UnknownHostException e) {
            System.err.println("Cannot find host: " + host);
        } catch (SocketTimeoutException e) {
            System.err.println("Connection timed out!");
        } catch (ConnectException e) {
            System.err.println("Cannot connect — is the server running?");
        } catch (IOException e) {
            System.err.println("I/O error: " + e.getMessage());
        }
    }
}
```

---

## 9. Socket vs. URL — When to Use Which

This is a common source of confusion. Here's a clear guide:

| Scenario | Use | Why |
|----------|-----|-----|
| Fetch a web page | `URL` / `HttpURLConnection` | HTTP protocol is built-in |
| Call a REST API | `HttpURLConnection` or HttpClient (Java 11+) | Handles HTTP verbs, headers, etc. |
| Download a file from URL | `URL.openStream()` | Quick and simple |
| Build a custom chat application | `Socket` / `ServerSocket` | Need your own protocol |
| Build a game server | `Socket` / `ServerSocket` | Custom protocol, low latency |
| Implement FTP, SMTP manually | `Socket` | Working at protocol level |
| Real-time bidirectional communication | `Socket` (or WebSocket) | Full-duplex communication |

> **Rule of thumb:** If HTTP does what you need, use `URL`/`HttpURLConnection`. If you need a **custom protocol** or require **persistent connections**, use `Socket`.

---

## 10. Exercises — Try It Yourself!

### Exercise 1: Enhanced Echo Server

Modify the Echo server to:
- **Uppercase** all echoed text
- Add a timestamp to each response
- Respond to the command `"quit"` by closing the connection

```java
// Hint: Your echo line becomes:
String timestamp = java.time.LocalTime.now().format(
    java.time.format.DateTimeFormatter.ofPattern("HH:mm:ss")
);
out.println("[" + timestamp + "] " + inputLine.toUpperCase());
```

### Exercise 2: Simple Calculator Server

Build a server that accepts math expressions like `"5 + 3"` and returns `"Result: 8"`.

```
Client sends: "5 + 3"
Server replies: "Result: 8"

Client sends: "10 * 4"
Server replies: "Result: 40"

Client sends: "quit"
Server replies: "Goodbye!" and closes
```

### Exercise 3: Multi-Client Chat Room

This is a bigger challenge! Build a chat server where:
- Multiple clients can connect
- When one client sends a message, all other clients see it
- Each client has a username

**Hint:** The server needs to keep a `List<PrintWriter>` of all connected clients' output streams. When a message comes in from one client, iterate through the list and `println()` to all of them!

---

## 11. Summary and What's Next

| Concept | Key Takeaway |
|---------|-------------|
| **Socket** | One endpoint of a TCP connection (IP + Port) |
| **ServerSocket** | Server-side listener — calls `accept()` for clients |
| **The Five-Step Pattern** | Open socket → get streams → read/write → close |
| **Protocol** | Agreement between client and server on message format |
| **try-with-resources** | Auto-closes sockets and streams |
| **Blocking I/O** | `accept()` and `readLine()` block until data arrives |
| **Multi-client** | One thread per client; main thread loops on `accept()` |
| **Timeouts** | Always set them to avoid infinite waits |

```
┌──────────────────────────────────────────────────────────────────┐
│                    SERIES ROADMAP                                    │
├──────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ✅ Part 1: Networking Basics, URLs, URLConnections                 │
│  ✅ Part 2: Socket Programming (YOU ARE HERE)                       │
│                                                                      │
│  📋 Part 3: Datagrams and Network Interfaces                       │
│     • UDP with DatagramSocket and DatagramPacket                    │
│     • Building a Quote-of-the-Day server                            │
│     • Broadcasting with MulticastSocket                             │
│     • Querying Network Interfaces                                   │
│                                                                      │
└──────────────────────────────────────────────────────────────────┘
```

> **💡 Up Next:** In Part 3, we explore the other side of networking — **UDP datagrams**. You'll build a Quote-of-the-Day server, learn about **multicasting** (broadcasting to many clients at once), and discover how to query your machine's **network interfaces**. It's the final piece of the Java networking puzzle! 🚀
