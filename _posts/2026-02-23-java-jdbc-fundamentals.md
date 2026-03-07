---
title: "Java JDBC Part 1: Connections, Statements, and ResultSets"
description: Learn the fundamentals of Java Database Connectivity — how to connect to a database, execute SQL, process results, handle exceptions, and understand the Coffee Break sample database
author: Vaibhav Gagneja
date: 2026-02-23 12:00:00 +0530
categories: [Development, Java]
tags: [java, jdbc, database, sql, mysql, derby, resultset, connections]
toc: true
image:
  path: https://images.unsplash.com/photo-1544383835-bda2bc66a55d
---

Most Java applications need to talk to a database at some point — whether it's storing user profiles, reading product catalogs, or running analytics queries. Java's **JDBC (Java Database Connectivity)** API is the standard way to make that happen.

In this three-part series, we'll cover everything from establishing your first database connection to advanced topics like connection pooling and distributed transactions. In Part 1, we start with the fundamentals: connecting to a database, executing SQL, and processing results.

---

## 1. What Is JDBC?

JDBC is a **Java API** that lets your Java programs talk to relational databases. It provides a standard set of interfaces so that your code looks the same regardless of whether you're using MySQL, PostgreSQL, Oracle, Derby, or any other RDBMS.

```
┌──────────────────────────────────────────────────────────────┐
│                    WHAT JDBC GIVES YOU                         │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Your Java Application                                         │
│       │                                                        │
│       │  Uses JDBC API (java.sql / javax.sql)                  │
│       ▼                                                        │
│  ┌──────────────────────────────────────────────────────┐     │
│  │              JDBC Driver Manager                      │     │
│  └──────────────┬──────────────┬──────────────┬─────────┘     │
│                 │              │              │                 │
│                 ▼              ▼              ▼                 │
│          ┌──────────┐  ┌──────────┐  ┌──────────────┐         │
│          │  MySQL   │  │  Derby   │  │  PostgreSQL  │         │
│          │  Driver  │  │  Driver  │  │    Driver    │         │
│          └────┬─────┘  └────┬─────┘  └──────┬───────┘         │
│               │             │               │                  │
│               ▼             ▼               ▼                  │
│          ┌──────────┐  ┌──────────┐  ┌──────────────┐         │
│          │  MySQL   │  │  Derby   │  │  PostgreSQL  │         │
│          │   DB     │  │   DB     │  │     DB       │         │
│          └──────────┘  └──────────┘  └──────────────┘         │
│                                                                │
│  Key insight: YOUR CODE only talks to the JDBC API.            │
│  The driver handles the database-specific details.             │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

The beauty of JDBC is that you write your code once against the JDBC API, and you can switch databases just by swapping drivers — your application code stays the same.

---

## 2. JDBC Driver Types

Not all JDBC drivers are built the same. There are four categories:

| Type | Name | How It Works | Example |
|------|------|-------------|---------|
| **Type 1** | JDBC-ODBC Bridge | Maps JDBC calls to ODBC calls. Requires a native ODBC library on the client machine. | JDBC-ODBC Bridge (deprecated) |
| **Type 2** | Native-API | Written partly in Java, partly in native code. Uses a database-specific client library. | Oracle OCI Driver |
| **Type 3** | Network Protocol | Pure Java client that talks to a middleware server, which then talks to the database. | Some application server drivers |
| **Type 4** | Thin / Direct-to-Database | Pure Java. Talks directly to the database using the database's network protocol. | MySQL Connector/J, Derby Embedded |

> **Note:** Type 4 drivers are the most common and the easiest to use — just drop a JAR file into your classpath and you're ready to go. Both MySQL Connector/J and the Derby drivers are Type 4.

---

## 3. The Five Steps of JDBC

Every database interaction in JDBC follows the same pattern:

```
┌──────────────────────────────────────────────────────────────┐
│               THE FIVE STEPS OF JDBC                           │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Step 1:  ESTABLISH A CONNECTION                               │
│           DriverManager.getConnection(url, user, password)     │
│                     │                                          │
│                     ▼                                          │
│  Step 2:  CREATE A STATEMENT                                   │
│           con.createStatement()                                │
│           con.prepareStatement(sql)                            │
│                     │                                          │
│                     ▼                                          │
│  Step 3:  EXECUTE THE QUERY                                    │
│           stmt.executeQuery(sql)     → returns ResultSet       │
│           stmt.executeUpdate(sql)    → returns int (row count) │
│                     │                                          │
│                     ▼                                          │
│  Step 4:  PROCESS THE RESULTS                                  │
│           while (rs.next()) { rs.getString("COL")... }         │
│                     │                                          │
│                     ▼                                          │
│  Step 5:  CLOSE THE CONNECTION                                 │
│           Use try-with-resources to auto-close                 │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

Here's a complete example. This method queries a table called `COFFEES` and prints every row:

```java
public static void viewTable(Connection con) throws SQLException {
    String query = "select COF_NAME, SUP_ID, PRICE, SALES, TOTAL from COFFEES";
    try (Statement stmt = con.createStatement()) {
        ResultSet rs = stmt.executeQuery(query);
        while (rs.next()) {
            String coffeeName = rs.getString("COF_NAME");
            int supplierID = rs.getInt("SUP_ID");
            float price = rs.getFloat("PRICE");
            int sales = rs.getInt("SALES");
            int total = rs.getInt("TOTAL");
            System.out.println(coffeeName + ", " + supplierID + ", " +
                               price + ", " + sales + ", " + total);
        }
    } catch (SQLException e) {
        JDBCTutorialUtilities.printSQLException(e);
    }
}
```

Notice the `try-with-resources` — the `Statement` is automatically closed when the block exits, even if an exception occurs. The `ResultSet` created from that statement is also closed automatically.

---

## 4. Establishing a Connection

Before you can do anything with a database, you need a `Connection` object. There are two ways to get one:

| Approach | Class | When to Use |
|----------|-------|-------------|
| **DriverManager** | `java.sql.DriverManager` | Simple applications, tutorials, quick prototyping |
| **DataSource** | `javax.sql.DataSource` | Production applications (supports pooling, distributed transactions) |

We cover `DataSource` in Part 3. For now, let's use `DriverManager`.

### Using DriverManager

The `DriverManager` class connects to a database using a **JDBC URL**. Here's how:

```java
public Connection getConnection() throws SQLException {
    Connection conn = null;
    Properties connectionProps = new Properties();
    connectionProps.put("user", this.userName);
    connectionProps.put("password", this.password);

    if (this.dbms.equals("mysql")) {
        conn = DriverManager.getConnection(
                   "jdbc:mysql://" + this.serverName +
                   ":" + this.portNumber + "/",
                   connectionProps);
    } else if (this.dbms.equals("derby")) {
        conn = DriverManager.getConnection(
                   "jdbc:derby:" + this.dbName +
                   ";create=true",
                   connectionProps);
    }
    System.out.println("Connected to database");
    return conn;
}
```

### Database Connection URLs

The URL format varies by database:

```
┌──────────────────────────────────────────────────────────────┐
│               DATABASE CONNECTION URL FORMATS                  │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  MySQL:                                                        │
│  jdbc:mysql://localhost:3306/mydb                               │
│  ─┬──  ─┬─── ────┬──── ─┬── ─┬──                              │
│   │     │        │      │    └── database name                 │
│   │     │        │      └─────── port (default: 3306)          │
│   │     │        └────────────── hostname                      │
│   │     └─────────────────────── sub-protocol (driver)         │
│   └───────────────────────────── protocol (always "jdbc")      │
│                                                                │
│  Derby (Embedded):                                             │
│  jdbc:derby:testdb;create=true                                 │
│  ─┬──  ──┬── ──┬── ─────┬─────                                │
│   │      │     │        └──── attribute: create if not exists  │
│   │      │     └─────────────  database name                   │
│   │      └───────────────────  sub-protocol                    │
│   └──────────────────────────  protocol                        │
│                                                                │
│  Derby (Network Client):                                       │
│  jdbc:derby://localhost:1527/mydb                               │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

> **Note:** With JDBC 4.0+, drivers are auto-loaded from the classpath. You no longer need to call `Class.forName("com.mysql.cj.jdbc.Driver")` — just make sure the driver JAR is on your classpath and `DriverManager` will find it.

---

## 5. Creating and Executing Statements

Once you have a connection, you create a `Statement` to send SQL to the database. JDBC provides three types:

| Statement Type | Use Case | Supports Parameters? |
|---------------|----------|---------------------|
| `Statement` | Simple SQL with no parameters | No |
| `PreparedStatement` | Precompiled SQL with input parameters. Prevents SQL injection. | Yes |
| `CallableStatement` | Execute stored procedures. Can have both input and output parameters. | Yes |

`PreparedStatement` extends `Statement`, and `CallableStatement` extends `PreparedStatement`, so each level adds capabilities on top of the previous one.

### Creating a Statement

```java
Statement stmt = con.createStatement();
```

### Executing: Which Method to Use?

| Method | Returns | Use When |
|--------|---------|----------|
| `executeQuery(sql)` | `ResultSet` | `SELECT` statements — you expect rows back |
| `executeUpdate(sql)` | `int` (row count) | `INSERT`, `UPDATE`, `DELETE`, or DDL (`CREATE TABLE`, etc.) |
| `execute(sql)` | `boolean` | When the query might return multiple `ResultSet` objects |

```java
// SELECT — returns data
ResultSet rs = stmt.executeQuery("SELECT * FROM COFFEES");

// INSERT — returns number of rows inserted
int count = stmt.executeUpdate(
    "INSERT INTO COFFEES VALUES('Kona', 150, 10.99, 0, 0)");

// DDL — returns 0 (no rows affected)
int n = stmt.executeUpdate(
    "CREATE TABLE TEMP (ID INT, NAME VARCHAR(40))");
```

> **Important:** When `executeUpdate` returns 0, it could mean either (a) the statement affected zero rows, or (b) it was a DDL statement. The return value alone doesn't tell you which — you need to know what statement you ran.

---

## 6. Working with ResultSet Objects

A `ResultSet` is a table of data returned by a query. You navigate it using a **cursor** — a pointer that starts *before* the first row and moves forward as you call `next()`.

```
┌──────────────────────────────────────────────────────────────┐
│                  ResultSet CURSOR MOVEMENT                     │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Initial position: BEFORE first row                            │
│                                                                │
│       ──► next() ──► next() ──► next() ──► next() ───►        │
│       │           │           │           │           │        │
│  [before]   [Row 1]     [Row 2]     [Row 3]    [after]        │
│  first row  COF_NAME:   COF_NAME:   COF_NAME:  last row       │
│             Colombian   French_R    Espresso                   │
│                                                                │
│  next() returns TRUE when positioned on a valid row            │
│  next() returns FALSE when past the last row                   │
│                                                                │
│  Typical pattern:                                              │
│  while (rs.next()) {                                           │
│      // process current row                                    │
│  }                                                             │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### ResultSet Types

You can control the cursor's capabilities when creating the statement:

| Type | Scrolling | Sensitivity to DB Changes |
|------|-----------|--------------------------|
| `TYPE_FORWARD_ONLY` (default) | Forward only (`next()`) | Depends on implementation |
| `TYPE_SCROLL_INSENSITIVE` | Forward, backward, absolute | Does NOT reflect changes made by others |
| `TYPE_SCROLL_SENSITIVE` | Forward, backward, absolute | Reflects changes made by others |

### ResultSet Concurrency

| Concurrency | Can Update Through ResultSet? |
|------------|------------------------------|
| `CONCUR_READ_ONLY` (default) | No — read-only |
| `CONCUR_UPDATABLE` | Yes — can update, insert, delete rows |

You set both when creating a statement:

```java
Statement stmt = con.createStatement(
    ResultSet.TYPE_SCROLL_SENSITIVE,
    ResultSet.CONCUR_UPDATABLE
);
```

### Cursor Holdability

What happens to open ResultSets when you call `commit()`?

| Holdability | Behavior on Commit |
|------------|-------------------|
| `HOLD_CURSORS_OVER_COMMIT` | ResultSet stays open after commit |
| `CLOSE_CURSORS_AT_COMMIT` | ResultSet is closed when commit is called |

The default varies by database. You can check what your database supports:

```java
DatabaseMetaData dbMetaData = conn.getMetaData();
System.out.println("Default holdability: " +
    dbMetaData.getResultSetHoldability());
System.out.println("Supports HOLD_CURSORS_OVER_COMMIT? " +
    dbMetaData.supportsResultSetHoldability(
        ResultSet.HOLD_CURSORS_OVER_COMMIT));
```

---

## 7. Retrieving Column Values

The `ResultSet` interface provides getter methods for every Java type. You can retrieve values by **column index** (1-based) or by **column name/alias**:

```java
// By column name (more readable)
String coffeeName = rs.getString("COF_NAME");
int supplierID    = rs.getInt("SUP_ID");
float price       = rs.getFloat("PRICE");

// By column index (more efficient)
String coffeeName = rs.getString(1);
int supplierID    = rs.getInt(2);
float price       = rs.getFloat(3);
```

### Common Getter Methods

| SQL Type | Getter Method | Java Type |
|----------|--------------|-----------|
| `VARCHAR`, `CHAR` | `getString()` | `String` |
| `INTEGER` | `getInt()` | `int` |
| `FLOAT`, `DOUBLE` | `getFloat()`, `getDouble()` | `float`, `double` |
| `NUMERIC`, `DECIMAL` | `getBigDecimal()` | `BigDecimal` |
| `DATE` | `getDate()` | `java.sql.Date` |
| `TIMESTAMP` | `getTimestamp()` | `java.sql.Timestamp` |
| `BOOLEAN` | `getBoolean()` | `boolean` |
| `BLOB` | `getBlob()` | `Blob` |
| `CLOB` | `getClob()` | `Clob` |

> **Note:** `getString()` can technically retrieve *any* SQL type — the driver will convert it to a String. This can be convenient, but you lose type safety and may need to convert back to a numeric type for calculations.

### Best Practices for Reading Columns

- Read columns **left to right** within each row for maximum portability
- Read each column **only once** per row
- If you use column names, be aware that when multiple columns share the same alias, the first match is returned — use column indices to be specific

---

## 8. The Coffee Break Database

The tutorial's sample application models a small coffee house called **The Coffee Break**. The database has the following tables:

### COFFEES Table

| COF_NAME | SUP_ID | PRICE | SALES | TOTAL |
|----------|--------|-------|-------|-------|
| Colombian | 101 | 7.99 | 0 | 0 |
| French_Roast | 49 | 8.99 | 0 | 0 |
| Espresso | 150 | 9.99 | 0 | 0 |
| Colombian_Decaf | 101 | 8.99 | 0 | 0 |
| French_Roast_Decaf | 49 | 9.99 | 0 | 0 |

```
┌──────────────────────────────────────────────────────────────┐
│                   COFFEES TABLE SCHEMA                         │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Column       Type           Constraint                        │
│  ──────────   ────────────   ─────────────────────             │
│  COF_NAME     VARCHAR(32)    PRIMARY KEY                       │
│  SUP_ID       INTEGER        FOREIGN KEY → SUPPLIERS(SUP_ID)  │
│  PRICE        NUMERIC(10,2)  NOT NULL                          │
│  SALES        INTEGER        NOT NULL                          │
│  TOTAL        INTEGER        NOT NULL                          │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### SUPPLIERS Table

| SUP_ID | SUP_NAME | STREET | CITY | STATE | ZIP |
|--------|----------|--------|------|-------|-----|
| 101 | Acme, Inc. | 99 Market Street | Groundsville | CA | 95199 |
| 49 | Superior Coffee | 1 Party Place | Mendocino | CA | 95460 |
| 150 | The High Ground | 100 Coffee Lane | Meadows | CA | 93966 |

### Other Tables

The database also includes:

| Table | Purpose |
|-------|---------|
| `COF_INVENTORY` | Tracks warehouse stock for each coffee type |
| `MERCH_INVENTORY` | Tracks non-coffee merchandise (cups, saucers, etc.) |
| `COFFEE_HOUSES` | Store locations with sales figures |
| `DATA_REPOSITORY` | Stores URLs referencing documents and other data |

### Creating the Tables

```sql
CREATE TABLE SUPPLIERS (
    SUP_ID    INTEGER     NOT NULL,
    SUP_NAME  VARCHAR(40) NOT NULL,
    STREET    VARCHAR(40) NOT NULL,
    CITY      VARCHAR(20) NOT NULL,
    STATE     CHAR(2)     NOT NULL,
    ZIP       CHAR(5),
    PRIMARY KEY (SUP_ID)
);

CREATE TABLE COFFEES (
    COF_NAME VARCHAR(32)   NOT NULL,
    SUP_ID   INT           NOT NULL,
    PRICE    NUMERIC(10,2) NOT NULL,
    SALES    INTEGER       NOT NULL,
    TOTAL    INTEGER       NOT NULL,
    PRIMARY KEY (COF_NAME),
    FOREIGN KEY (SUP_ID) REFERENCES SUPPLIERS (SUP_ID)
);
```

> **Important:** You must create `SUPPLIERS` before `COFFEES` because `COFFEES` has a foreign key referencing `SUPPLIERS`.

### Creating Tables with JDBC

```java
public void createTable() throws SQLException {
    String createString =
        "CREATE TABLE SUPPLIERS " +
        "(SUP_ID INTEGER NOT NULL, " +
        "SUP_NAME VARCHAR(40) NOT NULL, " +
        "STREET VARCHAR(40) NOT NULL, " +
        "CITY VARCHAR(20) NOT NULL, " +
        "STATE CHAR(2) NOT NULL, " +
        "ZIP CHAR(5), " +
        "PRIMARY KEY (SUP_ID))";

    try (Statement stmt = con.createStatement()) {
        stmt.executeUpdate(createString);
    } catch (SQLException e) {
        JDBCTutorialUtilities.printSQLException(e);
    }
}
```

### Populating Tables

```java
public void populateTable() throws SQLException {
    try (Statement stmt = con.createStatement()) {
        stmt.executeUpdate(
            "INSERT INTO SUPPLIERS " +
            "VALUES(49, 'Superior Coffee', '1 Party Place', " +
            "'Mendocino', 'CA', '95460')");
        stmt.executeUpdate(
            "INSERT INTO SUPPLIERS " +
            "VALUES(101, 'Acme, Inc.', '99 Market Street', " +
            "'Groundsville', 'CA', '95199')");
        stmt.executeUpdate(
            "INSERT INTO SUPPLIERS " +
            "VALUES(150, 'The High Ground', '100 Coffee Lane', " +
            "'Meadows', 'CA', '93966')");
    } catch (SQLException e) {
        JDBCTutorialUtilities.printSQLException(e);
    }
}
```

---

## 9. Handling SQL Exceptions

When JDBC encounters an error, it throws a `SQLException`. This isn't just a regular exception — it carries extra diagnostic information:

```
┌──────────────────────────────────────────────────────────────┐
│                  ANATOMY OF A SQLException                     │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌──────────────────────────────────────────────────────┐     │
│  │  getMessage()                                         │     │
│  │  "Table 'TESTDB.COFFEES' already exists"             │     │
│  ├──────────────────────────────────────────────────────┤     │
│  │  getSQLState()                                        │     │
│  │  "42Y55"  (ISO/ANSI standard 5-character code)       │     │
│  ├──────────────────────────────────────────────────────┤     │
│  │  getErrorCode()                                       │     │
│  │  30000  (vendor-specific error number)               │     │
│  ├──────────────────────────────────────────────────────┤     │
│  │  getCause()                                           │     │
│  │  Another Throwable that caused this exception        │     │
│  ├──────────────────────────────────────────────────────┤     │
│  │  getNextException()                                   │     │
│  │  Next SQLException in the chain (if multiple errors) │     │
│  └──────────────────────────────────────────────────────┘     │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Printing Full Exception Details

Here's a utility method that walks the entire exception chain:

```java
public static void printSQLException(SQLException ex) {
    for (Throwable e : ex) {
        if (e instanceof SQLException) {
            if (!ignoreSQLException(((SQLException)e).getSQLState())) {
                e.printStackTrace(System.err);
                System.err.println("SQLState: " +
                    ((SQLException)e).getSQLState());
                System.err.println("Error Code: " +
                    ((SQLException)e).getErrorCode());
                System.err.println("Message: " + e.getMessage());

                Throwable t = ex.getCause();
                while (t != null) {
                    System.out.println("Cause: " + t);
                    t = t.getCause();
                }
            }
        }
    }
}
```

### Ignoring Certain Exceptions

Sometimes you want to silently handle expected errors (like trying to drop a table that doesn't exist):

```java
public static boolean ignoreSQLException(String sqlState) {
    if (sqlState == null) {
        System.out.println("The SQL state is not defined!");
        return false;
    }
    // X0Y32: Jar file already exists in schema
    if (sqlState.equalsIgnoreCase("X0Y32")) return true;
    // 42Y55: Table already exists in schema
    if (sqlState.equalsIgnoreCase("42Y55")) return true;
    return false;
}
```

### SQL Warnings

`SQLWarning` is a subclass of `SQLException` for non-fatal issues. Warnings don't stop execution — they just alert you that something didn't go as planned:

```java
public static void printWarnings(SQLWarning warning) throws SQLException {
    if (warning != null) {
        System.out.println("\n---Warning---\n");
        while (warning != null) {
            System.out.println("Message: " + warning.getMessage());
            System.out.println("SQLState: " + warning.getSQLState());
            System.out.println("Vendor code: " + warning.getErrorCode());
            warning = warning.getNextWarning();
        }
    }
}
```

You retrieve warnings from the object that generated them:

```java
// Warnings from a Statement
SQLWarning warn = stmt.getWarnings();

// Warnings from a ResultSet
SQLWarning warn = rs.getWarnings();

// Warnings from a Connection
SQLWarning warn = con.getWarnings();
```

> **Note:** Executing a new statement automatically clears warnings from the previous statement. If you need to check for warnings, do it before executing the next statement.

### Categorized SQLExceptions

JDBC provides more specific subclasses for common error categories:

| Subclass | Meaning |
|----------|---------|
| `SQLNonTransientException` | Retrying the same operation will fail again (bad SQL syntax, table doesn't exist) |
| `SQLTransientException` | The error might go away if you retry (timeout, temporary resource issue) |
| `SQLRecoverableException` | The application can recover by retrying the operation or getting a new connection |
| `BatchUpdateException` | Error during a batch update — includes the update counts for all statements that ran before the failure |
| `SQLClientInfoException` | One or more client info properties couldn't be set on the connection |

---

## 10. Closing Resources — The Right Way

Leaving database connections, statements, or result sets open causes **resource leaks** — your application holds onto database handles, memory, and network connections unnecessarily.

### The Old Way (finally block)

```java
Statement stmt = null;
try {
    stmt = con.createStatement();
    ResultSet rs = stmt.executeQuery(query);
    // ... process results ...
} catch (SQLException e) {
    // handle error
} finally {
    if (stmt != null) stmt.close();  // always close!
}
```

### The Modern Way (try-with-resources)

```java
try (Statement stmt = con.createStatement()) {
    ResultSet rs = stmt.executeQuery(query);
    while (rs.next()) {
        // ... process results ...
    }
}  // stmt is auto-closed here, even if an exception was thrown
```

The `try-with-resources` approach is shorter, safer, and the recommended practice since Java 7.

```
┌──────────────────────────────────────────────────────────────┐
│              RESOURCE CLOSING ORDER                            │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  When a Statement is closed:                                   │
│    → Its ResultSet is also closed automatically                │
│                                                                │
│  When a Connection is closed:                                  │
│    → All its Statements are also closed                        │
│    → All ResultSets from those Statements are also closed      │
│                                                                │
│  But don't rely on this! Explicitly close each resource        │
│  as soon as you're done with it. The sooner you release        │
│  resources, the better your application performs.               │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## 11. Summary

| Concept | Key Takeaway |
|---------|-------------|
| **JDBC** | A standard Java API for database access — write once, connect to any database |
| **Driver Types** | Type 4 (pure Java) is most common; just add a JAR to your classpath |
| **Connection** | Use `DriverManager.getConnection(url, props)` for simple apps; `DataSource` for production |
| **Statement Types** | `Statement` for simple SQL, `PreparedStatement` for parameterized queries, `CallableStatement` for stored procedures |
| **Execute Methods** | `executeQuery()` for SELECT, `executeUpdate()` for INSERT/UPDATE/DELETE/DDL |
| **ResultSet** | A cursor-based table of results; navigate with `next()`, read with getter methods |
| **Exceptions** | `SQLException` carries SQLState, error code, message, and chained exceptions |
| **Resource Management** | Always close resources — prefer `try-with-resources` |

In **Part 2**, we'll dig deeper into `PreparedStatement` for parameterized queries and SQL injection prevention, transactions with commit/rollback/savepoints, and batch updates for efficient bulk operations.
