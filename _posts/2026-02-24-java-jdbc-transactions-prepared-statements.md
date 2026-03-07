---
title: "Java JDBC Part 2: Prepared Statements, Callable Statements, Transactions, and Batch Updates"
description: Master parameterized queries with PreparedStatement, call stored procedures with CallableStatement, control transactions with commit and rollback, use savepoints, understand isolation levels, and perform efficient batch updates
author: Vaibhav Gagneja
date: 2026-02-24 12:00:00 +0530
categories: [Development, Java]
tags: [java, jdbc, database, sql, prepared-statements, callable-statements, stored-procedures, transactions, batch-updates, savepoints]
toc: true
image:
  path: https://images.unsplash.com/photo-1558494949-ef010cbdcc31
---

In [Part 1](/posts/java-jdbc-fundamentals/), we covered the JDBC fundamentals — establishing connections, executing simple statements, reading results, and handling exceptions. But production database code demands more.

You need **prepared statements** to prevent SQL injection and improve performance. You need **transactions** to keep your data consistent when multiple operations must succeed or fail together. And you need **batch updates** when inserting or updating hundreds (or thousands) of rows without drowning the database in individual requests.

This post covers all three.

---

## 1. Using Prepared Statements

A `PreparedStatement` is a precompiled SQL statement that accepts **parameters**. Instead of embedding values directly into SQL strings (which is dangerous and slow), you use placeholders and set values programmatically.

### Why Not Just Use Statement?

Consider this code using a regular `Statement`:

```java
// This is DANGEROUS — SQL injection risk!
String query = "SELECT * FROM COFFEES WHERE COF_NAME = '" + userInput + "'";
Statement stmt = con.createStatement();
ResultSet rs = stmt.executeQuery(query);
```

If `userInput` is `"' OR '1'='1"`, the query becomes:

```sql
SELECT * FROM COFFEES WHERE COF_NAME = '' OR '1'='1'
```

That returns **every row** in the table. With a `PreparedStatement`, this attack is impossible:

```java
// SAFE — the parameter is treated as a literal value, never as SQL
String query = "SELECT * FROM COFFEES WHERE COF_NAME = ?";
PreparedStatement pstmt = con.prepareStatement(query);
pstmt.setString(1, userInput);  // userInput is escaped automatically
ResultSet rs = pstmt.executeQuery();
```

```
┌──────────────────────────────────────────────────────────────┐
│         STATEMENT vs PREPAREDSTATEMENT                         │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Statement (AVOID for user input):                             │
│  ──────────────────────────────────                            │
│  SQL string is built by concatenation                          │
│  "SELECT * FROM T WHERE name = '" + input + "'"               │
│     → Parsed and compiled EVERY TIME                           │
│     → User input is treated as SQL code                        │
│     → SQL INJECTION possible                                   │
│                                                                │
│  PreparedStatement (USE THIS):                                 │
│  ──────────────────────────────                                │
│  SQL template with ? placeholders                              │
│  "SELECT * FROM T WHERE name = ?"                              │
│     → Parsed and compiled ONCE, reused many times              │
│     → User input is treated as DATA, never as code             │
│     → SQL INJECTION impossible                                 │
│     → Better performance for repeated queries                  │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Creating and Using a PreparedStatement

Here's a complete example that updates coffee prices by a given percentage:

```java
public void updateCoffeeSales(HashMap<String, Integer> salesForWeek)
    throws SQLException {

    String updateString =
        "UPDATE COFFEES SET SALES = ? WHERE COF_NAME = ?";
    String updateStatement =
        "UPDATE COFFEES SET TOTAL = TOTAL + ? WHERE COF_NAME = ?";

    try (PreparedStatement updateSales = con.prepareStatement(updateString);
         PreparedStatement updateTotal = con.prepareStatement(updateStatement)) {

        for (Map.Entry<String, Integer> entry : salesForWeek.entrySet()) {
            // Set parameter 1 (the value)
            updateSales.setInt(1, entry.getValue());
            // Set parameter 2 (the coffee name)
            updateSales.setString(2, entry.getKey());
            updateSales.executeUpdate();

            updateTotal.setInt(1, entry.getValue());
            updateTotal.setString(2, entry.getKey());
            updateTotal.executeUpdate();
        }
    }
}
```

### Parameter Setter Methods

Parameters are numbered starting at **1** (not 0). Each SQL type has a corresponding setter method:

| Method | SQL Type | Java Type |
|--------|----------|-----------|
| `setString(pos, val)` | `VARCHAR`, `CHAR` | `String` |
| `setInt(pos, val)` | `INTEGER` | `int` |
| `setFloat(pos, val)` | `FLOAT` | `float` |
| `setDouble(pos, val)` | `DOUBLE` | `double` |
| `setBigDecimal(pos, val)` | `NUMERIC`, `DECIMAL` | `BigDecimal` |
| `setDate(pos, val)` | `DATE` | `java.sql.Date` |
| `setTimestamp(pos, val)` | `TIMESTAMP` | `java.sql.Timestamp` |
| `setBoolean(pos, val)` | `BOOLEAN` | `boolean` |
| `setBlob(pos, val)` | `BLOB` | `Blob` or `InputStream` |
| `setClob(pos, val)` | `CLOB` | `Clob` or `Reader` |
| `setNull(pos, sqlType)` | Any | Sets a `NULL` value |

> **Note:** Once you set a parameter value, it stays set for all subsequent executions of that `PreparedStatement` until you change it or call `clearParameters()`. This is why prepared statements are efficient in loops — the SQL is parsed once, and you just swap parameter values.

### Return Values

Just like `Statement`, `PreparedStatement` has:

| Method | When to Use | Returns |
|--------|------------|---------|
| `executeQuery()` | `SELECT` queries | `ResultSet` |
| `executeUpdate()` | `INSERT`, `UPDATE`, `DELETE`, DDL | `int` (row count) |
| `execute()` | When you don't know the statement type | `boolean` |

Notice there's no SQL string argument — the SQL was already provided in `con.prepareStatement(sql)`.

---

## 2. Using Callable Statements

A `CallableStatement` lets you call **stored procedures** — precompiled SQL routines stored in the database. If `PreparedStatement` is for parameterized queries, `CallableStatement` is for calling database-side functions and procedures that may have both input and output parameters.

### Creating Stored Procedures: Java DB vs MySQL

Stored procedures differ significantly between databases. Java DB (Derby) and MySQL have completely different approaches:

```
┌──────────────────────────────────────────────────────────────┐
│      STORED PROCEDURES: JAVA DB vs MYSQL                       │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Java DB (Derby):                                              │
│  ─────────────────                                             │
│  Procedures are written as JAVA METHODS                        │
│  and registered with CREATE PROCEDURE.                         │
│  The actual logic lives in a .java file.                       │
│                                                                │
│  ┌──────────┐   CREATE PROCEDURE  ┌────────────────────┐     │
│  │ Database │ ◄─── references ─── │ Java class method  │     │
│  │          │   EXTERNAL NAME     │ showSuppliers()    │     │
│  └──────────┘                     └────────────────────┘     │
│                                                                │
│  MySQL:                                                        │
│  ──────                                                        │
│  Procedures are written in SQL/PSM directly                    │
│  inside a BEGIN...END block.                                   │
│  The logic lives IN the database itself.                       │
│                                                                │
│  ┌──────────┐                                                 │
│  │ Database │  CREATE PROCEDURE ... BEGIN ... END              │
│  │  (has     │  All SQL logic is stored inside                 │
│  │  the code)│  the database itself                            │
│  └──────────┘                                                 │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

**Java DB** — the stored procedure body is a Java method. You write the method in a Java class, compile it, and then register it:

```java
// The Java method that becomes a stored procedure
// (in StoredProcedures.java)
public static void showSuppliers(ResultSet[] rs) throws SQLException {
    Connection con = DriverManager.getConnection("jdbc:default:connection");
    Statement stmt = null;
    String query =
        "SELECT SUPPLIERS.SUP_NAME, COFFEES.COF_NAME " +
        "FROM SUPPLIERS, COFFEES " +
        "WHERE SUPPLIERS.SUP_ID = COFFEES.SUP_ID " +
        "ORDER BY SUP_NAME";
    stmt = con.createStatement();
    rs[0] = stmt.executeQuery(query);
}
```

Then register it using SQL:

```sql
CREATE PROCEDURE SHOW_SUPPLIERS()
    PARAMETER STYLE JAVA
    LANGUAGE JAVA
    READS SQL DATA
    DYNAMIC RESULT SETS 1
    EXTERNAL NAME
      'com.oracle.tutorial.jdbc.StoredProcedures.showSuppliers'
```

**MySQL** — the logic is written directly in SQL:

```sql
DELIMITER //

CREATE PROCEDURE SHOW_SUPPLIERS()
BEGIN
    SELECT SUPPLIERS.SUP_NAME, COFFEES.COF_NAME
    FROM SUPPLIERS, COFFEES
    WHERE SUPPLIERS.SUP_ID = COFFEES.SUP_ID
    ORDER BY SUP_NAME;
END //

CREATE PROCEDURE RAISE_PRICE(
    IN coffeeName VARCHAR(32),
    IN maximumPercentage FLOAT,
    INOUT newPrice NUMERIC(10,2)
)
BEGIN
    SET newPrice = 0;
    SELECT PRICE INTO newPrice
    FROM COFFEES WHERE COF_NAME = coffeeName;

    SET newPrice = newPrice + (newPrice * maximumPercentage);
    UPDATE COFFEES SET PRICE = newPrice
    WHERE COF_NAME = coffeeName;
END //

DELIMITER ;
```

> **Note:** In MySQL, you need to change the `DELIMITER` when creating procedures, because the default delimiter `;` would end the `CREATE PROCEDURE` statement prematurely at the first semicolon inside the `BEGIN...END` block. After defining the procedure, you reset the delimiter back to `;`.

### Creating Stored Procedures via JDBC

You can also create stored procedures programmatically from Java:

```java
public void createProcedureShowSuppliers() throws SQLException {
    String queryDrop = "DROP PROCEDURE IF EXISTS SHOW_SUPPLIERS";

    String createProcedure =
        "CREATE PROCEDURE SHOW_SUPPLIERS() " +
        "BEGIN " +
        "SELECT SUPPLIERS.SUP_NAME, COFFEES.COF_NAME " +
        "FROM SUPPLIERS, COFFEES " +
        "WHERE SUPPLIERS.SUP_ID = COFFEES.SUP_ID " +
        "ORDER BY SUP_NAME; " +
        "END";

    try (Statement stmt = con.createStatement()) {
        stmt.execute(queryDrop);       // remove old version if it exists
        stmt.execute(createProcedure); // create the new one
    }
}
```

### Parameter Types

Stored procedures can have three kinds of parameters:

| Type | Direction | Description |
|------|-----------|-------------|
| `IN` | Caller → Procedure | Input value passed to the procedure. Set with `setXxx()` methods. |
| `OUT` | Procedure → Caller | Output value returned from the procedure. Register with `registerOutParameter()`, read with `getXxx()`. |
| `INOUT` | Both directions | Serves as both input and output. Set with `setXxx()`, register with `registerOutParameter()`, read with `getXxx()`. |

```
┌──────────────────────────────────────────────────────────────┐
│            CallableStatement PARAMETER FLOW                    │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Your Java Code                    Stored Procedure            │
│  ──────────────                    ────────────────            │
│                                                                │
│  IN parameters:                                                │
│  cs.setString(1, "Colombian")  ──────►  coffeeName             │
│  cs.setFloat(2, 0.10f)         ──────►  maximumPercentage      │
│                                                                │
│  OUT parameters:                                               │
│  float result = cs.getFloat(3) ◄──────  newPrice               │
│                                                                │
│  INOUT parameters:                                             │
│  cs.setFloat(3, 9.99f)         ──────►  newPrice (input)       │
│  float result = cs.getFloat(3) ◄──────  newPrice (output)      │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Calling a Stored Procedure

```java
public void callRaisePrice(String coffeeName, float maxPercent)
    throws SQLException {

    // The ? placeholders map to IN, OUT, and INOUT parameters
    try (CallableStatement cs =
            con.prepareCall("{call RAISE_PRICE(?, ?, ?)}")) {

        // Set IN parameters
        cs.setString(1, coffeeName);
        cs.setFloat(2, maxPercent);

        // Register INOUT parameter — set its input value AND
        // tell JDBC what type to expect back
        cs.setFloat(3, 0f);   // input value
        cs.registerOutParameter(3, Types.NUMERIC);  // output type

        cs.execute();

        // Read the output
        float newPrice = cs.getFloat(3);
        System.out.println("New price for " + coffeeName + ": $" + newPrice);
    }
}
```

### Calling Procedures That Return a ResultSet

Some stored procedures return result sets. Handle them the same way you would with a regular query:

```java
try (CallableStatement cs = con.prepareCall("{call SHOW_SUPPLIERS}")) {
    ResultSet rs = cs.executeQuery();
    while (rs.next()) {
        String supplier = rs.getString("SUP_NAME");
        String coffee   = rs.getString("COF_NAME");
        System.out.println(supplier + ": " + coffee);
    }
}
```

### The Call Syntax

JDBC uses a special **escape syntax** for calling stored procedures that works across all databases:

| Syntax | When to Use |
|--------|------------|
| `{call procedure_name(?, ?)}` | Procedure with no return value |
| `{? = call function_name(?, ?)}` | Function that returns a value (the first `?` is the return value) |
| `{call procedure_name}` | No parameters at all |

> **Note:** The escape syntax `{call ...}` is translated by the JDBC driver into the database-specific syntax. This means your code works the same whether the underlying database is Oracle, MySQL, or Derby.

### Complete Example: Stored Procedure with Multiple Features

Here's a more involved example — a procedure that gets the supplier name for a given coffee:

```java
public void getSupplierOfCoffee(String coffeeName) throws SQLException {
    try (CallableStatement cs = con.prepareCall(
            "{call GET_SUPPLIER_OF_COFFEE(?, ?)}")) {

        cs.setString(1, coffeeName);                       // IN
        cs.registerOutParameter(2, Types.VARCHAR);         // OUT

        cs.execute();

        String supplierName = cs.getString(2);
        System.out.println("Supplier of " + coffeeName +
                           ": " + supplierName);
    }
}
```

The MySQL procedure behind this:

```sql
CREATE PROCEDURE GET_SUPPLIER_OF_COFFEE(
    IN coffeeNameParam VARCHAR(32),
    OUT supplierNameParam VARCHAR(40)
)
BEGIN
    SELECT SUPPLIERS.SUP_NAME INTO supplierNameParam
    FROM SUPPLIERS, COFFEES
    WHERE COFFEES.COF_NAME = coffeeNameParam
      AND COFFEES.SUP_ID = SUPPLIERS.SUP_ID;
END
```

---

## 3. Modifying Data Through a ResultSet

In Part 1, we saw ResultSets as read-only views of query results. But if you create a statement with `CONCUR_UPDATABLE`, you can **modify data directly through the ResultSet** — updating existing rows, inserting new ones, and deleting rows.

### Updating Rows

```java
public void modifyPrices(float percentage) throws SQLException {
    try (Statement stmt = con.createStatement(
            ResultSet.TYPE_SCROLL_SENSITIVE,   // scrollable cursor
            ResultSet.CONCUR_UPDATABLE)) {     // allows updates

        ResultSet uprs = stmt.executeQuery("SELECT * FROM COFFEES");
        while (uprs.next()) {
            float oldPrice = uprs.getFloat("PRICE");
            float newPrice = oldPrice + (oldPrice * percentage);

            uprs.updateFloat("PRICE", newPrice);  // modify in buffer
            uprs.updateRow();                       // write to database
        }
    }
}
```

```
┌──────────────────────────────────────────────────────────────┐
│            UPDATING ROWS THROUGH A ResultSet                   │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Step 1: Position cursor on the row → rs.next()                │
│  Step 2: Modify column values       → rs.updateFloat(col, val)│
│  Step 3: Write changes to database  → rs.updateRow()           │
│                                                                │
│  Changed your mind?                                            │
│  Call rs.cancelRowUpdates() before updateRow() to revert.      │
│                                                                │
│  When you call updateRow():                                    │
│  ┌──────────┐                ┌──────────┐                     │
│  │ ResultSet │ ──updateRow──►│ Database │                     │
│  │  Buffer   │               │  Table   │                     │
│  │ PRICE=8.99│               │PRICE=8.99│                     │
│  └──────────┘                └──────────┘                     │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Inserting Rows

To insert a new row through a ResultSet, you use the **insert row buffer** — a special staging area:

```java
public void insertRow(String coffeeName, int supplierID,
                      float price, int sales, int total)
    throws SQLException {

    try (Statement stmt = con.createStatement(
            ResultSet.TYPE_SCROLL_SENSITIVE,
            ResultSet.CONCUR_UPDATABLE)) {

        ResultSet uprs = stmt.executeQuery("SELECT * FROM COFFEES");
        uprs.moveToInsertRow();             // move to insert row buffer

        uprs.updateString("COF_NAME", coffeeName);
        uprs.updateInt("SUP_ID", supplierID);
        uprs.updateFloat("PRICE", price);
        uprs.updateInt("SALES", sales);
        uprs.updateInt("TOTAL", total);

        uprs.insertRow();                   // write to database
        uprs.moveToCurrentRow();            // return to where we were
    }
}
```

```
┌──────────────────────────────────────────────────────────────┐
│           INSERTING A ROW VIA ResultSet                        │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Normal ResultSet rows:                                        │
│  ┌───────────────────────────────────────────────┐             │
│  │  Row 1: Colombian, 101, 7.99, 0, 0           │             │
│  │  Row 2: French_Roast, 49, 8.99, 0, 0         │  ← cursor  │
│  │  Row 3: Espresso, 150, 9.99, 0, 0            │             │
│  └───────────────────────────────────────────────┘             │
│                                                                │
│  moveToInsertRow():                                            │
│  ┌───────────────────────────────────────────────┐             │
│  │  INSERT BUFFER: (empty, waiting for values)   │  ← cursor  │
│  └───────────────────────────────────────────────┘             │
│                                                                │
│  After updateString/updateInt/... then insertRow():            │
│  The new row is written to the database.                       │
│                                                                │
│  moveToCurrentRow():                                           │
│  Cursor returns to where it was before moveToInsertRow().      │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Deleting Rows

```java
// Position cursor on the row to delete
rs.absolute(4);       // move to 4th row
rs.deleteRow();       // row is deleted from ResultSet AND from database
```

---

## 4. Batch Updates

When you need to execute many SQL statements (say, inserting 500 rows), sending them one at a time is wasteful — each one requires a round trip to the database. **Batch updates** bundle multiple statements into a single request.

### Batch with Statement

```java
public void batchUpdate() throws SQLException {
    con.setAutoCommit(false);       // required for batch operations

    try (Statement stmt = con.createStatement()) {
        stmt.addBatch("INSERT INTO COFFEES VALUES('Amaretto', 49, 9.99, 0, 0)");
        stmt.addBatch("INSERT INTO COFFEES VALUES('Hazelnut', 49, 9.99, 0, 0)");
        stmt.addBatch("INSERT INTO COFFEES VALUES('Amaretto_decaf', 49, 10.99, 0, 0)");
        stmt.addBatch("INSERT INTO COFFEES VALUES('Hazelnut_decaf', 49, 10.99, 0, 0)");

        int[] updateCounts = stmt.executeBatch();
        con.commit();               // commit all inserts at once
    } catch (BatchUpdateException e) {
        // Handle partial failures
    } finally {
        con.setAutoCommit(true);
    }
}
```

### Batch with PreparedStatement

For parameterized batch inserts — the most efficient approach:

```java
public void batchInsertCoffees(List<Coffee> coffees) throws SQLException {
    con.setAutoCommit(false);

    String sql = "INSERT INTO COFFEES VALUES(?, ?, ?, ?, ?)";
    try (PreparedStatement pstmt = con.prepareStatement(sql)) {
        for (Coffee c : coffees) {
            pstmt.setString(1, c.getName());
            pstmt.setInt(2, c.getSupplierID());
            pstmt.setFloat(3, c.getPrice());
            pstmt.setInt(4, c.getSales());
            pstmt.setInt(5, c.getTotal());
            pstmt.addBatch();       // add to batch, don't execute yet
        }

        int[] counts = pstmt.executeBatch();   // send all at once
        con.commit();
    } catch (BatchUpdateException e) {
        // ...
    } finally {
        con.setAutoCommit(true);
    }
}
```

### Handling BatchUpdateException

If any statement in a batch fails, a `BatchUpdateException` is thrown. The key property is `getUpdateCounts()`, which tells you what happened to each statement:

| Value | Meaning |
|-------|---------|
| `>= 0` | Statement succeeded, value is the row count |
| `Statement.SUCCESS_NO_INFO` | Statement succeeded, but no row count available |
| `Statement.EXECUTE_FAILED` | Statement failed |

```java
catch (BatchUpdateException e) {
    int[] updateCounts = e.getUpdateCounts();
    for (int i = 0; i < updateCounts.length; i++) {
        if (updateCounts[i] == Statement.EXECUTE_FAILED) {
            System.err.println("Statement " + i + " failed!");
        }
    }
    con.rollback();     // undo all changes if any failed
}
```

> **Note:** Whether the driver continues executing after a batch failure or stops immediately depends on the JDBC driver implementation. Check your driver's documentation.

---

## 5. Using Transactions

A **transaction** groups multiple SQL statements into a single unit of work. Either **all** of them succeed, or **none** of them take effect. This is essential for data integrity.

### The Classic Example

Imagine The Coffee Break wants to update weekly sales. For each coffee type, two operations must happen:

1. Set the `SALES` column to the week's sales number
2. Add that number to the running `TOTAL`

If the first statement succeeds but the second fails, the data is inconsistent — `SALES` shows the new number but `TOTAL` hasn't been updated. Transactions prevent this.

```
┌──────────────────────────────────────────────────────────────┐
│                 WHY TRANSACTIONS MATTER                        │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  WITHOUT transactions:                                         │
│  ┌────────────────────────────────────────────┐               │
│  │  UPDATE COFFEES SET SALES = 50             │  → Success    │
│  │     WHERE COF_NAME = 'Colombian'           │               │
│  │                                            │               │
│  │  UPDATE COFFEES SET TOTAL = TOTAL + 50     │  → FAILS!     │
│  │     WHERE COF_NAME = 'Colombian'           │               │
│  └────────────────────────────────────────────┘               │
│  Result: SALES = 50, but TOTAL is unchanged.                  │
│  Data is now INCONSISTENT.                                     │
│                                                                │
│  WITH transactions:                                            │
│  ┌────────────────────────────────────────────┐               │
│  │  BEGIN TRANSACTION                         │               │
│  │  UPDATE COFFEES SET SALES = 50 ...         │  → Success    │
│  │  UPDATE COFFEES SET TOTAL = TOTAL + 50 ... │  → FAILS!     │
│  │  ROLLBACK (undo everything)                │               │
│  └────────────────────────────────────────────┘               │
│  Result: SALES and TOTAL are both unchanged.                  │
│  Data is CONSISTENT.                                           │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Auto-Commit Mode

By default, JDBC runs in **auto-commit mode** — every individual statement is committed immediately after it's executed. To use transactions, you must disable this:

```java
con.setAutoCommit(false);   // disable auto-commit → manual transactions
```

Now statements are grouped: nothing is committed until you explicitly call `commit()`, and you can undo everything with `rollback()`.

### Complete Transaction Example

```java
public void updateCoffeeSales(HashMap<String, Integer> salesForWeek)
    throws SQLException {

    String updateSalesSQL =
        "UPDATE COFFEES SET SALES = ? WHERE COF_NAME = ?";
    String updateTotalSQL =
        "UPDATE COFFEES SET TOTAL = TOTAL + ? WHERE COF_NAME = ?";

    try (PreparedStatement updateSales = con.prepareStatement(updateSalesSQL);
         PreparedStatement updateTotal = con.prepareStatement(updateTotalSQL)) {

        con.setAutoCommit(false);       // start transaction

        for (Map.Entry<String, Integer> entry : salesForWeek.entrySet()) {
            updateSales.setInt(1, entry.getValue());
            updateSales.setString(2, entry.getKey());
            updateSales.executeUpdate();

            updateTotal.setInt(1, entry.getValue());
            updateTotal.setString(2, entry.getKey());
            updateTotal.executeUpdate();
        }

        con.commit();                   // all succeeded → make permanent

    } catch (SQLException e) {
        // Something failed → undo everything
        if (con != null) {
            try {
                System.err.println("Transaction failed, rolling back.");
                con.rollback();
            } catch (SQLException ex) {
                JDBCTutorialUtilities.printSQLException(ex);
            }
        }
    } finally {
        con.setAutoCommit(true);
    }
}
```

---

## 6. Transaction Isolation Levels

When multiple users access the same database simultaneously, their transactions can interfere with each other. **Isolation levels** control how much one transaction can "see" of another transaction's uncommitted work.

### The Three Phenomena

| Phenomenon | What Happens |
|-----------|-------------|
| **Dirty read** | Transaction A reads data that Transaction B has modified but **not yet committed**. If B rolls back, A has read data that never existed. |
| **Non-repeatable read** | Transaction A reads a row, then Transaction B modifies and commits that row. When A reads the same row again, it gets different values. |
| **Phantom read** | Transaction A reads rows matching a condition. Transaction B inserts new rows that match the same condition. When A repeats the query, it gets additional "phantom" rows. |

### Isolation Level Comparison

| Level | Dirty Reads | Non-Repeatable Reads | Phantom Reads | Performance |
|-------|------------|---------------------|---------------|-------------|
| `TRANSACTION_READ_UNCOMMITTED` | Allowed | Allowed | Allowed | Fastest |
| `TRANSACTION_READ_COMMITTED` | Prevented | Allowed | Allowed | Fast |
| `TRANSACTION_REPEATABLE_READ` | Prevented | Prevented | Allowed | Moderate |
| `TRANSACTION_SERIALIZABLE` | Prevented | Prevented | Prevented | Slowest |

```java
// Set isolation level BEFORE starting a transaction
con.setTransactionIsolation(Connection.TRANSACTION_READ_COMMITTED);
con.setAutoCommit(false);
// ... perform operations ...
con.commit();
```

> **Note:** Higher isolation means safer data but slower performance. Most databases default to `READ_COMMITTED`, which is a good balance for typical applications. Use `SERIALIZABLE` only when absolute consistency is critical — it essentially makes transactions run one at a time.

You can check what your database supports:

```java
DatabaseMetaData meta = con.getMetaData();
System.out.println("Default isolation: " +
    con.getTransactionIsolation());
System.out.println("Supports SERIALIZABLE? " +
    meta.supportsTransactionIsolationLevel(
        Connection.TRANSACTION_SERIALIZABLE));
```

---

## 7. Savepoints

Sometimes a transaction does many things, and you don't want to roll back *everything* when a single step fails. **Savepoints** let you mark positions within a transaction that you can roll back to — like checkpoints in a video game.

```
┌──────────────────────────────────────────────────────────────┐
│                   HOW SAVEPOINTS WORK                          │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  con.setAutoCommit(false);                                     │
│       │                                                        │
│       ▼                                                        │
│  Statement 1: INSERT INTO COFFEES ...        (succeeds)        │
│  Statement 2: INSERT INTO COFFEES ...        (succeeds)        │
│       │                                                        │
│       ▼                                                        │
│  Savepoint sp1 = con.setSavepoint("after_coffees")             │
│       │                                     ▲                  │
│       ▼                                     │                  │
│  Statement 3: UPDATE SUPPLIERS ...           │  (fails!)       │
│       │                                     │                  │
│       └───── con.rollback(sp1) ─────────────┘                  │
│              (Undoes statement 3, keeps 1 and 2)               │
│       │                                                        │
│       ▼                                                        │
│  Statement 4: UPDATE SUPPLIERS ... (retry with different data) │
│       │                                                        │
│       ▼                                                        │
│  con.commit()   (commits statements 1, 2, and 4)              │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Using Savepoints in Code

```java
public void insertWithSavepoint() throws SQLException {
    con.setAutoCommit(false);

    Statement stmt = con.createStatement();

    // Phase 1: Insert coffees
    stmt.executeUpdate("INSERT INTO COFFEES VALUES('Amaretto', 49, 9.99, 0, 0)");
    stmt.executeUpdate("INSERT INTO COFFEES VALUES('Hazelnut', 49, 9.99, 0, 0)");

    // Create a savepoint — we can roll back to here if phase 2 fails
    Savepoint sp1 = con.setSavepoint("coffee_inserts");

    // Phase 2: Insert decaf versions
    int decafCount = stmt.executeUpdate(
        "INSERT INTO COFFEES VALUES('Amaretto_decaf', 49, 10.99, 0, 0)");
    int hazelDecafCount = stmt.executeUpdate(
        "INSERT INTO COFFEES VALUES('Hazelnut_decaf', 49, 10.99, 0, 0)");

    if (decafCount != 1 || hazelDecafCount != 1) {
        // Phase 2 didn't go well — roll back only phase 2
        con.rollback(sp1);
    }

    con.commit();                // commit whatever survived
    con.setAutoCommit(true);
}
```

### Key Rules for Savepoints

| Rule | Detail |
|------|--------|
| **Naming is optional** | `con.setSavepoint()` creates an unnamed savepoint; `con.setSavepoint("name")` creates a named one |
| **Rolling back** | `con.rollback(sp)` undoes everything after the savepoint but does NOT commit |
| **Releasing** | `con.releaseSavepoint(sp)` removes the savepoint from the transaction — the work becomes part of the regular transaction |
| **After rollback** | A released savepoint can no longer be rolled back to — you'll get a `SQLException` |
| **Full rollback** | `con.rollback()` (without arguments) still undoes the entire transaction, including everything before the savepoint |

> **Note:** Savepoints are part of the JDBC 3.0 specification. Check `DatabaseMetaData.supportsSavepoints()` to verify your database supports them.

---

## 8. When to Use What

Here's a decision guide for choosing the right approach:

```
┌──────────────────────────────────────────────────────────────┐
│                   DECISION GUIDE                               │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  "I need to run a simple query once"                           │
│     → Use Statement                                            │
│                                                                │
│  "I need to run the same query with different values"          │
│     → Use PreparedStatement                                    │
│                                                                │
│  "I need to accept user input in a query"                      │
│     → Use PreparedStatement (prevents SQL injection)           │
│                                                                │
│  "I need to insert/update many rows at once"                   │
│     → Use PreparedStatement + batch updates                    │
│                                                                │
│  "Multiple operations must succeed or fail together"           │
│     → Use transactions (setAutoCommit(false) + commit/rollback)│
│                                                                │
│  "Only part of a large transaction should be retried"          │
│     → Use transactions + savepoints                            │
│                                                                │
│  "I need to call a stored procedure"                           │
│     → Use CallableStatement                                    │
│                                                                │
│  "I need to modify result data directly"                       │
│     → Use updatable ResultSet (CONCUR_UPDATABLE)               │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## 9. Summary

| Concept | Key Takeaway |
|---------|-------------|
| **PreparedStatement** | Precompiled SQL with `?` placeholders — prevents SQL injection and improves performance |
| **CallableStatement** | Call stored procedures with IN, OUT, and INOUT parameters using `{call ...}` escape syntax |
| **Parameter Setting** | Use typed setter methods (`setString`, `setInt`, etc.) — parameters are 1-indexed |
| **Updatable ResultSet** | Create with `CONCUR_UPDATABLE` — can update, insert, and delete rows directly |
| **Batch Updates** | Bundle multiple statements with `addBatch()` and send them in one call with `executeBatch()` |
| **Transactions** | Disable auto-commit, execute statements, then `commit()` or `rollback()` as a unit |
| **Isolation Levels** | Control visibility of uncommitted data between concurrent transactions |
| **Savepoints** | Named checkpoints within a transaction — allow partial rollback |

In **Part 3**, we'll explore `RowSet` objects (connected and disconnected), `DataSource` for production-grade connection management, connection pooling, and distributed transactions.
