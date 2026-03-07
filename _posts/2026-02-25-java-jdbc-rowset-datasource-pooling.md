---
title: "Java JDBC Part 3: RowSet Objects, DataSource, and Connection Pooling"
description: Explore connected and disconnected RowSets (JdbcRowSet, CachedRowSet, WebRowSet, JoinRowSet, FilteredRowSet), DataSource objects for production-grade connections, and connection pooling for high-performance applications
author: Vaibhav Gagneja
date: 2026-02-25 12:00:00 +0530
categories: [Development, Java]
tags: [java, jdbc, database, rowset, datasource, connection-pooling, cachedrowset, webrowset]
toc: true
image:
  path: https://images.unsplash.com/photo-1551288049-bebda4e38f71
---

In [Part 1](/posts/java-jdbc-fundamentals/) we covered the JDBC fundamentals, and in [Part 2](/posts/java-jdbc-transactions-prepared-statements/) we mastered prepared statements and transactions. Now we tackle the more advanced side of JDBC: **RowSet objects** that bring flexibility and JavaBeans compliance to your data access, **DataSource** for production-grade connection management, and **connection pooling** for performance at scale.

---

## 1. What Are RowSet Objects?

A `RowSet` is a wrapper around a `ResultSet` that adds several capabilities:

```
┌──────────────────────────────────────────────────────────────┐
│             ResultSet vs RowSet                                │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  ResultSet:                                                    │
│  ──────────                                                    │
│  - Tied to a Statement and Connection                          │
│  - Must keep connection open while reading                     │
│  - Not a JavaBean                                              │
│  - Not serializable                                            │
│  - Forward-only by default                                     │
│                                                                │
│  RowSet:                                                       │
│  ────────                                                      │
│  - Is a JavaBean (properties + events)                         │
│  - Scrollable and updatable by default                         │
│  - Serializable — can pass over a network                      │
│  - Can work WITHOUT a live database connection                 │
│  - Supports event notifications (listeners)                    │
│                                                                │
│  RowSet extends ResultSet, so everything you know              │
│  about ResultSet still applies.                                │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### JavaBeans Properties

Because a `RowSet` is a JavaBean, you can set its properties individually instead of passing everything to a constructor:

```java
// Traditional JDBC: everything wired together
Connection con = DriverManager.getConnection(url, user, pass);
Statement stmt = con.createStatement();
ResultSet rs = stmt.executeQuery(query);

// RowSet: set properties individually
JdbcRowSet jrs = RowSetProvider.newFactory().createJdbcRowSet();
jrs.setUrl(url);
jrs.setUsername(user);
jrs.setPassword(pass);
jrs.setCommand(query);
jrs.execute();    // connects, executes, and returns results
```

### Event Notification

RowSet objects fire events when certain things happen, and any registered listener is notified:

| Event | When It Fires |
|-------|--------------|
| `cursorMoved` | The cursor moves to a different row |
| `rowChanged` | A row has been inserted, updated, or deleted |
| `rowSetChanged` | The entire RowSet contents have changed (e.g., new query executed) |

This is useful in GUI applications — a table display component can listen for changes and refresh itself automatically.

---

## 2. Connected vs Disconnected RowSets

This is the key architectural distinction:

```
┌──────────────────────────────────────────────────────────────┐
│          CONNECTED vs DISCONNECTED RowSets                     │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  CONNECTED:                                                    │
│  ┌──────────┐     always connected     ┌──────────────┐      │
│  │ JdbcRowSet│ ◄───────────────────────►│   Database   │      │
│  └──────────┘                           └──────────────┘      │
│  Changes go to the database immediately.                       │
│  Behaves like a ResultSet with added features.                 │
│                                                                │
│  DISCONNECTED:                                                 │
│  ┌────────────┐  connect   ┌──────────────┐                   │
│  │CachedRowSet│───────────►│   Database   │                   │
│  │            │  fetch data └──────────────┘                   │
│  │            │◄──── data ──────┘                              │
│  │            │  disconnect                                    │
│  │   (works   │                                                │
│  │  offline)  │                                                │
│  │            │  reconnect  ┌──────────────┐                   │
│  │            │────────────►│   Database   │                   │
│  │            │  sync       └──────────────┘                   │
│  └────────────┘  changes                                       │
│                                                                │
│  Uses a connection only briefly — to fetch and to sync.        │
│  Can work with data completely offline.                        │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

| Feature | Connected (`JdbcRowSet`) | Disconnected (`CachedRowSet`, etc.) |
|---------|------------------------|-------------------------------------|
| **Connection** | Always open | Opens briefly, then disconnects |
| **Memory** | Low — data stays in DB | Higher — data cached in memory |
| **Changes** | Immediate to database | Buffered, synced later with `acceptChanges()` |
| **Thin client** | No — needs live connection | Yes — ideal for mobile, web |
| **Serializable** | No | Yes — can pass over network |

### The RowSet Family

| Type | Connected? | Primary Use |
|------|-----------|-------------|
| `JdbcRowSet` | Yes | Enhanced ResultSet with JavaBeans support |
| `CachedRowSet` | No | In-memory data cache, works offline |
| `WebRowSet` | No | XML import/export of tabular data |
| `JoinRowSet` | No | SQL JOIN across multiple disconnected RowSets |
| `FilteredRowSet` | No | Filtered views without re-querying the database |

---

## 3. Using JdbcRowSet

A `JdbcRowSet` is the simplest RowSet — it's essentially a `ResultSet` with JavaBeans compliance. It's always connected to the database, and all operations go directly to the database.

### Creating a JdbcRowSet

Use `RowSetProvider` (the modern approach):

```java
// From scratch — set properties, then execute
JdbcRowSet jdbcRs = RowSetProvider.newFactory().createJdbcRowSet();
jdbcRs.setUrl("jdbc:mysql://localhost:3306/testdb");
jdbcRs.setUsername("root");
jdbcRs.setPassword("secret");
jdbcRs.setCommand("SELECT * FROM COFFEES");
jdbcRs.execute();

// From an existing ResultSet
JdbcRowSet jdbcRs = RowSetProvider.newFactory().createJdbcRowSet(rs);
```

### Default Capabilities

Unlike a plain `ResultSet` where you specify scrollability and updatability when creating a statement, a `JdbcRowSet` comes with these defaults:

| Property | Default |
|----------|---------|
| **Type** | `TYPE_SCROLL_INSENSITIVE` (scrollable) |
| **Concurrency** | `CONCUR_UPDATABLE` (can modify rows) |
| **Escape processing** | Enabled |

### Navigating and Reading

```java
jdbcRs.execute();

// Forward iteration (same as ResultSet)
while (jdbcRs.next()) {
    System.out.println(jdbcRs.getString("COF_NAME") + " " +
                       jdbcRs.getFloat("PRICE"));
}

// Move to a specific row (scrollable!)
jdbcRs.absolute(3);           // go to row 3
System.out.println(jdbcRs.getString("COF_NAME"));

jdbcRs.previous();             // go back one row
jdbcRs.first();                // go to first row
jdbcRs.last();                 // go to last row
```

### Updating Data

Because a `JdbcRowSet` is connected and updatable by default, changes go directly to the database:

```java
jdbcRs.absolute(3);                      // move to row 3
jdbcRs.updateFloat("PRICE", 12.99f);     // change the price
jdbcRs.updateRow();                       // write to database immediately
```

### Inserting and Deleting

```java
// Insert
jdbcRs.moveToInsertRow();
jdbcRs.updateString("COF_NAME", "House_Blend");
jdbcRs.updateInt("SUP_ID", 49);
jdbcRs.updateFloat("PRICE", 7.99f);
jdbcRs.updateInt("SALES", 0);
jdbcRs.updateInt("TOTAL", 0);
jdbcRs.insertRow();
jdbcRs.moveToCurrentRow();

// Delete
jdbcRs.last();
jdbcRs.deleteRow();
```

---

## 4. Using CachedRowSet

A `CachedRowSet` is the workhorse of disconnected RowSets. It fetches data into memory, disconnects from the database, lets you work with the data offline, and then syncs changes back when you're ready.

### Why Use CachedRowSet?

```
┌──────────────────────────────────────────────────────────────┐
│             WHEN TO USE CachedRowSet                           │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Scenario 1: THIN CLIENTS                                      │
│    PDA, mobile app, or embedded device with limited memory     │
│    → Fetch a page of data, disconnect, display to user         │
│    → User edits → reconnect briefly to sync changes            │
│                                                                │
│  Scenario 2: SENDING DATA OVER NETWORK                         │
│    You need to send query results to another tier              │
│    → CachedRowSet is serializable (ResultSet is NOT)           │
│    → Send it over RMI, sockets, or message queue               │
│                                                                │
│  Scenario 3: SCROLLABLE ResultSet NOT SUPPORTED               │
│    Your driver doesn't support scrollable ResultSets           │
│    → Wrap the forward-only ResultSet in a CachedRowSet         │
│    → Now you have full scrollability without driver support     │
│                                                                │
│  Scenario 4: REDUCE CONNECTION LOAD                            │
│    Database has limited connection slots                        │
│    → Hold the connection only while fetching                   │
│    → Release it immediately so others can use it               │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Setting Up a CachedRowSet

```java
CachedRowSet crs = RowSetProvider.newFactory().createCachedRowSet();
crs.setUrl("jdbc:mysql://localhost:3306/testdb");
crs.setUsername("root");
crs.setPassword("secret");
crs.setCommand("SELECT * FROM COFFEES");
crs.execute();    // fetches data, then DISCONNECTS
```

You can also populate it from an existing `ResultSet`:

```java
ResultSet rs = stmt.executeQuery("SELECT * FROM COFFEES");
CachedRowSet crs = RowSetProvider.newFactory().createCachedRowSet();
crs.populate(rs);    // copies all data into memory
```

### Controlling How Much Data to Fetch

For large tables, you don't want to load everything into memory:

```java
crs.setPageSize(20);    // fetch 20 rows at a time
crs.execute();          // fetches first 20 rows

// Process page 1...

while (crs.nextPage()) {
    // Process next 20 rows...
}
```

### The Reader and Writer

Internally, a `CachedRowSet` uses two helper objects:

| Object | Purpose |
|--------|---------|
| **SyncProvider Reader** | Reads data from the database into the `CachedRowSet` |
| **SyncProvider Writer** | Writes modified data from the `CachedRowSet` back to the database |

You generally don't interact with these directly — they work behind the scenes when you call `execute()` and `acceptChanges()`.

### Syncing Changes Back to the Database

After modifying data offline, call `acceptChanges()` to push changes to the database:

```java
// Modify data (disconnected — no live connection needed)
crs.absolute(3);
crs.updateFloat("PRICE", 11.99f);
crs.updateRow();

// Reconnect and sync
crs.acceptChanges();    // opens connection, writes changes, closes connection
```

If you used properties (`setUrl`, `setUsername`, `setPassword`) when creating the `CachedRowSet`, it will reconnect using those. Otherwise, you can pass a `Connection`:

```java
crs.acceptChanges(con);
```

### Handling Sync Conflicts

What if someone else modified the same row while you were disconnected? This is a **sync conflict**. The `CachedRowSet` provides a `SyncResolver` to handle them:

```java
try {
    crs.acceptChanges();
} catch (SyncProviderException spe) {
    SyncResolver resolver = spe.getSyncResolver();

    while (resolver.nextConflict()) {
        int status = resolver.getStatus();

        if (status == SyncResolver.UPDATE_ROW_CONFLICT) {
            int row = resolver.getRow();
            crs.absolute(row);

            // Compare your value with what's now in the database
            float yourPrice = crs.getFloat("PRICE");
            float dbPrice = resolver.getConflictValue("PRICE");

            // Decide which value wins
            if (yourPrice > dbPrice) {
                resolver.setResolvedValue("PRICE", yourPrice);
            } else {
                resolver.setResolvedValue("PRICE", dbPrice);
            }
        }
    }
}
```

```
┌──────────────────────────────────────────────────────────────┐
│                 SYNC CONFLICT RESOLUTION                       │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Time 1: You fetch PRICE = 9.99                               │
│  Time 2: Someone else changes PRICE to 10.99 in the DB        │
│  Time 3: You change PRICE to 11.99 in your CachedRowSet       │
│  Time 4: You call acceptChanges()                              │
│                                                                │
│  Conflict detected:                                            │
│    Your value:     11.99                                       │
│    Database value:  10.99  (changed since you fetched)         │
│    Original value:   9.99  (what was there when you fetched)   │
│                                                                │
│  You decide → resolver.setResolvedValue("PRICE", yourChoice)  │
│                                                                │
│  Conflict types:                                               │
│    UPDATE_ROW_CONFLICT  — you and someone else updated         │
│    DELETE_ROW_CONFLICT  — you updated, someone else deleted    │
│    INSERT_ROW_CONFLICT  — your insert conflicts with existing  │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Key Columns

For the writer to identify which rows to update in the database, you must set **key columns** — typically the primary key:

```java
int[] keys = {1};    // column 1 is the primary key
crs.setKeyColumns(keys);
```

---

## 5. WebRowSet — XML Representation

A `WebRowSet` extends `CachedRowSet` by adding the ability to read and write itself as an **XML document**. This makes it easy to transfer tabular data over HTTP or store query results as XML files.

### Writing to XML

```java
WebRowSet wrs = RowSetProvider.newFactory().createWebRowSet();
wrs.setUrl("jdbc:mysql://localhost:3306/testdb");
wrs.setUsername("root");
wrs.setPassword("secret");
wrs.setCommand("SELECT * FROM COFFEES WHERE PRICE > ?");
wrs.setFloat(1, 8.00f);
wrs.execute();

// Write to XML file
FileOutputStream fos = new FileOutputStream("coffees.xml");
wrs.writeXml(fos);
fos.close();
```

The XML output contains three sections:

```
┌──────────────────────────────────────────────────────────────┐
│                WebRowSet XML STRUCTURE                         │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  <webRowSet>                                                   │
│    <properties>                                                │
│      Connection URL, username, command, isolation level,       │
│      type, concurrency, max rows, and other settings           │
│    </properties>                                               │
│                                                                │
│    <metadata>                                                  │
│      Column count, column names, types, precision,             │
│      table name, catalog name, etc.                            │
│    </metadata>                                                 │
│                                                                │
│    <data>                                                      │
│      <currentRow>                                              │
│        <columnValue>Colombian</columnValue>                    │
│        <columnValue>101</columnValue>                          │
│        <columnValue>7.99</columnValue>                         │
│      </currentRow>                                             │
│      <currentRow>                                              │
│        <columnValue>French_Roast</columnValue>                 │
│        ...                                                     │
│      </currentRow>                                             │
│    </data>                                                     │
│  </webRowSet>                                                  │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Reading from XML

```java
WebRowSet wrs = RowSetProvider.newFactory().createWebRowSet();
FileInputStream fis = new FileInputStream("coffees.xml");
wrs.readXml(fis);
fis.close();

// Now use wrs like any other RowSet
while (wrs.next()) {
    System.out.println(wrs.getString("COF_NAME"));
}
```

---

## 6. JoinRowSet — SQL JOINs Without a Database Connection

A `JoinRowSet` lets you combine data from multiple disconnected RowSets using `SQL JOIN` semantics — without any database connection at all.

```java
CachedRowSet coffees = RowSetProvider.newFactory().createCachedRowSet();
coffees.setCommand("SELECT * FROM COFFEES");
// ... set URL, execute ...

CachedRowSet suppliers = RowSetProvider.newFactory().createCachedRowSet();
suppliers.setCommand("SELECT * FROM SUPPLIERS");
// ... set URL, execute ...

// Create a JoinRowSet and add both tables
JoinRowSet jrs = RowSetProvider.newFactory().createJoinRowSet();
jrs.addRowSet(coffees, "SUP_ID");      // join column in COFFEES
jrs.addRowSet(suppliers, "SUP_ID");    // join column in SUPPLIERS

// Iterate over the joined result — no database connection needed!
while (jrs.next()) {
    String coffeeName = jrs.getString("COF_NAME");
    String supplierName = jrs.getString("SUP_NAME");
    System.out.println(coffeeName + " supplied by " + supplierName);
}
```

This is equivalent to:

```sql
SELECT * FROM COFFEES, SUPPLIERS
WHERE COFFEES.SUP_ID = SUPPLIERS.SUP_ID
```

> **Note:** `JoinRowSet` supports `INNER_JOIN` (default), `LEFT_OUTER_JOIN`, `RIGHT_OUTER_JOIN`, `CROSS_JOIN`, and `FULL_JOIN`. Set the type with `jrs.setJoinType(JoinRowSet.INNER_JOIN)`.

---

## 7. FilteredRowSet — Client-Side Filtering

A `FilteredRowSet` lets you filter data without hitting the database again. You define a `Predicate` that controls which rows are visible:

```java
public class RangeFilter implements Predicate {
    private final float lo;
    private final float hi;
    private final String columnName;

    public RangeFilter(float lo, float hi, String columnName) {
        this.lo = lo;
        this.hi = hi;
        this.columnName = columnName;
    }

    @Override
    public boolean evaluate(RowSet rs) {
        try {
            float value = ((CachedRowSet)rs).getFloat(columnName);
            return value >= lo && value <= hi;
        } catch (SQLException e) {
            return false;
        }
    }

    // Also implement evaluate(Object, int) and evaluate(Object, String)
    @Override
    public boolean evaluate(Object value, int column) throws SQLException {
        // ... similar logic
        return true;
    }

    @Override
    public boolean evaluate(Object value, String columnName) throws SQLException {
        // ... similar logic
        return true;
    }
}
```

Using the filter:

```java
FilteredRowSet frs = RowSetProvider.newFactory().createFilteredRowSet();
frs.setCommand("SELECT * FROM COFFEES");
// ... set URL, execute ...

// Show only coffees priced between $8 and $10
frs.setFilter(new RangeFilter(8.0f, 10.0f, "PRICE"));

// Only matching rows are visible
while (frs.next()) {
    System.out.println(frs.getString("COF_NAME") + " $" +
                       frs.getFloat("PRICE"));
}

// Remove the filter — all rows visible again
frs.setFilter(null);
```

```
┌──────────────────────────────────────────────────────────────┐
│          FilteredRowSet vs SQL WHERE                           │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  SQL WHERE:                                                    │
│    SELECT * FROM COFFEES WHERE PRICE BETWEEN 8 AND 10          │
│    → Requires a database connection                            │
│    → Returns only matching rows from DB                        │
│                                                                │
│  FilteredRowSet:                                               │
│    frs.setFilter(new RangeFilter(8, 10, "PRICE"))              │
│    → No database connection needed                             │
│    → All data is in memory; filter just hides rows             │
│    → Can change filters instantly without re-querying          │
│    → Data already fetched — this is CLIENT-SIDE filtering      │
│                                                                │
│  Use FilteredRowSet when:                                      │
│    - You already have the data locally                         │
│    - You need to switch between different views quickly        │
│    - You want to avoid repeated database round trips           │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## 8. DataSource Objects

In Part 1, we used `DriverManager` to get connections. That works for tutorials and small applications, but production systems use **DataSource** objects instead.

### Why DataSource Over DriverManager?

| Feature | DriverManager | DataSource |
|---------|--------------|------------|
| **Connection URL** | Hardcoded in your application | Stored externally (JNDI directory) |
| **Code changes to switch DB** | Must recompile | Just change the JNDI config — no code changes |
| **Connection pooling** | Not supported | Supported |
| **Distributed transactions** | Not supported | Supported |
| **Production-ready** | Not recommended | The standard approach |

```
┌──────────────────────────────────────────────────────────────┐
│          DriverManager vs DataSource                           │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  DriverManager approach:                                       │
│  ┌──────────┐                     ┌──────────┐               │
│  │ Your App │───getConnection()──►│ Database │               │
│  │          │   (URL hardcoded)   │          │               │
│  └──────────┘                     └──────────┘               │
│  Problem: URL, driver class, and config are in your code.     │
│  Changing the database requires recompiling.                   │
│                                                                │
│  DataSource approach:                                          │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐             │
│  │ Your App │──►  │  JNDI    │──►  │DataSource│             │
│  │          │     │Directory │     │          │             │
│  └──────────┘     └──────────┘     │   ┌──────┴────┐       │
│  Looks up                          │   │Connection │       │
│  "jdbc/CoffeesDB"                  │   │  Pool     │       │
│  by name                           └───┴───────────┘       │
│                                          │                   │
│                                          ▼                   │
│                                    ┌──────────┐             │
│                                    │ Database │             │
│                                    └──────────┘             │
│                                                                │
│  Your code only knows the JNDI name. Everything else is       │
│  configured by the system administrator.                       │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Deploying a DataSource (Admin Side)

A system administrator registers the DataSource with a JNDI naming service:

```java
// This is done by the admin/deployer, NOT by your application code
VendorDataSource vds = new VendorDataSource();
vds.setServerName("localhost");
vds.setDatabaseName("testdb");
vds.setPortNumber(3306);
vds.setDescription("Coffee Break Database");

Context ctx = new InitialContext();
ctx.bind("jdbc/CoffeesDB", vds);
```

### Using a DataSource (Application Side)

Your application code simply looks up the DataSource by name:

```java
Context ctx = new InitialContext();
DataSource ds = (DataSource) ctx.lookup("jdbc/CoffeesDB");
Connection con = ds.getConnection("user", "password");
```

The application never knows the server name, port, or database name — it only knows the logical name `jdbc/CoffeesDB`. If the DBA moves the database to a different server, they update the JNDI entry and your application continues working without any code changes.

---

## 9. Connection Pooling

Creating database connections is expensive — it involves network handshakes, authentication, and resource allocation. In a web application handling hundreds of requests per second, opening a new connection for each request would cripple performance.

**Connection pooling** solves this by maintaining a pool of pre-created connections that are reused:

```
┌──────────────────────────────────────────────────────────────┐
│             HOW CONNECTION POOLING WORKS                       │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  WITHOUT pooling:                                              │
│  Request 1 → Open connection → Use → Close → (destroyed)      │
│  Request 2 → Open connection → Use → Close → (destroyed)      │
│  Request 3 → Open connection → Use → Close → (destroyed)      │
│  Each open/close takes ~100-500ms                              │
│                                                                │
│  WITH pooling:                                                 │
│  ┌──────────────── Connection Pool ─────────────────┐         │
│  │                                                    │         │
│  │  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐         │         │
│  │  │ Con1 │  │ Con2 │  │ Con3 │  │ Con4 │         │         │
│  │  │(idle)│  │(busy)│  │(idle)│  │(busy)│         │         │
│  │  └──────┘  └──────┘  └──────┘  └──────┘         │         │
│  │                                                    │         │
│  └────────────────────────────────────────────────────┘         │
│                                                                │
│  Request 1 → Borrow Con1 → Use → Return to pool (reused!)     │
│  Request 2 → Borrow Con1 → Use → Return to pool (reused!)     │
│  Connections are created ONCE and reused thousands of times.   │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### How It Works in JDBC

From the application's perspective, nothing changes — you still call `ds.getConnection()`. But behind the scenes, the `DataSource` is actually a `ConnectionPoolDataSource` that manages a pool:

```java
// Application code — EXACTLY the same as without pooling!
DataSource ds = (DataSource) ctx.lookup("jdbc/CoffeesDB");
Connection con = ds.getConnection("user", "password");

// ... use the connection ...

con.close();    // Does NOT actually close the connection!
                // Returns it to the pool for reuse.
```

```
┌──────────────────────────────────────────────────────────────┐
│          WHAT HAPPENS WHEN YOU CALL close()                    │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Without pooling:                                              │
│  con.close()  →  Connection destroyed, resources freed         │
│                                                                │
│  With pooling:                                                 │
│  con.close()  →  Connection returned to Pool                   │
│                  Marked as "available" for next request         │
│                  Physical connection stays open                 │
│                                                                │
│  The connection you get is actually a                           │
│  PooledConnection wrapper:                                     │
│                                                                │
│  ┌────────────────────────────────────┐                       │
│  │       PooledConnection             │                       │
│  │  ┌──────────────────────────────┐  │                       │
│  │  │   Physical Connection        │  │                       │
│  │  │   (the real DB connection)   │  │                       │
│  │  └──────────────────────────────┘  │                       │
│  │                                    │                       │
│  │  Fires events when "closed"        │                       │
│  │  Pool listens and reclaims it      │                       │
│  └────────────────────────────────────┘                       │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Deploying a Pooled DataSource

The admin deploys a `ConnectionPoolDataSource` implementation:

```java
// Admin/deployer configuration
ConnectionPoolDataSource cpds = new VendorConnectionPoolDS();
cpds.setServerName("localhost");
cpds.setDatabaseName("testdb");
cpds.setPortNumber(3306);

Context ctx = new InitialContext();
ctx.bind("jdbc/pool/CoffeesDB", cpds);
```

The application still uses it through a regular `DataSource` lookup — the pooling is transparent.

---

## 10. Distributed Transactions

In some scenarios, a single transaction must span **multiple databases** — for example, transferring inventory data from one database to another. Regular transactions only work within a single connection. **Distributed transactions** coordinate across multiple connections.

```
┌──────────────────────────────────────────────────────────────┐
│            DISTRIBUTED TRANSACTION — TWO-PHASE COMMIT          │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Transaction Manager                                           │
│       │                                                        │
│       ├──► Database A: DELETE FROM inventory                   │
│       │                 WHERE item = 'Coffee Maker'            │
│       │                                                        │
│       ├──► Database B: INSERT INTO inventory                   │
│       │                 VALUES('Coffee Maker', ...)            │
│       │                                                        │
│       │   Phase 1: PREPARE                                     │
│       │     Ask both databases: "Can you commit?"              │
│       │     DB A: "Yes"                                        │
│       │     DB B: "Yes"                                        │
│       │                                                        │
│       │   Phase 2: COMMIT                                      │
│       │     Tell both databases: "Commit now!"                 │
│       │     Both commit — transaction complete.                │
│       │                                                        │
│       │   If Phase 1 had a "No":                               │
│       │     Phase 2: ROLLBACK all participants.                │
│       │                                                        │
└──────────────────────────────────────────────────────────────┘
```

JDBC supports distributed transactions through `XADataSource`:

```java
// Admin deploys XADataSource
XADataSource xads = new VendorXADataSource();
xads.setServerName("localhost");
xads.setDatabaseName("testdb");
xads.setPortNumber(3306);

Context ctx = new InitialContext();
ctx.bind("jdbc/xa/CoffeesDB", xads);
```

The application code looks identical — the transaction manager handles the two-phase commit protocol automatically:

```java
// Your code is unchanged — the EJB container or
// transaction manager handles distributed coordination
DataSource ds = (DataSource) ctx.lookup("jdbc/xa/CoffeesDB");
Connection con = ds.getConnection();
// ... SQL operations ...
// commit/rollback managed by container
```

> **Important:** When a connection is part of a distributed transaction, you should not call `con.setAutoCommit(false)`, `con.commit()`, or `con.rollback()` yourself. The transaction manager controls these. Calling them manually will throw a `SQLException`.

---

## 11. Choosing the Right Approach

```
┌──────────────────────────────────────────────────────────────┐
│                   DECISION GUIDE                               │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  "I need a simple, always-connected cursor"                    │
│     → Use JdbcRowSet                                           │
│                                                                │
│  "I need to work with data offline / pass over network"        │
│     → Use CachedRowSet                                         │
│                                                                │
│  "I need to export/import data as XML"                         │
│     → Use WebRowSet                                            │
│                                                                │
│  "I need to join data from multiple sources without SQL"       │
│     → Use JoinRowSet                                           │
│                                                                │
│  "I need to filter data without re-querying"                   │
│     → Use FilteredRowSet                                       │
│                                                                │
│  "I need connection management for a production app"           │
│     → Use DataSource + JNDI                                    │
│                                                                │
│  "I need to handle many concurrent requests efficiently"       │
│     → Use Connection Pooling                                   │
│                                                                │
│  "My transaction spans multiple databases"                     │
│     → Use Distributed Transactions (XADataSource)              │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## 12. Summary

| Concept | Key Takeaway |
|---------|-------------|
| **RowSet** | Extends ResultSet with JavaBeans compliance, event notifications, and serializability |
| **JdbcRowSet** | Connected RowSet — essentially a better ResultSet with default scrollability and updatability |
| **CachedRowSet** | Disconnected — fetches data into memory, works offline, syncs back with `acceptChanges()` |
| **WebRowSet** | CachedRowSet + XML read/write — ideal for web service data exchange |
| **JoinRowSet** | Joins multiple disconnected RowSets in memory — SQL JOIN without a database connection |
| **FilteredRowSet** | Client-side filtering with Predicate — switch views without re-querying |
| **DataSource** | Production-grade connection factory — database details stored in JNDI, not hardcoded |
| **Connection Pooling** | Reuse pre-created connections — `close()` returns to pool instead of destroying |
| **Distributed Transactions** | Two-phase commit across multiple databases — managed by XADataSource and a transaction manager |

This wraps up the core JDBC series. From establishing your first connection in Part 1, through transactions and batch updates in Part 2, to RowSets and connection pooling here in Part 3, you now have a solid foundation for building database-driven Java applications.

In **[Part 4](/posts/java-jdbc-sqlxml-lobs-type-mappings/)**, we'll explore advanced data types — working with SQLXML for XML data, handling Large Objects (BLOBs and CLOBs), and creating customized type mappings.
