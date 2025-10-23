# Database Indexing - Complete Interview Notes

## Table of Contents
1. [Understanding Page, Slot, and Offset](#understanding-page-slot-and-offset)
2. [B-Tree Structure with Examples](#b-tree-structure-with-examples)
3. [B+Tree Structure with Examples](#b-tree-structure-with-examples-1)
4. [How Database Indexing Actually Works](#how-database-indexing-actually-works)
5. [Three Real Scenarios with Examples](#three-real-scenarios-with-examples)

---

## 1. Understanding Page, Slot, and Offset

Before understanding indexes, you need to know how database actually stores data on disk.

### What is a Page?

Think of a page as a **fixed-size container** where database stores data. 

- **Size**: Typically 8 KB (8192 bytes)
- **Pages are numbered**: Page 0, Page 1, Page 2, etc.
- Database reads/writes data in **complete pages** (not individual rows)

**Simple Analogy**: Think of a page like a notebook page - it has a fixed size, and you write multiple entries on it.

### Structure of a Page

Every page has three main sections:

```
┌─────────────────────────────────────────────┐
│         PAGE HEADER (96 bytes)              │ ← Metadata about page
├─────────────────────────────────────────────┤
│         ACTUAL ROW DATA                     │ ← Your table rows stored here
│         (Grows downward ↓)                  │
│                                             │
│         FREE SPACE                          │
│                                             │
│         (Grows upward ↑)                    │
│         SLOT ARRAY / ROW OFFSET ARRAY       │ ← Pointers to each row
└─────────────────────────────────────────────┘
```

### What is a Slot?

A **slot** is like an **index entry** that tells you where a specific row starts within the page.

- Each slot = **2 bytes**
- Slots are stored at the **bottom of the page**
- Each slot contains the **offset** (position) where a row begins

**Simple Analogy**: Think of slots like a table of contents at the back of a chapter, telling you which line each paragraph starts on.

### What is an Offset?

An **offset** is simply a **number indicating the starting position** of a row within the page.

- Measured in bytes from the start of the page
- Example: Offset 150 means the row starts at byte position 150 in the page

### Real Example: Page with 3 Rows

```
Page Structure:

Byte 0-95:    [Page Header - metadata]

Byte 96:      Start of Row 1 → "1, John, 25, IT"
Byte 150:     Start of Row 2 → "2, Sarah, 30, HR"  
Byte 210:     Start of Row 3 → "3, Mike, 28, Finance"

Byte 8186:    Slot 2 = 210 (points to Row 3)
Byte 8188:    Slot 1 = 150 (points to Row 2)
Byte 8190:    Slot 0 = 96  (points to Row 1)
```

**How it works:**
- Database wants Row 2
- Looks at Slot 1 → finds offset 150
- Goes to byte position 150 → reads Row 2 data

### What is RID (Row Identifier)?

When a table has **no clustered index** (called a Heap table), each row is identified by a **RID**.

**RID Structure:**
```
RID = FileID : PageID : SlotID
      (2 bytes) (4 bytes) (2 bytes)
Total = 8 bytes
```

**Real Example:**
```
RID = 1:230:2

Means:
- File 1 (database can have multiple files)
- Page 230 (in that file)
- Slot 2 (on that page)
```

**Simple Analogy**: RID is like a complete address:
- FileID = City
- PageID = Street number
- SlotID = Apartment number

---

## 2. B-Tree Structure with Examples

### What is a B-Tree?

A **B-Tree** is a tree data structure where:
- Each node can store **multiple keys** (not just one)
- Each node can have **multiple children** (not just two)
- Tree stays **balanced** automatically

### Why Multiple Keys Per Node?

**Traditional Binary Tree Problem:**
```
Binary Tree (one key per node):
         50
        /  \
      30    70
     / \    / \
   20  40 60  80

Height = 3 levels
For 1 million records → Height = 20 levels
= 20 disk reads! (SLOW)
```

**B-Tree Solution:**
```
B-Tree (multiple keys per node):
        [30, 50, 70]
       /    |    |    \
   [20]  [40]  [60]  [80]

Height = 2 levels
For 1 million records → Height = 3-4 levels
= 3-4 disk reads! (FAST)
```

### Understanding "Keys" in B-Tree

When we say **"keys"** in database context:
- Keys = **actual values** from your indexed column
- Example: If you index on EmployeeID, keys are actual IDs like 10, 20, 30

**Not pointers** - they are actual data values!

### B-Tree Node Structure

Each node contains:
1. **Keys** (actual values)
2. **Data** or **pointer to data**
3. **Child pointers** (pointing to other nodes)

```
Internal Node Structure:

[Pointer] [Key1] [Pointer] [Key2] [Pointer] [Key3] [Pointer]
    ↓       10       ↓       20       ↓       30       ↓
  Child1          Child2           Child3          Child4
```

### Order of B-Tree

**Order** = Maximum number of children a node can have

**Order 3 B-Tree** = Each node can have maximum 3 children

Rules for Order 3:
- Maximum keys per node = Order - 1 = **2 keys**
- Maximum children per node = **3 children**
- Minimum children per node = ⌈3/2⌉ = **2 children** (except root)

### Complete Order 2 B-Tree Example

**Order 2 means:**
- Max keys per node = 1
- Max children per node = 2

**Example: Index on EmployeeID**

```
Step 1: Insert EmployeeID = 10

Root Node:
[10]

Step 2: Insert 20

Root Node:
[10, 20]  ← Node is FULL (max 1 key for order 2)
            Node must SPLIT!

After Split:
         [20]           ← 20 moves up
        /    \
      [10]  (empty)    ← 10 stays left

Step 3: Insert 30

         [20]
        /    \
      [10]  [30]       ← 30 goes to right

Step 4: Insert 5

         [20]
        /    \
    [5,10]  [30]       ← 5 inserted on left
    
Left node FULL! Must split:

         [10, 20]       ← Both 10 and 20 at root now
        /   |    \
      [5] [15] [30]

And so on...
```

### Complete Order 3 B-Tree Example

**Order 3 means:**
- Max keys per node = 2
- Max children per node = 3

**Example: Building index on Student IDs**

**Keys to insert: 10, 20, 30, 40, 25, 15**

```
Step 1: Insert 10, 20
Root: [10, 20]  ← Can fit 2 keys

Step 2: Insert 30
Root: [10, 20, 30]  ← FULL! Must split

After split:
           [20]              ← Middle key moves up
          /    \
       [10]    [30]          ← Split into 2 nodes

Step 3: Insert 40
           [20]
          /    \
       [10]   [30, 40]       ← 40 added to right

Step 4: Insert 25
           [20]
          /    \
       [10]   [25, 30, 40]   ← FULL! Right node must split

After split:
           [20, 30]          ← 30 moves up to root
          /    |    \
       [10]  [25]  [40]      ← Split right node

Step 5: Insert 15
           [20, 30]
          /    |    \
     [10,15] [25]  [40]      ← 15 added to leftmost node
```

**Final B-Tree Structure:**
```
            [20, 30]          ← Root (internal node)
           /    |    \
      [10,15] [25]  [40]     ← Leaf nodes
```

**Key Properties:**
- **All levels** store keys AND data (or pointers to data)
- Data can be found at **any level** (root, internal, or leaf)
- Tree is **balanced** - all leaf nodes at same level

### How B-Tree Search Works

**Search for StudentID = 25:**

```
Step 1: Start at root [20, 30]
        25 > 20 and 25 < 30
        → Go to middle child

Step 2: Reach node [25]
        Found! Data is right here.

Total: 2 disk reads
```

---

## 3. B+Tree Structure with Examples

### What is a B+Tree?

B+Tree is a **variation** of B-Tree with important differences:

**Main Differences:**
1. **Only leaf nodes store actual data**
2. **Internal nodes store only keys** (for navigation)
3. **Leaf nodes are linked** together (doubly-linked list)

### Why B+Tree is Better for Databases

**Advantages:**
1. **More keys fit in internal nodes** (no data stored there)
   - Shorter tree = Fewer disk reads
2. **Range queries are fast** (just follow leaf links)
3. **All data at same level** (predictable performance)

### Complete Order 3 B+Tree Example

**Order 3 means:**
- Max keys per node = 2
- Max children per node = 3

**Keys to insert: 10, 20, 30, 40, 25, 15**

```
Step 1: Insert 10, 20
Leaf: [10:Data10, 20:Data20]  ← Leaf stores key + data

Step 2: Insert 30
Leaf: [10:D10, 20:D20, 30:D30]  ← FULL! Must split

After split:
         [20]                    ← Internal node (key ONLY)
        /    \
  [10:D10]  [20:D20, 30:D30]   ← Leaf nodes (key + data)
      ↔           ↔              ← Leaves are LINKED

Note: Key 20 appears in BOTH internal node and leaf

Step 3: Insert 40
         [20]
        /    \
  [10:D10]  [20:D20, 30:D30, 40:D40]  ← Right leaf FULL!

After split:
         [20, 30]                ← Internal node
        /    |    \
  [10:D10] [20:D20] [30:D30, 40:D40]  ← 3 leaf nodes
      ↔        ↔          ↔            ← All linked

Step 4: Insert 25
         [20, 30]
        /    |    \
  [10:D10] [20:D20] [25:D25, 30:D30, 40:D40]  ← FULL!

After split:
         [20, 25, 30]          ← Internal node updated
        /    |    |    \
 [10:D10] [20:D20] [25:D25] [30:D30, 40:D40]
     ↔        ↔        ↔          ↔

Step 5: Insert 15
         [20, 25, 30]
        /    |    |    \
[10:D10,    [20:D20] [25:D25] [30:D30, 40:D40]
 15:D15]
     ↔        ↔        ↔          ↔
```

**Final B+Tree Structure:**
```
INTERNAL LEVEL:
         [20, 25, 30]          ← Keys only, no data
        /    |    |    \

LEAF LEVEL (all linked):
[10:D10,  ↔  [20:D20]  ↔  [25:D25]  ↔  [30:D30,
 15:D15]                                 40:D40]

All actual data is at leaf level!
```

### How B+Tree Search Works

**Search for key = 25:**

```
Step 1: Start at root [20, 25, 30]
        25 = 25
        → Go to 3rd child

Step 2: Reach leaf [25:Data25]
        Found data!

Total: 2 disk reads
```

**Range query: Find all keys from 20 to 40:**

```
Step 1: Find key 20 (navigate to leaf)
Step 2: Follow linked list from leaf to leaf
        [20] → [25] → [30, 40]
        
Done! Just sequential read through leaves.
```

---

## 4. How Database Indexing Actually Works

### Index as a Separate Structure

**Important Concept**: An index is a **separate data structure** (usually B+Tree) built on top of your table.

```
Your Table:                     Index on EmployeeID:
┌─────────────┐                ┌──────────────┐
│ Page 0      │                │  Index Root  │
│ Rows 1-100  │                │    [500]     │
├─────────────┤                ├──────────────┤
│ Page 1      │                │ [100] [1000] │
│ Rows 101-200│                ├──────────────┤
├─────────────┤                │ Leaf Nodes   │
│ Page 2      │  ←─ Points to ─│ [1][2]...[99]│
│ Rows 201-300│                └──────────────┘
└─────────────┘
```

### Clustered Index vs Non-Clustered Index

This is the **most important concept** to understand!

---

## 5. Three Real Scenarios with Examples

Let's understand with a **real Employee table**.

**Table: Employees**
```
Columns: EmployeeID, Name, Age, Department, Salary
```

---

### Scenario 1: Table WITHOUT Primary Key (HEAP)

**What happens:**
- No primary key defined
- No clustered index created
- Table stored as **HEAP** (unordered collection)

**Create Table:**
```sql
CREATE TABLE Employees (
    EmployeeID INT,
    Name VARCHAR(50),
    Age INT,
    Department VARCHAR(50),
    Salary DECIMAL(10,2)
);

-- Insert data
INSERT INTO Employees VALUES 
(5, 'John', 30, 'IT', 75000),
(2, 'Sarah', 28, 'HR', 65000),
(8, 'Mike', 35, 'IT', 85000),
(1, 'Emma', 26, 'Finance', 60000);
```

**Physical Storage (Heap - No Order):**
```
┌──────────────────────────────────┐
│ Page 100 (File 1)                │
│ Slot 0 → Row: 5,John,30,IT,75000 │  ← RID = 1:100:0
│ Slot 1 → Row: 2,Sarah,28,HR,65000│  ← RID = 1:100:1
├──────────────────────────────────┤
│ Page 101 (File 1)                │
│ Slot 0 → Row: 8,Mike,35,IT,85000 │  ← RID = 1:101:0
│ Slot 1 → Row: 1,Emma,26,Fin,60000│  ← RID = 1:101:1
└──────────────────────────────────┘

Rows stored in INSERT order (no sorting)
Each row identified by RID = FileID:PageID:SlotID
```

**Now Create Non-Clustered Index:**
```sql
CREATE NONCLUSTERED INDEX idx_name 
ON Employees(Name);
```

**Index Structure (B+Tree):**
```
Non-Clustered Index on Name:

Root Node:
    [Mike]
    /      \

Internal Level:
[Emma, John]  [Sarah]

Leaf Level (contains Name + RID):
[Emma → 1:101:1] ← [John → 1:100:0] ← [Mike → 1:101:0] ← [Sarah → 1:100:1]
    ↓ RID             ↓ RID              ↓ RID              ↓ RID
Points to heap     Points to heap     Points to heap     Points to heap
```

**Query: Find employee named "John"**
```sql
SELECT * FROM Employees WHERE Name = 'John';
```

**Execution Steps:**
```
Step 1: Search index for 'John'
        Navigate B+Tree → Find leaf node
        Leaf contains: [John → 1:100:0]

Step 2: Use RID = 1:100:0 to fetch actual row
        RID means: File 1, Page 100, Slot 0
        
Step 3: Go to File 1, read Page 100, access Slot 0
        Slot 0 points to offset where John's row starts
        
Step 4: Read complete row: 5,John,30,IT,75000

Total: 
- 3 disk reads (index navigation)
- 1 disk read (heap page)
= 4 disk reads
```

**Key Points:**
- Heap table uses **RID (8 bytes)** to identify rows
- RID = FileID:PageID:SlotID
- Non-clustered index stores: **Index Key + RID**
- Two lookups needed: Index → RID → Heap table

---

### Scenario 2: Table WITH Primary Key (Clustered Index)

**What happens:**
- Primary key automatically creates **CLUSTERED INDEX**
- Table data is **physically sorted** by primary key
- Data stored in **B+Tree structure** (not heap)

**Create Table:**
```sql
CREATE TABLE Employees (
    EmployeeID INT PRIMARY KEY,  ← Creates CLUSTERED INDEX
    Name VARCHAR(50),
    Age INT,
    Department VARCHAR(50),
    Salary DECIMAL(10,2)
);

-- Insert data (same as before)
INSERT INTO Employees VALUES 
(5, 'John', 30, 'IT', 75000),
(2, 'Sarah', 28, 'HR', 65000),
(8, 'Mike', 35, 'IT', 85000),
(1, 'Emma', 26, 'Finance', 60000);
```

**Physical Storage (Clustered Index - Sorted by EmployeeID):**

```
Clustered Index B+Tree Structure:

ROOT NODE (Internal):
    [5]
   /   \

INTERNAL LEVEL:
[2]      [8]

LEAF LEVEL (contains ACTUAL TABLE DATA):

Page 200:                           Page 201:
Slot 0 → [1,Emma,26,Finance,60000]  Slot 0 → [5,John,30,IT,75000]
Slot 1 → [2,Sarah,28,HR,65000]      Slot 1 → [8,Mike,35,IT,85000]

The leaf nodes of the clustered index ARE the table data!
Sorted by EmployeeID: 1, 2, 5, 8
```

**Important**: 
- Leaf level = Actual table data pages
- Data physically sorted by EmployeeID
- No separate heap storage

**Query: Find EmployeeID = 5**
```sql
SELECT * FROM Employees WHERE EmployeeID = 5;
```

**Execution Steps:**
```
Step 1: Search clustered index B+Tree for key = 5
        Navigate: Root [5] → go right → Leaf node

Step 2: Reach leaf node (Page 201, Slot 0)
        Data is RIGHT HERE: 5,John,30,IT,75000

Step 3: Return row

Total: 2-3 disk reads (just B+Tree navigation)
No additional lookup needed!
```

---

### Scenario 3: Clustered Index + Non-Clustered Index

**What happens:**
- Table has clustered index (from primary key)
- We add additional non-clustered index on another column
- Non-clustered index points to **clustered index key** (not RID)

**Same Table as Scenario 2, Add Non-Clustered Index:**
```sql
-- Table already has PRIMARY KEY on EmployeeID (clustered)

-- Add non-clustered index on Name
CREATE NONCLUSTERED INDEX idx_name 
ON Employees(Name);
```

**Storage Structure:**

**Clustered Index (Primary Key on EmployeeID):**
```
Leaf Level = Actual Data (sorted by EmployeeID):
[1,Emma,26,Finance,60000] → [2,Sarah,28,HR,65000] → 
[5,John,30,IT,75000] → [8,Mike,35,IT,85000]
```

**Non-Clustered Index on Name:**
```
Non-Clustered Index B+Tree:

Leaf Level (contains Name + EmployeeID):
[Emma → 1] ← [John → 5] ← [Mike → 8] ← [Sarah → 2]
    ↓            ↓            ↓            ↓
Points to      Points to    Points to    Points to
EmployeeID 1   EmployeeID 5 EmployeeID 8 EmployeeID 2
in clustered   in clustered in clustered in clustered
index          index        index        index
```

**Key Insight**: 
- Non-clustered index stores: **Index Key + Clustered Key**
- It does NOT store RID (because table is not a heap)
- It stores the **primary key value** (EmployeeID)

**Query: Find employee named "John"**
```sql
SELECT * FROM Employees WHERE Name = 'John';
```

**Execution Steps:**
```
Step 1: Search non-clustered index on Name
        Navigate B+Tree → Find 'John'
        Leaf node contains: [John → 5]
        Meaning: John has EmployeeID = 5

Step 2: Use EmployeeID = 5 to search clustered index
        Navigate clustered index B+Tree for key = 5
        
Step 3: Reach leaf of clustered index
        Leaf contains actual data: 5,John,30,IT,75000

Step 4: Return row

Total: 
- 2-3 disk reads (non-clustered index navigation)
- 2-3 disk reads (clustered index navigation)  
= 4-6 disk reads

This is called a "KEY LOOKUP"
```

---

## Summary Comparison: All Three Scenarios

### Scenario 1: Heap + Non-Clustered Index

```
Table Structure:
Heap (unordered pages)

Non-Clustered Index:
Index Key → RID (FileID:PageID:SlotID) → Heap Table

RID = 8 bytes
```

**Example:**
```
Index: [Name='John' → RID=1:100:3]
        ↓
Go to File 1, Page 100, Slot 3
        ↓
Read actual row data
```

---

### Scenario 2: Clustered Index Only

```
Table Structure:
Clustered Index B+Tree (leaf = data)

No separate index needed for primary key lookups
```

**Example:**
```
Search clustered index for EmployeeID=5
        ↓
Navigate B+Tree to leaf
        ↓
Leaf contains actual row data (no additional lookup)
```

---

### Scenario 3: Clustered Index + Non-Clustered Index

```
Table Structure:
Clustered Index B+Tree (leaf = data)

Non-Clustered Index:
Index Key → Clustered Key Value → Clustered Index

Clustered Key = typically 4 bytes (INT)
```

**Example:**
```
Non-Clustered Index: [Name='John' → EmployeeID=5]
        ↓
Search clustered index for EmployeeID=5
        ↓
Navigate clustered index B+Tree to leaf
        ↓
Leaf contains actual row data
```

---

## Hidden Clustered Index (MySQL/InnoDB Special Case)

**What if no primary key is defined?**

Different databases handle this differently:

**SQL Server:**
- Table becomes a HEAP
- Uses RID for row identification

**MySQL (InnoDB):**
- **Automatically creates hidden clustered index**
- Uses hidden 6-byte **ROW_ID** column
- You never see this column

```sql
-- MySQL InnoDB
CREATE TABLE Employees (
    Name VARCHAR(50),
    Age INT
);
-- No primary key!

-- InnoDB creates hidden column:
-- ROW_ID (6 bytes, auto-increment)
-- Creates clustered index on ROW_ID

-- Behind the scenes:
CREATE TABLE Employees (
    ROW_ID BIGINT PRIMARY KEY,  ← Hidden, auto-added
    Name VARCHAR(50),
    Age INT
);
```

---

## Uniquifier (Non-Unique Clustered Index)

**What if clustered index is NOT unique?**

SQL Server adds a hidden **4-byte UNIQUIFIER** column.

```sql
CREATE TABLE Employees (
    Department VARCHAR(50),
    Name VARCHAR(50)
);

-- Create NON-UNIQUE clustered index
CREATE CLUSTERED INDEX idx_dept 
ON Employees(Department);
```

**What happens internally:**
```
Insert rows:
(IT, John)
(IT, Sarah)    ← Duplicate Department!
(HR, Mike)
(IT, Emma)     ← Duplicate Department!

SQL Server adds hidden uniquifier:

Department  Name   Uniquifier (hidden)
IT          John   (not added - first occurrence)
IT          Sarah  1          ← Added because duplicate
HR          Mike   (not added - first occurrence)
IT          Emma   2          ← Added because duplicate

Actual clustered index key = (Department + Uniquifier)
Makes every row unique: (IT,0), (IT,1), (HR,0), (IT,2)
```

**Non-clustered indexes** on this table will store:
- Index key + (Department + Uniquifier)
- Uses 4 extra bytes per index entry!

---

## Quick Reference: Index Types

| Scenario | Table Structure | Row Locator | Size |
|----------|----------------|-------------|------|
| **Heap + Non-Clustered Index** | Heap (unordered) | RID (FileID:PageID:SlotID) | 8 bytes |
| **Clustered Index Only** | B+Tree (sorted) | N/A (data at leaf) | 0 bytes |
| **Clustered + Non-Clustered** | B+Tree (sorted) | Clustered Key (e.g., INT) | 4 bytes |
| **Non-Unique Clustered + Non-Clustered** | B+Tree (sorted) | Clustered Key + Uniquifier | 8 bytes |

---

## Important Interview Points

### 1. RID vs Clustered Key

**RID (Row Identifier):**
- Used in **HEAP tables** (no clustered index)
- Format: FileID (2 bytes) + PageID (4 bytes) + SlotID (2 bytes) = **8 bytes**
- Physical pointer - tells exact location on disk

**Clustered Key:**
- Used when table has **clustered index**
- Just the primary key value (e.g., INT = 4 bytes)
- Logical pointer - search clustered index to find row

### 2. Why Clustered Index is Faster

**Heap + Non-Clustered:**
```
Query → Non-Clustered Index → RID → Heap Table
        3 reads              1 read    = 4 reads
```

**Clustered Index:**
```
Query → Clustered Index → Data is right there!
        2-3 reads           = 2-3 reads
```

### 3. Why Only One Clustered Index?

- Clustered index defines **physical order** of data
- Data can only be sorted **one way** physically
- Multiple non-clustered indexes possible (they're separate structures)

### 4. Page, Slot, Offset Simplified

```
Page = Container (8 KB)
Slot = Index entry pointing to row (2 bytes)
Offset = Starting position of row in bytes

RID = File 1, Page 230, Slot 5
      ↓      ↓         ↓
    Which  Which     Which row
    file?  page?     on page?
```

### 5. B-Tree Order Explained

```
Order 3 B-Tree:
- Max 2 keys per node
- Max 3 children per node

Node: [Key1, Key2]
       /    |    \
   Child1 Child2 Child3
   
   Values < Key1
          Key1 ≤ Values < Key2
                  Values ≥ Key2
```

---

## Common Interview Questions

**Q: What is the difference between clustered and non-clustered index?**

**A:** 
- **Clustered index**: Defines physical order of table data. Leaf nodes contain actual data rows. Only one per table.
- **Non-clustered index**: Separate structure from table. Leaf nodes contain index key + pointer (RID or clustered key). Multiple allowed per table.

**Q: What is RID?**

**A:** RID (Row Identifier) is an 8-byte pointer used in heap tables. Format: FileID:PageID:SlotID. It tells the exact physical location of a row on disk.

**Q: Why does non-clustered index on a clustered table use clustered key instead of RID?**

**A:** Because the table data is organized as a B+Tree (clustered index), not a heap. The clustered key uniquely identifies each row. It's also smaller (e.g., 4 bytes for INT) than RID (8 bytes).

**Q: What happens when you don't define a primary key?**

**A:** 
- **SQL Server**: Table becomes a HEAP. Non-clustered indexes use RID.
- **MySQL InnoDB**: Automatically creates hidden clustered index on hidden ROW_ID column.

**Q: Explain B-Tree vs B+Tree in indexing.**

**A:**
- **B-Tree**: Data stored at all levels (internal + leaf nodes). Faster for exact match if found at internal level.
- **B+Tree**: Data stored only at leaf level. Internal nodes only have keys for navigation. Leaf nodes are linked. Better for range queries and sequential scans. Most databases use B+Tree.

---

## Conclusion

**Remember these key concepts:**

1. **Page** = 8KB storage unit, contains rows + slot array
2. **Slot** = 2-byte pointer to row offset within page
3. **RID** = FileID:PageID:SlotID (8 bytes) - used in heap tables
4. **Clustered Index** = Data sorted physically, leaf nodes ARE the data
5. **Non-Clustered Index** = Separate structure, points to RID or clustered key
6. **B+Tree** = Most common structure, data only at leaves, linked for range queries

Understanding these fundamentals will help you ace database indexing questions in interviews!