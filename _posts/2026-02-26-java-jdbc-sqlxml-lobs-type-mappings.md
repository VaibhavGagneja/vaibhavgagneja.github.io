---
title: "Java JDBC Part 4: SQLXML, Large Objects, and Custom Type Mappings"
description: Work with XML data via SQLXML objects, read and write BLOBs and CLOBs for binary and text large objects, and map SQL types to custom Java classes with custom type mappings
author: Vaibhav Gagneja
date: 2026-02-26 12:00:00 +0530
categories: [Development, Java]
tags: [java, jdbc, database, sql, sqlxml, blob, clob, type-mapping, large-objects, xml]
toc: true
image:
  path: https://images.unsplash.com/photo-1544383835-bda2bc66a55d
---

In [Part 1](/posts/java-jdbc-fundamentals/) we covered connections and basic operations, [Part 2](/posts/java-jdbc-transactions-prepared-statements/) handled prepared statements and transactions, and [Part 3](/posts/java-jdbc-rowset-datasource-pooling/) explored RowSets and connection pooling. This final part tackles advanced data types — storing XML directly in the database, handling binary and text large objects, and mapping SQL types to your own Java classes.

---

## 1. Using SQLXML Objects

The `SQLXML` interface was introduced in JDBC 4.0 and provides a way to interact with SQL `XML` data directly. Instead of treating XML as plain strings, SQLXML lets you work with XML data through streams, DOM trees, SAX parsers, or StAX cursors.

### What Is SQLXML?

```
┌──────────────────────────────────────────────────────────────┐
│                      SQLXML OBJECT                             │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Database column of type XML                                   │
│  ┌──────────────────────────────────────┐                     │
│  │ <rss>                                │                     │
│  │   <channel>                          │                     │
│  │     <title>Coffee News</title>       │                     │
│  │     <item>                           │                     │
│  │       <title>New Blend</title>       │                     │
│  │     </item>                          │                     │
│  │   </channel>                         │                     │
│  │ </rss>                               │                     │
│  └──────────────────────────────────────┘                     │
│                                                                │
│  How to access it from Java:                                   │
│                                                                │
│  ┌─────────────┐                                              │
│  │   SQLXML     │─── getString()      → plain XML string      │
│  │   object     │─── getSource()      → DOM, SAX, or StAX     │
│  │              │─── getBinaryStream() → raw bytes             │
│  │              │─── getCharStream()   → character stream      │
│  └─────────────┘                                              │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Creating and Inserting an SQLXML Value

```java
public void insertXMLData(Connection con, String coffeeName, String xmlData)
    throws SQLException {

    // Create a SQLXML object
    SQLXML sqlxml = con.createSQLXML();
    sqlxml.setString(xmlData);

    // Insert it into a table with an XML column
    String sql = "INSERT INTO RSS_FEEDS (COF_NAME, RSS_FEED_XML) VALUES (?, ?)";
    try (PreparedStatement pstmt = con.prepareStatement(sql)) {
        pstmt.setString(1, coffeeName);
        pstmt.setSQLXML(2, sqlxml);
        pstmt.executeUpdate();
    }
}
```

### Retrieving SQLXML Data

```java
public void readXMLData(Connection con) throws SQLException {
    String sql = "SELECT COF_NAME, RSS_FEED_XML FROM RSS_FEEDS";
    try (Statement stmt = con.createStatement();
         ResultSet rs = stmt.executeQuery(sql)) {

        while (rs.next()) {
            SQLXML xmlVal = rs.getSQLXML("RSS_FEED_XML");
            String xmlString = xmlVal.getString();
            System.out.println(rs.getString("COF_NAME") + ":");
            System.out.println(xmlString);

            // Important: free the SQLXML object when done
            xmlVal.free();
        }
    }
}
```

> **Important:** Always call `sqlxml.free()` when you're done. SQLXML objects hold onto database resources that won't be released until `free()` is called or the object is garbage collected.

### Working with SQLXML Through Streams

For large XML documents, working with strings is inefficient. Use streams instead:

```java
// Writing XML via a stream
SQLXML sqlxml = con.createSQLXML();
OutputStream os = sqlxml.setBinaryStream();
// Write your XML data to the OutputStream...
os.close();

// Reading XML via a stream
SQLXML xmlVal = rs.getSQLXML("RSS_FEED_XML");
InputStream is = xmlVal.getBinaryStream();
// Parse the InputStream with any XML parser...
is.close();
xmlVal.free();
```

### Using DOM, SAX, or StAX with SQLXML

SQLXML supports four common XML processing approaches:

| Method | Returns | When to Use |
|--------|---------|-------------|
| `getString()` / `setString()` | `String` | Small XML documents where memory is not a concern |
| `getBinaryStream()` / `setBinaryStream()` | `InputStream` / `OutputStream` | Streaming large XML as raw bytes |
| `getCharacterStream()` / `setCharacterStream()` | `Reader` / `Writer` | Streaming large XML as characters |
| `getSource(cls)` / `setResult(cls)` | `Source` / `Result` | Full DOM/SAX/StAX processing |

Using XSLT to transform SQLXML data:

```java
// Get XML as a Source for XSLT transformation
SQLXML xmlVal = rs.getSQLXML("RSS_FEED_XML");
Source source = xmlVal.getSource(DOMSource.class);

// Transform using XSLT
TransformerFactory tf = TransformerFactory.newInstance();
Transformer transformer = tf.newTransformer(
    new StreamSource(new File("transform.xslt")));

DOMResult result = new DOMResult();
transformer.transform(source, result);

// The result is a DOM Document you can work with
Document doc = (Document) result.getNode();
xmlVal.free();
```

---

## 2. Using Large Objects (LOBs)

Databases need to store more than numbers and short strings. **Large Objects** handle data that's too big for regular column types — think images, PDFs, audio files, or long text documents.

### BLOB, CLOB, and NCLOB

| Type | Full Name | Stores | Java Interface |
|------|-----------|--------|----------------|
| **BLOB** | Binary Large Object | Binary data (images, PDFs, audio) | `java.sql.Blob` |
| **CLOB** | Character Large Object | Large text (articles, logs, XML) | `java.sql.Clob` |
| **NCLOB** | National Character Large Object | Large text in national character set (Unicode) | `java.sql.NClob` |

```
┌──────────────────────────────────────────────────────────────┐
│                    LARGE OBJECT TYPES                           │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  BLOB:                                                         │
│  ┌───────────────────────────────────┐                        │
│  │ 0110 1001 0010 1110 1101 0011 ... │  Raw binary bytes      │
│  │  (images, PDFs, compressed data)  │                        │
│  └───────────────────────────────────┘                        │
│                                                                │
│  CLOB:                                                         │
│  ┌───────────────────────────────────┐                        │
│  │ "A long article about Colombian   │  UTF-8 text            │
│  │  coffee that spans thousands of   │                        │
│  │  characters..."                   │                        │
│  └───────────────────────────────────┘                        │
│                                                                │
│  NCLOB:                                                        │
│  ┌───────────────────────────────────┐                        │
│  │ "コーヒーについての記事..."          │  National charset text │
│  └───────────────────────────────────┘                        │
│                                                                │
│  Key point: LOBs are POINTERS in the database.                │
│  The actual data may be stored separately from the row.        │
│  Accessing LOB data is a STREAMING operation.                  │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Storing a CLOB (Text)

The Coffee Break wants to store a detailed description for each coffee. Since descriptions can be very long, they use a `CLOB` column:

```java
public void setCoffeeDescription(String coffeeName, String fileName)
    throws SQLException, IOException {

    String sql = "UPDATE COFFEES SET DESCRIPTION = ? WHERE COF_NAME = ?";

    try (PreparedStatement pstmt = con.prepareStatement(sql)) {

        // Read the description from a text file
        File file = new File(fileName);
        FileReader reader = new FileReader(file);

        // Set the CLOB parameter — stream the file contents
        pstmt.setCharacterStream(1, reader, (int) file.length());
        pstmt.setString(2, coffeeName);
        pstmt.executeUpdate();

        reader.close();
    }
}
```

### Reading a CLOB

```java
public String getCoffeeDescription(String coffeeName) throws SQLException {
    String sql = "SELECT DESCRIPTION FROM COFFEES WHERE COF_NAME = ?";

    try (PreparedStatement pstmt = con.prepareStatement(sql)) {
        pstmt.setString(1, coffeeName);
        ResultSet rs = pstmt.executeQuery();

        if (rs.next()) {
            Clob descClob = rs.getClob("DESCRIPTION");
            // Convert to string (for small CLOBs)
            String description = descClob.getSubString(1, (int) descClob.length());
            descClob.free();    // release resources
            return description;
        }
    }
    return null;
}
```

For large CLOBs, use a stream instead of loading everything into memory:

```java
Clob descClob = rs.getClob("DESCRIPTION");
Reader reader = descClob.getCharacterStream();
BufferedReader br = new BufferedReader(reader);

String line;
while ((line = br.readLine()) != null) {
    System.out.println(line);
}
br.close();
descClob.free();
```

### Storing a BLOB (Binary Data)

```java
public void setCoffeeImage(String coffeeName, String imagePath)
    throws SQLException, IOException {

    String sql = "UPDATE COFFEES SET IMAGE = ? WHERE COF_NAME = ?";

    try (PreparedStatement pstmt = con.prepareStatement(sql)) {
        File imageFile = new File(imagePath);
        FileInputStream fis = new FileInputStream(imageFile);

        pstmt.setBinaryStream(1, fis, (int) imageFile.length());
        pstmt.setString(2, coffeeName);
        pstmt.executeUpdate();

        fis.close();
    }
}
```

### Reading a BLOB

```java
public void getCoffeeImage(String coffeeName, String outputPath)
    throws SQLException, IOException {

    String sql = "SELECT IMAGE FROM COFFEES WHERE COF_NAME = ?";

    try (PreparedStatement pstmt = con.prepareStatement(sql)) {
        pstmt.setString(1, coffeeName);
        ResultSet rs = pstmt.executeQuery();

        if (rs.next()) {
            Blob imageBlob = rs.getBlob("IMAGE");
            InputStream is = imageBlob.getBinaryStream();

            // Write to a file
            FileOutputStream fos = new FileOutputStream(outputPath);
            byte[] buffer = new byte[4096];
            int bytesRead;
            while ((bytesRead = is.read(buffer)) != -1) {
                fos.write(buffer, 0, bytesRead);
            }

            fos.close();
            is.close();
            imageBlob.free();
        }
    }
}
```

### Creating a BLOB or CLOB Directly

You can create LOB objects from the `Connection` and then populate them:

```java
// Create a Blob
Blob blob = con.createBlob();
OutputStream os = blob.setBinaryStream(1);   // position starts at 1
// write bytes to os...
os.close();

// Create a Clob
Clob clob = con.createClob();
Writer writer = clob.setCharacterStream(1);
writer.write("Long text content here...");
writer.close();

// Use them in a PreparedStatement
pstmt.setBlob(1, blob);
pstmt.setClob(2, clob);
pstmt.executeUpdate();

// Release resources when done
blob.free();
clob.free();
```

### LOB Lifecycle Rules

| Rule | Detail |
|------|--------|
| **Validity** | A LOB's data is valid at least until the transaction in which it was created ends |
| **Free resources** | Always call `lob.free()` when done — LOBs hold database resources |
| **Stream access** | Use streams for large data — loading entire LOBs into memory defeats the purpose |
| **Transaction scope** | In auto-commit mode, each statement is its own transaction — the LOB may become invalid after `executeUpdate()` returns |

> **Note:** Some JDBC drivers let you use `setString()` or `setBytes()` for small LOB values, but for portability and large data, always use the streaming API (`setCharacterStream`, `setBinaryStream`).

---

## 3. Using Datalink and ROWID

Beyond LOBs and XML, JDBC supports two more specialized column types worth knowing about.

### DATALINK — Referencing External Files

A `DATALINK` column stores a URL that points to a resource outside the database — a file on a server, a web page, or any URI-addressable resource.

```java
// Reading a DATALINK value
String sql = "SELECT DOCUMENT_NAME, URL FROM DATA_REPOSITORY";
try (Statement stmt = con.createStatement();
     ResultSet rs = stmt.executeQuery(sql)) {

    while (rs.next()) {
        String docName = rs.getString("DOCUMENT_NAME");
        java.net.URL url = rs.getURL("URL");
        System.out.println(docName + " -> " + url.toString());
    }
}
```

```java
// Inserting a DATALINK value
String sql = "INSERT INTO DATA_REPOSITORY (DOCUMENT_NAME, URL) VALUES (?, ?)";
try (PreparedStatement pstmt = con.prepareStatement(sql)) {
    pstmt.setString(1, "CoffeeBlog");
    pstmt.setURL(2, new URL("https://coffeebreak.example.com/blog"));
    pstmt.executeUpdate();
}
```

### ROWID — Database Row Identifiers

A `ROWID` is a value that uniquely identifies a row in the database. It's database-specific and not part of standard SQL, but it can be used for fast row access:

```java
// Retrieve ROWID
ResultSet rs = stmt.executeQuery("SELECT ROWID, * FROM COFFEES");
while (rs.next()) {
    RowId rowId = rs.getRowId(1);
    System.out.println("RowID: " + rowId.toString());
}

// Use ROWID to access a specific row
PreparedStatement pstmt = con.prepareStatement(
    "SELECT * FROM COFFEES WHERE ROWID = ?");
pstmt.setRowId(1, rowId);
```

| Property | Detail |
|----------|--------|
| `ROWID_UNSUPPORTED` | Database does not support ROWIDs |
| `ROWID_VALID_OTHER` | ROWID validity is implementation-dependent |
| `ROWID_VALID_TRANSACTION` | Valid only for the duration of the transaction |
| `ROWID_VALID_SESSION` | Valid for the duration of the session |
| `ROWID_VALID_FOREVER` | Valid for the lifetime of the row |

```java
// Check ROWID support
DatabaseMetaData meta = con.getMetaData();
RowIdLifetime lifetime = meta.getRowIdLifetime();
if (lifetime == RowIdLifetime.ROWID_UNSUPPORTED) {
    System.out.println("This database does not support ROWIDs");
}
```

---

## 4. Customized Type Mappings

JDBC provides a way to map SQL structured types (`STRUCT`, `DISTINCT`, user-defined types) to your own Java classes. Instead of working with generic `Struct` objects, you can work with proper Java objects like `Address` or `Manager`.

### How It Works

```
┌──────────────────────────────────────────────────────────────┐
│              CUSTOM TYPE MAPPING FLOW                           │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Database (SQL Structured Type):                               │
│  CREATE TYPE ADDRESS AS (                                      │
│      NUM INTEGER, STREET VARCHAR(40),                          │
│      CITY VARCHAR(20), STATE CHAR(2), ZIP CHAR(5)              │
│  )                                                             │
│                                                                │
│                    │                                           │
│                    ▼  Without custom mapping                   │
│                                                                │
│  Java gets a generic Struct object:                            │
│  Struct address = (Struct) rs.getObject("ADDRESS");            │
│  Object[] attrs = address.getAttributes();                     │
│  // attrs[0] = number, attrs[1] = street, etc.                 │
│  // No type safety, easy to get index wrong                    │
│                                                                │
│                    │                                           │
│                    ▼  With custom mapping                      │
│                                                                │
│  Java gets YOUR class directly:                                │
│  Address address = (Address) rs.getObject("ADDRESS");          │
│  address.getStreet();   // type-safe!                          │
│  address.getCity();     // no index guessing                   │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Step 1: Define the SQL Type

```sql
CREATE TYPE ADDRESS
(
    NUM INTEGER,
    STREET VARCHAR(40),
    CITY VARCHAR(20),
    STATE CHAR(2),
    ZIP CHAR(5)
);
```

### Step 2: Create a Java Class That Implements SQLData

```java
public class Address implements SQLData {
    private String sqlType;
    private int num;
    private String street;
    private String city;
    private String state;
    private String zip;

    // Called by JDBC to determine the SQL type name
    @Override
    public String getSQLTypeName() throws SQLException {
        return sqlType;
    }

    // Called by JDBC when reading from the database
    @Override
    public void readSQL(SQLInput stream, String typeName) throws SQLException {
        this.sqlType = typeName;
        this.num    = stream.readInt();
        this.street = stream.readString();
        this.city   = stream.readString();
        this.state  = stream.readString();
        this.zip    = stream.readString();
    }

    // Called by JDBC when writing to the database
    @Override
    public void writeSQL(SQLOutput stream) throws SQLException {
        stream.writeInt(num);
        stream.writeString(street);
        stream.writeString(city);
        stream.writeString(state);
        stream.writeString(zip);
    }

    // Standard getters and setters...
    public String getStreet() { return street; }
    public String getCity()   { return city; }
    public String getState()  { return state; }
    public String getZip()    { return zip; }
}
```

> **Important:** The order of `readXxx()` calls in `readSQL` must match the order of attributes in the SQL type definition. Similarly, the order of `writeXxx()` calls in `writeSQL` must match. Getting the order wrong will silently produce wrong values.

### Step 3: Register the Mapping

```java
// Tell the Connection how to map the SQL type to your Java class
Map<String, Class<?>> typeMap = con.getTypeMap();
typeMap.put("ADDRESS", Address.class);
con.setTypeMap(typeMap);
```

### Step 4: Use It

Now when you call `getObject()` on a column of type `ADDRESS`, JDBC automatically creates an `Address` instance:

```java
String sql = "SELECT STORE_NAME, ADDRESS FROM STORE_INFORMATION";
try (Statement stmt = con.createStatement();
     ResultSet rs = stmt.executeQuery(sql)) {

    while (rs.next()) {
        String storeName = rs.getString("STORE_NAME");
        // JDBC uses the type map to convert automatically
        Address addr = (Address) rs.getObject("ADDRESS");

        System.out.println(storeName + ": " +
            addr.getStreet() + ", " +
            addr.getCity() + ", " +
            addr.getState() + " " +
            addr.getZip());
    }
}
```

### Nested Structured Types

SQL structured types can contain other structured types. For example, a `MANAGER` type that contains an `ADDRESS`:

```sql
CREATE TYPE MANAGER
(
    MGR_ID INTEGER,
    LAST_NAME VARCHAR(40),
    FIRST_NAME VARCHAR(40),
    PHONE VARCHAR(20),
    ADDRESS ADDRESS      -- nested structured type!
);
```

The Java class mirrors this nesting:

```java
public class Manager implements SQLData {
    private int mgrId;
    private String lastName;
    private String firstName;
    private String phone;
    private Address address;    // nested!

    @Override
    public void readSQL(SQLInput stream, String typeName) throws SQLException {
        this.mgrId     = stream.readInt();
        this.lastName  = stream.readString();
        this.firstName = stream.readString();
        this.phone     = stream.readString();
        // readObject() uses the type map to create an Address
        this.address   = (Address) stream.readObject();
    }

    @Override
    public void writeSQL(SQLOutput stream) throws SQLException {
        stream.writeInt(mgrId);
        stream.writeString(lastName);
        stream.writeString(firstName);
        stream.writeString(phone);
        stream.writeObject(address);    // uses writeSQL of Address
    }

    // ...
}
```

Register both mappings:

```java
Map<String, Class<?>> typeMap = con.getTypeMap();
typeMap.put("ADDRESS", Address.class);
typeMap.put("MANAGER", Manager.class);
con.setTypeMap(typeMap);
```

### DISTINCT Types

A `DISTINCT` type is an alias for an existing SQL type, giving it a domain-specific name:

```sql
CREATE TYPE MONEY AS NUMERIC(10, 2);
```

You can map it to a custom Java class the same way:

```java
public class Money implements SQLData {
    private BigDecimal amount;

    @Override
    public void readSQL(SQLInput stream, String typeName) throws SQLException {
        this.amount = stream.readBigDecimal();
    }

    @Override
    public void writeSQL(SQLOutput stream) throws SQLException {
        stream.writeBigDecimal(amount);
    }

    public BigDecimal getAmount() { return amount; }
}
```

### Using Without Mapping: Struct

If you don't want to create custom classes, you can work with the generic `Struct` interface:

```java
Struct address = (Struct) rs.getObject("ADDRESS");
Object[] attrs = address.getAttributes();

int num        = (Integer) attrs[0];
String street  = (String) attrs[1];
String city    = (String) attrs[2];
String state   = (String) attrs[3];
String zip     = (String) attrs[4];
```

| Approach | Pros | Cons |
|----------|------|------|
| **Custom mapping (`SQLData`)** | Type-safe, self-documenting, encapsulated | More code to write upfront |
| **Generic (`Struct`)** | Quick, no extra classes needed | Index-based, fragile, no type safety |

---

## 5. SQL Type Mapping Reference

Here's a complete reference of how SQL types map to Java types and the corresponding getter methods:

| SQL Type | Java Type | ResultSet Getter | PreparedStatement Setter |
|----------|-----------|-------------------|--------------------------|
| `CHAR`, `VARCHAR` | `String` | `getString()` | `setString()` |
| `INTEGER` | `int` | `getInt()` | `setInt()` |
| `BIGINT` | `long` | `getLong()` | `setLong()` |
| `SMALLINT` | `short` | `getShort()` | `setShort()` |
| `FLOAT`, `DOUBLE` | `double` | `getDouble()` | `setDouble()` |
| `REAL` | `float` | `getFloat()` | `setFloat()` |
| `NUMERIC`, `DECIMAL` | `BigDecimal` | `getBigDecimal()` | `setBigDecimal()` |
| `BOOLEAN` | `boolean` | `getBoolean()` | `setBoolean()` |
| `DATE` | `java.sql.Date` | `getDate()` | `setDate()` |
| `TIME` | `java.sql.Time` | `getTime()` | `setTime()` |
| `TIMESTAMP` | `java.sql.Timestamp` | `getTimestamp()` | `setTimestamp()` |
| `BLOB` | `Blob` | `getBlob()` | `setBlob()` |
| `CLOB` | `Clob` | `getClob()` | `setClob()` |
| `NCLOB` | `NClob` | `getNClob()` | `setNClob()` |
| `ARRAY` | `Array` | `getArray()` | `setArray()` |
| `XML` | `SQLXML` | `getSQLXML()` | `setSQLXML()` |
| `ROWID` | `RowId` | `getRowId()` | `setRowId()` |
| `DATALINK` | `URL` | `getURL()` | `setURL()` |
| `STRUCT` | `Struct` or custom `SQLData` | `getObject()` | `setObject()` |
| `DISTINCT` | Underlying type | Depends on base type | Depends on base type |

> **Note:** `java.sql.Date`, `java.sql.Time`, and `java.sql.Timestamp` are subclasses of `java.util.Date`. In modern Java (8+), you can convert them to `LocalDate`, `LocalTime`, and `LocalDateTime` respectively using the `toLocalDate()`, `toLocalTime()`, and `toLocalDateTime()` methods.

---

## 6. When to Use What

```
┌──────────────────────────────────────────────────────────────┐
│                    DECISION GUIDE                               │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  "I need to store/read XML data in the database"               │
│     → Use SQLXML objects                                       │
│     → Small XML: getString()/setString()                       │
│     → Large XML: streams or Source/Result                      │
│                                                                │
│  "I need to store images, PDFs, or binary files"               │
│     → Use BLOB with setBinaryStream()                          │
│                                                                │
│  "I need to store large text (articles, logs, descriptions)"   │
│     → Use CLOB with setCharacterStream()                       │
│                                                                │
│  "I need to store multilingual text"                           │
│     → Use NCLOB                                                │
│                                                                │
│  "I have a SQL user-defined type (STRUCT)"                     │
│     → Implement SQLData for type-safe custom mapping           │
│     → Or use Struct for quick, generic access                  │
│                                                                │
│  "I need to reference an external file from a row"             │
│     → Use DATALINK (stores a URL in the database)              │
│                                                                │
│  "I need a fast handle to a specific database row"             │
│     → Use ROWID (if your database supports it)                 │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## 7. Summary

| Concept | Key Takeaway |
|---------|-------------|
| **SQLXML** | JDBC 4.0 interface for reading/writing XML data in SQL XML columns — supports strings, streams, DOM, SAX, and StAX |
| **BLOB** | Binary Large Object — use `setBinaryStream()` / `getBinaryStream()` for images, files, and binary data |
| **CLOB / NCLOB** | Character Large Object — use `setCharacterStream()` / `getCharacterStream()` for large text data |
| **Datalink** | Stores a URL reference to an external resource — read with `getURL()` |
| **ROWID** | Database-specific row identifier — check `getRowIdLifetime()` for validity scope |
| **SQLData** | Interface for mapping SQL structured types to custom Java classes — type-safe and self-documenting |
| **Struct** | Generic representation of SQL structured types — quick but index-based and fragile |
| **Type Map** | `Connection.setTypeMap()` registers SQL type name to Java class mappings |
| **DISTINCT** | SQL alias for an existing type — can be mapped to a custom Java class via SQLData |

This concludes the JDBC series. Across all four parts, we've covered everything from your first `getConnection()` call to handling custom SQL types. The JDBC API is large, but the core patterns are consistent: get a connection, create a statement, execute, process results, and clean up resources.
