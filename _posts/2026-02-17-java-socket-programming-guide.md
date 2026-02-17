---
title: "Java Socket Programming: Build a Real-Time Chat Application from Scratch"
description: Master Java socket programming with TCP/IP fundamentals, ServerSocket and Socket APIs, multithreading for networking, and a complete hands-on chat application tutorial
author: Vaibhav Gagneja
date: 2026-02-17 12:00:00 +0530
categories: [Development, Java]
tags: [java, sockets, networking, tcp, chat-application, multithreading, server-client]
toc: true
image:
  path: https://images.unsplash.com/photo-1558494949-ef010cbdcc31
---

Networking is the backbone of modern software — from the web browser you're reading this in, to messaging apps, multiplayer games, and microservices talking to each other. At the heart of all network communication in Java lies **Socket Programming**.

In this hands-on guide, we'll go from zero to building a **fully functional real-time chat application** using Java sockets. Along the way, you'll learn TCP/IP fundamentals, the `ServerSocket` and `Socket` APIs, multithreaded networking, and production best practices.

---

## 1. What Is a Socket?

A **socket** is one endpoint of a two-way communication link between two programs running on a network. Think of it like a **telephone** — each side needs one to talk and listen.

```
┌─────────────────────────────────────────────────────────────────┐
│                    SOCKET: THE TELEPHONE ANALOGY                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                     │
│   Machine A (Server)              Machine B (Client)               │
│   ┌────────────────┐              ┌────────────────┐               │
│   │   Application  │              │   Application  │               │
│   │       │        │              │       │        │               │
│   │   ┌───┴────┐   │              │   ┌───┴────┐   │               │
│   │   │ Socket │◄──┼──── TCP ─────┼──►│ Socket │   │               │
│   │   └────────┘   │  Connection  │   └────────┘   │               │
│   └────────────────┘              └────────────────┘               │
│                                                                     │
│   A socket = IP Address + Port Number                              │
│   Example:  192.168.1.10:8080 ◄──► 192.168.1.20:54321            │
│                                                                     │
└─────────────────────────────────────────────────────────────────┘
```

### Key Socket Concepts

| Concept | Description |
|---------|-------------|
| **IP Address** | Identifies a machine on the network (e.g., `192.168.1.10`) |
| **Port Number** | Identifies a specific application on that machine (0–65535) |
| **Socket** | The combination of IP + Port — a unique communication endpoint |
| **Connection** | A live link between two sockets for data exchange |

> **💡 Port Number Rules:**
> - Ports 0–1023 are **well-known** (HTTP=80, HTTPS=443, FTP=21)
> - Ports 1024–49151 are **registered** (MySQL=3306, PostgreSQL=5432)
> - Ports 49152–65535 are **dynamic/ephemeral** — used by clients automatically

---

## 2. TCP vs UDP — Choosing Your Protocol

Before writing any code, you need to decide: **TCP or UDP?**

```
┌──────────────────────────────────────────────────────────────────┐
│                       TCP vs UDP AT A GLANCE                       │
├──────────────────────────────────────────────────────────────────┤
│                                                                      │
│  TCP (Transmission Control Protocol)                                │
│  ════════════════════════════════════                                │
│  🤝 Connection-oriented (handshake first)                           │
│  ✅ Reliable delivery (guaranteed, in-order)                        │
│  📦 Stream-based (continuous flow of bytes)                         │
│  🐢 Slower but safe                                                 │
│                                                                      │
│  UDP (User Datagram Protocol)                                       │
│  ════════════════════════════════                                    │
│  🏃 Connectionless (fire and forget)                                │
│  ⚠️  Unreliable delivery (packets may be lost/reordered)           │
│  📨 Datagram-based (discrete packets)                               │
│  ⚡ Faster but risky                                                │
│                                                                      │
└──────────────────────────────────────────────────────────────────┘
```

| Feature | TCP | UDP |
|---------|-----|-----|
| **Connection** | Required (3-way handshake) | Not required |
| **Reliability** | Guaranteed delivery | Best-effort delivery |
| **Ordering** | Messages arrive in order | No order guarantee |
| **Speed** | Slower (overhead for reliability) | Faster (minimal overhead) |
| **Use Cases** | Chat apps, file transfer, web, email | Video streaming, gaming, DNS |

> **For our chat application**, we'll use **TCP** — because we need every message to arrive reliably and in order. You don't want "Hey, are you free tonight?" arriving after "See you there!" 😄

---

## 3. The TCP Three-Way Handshake

Before any data flows, TCP establishes a connection through a **three-way handshake**:

```
┌──────────────────────────────────────────────────────────────────┐
│                  TCP THREE-WAY HANDSHAKE                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                      │
│   Client                                             Server         │
│     │                                                  │            │
│     │  ───── SYN (seq=100) ─────────────────────►     │  Step 1    │
│     │  "Hey, I want to connect!"                      │            │
│     │                                                  │            │
│     │  ◄──── SYN-ACK (seq=200, ack=101) ──────       │  Step 2    │
│     │  "Sure! I acknowledge your request."             │            │
│     │                                                  │            │
│     │  ───── ACK (ack=201) ─────────────────────►     │  Step 3    │
│     │  "Great, let's start talking!"                   │            │
│     │                                                  │            │
│     │  ◄═══════ CONNECTION ESTABLISHED ═══════►       │            │
│     │                                                  │            │
│     │  ◄────── Data Exchange Begins ──────────►       │            │
│                                                                      │
└──────────────────────────────────────────────────────────────────┘
```

Once the handshake completes, both sides can freely send and receive data through their respective sockets.

---

## 4. Java Socket API: The Building Blocks

Java provides two key classes in `java.net` for TCP socket programming:

```
┌──────────────────────────────────────────────────────────────────┐
│                 JAVA SOCKET API OVERVIEW                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                      │
│   SERVER SIDE                        CLIENT SIDE                    │
│   ═══════════                        ═══════════                    │
│   ┌──────────────────┐              ┌──────────────────┐           │
│   │   ServerSocket   │              │     Socket       │           │
│   │   (java.net)     │              │   (java.net)     │           │
│   ├──────────────────┤              ├──────────────────┤           │
│   │ • Listens on a   │              │ • Connects to a  │           │
│   │   specific port  │              │   server's IP    │           │
│   │ • Waits for      │              │   and port       │           │
│   │   client to      │◄── accept ──│ • Sends/receives │           │
│   │   connect        │   creates   │   data through   │           │
│   │ • Creates a      │   Socket    │   I/O streams    │           │
│   │   Socket for     │────────────►│                  │           │
│   │   each client    │              │                  │           │
│   └──────────────────┘              └──────────────────┘           │
│                                                                      │
│   Key I/O Streams (from Socket):                                    │
│   • getInputStream()  → DataInputStream  (read from remote)        │
│   • getOutputStream() → DataOutputStream (write to remote)         │
│                                                                      │
└──────────────────────────────────────────────────────────────────┘
```

### ServerSocket — Essential Methods

| Method | Description |
|--------|-------------|
| `ServerSocket(int port)` | Creates a server socket bound to the specified port |
| `accept()` | Blocks and waits for a client connection; returns a `Socket` |
| `close()` | Closes the server socket |
| `getInetAddress()` | Returns the local address to which this socket is bound |
| `isClosed()` | Returns whether the socket is closed |

### Socket — Essential Methods

| Method | Description |
|--------|-------------|
| `Socket(String host, int port)` | Creates a socket and connects to the specified host/port |
| `getInputStream()` | Returns an input stream for reading data from the remote side |
| `getOutputStream()` | Returns an output stream for writing data to the remote side |
| `close()` | Closes this socket |
| `getInetAddress()` | Returns the address to which the socket is connected |
| `getPort()` | Returns the remote port number |
| `isConnected()` | Returns whether the socket is connected |

---

## 5. Hello, Network! — Your First Socket Program

Let's start with the absolute basics: a server that accepts one client and they exchange a single message.

### Step 1: The Server

```java
import java.io.DataInputStream;
import java.io.DataOutputStream;
import java.io.IOException;
import java.net.ServerSocket;
import java.net.Socket;

public class SimpleServer {
    public static void main(String[] args) {
        System.out.println("Server starting...");

        // try-with-resources ensures sockets are closed automatically
        try (ServerSocket serverSocket = new ServerSocket(5000)) {

            System.out.println("Waiting for a client on port 5000...");
            Socket clientSocket = serverSocket.accept();  // BLOCKS here
            System.out.println("Client connected!");

            // Set up I/O streams
            DataInputStream input = new DataInputStream(clientSocket.getInputStream());
            DataOutputStream output = new DataOutputStream(clientSocket.getOutputStream());

            // Read a message from the client
            String received = input.readUTF();
            System.out.println("Client says: " + received);

            // Send a response
            output.writeUTF("Hello from the server!");
            output.flush();

            clientSocket.close();
            System.out.println("Connection closed.");

        } catch (IOException e) {
            System.err.println("Server error: " + e.getMessage());
        }
    }
}
```

### Step 2: The Client

```java
import java.io.DataInputStream;
import java.io.DataOutputStream;
import java.io.IOException;
import java.net.Socket;

public class SimpleClient {
    public static void main(String[] args) {
        System.out.println("Connecting to server...");

        try (Socket socket = new Socket("localhost", 5000)) {

            System.out.println("Connected!");

            // Set up I/O streams
            DataOutputStream output = new DataOutputStream(socket.getOutputStream());
            DataInputStream input = new DataInputStream(socket.getInputStream());

            // Send a message to the server
            output.writeUTF("Hello from the client!");
            output.flush();

            // Read the server's response
            String response = input.readUTF();
            System.out.println("Server says: " + response);

        } catch (IOException e) {
            System.err.println("Client error: " + e.getMessage());
        }
    }
}
```

### How to Run

```
┌──────────────────────────────────────────────────────────────────┐
│                    RUNNING YOUR FIRST SOCKET APP                    │
├──────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Terminal 1 (Server):                                               │
│  $ javac SimpleServer.java                                          │
│  $ java SimpleServer                                                │
│  Server starting...                                                 │
│  Waiting for a client on port 5000...                               │
│  █  (server is BLOCKING here, waiting for client)                   │
│                                                                      │
│  Terminal 2 (Client):                                               │
│  $ javac SimpleClient.java                                          │
│  $ java SimpleClient                                                │
│  Connecting to server...                                            │
│  Connected!                                                          │
│  Server says: Hello from the server!                                │
│                                                                      │
│  Back in Terminal 1:                                                │
│  Client connected!                                                   │
│  Client says: Hello from the client!                                │
│  Connection closed.                                                  │
│                                                                      │
│  ⚠️  ALWAYS start the server FIRST, then the client!               │
│                                                                      │
└──────────────────────────────────────────────────────────────────┘
```

### What Just Happened?

```
┌──────────────────────────────────────────────────────────────────┐
│                     EXECUTION FLOW                                  │
├──────────────────────────────────────────────────────────────────┤
│                                                                      │
│  SERVER                              CLIENT                         │
│  ══════                              ══════                         │
│  1. Create ServerSocket(5000)                                       │
│  2. accept() — BLOCKS ⏸️                                           │
│                                      3. Create Socket("localhost",  │
│                                         5000) — triggers handshake  │
│  4. accept() returns Socket ✅                                      │
│  5. Open streams                     6. Open streams                │
│                                      7. writeUTF("Hello...")        │
│  8. readUTF() → receives message                                    │
│  9. writeUTF("Hello from server")                                   │
│                                     10. readUTF() → gets response   │
│ 11. Close                           12. Close                       │
│                                                                      │
└──────────────────────────────────────────────────────────────────┘
```

---

## 6. Understanding `DataInputStream` and `DataOutputStream`

In our socket programs, we use `DataInputStream` and `DataOutputStream` to send and receive data. But why these specifically?

### Why Not Just Use Raw Streams?

| Approach | Limitation |
|----------|-----------|
| `InputStream.read()` / `OutputStream.write()` | Reads/writes only **raw bytes** — no structure |
| `BufferedReader` / `PrintWriter` | Good for text, but not for mixed data types |
| **`DataInputStream` / `DataOutputStream`** | ✅ Reads/writes **Java primitives + strings** in binary format |

### The `writeUTF()` / `readUTF()` Pair

This is the workhorse method for socket communication:

```java
// SENDING side
DataOutputStream dos = new DataOutputStream(socket.getOutputStream());
dos.writeUTF("Hello, World!");  // Writes: [2-byte length] + [UTF-8 bytes]
dos.flush();                     // Force send immediately

// RECEIVING side
DataInputStream dis = new DataInputStream(socket.getInputStream());
String message = dis.readUTF();  // Reads the length, then that many bytes
// message = "Hello, World!"
```

```
┌──────────────────────────────────────────────────────────────────┐
│                  HOW writeUTF / readUTF WORKS                      │
├──────────────────────────────────────────────────────────────────┤
│                                                                      │
│  writeUTF("Hi")                                                     │
│                                                                      │
│  Wire format:                                                       │
│  ┌──────────┬──────────────────┐                                    │
│  │ 00 02    │  48 69           │                                    │
│  │ (length) │  ("Hi" in UTF-8) │                                    │
│  └──────────┴──────────────────┘                                    │
│   2 bytes       N bytes                                             │
│                                                                      │
│  readUTF():                                                         │
│  1. Read 2 bytes → length = 2                                       │
│  2. Read 2 bytes → "Hi"                                             │
│  3. Return "Hi" as a Java String ✅                                 │
│                                                                      │
│  🔑 This is self-delimiting — the receiver knows exactly            │
│     how many bytes to read. No confusion about message              │
│     boundaries!                                                      │
│                                                                      │
└──────────────────────────────────────────────────────────────────┘
```

### Other Useful DataStream Methods

| Write Method | Read Method | Data Type |
|-------------|-------------|-----------|
| `writeUTF(String)` | `readUTF()` | Modified UTF-8 string |
| `writeInt(int)` | `readInt()` | 4-byte integer |
| `writeDouble(double)` | `readDouble()` | 8-byte double |
| `writeBoolean(boolean)` | `readBoolean()` | 1-byte boolean |
| `writeLong(long)` | `readLong()` | 8-byte long |

> **⚠️ Golden Rule:** Always read in the **exact same order** you write. If you `writeInt()` then `writeUTF()`, you must `readInt()` then `readUTF()` on the other side.

---

## 7. Adding Continuous Communication

Our first example exchanged just one message. Let's make it a proper back-and-forth conversation:

### Continuous Server

```java
import java.io.DataInputStream;
import java.io.DataOutputStream;
import java.io.IOException;
import java.net.InetAddress;
import java.net.ServerSocket;
import java.net.Socket;
import java.util.Scanner;

public class ChatServer {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);

        try (ServerSocket serverSocket = new ServerSocket(5000)) {

            // Display the server's IP address for the client to use
            String serverIP = InetAddress.getLocalHost().getHostAddress();
            System.out.println("Server started at IP: " + serverIP);
            System.out.println("Waiting for a client...");

            Socket clientSocket = serverSocket.accept();
            System.out.println("Client connected!");
            System.out.println("──────────────────────────────────");

            DataInputStream input = new DataInputStream(clientSocket.getInputStream());
            DataOutputStream output = new DataOutputStream(clientSocket.getOutputStream());

            // Communication loop
            while (true) {
                // Read message from client
                String received = input.readUTF();
                System.out.println("Client: " + received);

                if (received.equalsIgnoreCase("exit")) {
                    System.out.println("Client disconnected.");
                    break;
                }

                // Type and send a response
                System.out.print("You: ");
                String reply = scanner.nextLine();
                output.writeUTF(reply);
                output.flush();

                if (reply.equalsIgnoreCase("exit")) {
                    System.out.println("Server shutting down.");
                    break;
                }
            }

            clientSocket.close();

        } catch (IOException e) {
            System.err.println("Server error: " + e.getMessage());
        }
    }
}
```

### Continuous Client

```java
import java.io.DataInputStream;
import java.io.DataOutputStream;
import java.io.IOException;
import java.net.Socket;
import java.util.Scanner;

public class ChatClient {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);

        System.out.print("Enter server IP address: ");
        String serverIP = scanner.nextLine();

        try (Socket socket = new Socket(serverIP, 5000)) {

            System.out.println("Connected to server!");
            System.out.println("──────────────────────────────────");

            DataOutputStream output = new DataOutputStream(socket.getOutputStream());
            DataInputStream input = new DataInputStream(socket.getInputStream());

            while (true) {
                // Type and send a message
                System.out.print("You: ");
                String message = scanner.nextLine();
                output.writeUTF(message);
                output.flush();

                if (message.equalsIgnoreCase("exit")) {
                    System.out.println("Disconnecting...");
                    break;
                }

                // Wait for the server's reply
                String response = input.readUTF();
                System.out.println("Server: " + response);

                if (response.equalsIgnoreCase("exit")) {
                    System.out.println("Server disconnected.");
                    break;
                }
            }

        } catch (IOException e) {
            System.err.println("Connection error: " + e.getMessage());
        }
    }
}
```

### The Problem with This Approach

This works, but there's a fundamental flaw:

```
┌──────────────────────────────────────────────────────────────────┐
│              THE TURN-BASED COMMUNICATION PROBLEM                   │
├──────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Current flow (BLOCKING):                                           │
│                                                                      │
│  Client sends → Server reads → Server sends → Client reads → ...   │
│                                                                      │
│  ⚠️ Problems:                                                       │
│  1. Client must WAIT for server to reply before sending again       │
│  2. Server must WAIT for client to send before it can reply         │
│  3. Neither can send two messages in a row                          │
│  4. Feels like walkie-talkie, not a real chat!                      │
│                                                                      │
│  What we want (NON-BLOCKING):                                       │
│                                                                      │
│  Client can send ──── anytime ────► Server reads immediately        │
│  Client reads ◄────── anytime ──── Server can send                  │
│                                                                      │
│  Solution: Use SEPARATE THREADS for reading and writing!            │
│                                                                      │
└──────────────────────────────────────────────────────────────────┘
```

> **💡 Key Insight:** `readUTF()` is a **blocking call** — it freezes the thread until data arrives. To send and receive simultaneously, we need **multithreading**.

---

## 8. The Solution: Multithreaded Chat

The idea is simple: use **one thread for reading** and the **main thread for writing**. This way, incoming messages display instantly while you're typing.

```
┌──────────────────────────────────────────────────────────────────┐
│              MULTITHREADED ARCHITECTURE                              │
├──────────────────────────────────────────────────────────────────┤
│                                                                      │
│   SERVER APPLICATION                  CLIENT APPLICATION            │
│   ┌──────────────────┐              ┌──────────────────┐           │
│   │  Main Thread     │              │  Main Thread     │           │
│   │  ┌────────────┐  │              │  ┌────────────┐  │           │
│   │  │ Write msgs │──┼──── TCP ─────┼──│ Read msgs  │  │           │
│   │  └────────────┘  │  Connection  │  └────────────┘  │           │
│   │                  │              │                  │           │
│   │  Reader Thread   │              │  Reader Thread   │           │
│   │  ┌────────────┐  │              │  ┌────────────┐  │           │
│   │  │ Read msgs  │◄─┼──── TCP ─────┼──│ Write msgs │  │           │
│   │  └────────────┘  │  Connection  │  └────────────┘  │           │
│   └──────────────────┘              └──────────────────┘           │
│                                                                      │
│   Both sides have:                                                  │
│   • Main thread → handles user input & sends messages              │
│   • Reader thread → continuously listens for incoming messages     │
│                                                                      │
└──────────────────────────────────────────────────────────────────┘
```

---

## 9. Building the Chat Application — Server with GUI

Now let's build a proper chat application with a graphical interface using **Swing**. We'll add a text area for message history, a text field for input, and a dedicated thread for reading incoming messages.

### The Server Application

```java
import java.awt.BorderLayout;
import java.awt.Color;
import java.awt.Font;
import java.awt.event.ActionEvent;
import java.awt.event.ActionListener;
import java.io.DataInputStream;
import java.io.DataOutputStream;
import java.io.IOException;
import java.net.InetAddress;
import java.net.ServerSocket;
import java.net.Socket;
import javax.swing.JFrame;
import javax.swing.JScrollPane;
import javax.swing.JTextArea;
import javax.swing.JTextField;

public class ChatServerGUI {
    // ──── GUI Components ────
    private JFrame frame;
    private JTextArea messageArea;
    private JScrollPane scrollPane;
    private JTextField inputField;

    // ──── Networking Components ────
    private ServerSocket serverSocket;
    private Socket clientSocket;
    private DataInputStream inputStream;
    private DataOutputStream outputStream;

    // ──── Reader Thread ────
    // This thread continuously reads messages from the client.
    // It runs in the background so the main/GUI thread stays responsive.
    private Thread readerThread = new Thread() {
        @Override
        public void run() {
            while (true) {
                try {
                    String message = inputStream.readUTF();
                    displayMessage("Client: " + message);
                } catch (IOException e) {
                    displayMessage("⚠ Client disconnected.");
                    break;
                }
            }
        }
    };

    public ChatServerGUI() {
        // ──── Build the GUI ────
        frame = new JFrame("Chat Server");
        frame.setSize(500, 500);
        frame.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);

        // Message display area
        messageArea = new JTextArea();
        messageArea.setEditable(false);
        messageArea.setFont(new Font("Segoe UI", Font.PLAIN, 14));
        messageArea.setBackground(new Color(240, 240, 240));
        scrollPane = new JScrollPane(messageArea);
        frame.add(scrollPane, BorderLayout.CENTER);

        // Input field at the bottom
        inputField = new JTextField();
        inputField.setFont(new Font("Segoe UI", Font.PLAIN, 14));
        inputField.setEditable(false);  // Disabled until client connects
        inputField.addActionListener(new ActionListener() {
            @Override
            public void actionPerformed(ActionEvent e) {
                String text = inputField.getText().trim();
                if (!text.isEmpty()) {
                    sendMessage(text);
                    displayMessage("You: " + text);
                    inputField.setText("");
                }
            }
        });
        frame.add(inputField, BorderLayout.SOUTH);

        frame.setVisible(true);
    }

    /**
     * Starts the server: binds to a port, displays its IP,
     * and blocks until a client connects.
     */
    public void waitForClient() {
        try {
            String ipAddress = InetAddress.getLocalHost().getHostAddress();

            serverSocket = new ServerSocket(5000);
            messageArea.setText("Server started.\n");
            messageArea.append("Your IP Address: " + ipAddress + "\n");
            messageArea.append("Waiting for a client to connect...\n");

            clientSocket = serverSocket.accept();  // BLOCKS

            messageArea.append("━━ Client Connected! ━━\n\n");
            inputField.setEditable(true);  // Enable input now

        } catch (IOException e) {
            displayMessage("Server startup error: " + e.getMessage());
        }
    }

    /**
     * Sets up the I/O streams and starts the reader thread.
     * Call this AFTER the client has connected.
     */
    public void initializeStreams() {
        try {
            inputStream = new DataInputStream(clientSocket.getInputStream());
            outputStream = new DataOutputStream(clientSocket.getOutputStream());
        } catch (IOException e) {
            displayMessage("Stream initialization error: " + e.getMessage());
        }
        readerThread.start();
    }

    /** Sends a message to the connected client. */
    private void sendMessage(String message) {
        try {
            outputStream.writeUTF(message);
            outputStream.flush();
        } catch (IOException e) {
            displayMessage("Send error: " + e.getMessage());
        }
    }

    /** Appends a message to the chat window. */
    private void displayMessage(String message) {
        messageArea.append(message + "\n");
        // Auto-scroll to bottom
        messageArea.setCaretPosition(messageArea.getDocument().getLength());
    }

    // ──── Entry Point ────
    public static void main(String[] args) {
        ChatServerGUI server = new ChatServerGUI();
        server.waitForClient();
        server.initializeStreams();
    }
}
```

### How the Server Works — Step by Step

```
┌──────────────────────────────────────────────────────────────────┐
│                   SERVER LIFECYCLE                                   │
├──────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. main() → new ChatServerGUI()                                    │
│     └─► Creates the Swing GUI (window, text area, input field)      │
│     └─► Input field is DISABLED (no one to talk to yet)             │
│                                                                      │
│  2. waitForClient()                                                 │
│     └─► Creates ServerSocket on port 5000                           │
│     └─► Displays IP address (so client knows where to connect)      │
│     └─► Calls accept() — BLOCKS until a client connects             │
│     └─► On connection: enables input field                          │
│                                                                      │
│  3. initializeStreams()                                              │
│     └─► Wraps socket streams in DataInputStream/DataOutputStream    │
│     └─► Starts readerThread — now running in background             │
│                                                                      │
│  4. Two things happen in parallel:                                  │
│     ┌─────────────────────────────────────────────┐                 │
│     │ MAIN/GUI THREAD: User types in inputField   │                 │
│     │ → ActionListener fires → sendMessage()      │                 │
│     │ → writeUTF() sends to client                │                 │
│     ├─────────────────────────────────────────────┤                 │
│     │ READER THREAD: Loops calling readUTF()      │                 │
│     │ → Blocks until message arrives              │                 │
│     │ → Calls displayMessage() to show it         │                 │
│     └─────────────────────────────────────────────┘                 │
│                                                                      │
└──────────────────────────────────────────────────────────────────┘
```

---

## 10. Building the Chat Application — Client with GUI

### The Client Application

```java
import java.awt.BorderLayout;
import java.awt.Color;
import java.awt.Font;
import java.awt.event.ActionEvent;
import java.awt.event.ActionListener;
import java.io.DataInputStream;
import java.io.DataOutputStream;
import java.io.IOException;
import java.net.Socket;
import javax.swing.JFrame;
import javax.swing.JOptionPane;
import javax.swing.JScrollPane;
import javax.swing.JTextArea;
import javax.swing.JTextField;

public class ChatClientGUI {
    // ──── GUI Components ────
    private JFrame frame;
    private JTextArea messageArea;
    private JScrollPane scrollPane;
    private JTextField inputField;

    // ──── Networking Components ────
    private Socket socket;
    private DataInputStream inputStream;
    private DataOutputStream outputStream;

    private String serverAddress;

    // ──── Reader Thread ────
    private Thread readerThread = new Thread() {
        @Override
        public void run() {
            while (true) {
                try {
                    String message = inputStream.readUTF();
                    displayMessage("Server: " + message);
                } catch (IOException e) {
                    displayMessage("⚠ Connection lost.");
                    break;
                }
            }
        }
    };

    public ChatClientGUI() {
        // Prompt user for the server's IP address
        serverAddress = JOptionPane.showInputDialog(
            null,
            "Enter the server's IP address:",
            "Connect to Server",
            JOptionPane.QUESTION_MESSAGE
        );

        // Validate input
        if (serverAddress == null || serverAddress.trim().isEmpty()) {
            System.out.println("No IP address provided. Exiting.");
            System.exit(0);
        }

        // Connect to the server
        connectToServer();

        // ──── Build the GUI ────
        frame = new JFrame("Chat Client");
        frame.setSize(500, 500);
        frame.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);

        messageArea = new JTextArea();
        messageArea.setEditable(false);
        messageArea.setFont(new Font("Segoe UI", Font.PLAIN, 14));
        messageArea.setBackground(new Color(240, 240, 240));
        messageArea.setText("Connected to server at " + serverAddress + "\n");
        messageArea.append("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━\n\n");
        scrollPane = new JScrollPane(messageArea);
        frame.add(scrollPane, BorderLayout.CENTER);

        inputField = new JTextField();
        inputField.setFont(new Font("Segoe UI", Font.PLAIN, 14));
        inputField.addActionListener(new ActionListener() {
            @Override
            public void actionPerformed(ActionEvent e) {
                String text = inputField.getText().trim();
                if (!text.isEmpty()) {
                    sendMessage(text);
                    displayMessage("You: " + text);
                    inputField.setText("");
                }
            }
        });
        frame.add(inputField, BorderLayout.SOUTH);

        frame.setVisible(true);
    }

    /** Establishes the TCP connection to the server. */
    private void connectToServer() {
        try {
            socket = new Socket(serverAddress, 5000);
        } catch (IOException e) {
            JOptionPane.showMessageDialog(
                null,
                "Could not connect to server at " + serverAddress + "\n" + e.getMessage(),
                "Connection Failed",
                JOptionPane.ERROR_MESSAGE
            );
            System.exit(1);
        }
    }

    /** Sets up I/O streams and starts the reader thread. */
    public void initializeStreams() {
        try {
            inputStream = new DataInputStream(socket.getInputStream());
            outputStream = new DataOutputStream(socket.getOutputStream());
        } catch (IOException e) {
            displayMessage("Stream error: " + e.getMessage());
        }
        readerThread.start();
    }

    private void sendMessage(String message) {
        try {
            outputStream.writeUTF(message);
            outputStream.flush();
        } catch (IOException e) {
            displayMessage("Send error: " + e.getMessage());
        }
    }

    private void displayMessage(String message) {
        messageArea.append(message + "\n");
        messageArea.setCaretPosition(messageArea.getDocument().getLength());
    }

    // ──── Entry Point ────
    public static void main(String[] args) {
        ChatClientGUI client = new ChatClientGUI();
        client.initializeStreams();
    }
}
```

---

## 11. Running the Complete Application

Here's the step-by-step process to run your chat app:

```
┌──────────────────────────────────────────────────────────────────┐
│                 RUNNING THE CHAT APPLICATION                        │
├──────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Step 1: Compile both files                                         │
│  ─────────────────────────────                                      │
│  $ javac ChatServerGUI.java                                         │
│  $ javac ChatClientGUI.java                                         │
│                                                                      │
│  Step 2: Start the Server FIRST                                     │
│  ──────────────────────────────                                     │
│  $ java ChatServerGUI                                               │
│  → A window appears showing the server's IP address                 │
│  → Note down the IP (e.g., 192.168.1.42)                           │
│                                                                      │
│  Step 3: Start the Client                                           │
│  ────────────────────────────                                       │
│  $ java ChatClientGUI                                               │
│  → A dialog box asks for the IP address                             │
│  → Enter the server's IP (or "localhost" if same machine)           │
│  → Client window appears after successful connection                │
│                                                                      │
│  Step 4: Chat!                                                      │
│  ─────────────                                                      │
│  → Type a message in either window's input field                    │
│  → Press ENTER to send                                              │
│  → Messages appear in both windows instantly!                       │
│                                                                      │
│  ┌───────────────────┐    ┌───────────────────┐                     │
│  │  Chat Server      │    │  Chat Client      │                     │
│  │                   │    │                   │                     │
│  │  Client: Hey!     │    │  Connected to...  │                     │
│  │  You: Hi there!   │    │  You: Hey!        │                     │
│  │  Client: How are  │    │  Server: Hi there!│                     │
│  │  you?             │    │  You: How are you?│                     │
│  │───────────────────│    │───────────────────│                     │
│  │ [Type here...   ] │    │ [Type here...   ] │                     │
│  └───────────────────┘    └───────────────────┘                     │
│                                                                      │
└──────────────────────────────────────────────────────────────────┘
```

> **💡 Testing on the same machine?** Use `localhost` or `127.0.0.1` as the IP address.

---

## 12. Anatomy of the Application — How It All Fits Together

Let's trace the complete lifecycle of a single message:

```
┌──────────────────────────────────────────────────────────────────┐
│               MESSAGE JOURNEY: CLIENT → SERVER                      │
├──────────────────────────────────────────────────────────────────┤
│                                                                      │
│  CLIENT SIDE                                                        │
│  ═══════════                                                        │
│  1. User types "Hello!" in inputField and presses ENTER             │
│  2. ActionListener fires                                            │
│  3. sendMessage("Hello!"):                                          │
│     └─► outputStream.writeUTF("Hello!")                             │
│         └─► Encodes as [00 06][48 65 6C 6C 6F 21]                  │
│         └─► Sends bytes over TCP socket                             │
│  4. displayMessage("You: Hello!") — shows locally                  │
│                                                                      │
│  ─── network transmission (bytes travel over TCP) ───              │
│                                                                      │
│  SERVER SIDE                                                        │
│  ═══════════                                                        │
│  5. readerThread is blocked on inputStream.readUTF()                │
│  6. Bytes arrive → readUTF() decodes → returns "Hello!"            │
│  7. displayMessage("Client: Hello!") — shows on server             │
│                                                                      │
│  Total time: typically < 1 millisecond on a local network!          │
│                                                                      │
└──────────────────────────────────────────────────────────────────┘
```

### Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                    COMPLETE ARCHITECTURE                             │
├──────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────── ChatServerGUI ────────────┐                           │
│  │                                       │                           │
│  │  ┌──────────────────────────────┐    │                           │
│  │  │   JFrame (Server Window)     │    │                           │
│  │  │  ┌────────────────────────┐  │    │                           │
│  │  │  │ JTextArea (messages)   │  │    │                           │
│  │  │  │  via JScrollPane       │  │    │                           │
│  │  │  └────────────────────────┘  │    │                           │
│  │  │  ┌────────────────────────┐  │    │                           │
│  │  │  │ JTextField (input)     │  │    │     ┌───────────┐        │
│  │  │  └────────────────────────┘  │    │     │           │        │
│  │  └──────────────────────────────┘    │     │  TCP/IP   │        │
│  │                                       │     │  Network  │        │
│  │  ServerSocket ──► accept() ──► Socket │◄───►│  Port     │        │
│  │  DataInputStream  (reads from client) │     │  5000     │        │
│  │  DataOutputStream (writes to client)  │     │           │        │
│  │  readerThread (background read loop)  │     └───────────┘        │
│  └───────────────────────────────────────┘          ▲               │
│                                                      │               │
│  ┌─────────── ChatClientGUI ────────────┐           │               │
│  │                                       │           │               │
│  │  Socket("IP", 5000) ─────────────────┼───────────┘               │
│  │  DataOutputStream (writes to server)  │                           │
│  │  DataInputStream  (reads from server) │                           │
│  │  readerThread (background read loop)  │                           │
│  │                                       │                           │
│  │  ┌──────────────────────────────┐    │                           │
│  │  │   JFrame (Client Window)     │    │                           │
│  │  │   JTextArea + JScrollPane    │    │                           │
│  │  │   JTextField (input)         │    │                           │
│  │  └──────────────────────────────┘    │                           │
│  └───────────────────────────────────────┘                           │
│                                                                      │
└──────────────────────────────────────────────────────────────────┘
```

---

## 13. Key Concepts Deep Dive

### Why Does `accept()` Block?

`ServerSocket.accept()` is a **blocking method** — it pauses the thread's execution and waits until a client connects. This is by design:

```java
// This line HALTS execution until a client shows up
Socket clientSocket = serverSocket.accept();  // ⏸️ BLOCKED HERE
// Code below only runs AFTER a client connects
```

> **💡 Think of it like a receptionist** sitting at a desk with nothing to do until a visitor walks in. The receptionist doesn't keep checking — they just wait.

### Why Use a Separate Thread for Reading?

Without a separate thread, your GUI would **freeze** while waiting for messages:

```
┌──────────────────────────────────────────────────────────────────┐
│           WITHOUT SEPARATE READER THREAD (BAD ❌)                   │
├──────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Single Thread:                                                     │
│  1. Wait for message... ⏸️ (GUI FROZEN! Can't type!)               │
│  2. Message arrives → display it                                    │
│  3. Now you can type and send                                       │
│  4. Wait for message... ⏸️ (GUI FROZEN AGAIN!)                     │
│                                                                      │
├──────────────────────────────────────────────────────────────────┤
│           WITH SEPARATE READER THREAD (GOOD ✅)                     │
├──────────────────────────────────────────────────────────────────┤
│                                                                      │
│  GUI Thread:       Type → Send → Type → Send → Type (always free)  │
│  Reader Thread:    Wait → Read → Display → Wait → Read → Display   │
│                                                                      │
│  Both run SIMULTANEOUSLY — GUI never freezes!                       │
│                                                                      │
└──────────────────────────────────────────────────────────────────┘
```

### The `flush()` Mystery Explained

When you write to a stream, data may sit in an **internal buffer** instead of being sent immediately. `flush()` forces the buffer to empty:

```java
outputStream.writeUTF("Hello!");
// Data might still be sitting in the buffer!

outputStream.flush();
// NOW it's guaranteed to be sent over the network!
```

| Scenario | Without `flush()` | With `flush()` |
|----------|-------------------|----------------|
| Short message | Might sit in buffer | Sent immediately |
| Stream closing | Flushed automatically on `close()` | Already sent |
| Time-sensitive data | Could appear delayed | No delay |

> **Best Practice:** Always call `flush()` after writing to a network stream. For file I/O, it's less critical since closing the stream flushes automatically.

---

## 14. Error Handling and Robustness

Our basic application works, but production code needs better error handling. Here's an improved approach:

### Graceful Disconnection Handling

```java
// In the reader thread — handle disconnection gracefully
private Thread readerThread = new Thread() {
    @Override
    public void run() {
        try {
            while (!Thread.currentThread().isInterrupted()) {
                String message = inputStream.readUTF();
                displayMessage("Remote: " + message);
            }
        } catch (IOException e) {
            // This fires when the remote side disconnects
            displayMessage("⚠ Connection closed: " + e.getMessage());
        } finally {
            cleanup();
        }
    }
};

// Resource cleanup method
private void cleanup() {
    try {
        if (inputStream != null)  inputStream.close();
        if (outputStream != null) outputStream.close();
        if (socket != null)       socket.close();
    } catch (IOException e) {
        System.err.println("Cleanup error: " + e.getMessage());
    }
}
```

### Common Errors and Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| `ConnectException: Connection refused` | Server isn't running or wrong port | Start server first, verify port number |
| `BindException: Address already in use` | Another process is using that port | Choose a different port or kill the process |
| `SocketException: Connection reset` | Remote side closed unexpectedly | Add try-catch around read calls |
| `UnknownHostException` | Invalid IP address / hostname | Verify the address is correct |
| `SocketTimeoutException` | Connection attempt took too long | Check network, increase timeout |

### Setting Connection Timeouts

```java
// Client-side: don't wait forever to connect
Socket socket = new Socket();
socket.connect(
    new java.net.InetSocketAddress("192.168.1.42", 5000),
    5000  // 5-second timeout
);

// Server-side: don't wait forever for clients
serverSocket.setSoTimeout(30000);  // 30-second timeout on accept()
try {
    Socket client = serverSocket.accept();
} catch (java.net.SocketTimeoutException e) {
    System.out.println("No client connected within 30 seconds.");
}
```

---

## 15. Exercise: Extend the Chat Application

Now it's your turn! Try these exercises to deepen your understanding:

### Exercise 1: Add Timestamps 🕐

Add a timestamp to each message displayed in the chat window.

**Hint:**
```java
import java.time.LocalTime;
import java.time.format.DateTimeFormatter;

DateTimeFormatter formatter = DateTimeFormatter.ofPattern("HH:mm:ss");
String timestamp = LocalTime.now().format(formatter);
// Output: "[14:30:45] Client: Hello!"
```

### Exercise 2: Add an "Online" Status Indicator 🟢

Show "Client Connected" / "Client Disconnected" in the window title bar.

**Hint:** Use `frame.setTitle("Chat Server — 🟢 Online");`

### Exercise 3: Multi-Client Server 👥

Modify the server to accept **multiple clients** simultaneously. This requires:
1. A `while(true)` loop around `accept()`
2. A new thread for **each client** that connects
3. Broadcasting messages from one client to all others

**Skeleton:**
```java
import java.io.*;
import java.net.*;
import java.util.*;

public class MultiClientServer {
    // Track all connected client output streams
    private static Set<DataOutputStream> clientStreams =
        Collections.synchronizedSet(new HashSet<>());

    public static void main(String[] args) throws IOException {
        ServerSocket serverSocket = new ServerSocket(5000);
        System.out.println("Multi-client server started on port 5000");

        while (true) {
            Socket clientSocket = serverSocket.accept();
            System.out.println("New client connected!");

            DataOutputStream dos = new DataOutputStream(
                clientSocket.getOutputStream()
            );
            clientStreams.add(dos);

            // Handle this client in a new thread
            new Thread(new ClientHandler(clientSocket, dos)).start();
        }
    }

    /** Sends a message to ALL connected clients */
    static void broadcast(String message, DataOutputStream sender) {
        synchronized (clientStreams) {
            for (DataOutputStream dos : clientStreams) {
                if (dos != sender) {  // Don't echo back to sender
                    try {
                        dos.writeUTF(message);
                        dos.flush();
                    } catch (IOException e) {
                        // Client disconnected — will be cleaned up
                    }
                }
            }
        }
    }
}

class ClientHandler implements Runnable {
    private Socket socket;
    private DataOutputStream myOutputStream;

    ClientHandler(Socket socket, DataOutputStream dos) {
        this.socket = socket;
        this.myOutputStream = dos;
    }

    @Override
    public void run() {
        try {
            DataInputStream dis = new DataInputStream(
                socket.getInputStream()
            );
            while (true) {
                String message = dis.readUTF();
                System.out.println("Received: " + message);
                MultiClientServer.broadcast(message, myOutputStream);
            }
        } catch (IOException e) {
            System.out.println("Client disconnected.");
            MultiClientServer.clientStreams.remove(myOutputStream);
        }
    }
}
```

### Exercise 4: File Transfer 📁

Add the ability to send files between the client and server. Use `DataOutputStream.writeInt()` for the file size, followed by writing the file bytes in chunks.

---

## 16. Production Best Practices

If you're taking socket programming into a real project, keep these in mind:

### Thread Safety with Swing

Swing is **not thread-safe**. When updating the GUI from the reader thread, use `SwingUtilities`:

```java
// ❌ WRONG — updating Swing from a non-EDT thread
messageArea.append(message + "\n");

// ✅ CORRECT — schedule update on the Event Dispatch Thread
javax.swing.SwingUtilities.invokeLater(() -> {
    messageArea.append(message + "\n");
    messageArea.setCaretPosition(messageArea.getDocument().getLength());
});
```

### Resource Management Checklist

| Resource | When to Close | How |
|----------|--------------|-----|
| `Socket` | When communication is done | `socket.close()` or try-with-resources |
| `ServerSocket` | When server shuts down | `serverSocket.close()` |
| `DataInputStream` | Before closing socket | `dis.close()` |
| `DataOutputStream` | Before closing socket | `dos.close()` (also flushes) |
| `Thread` | When no longer needed | Use `interrupt()` + check `isInterrupted()` |

### Architecture Recommendations

```
┌──────────────────────────────────────────────────────────────────┐
│              LEVELING UP YOUR SOCKET APPLICATION                    │
├──────────────────────────────────────────────────────────────────┤
│                                                                      │
│  BEGINNER → What we built today                                     │
│  • Single server, single client                                     │
│  • Direct socket communication                                     │
│  • Blocking I/O with threads                                        │
│                                                                      │
│  INTERMEDIATE → Production-ready                                    │
│  • Multi-client server with thread pool (ExecutorService)           │
│  • Protocol design (message types, headers, payloads)               │
│  • Connection heartbeats and timeouts                               │
│  • Proper Swing threading (SwingUtilities.invokeLater)              │
│                                                                      │
│  ADVANCED → Enterprise-grade                                        │
│  • Java NIO (Non-blocking I/O with Selectors)                       │
│  • Netty framework for high-performance networking                  │
│  • WebSockets for browser-based chat                                │
│  • Message queues (RabbitMQ, Kafka) for scalability                 │
│  • TLS/SSL encryption for security                                  │
│                                                                      │
└──────────────────────────────────────────────────────────────────┘
```

---

## 17. Quick Reference Card

### Server Setup Pattern

```java
// 1. Create ServerSocket
ServerSocket server = new ServerSocket(PORT);

// 2. Wait for client
Socket client = server.accept();

// 3. Set up streams
DataInputStream in = new DataInputStream(client.getInputStream());
DataOutputStream out = new DataOutputStream(client.getOutputStream());

// 4. Communicate
String msg = in.readUTF();      // Read
out.writeUTF("response");      // Write
out.flush();                     // Send

// 5. Cleanup
in.close(); out.close(); client.close(); server.close();
```

### Client Setup Pattern

```java
// 1. Connect to server
Socket socket = new Socket("server_ip", PORT);

// 2. Set up streams
DataOutputStream out = new DataOutputStream(socket.getOutputStream());
DataInputStream in = new DataInputStream(socket.getInputStream());

// 3. Communicate
out.writeUTF("hello");          // Write
out.flush();                     // Send
String reply = in.readUTF();    // Read

// 4. Cleanup
out.close(); in.close(); socket.close();
```

### Key Classes Cheat Sheet

| Class | Package | Role |
|-------|---------|------|
| `ServerSocket` | `java.net` | Listens for client connections on a port |
| `Socket` | `java.net` | Represents one end of a TCP connection |
| `DataInputStream` | `java.io` | Reads Java primitives from a stream |
| `DataOutputStream` | `java.io` | Writes Java primitives to a stream |
| `InetAddress` | `java.net` | Represents an IP address |
| `Thread` | `java.lang` | Enables concurrent execution |

---

## 18. Summary

Let's recap the complete journey we've taken:

| Section | What You Learned |
|---------|-----------------|
| **Sockets** | An endpoint = IP + Port; the foundation of network I/O |
| **TCP vs UDP** | TCP = reliable and ordered; UDP = fast but unreliable |
| **Three-Way Handshake** | SYN → SYN-ACK → ACK establishes the TCP connection |
| **ServerSocket** | Creates a listener that waits for incoming connections |
| **Socket** | Connects to a server; provides I/O streams for communication |
| **DataStreams** | `writeUTF()`/`readUTF()` for sending/receiving strings over sockets |
| **Blocking I/O** | `accept()` and `readUTF()` block until data is available |
| **Multithreading** | Separate threads for reading and writing prevent GUI freezes |
| **GUI Integration** | Swing components for a visual chat interface |
| **Error Handling** | Timeouts, graceful disconnection, resource cleanup |

### What's Next?

- 🔄 **Java NIO** — Non-blocking I/O for handling thousands of connections
- 🔐 **SSL/TLS Sockets** — Encrypted communication using `SSLSocket`
- 🌐 **WebSockets** — Real-time communication in web applications
- 📡 **RMI (Remote Method Invocation)** — Calling methods on remote objects

> **💡 Final Thought:** Every time you open a website, send a message, or stream a video — sockets are working behind the scenes. Now you know exactly how they work. Go build something amazing! 🚀
