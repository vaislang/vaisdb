# Stage 1: On-Disk Format Decisions

> **Status**: Design Complete
> **Impact**: IMMUTABLE after first user - these decisions cannot be changed without full migration
> **Last Updated**: 2026-02-02

---

## 1. Unified Page Header Format (48 bytes)

All 4 engines (relational, vector, graph, full-text) share this header. Every page on disk begins with exactly these 48 bytes.

### Layout

```
Offset  Size  Field               Type    Description
──────  ────  ─────               ────    ───────────
0       4     page_id             u32     Unique page identifier (0 = meta page)
4       1     page_type           u8      Page type (see registry below)
5       1     engine_tag          u8      Engine that owns this page
6       8     page_lsn            u64     LSN of last WAL record that modified this page
14      8     txn_id              u64     Transaction that last modified this page
22      4     checksum            u32     CRC32C of entire page (excluding this field)
26      2     flags               u16     Bit flags (see below)
28      2     free_space_offset   u16     Offset to start of free space within page
30      2     item_count          u16     Number of items/tuples/entries in this page
32      4     prev_page           u32     Previous page in chain (0 = none)
36      4     next_page           u32     Next page in chain (0 = none)
40      4     overflow_page       u32     First overflow page (0 = none)
44      1     compression_algo    u8      0=none, 1=lz4, 2=zstd
45      1     format_version      u8      Page format version (starts at 1)
46      2     reserved            u16     Reserved for future use (must be 0)
```

**Total: 48 bytes**

### Design Rationale

- **page_id (u32)**: Supports up to 4 billion pages. At 8KB page size = 32TB max database. At 16KB = 64TB. Sufficient for initial release.
- **page_type + engine_tag**: Separated to allow the same page type concept across engines (e.g., "leaf node" exists in B+Tree and HNSW) while still knowing which engine owns it.
- **page_lsn (u64)**: Critical for crash recovery. Buffer pool compares page_lsn to WAL to determine if page needs redo.
- **checksum (u32)**: CRC32C chosen for hardware acceleration (Intel CRC32 instruction, ARM CRC extension). Computed over entire page with checksum field zeroed. Verified on every page read.
- **format_version (u8)**: Enables lazy online migration. Reader checks version, converts old format on access, writes back new format. No dump/restore needed.
- **free_space_offset**: Points to the boundary between used and free space within the page body. Grows from top (items) and bottom (data), meeting in the middle.

### Flags (u16 bit field)

```
Bit 0:  IS_DIRTY          Page has uncommitted modifications
Bit 1:  IS_LEAF            Leaf node (B+Tree, HNSW layer 0)
Bit 2:  IS_ROOT            Root node
Bit 3:  IS_OVERFLOW        This page is an overflow continuation
Bit 4:  IS_COMPRESSED      Page body is compressed
Bit 5:  NEEDS_COMPACTION   Page has excessive dead space
Bit 6:  IS_PINNED          Do not evict from buffer pool
Bit 7:  HAS_TOMBSTONES     Page contains soft-deleted entries
Bits 8-15: Reserved (must be 0)
```

### Checksum Calculation

```
fn calculate_checksum(page: &[u8; PAGE_SIZE]) -> u32 {
    // Zero out checksum field (offset 22, 4 bytes)
    let mut buf = page.clone();
    buf[22..26] = [0, 0, 0, 0];
    crc32c(&buf)
}
```

Checksum is verified on every read from disk. Mismatch indicates corruption → return error, do not serve corrupted data.

---

## 2. Page Type Registry

Each page type has a unique u8 identifier. Engine tag disambiguates when needed.

### Page Types

```
Value  Name                Engine    Description
─────  ────                ──────    ───────────
0x00   META                common    Database metadata (page 0 only)
0x01   FREELIST            common    Free page tracking
0x02   CATALOG             common    Schema catalog pages

0x10   DATA                sql       Heap data pages (tuples)
0x11   BTREE_INTERNAL      sql       B+Tree internal node
0x12   BTREE_LEAF          sql       B+Tree leaf node

0x20   HNSW_META           vector    HNSW index metadata (parameters, entry point)
0x21   HNSW_NODE           vector    HNSW node data (vector + neighbor lists)
0x22   HNSW_LAYER          vector    HNSW layer structure
0x23   VECTOR_DATA         vector    Raw vector storage (for overflow)

0x30   GRAPH_NODE          graph     Graph node storage
0x31   GRAPH_ADJ           graph     Adjacency list pages
0x32   GRAPH_PROPERTY      graph     Node/edge property storage

0x40   INVERTED_DICT       fulltext  Dictionary B+Tree pages
0x41   INVERTED_POSTING    fulltext  Posting list pages
0x42   INVERTED_META       fulltext  Full-text index metadata

0x50   WAL_SEGMENT         common    WAL segment (not in main data file)
0x51   UNDO_LOG            common    Undo log pages

0xFE   OVERFLOW            common    Overflow continuation page
0xFF   UNUSED              common    Unallocated page
```

### Engine Tags

```
Value  Engine
─────  ──────
0x00   common (shared infrastructure)
0x01   sql (relational engine)
0x02   vector (vector engine)
0x03   graph (graph engine)
0x04   fulltext (full-text engine)
```

### Extension Strategy

- Page types 0x60-0xEF are reserved for future engines
- Engine tags 0x05-0x0F are reserved for future engines
- Any new page type must be registered here before use

---

## 3. Default Page Size

### Decision: 8KB default, 16KB for vector-heavy workloads

```
Page size is set at CREATE DATABASE and is IMMUTABLE for the lifetime of the database.
```

### Trade-off Analysis

| Factor | 4KB | 8KB (default) | 16KB (vector) |
|--------|-----|---------------|---------------|
| B+Tree fanout | Low | Good | High |
| Vector fit (1536-dim f32) | No (6KB > 4KB) | No (6KB > 8KB body) | Yes (6KB < ~16KB body) |
| Vector fit (768-dim f32) | No (3KB) | Yes | Yes |
| I/O amplification | Low | Medium | Higher |
| Buffer pool memory efficiency | Good | Good | Lower |
| Overflow frequency | High | Medium | Low |

### Vector Size Reference

```
Dimensions   Bytes (f32)   Fits in 8KB body?   Fits in 16KB body?
384          1,536         Yes                  Yes
768          3,072         Yes                  Yes
1024         4,096         Yes                  Yes
1536         6,144         No (overflow)        Yes
3072         12,288        No (overflow)        No (overflow)
```

### Decision Rationale

- **8KB default**: Good balance for mixed workloads. Most vectors (up to 1024-dim) fit in a single page. B+Tree fanout is reasonable. Matches common filesystem block sizes.
- **16KB option**: For workloads dominated by 1536-dim vectors (OpenAI embeddings). Eliminates overflow for the most common large embedding size.
- Vectors that don't fit in page body use overflow pages (linked list via `overflow_page` in header).

### Page Body Size

```
Page body = PAGE_SIZE - 48 (header) = 8144 bytes (8KB) or 16336 bytes (16KB)
```

---

## 4. File Layout Strategy

### Decision: Bundle Directory (`.vaisdb/`)

A VaisDB database is a **directory** that appears as a single logical unit.

```
mydb.vaisdb/
├── data.vdb        # Main data file: heap pages, B+Tree, catalog
├── vectors.vdb     # Vector data: HNSW nodes, vector storage
├── graph.vdb       # Graph data: nodes, adjacency lists
├── fulltext.vdb    # Full-text: dictionary, posting lists
├── wal/
│   ├── 000001.wal  # WAL segment files (64MB each)
│   ├── 000002.wal
│   └── ...
├── undo/
│   └── undo.vdb    # Undo log for MVCC
├── meta.vdb        # Database metadata, configuration
└── lock            # flock file for embedded mode
```

### Design Rationale

- **Separate files per engine**: Enables I/O parallelism. Vector queries don't contend with SQL writes at the filesystem level.
- **Unified page format**: All `.vdb` files use the same page format with the same 48-byte header. Only the page types differ.
- **WAL directory**: Separate WAL segments for easy archiving (PITR). Sequential writes are isolated from random data access.
- **Undo directory**: Undo log separated from data for write pattern optimization.
- **lock file**: `flock()` on this file prevents multiple processes opening the same DB in embedded mode.

### Page ID Space

Each file has its own page ID space starting from 0. Pages are addressed as `(file_id, page_id)`:

```
file_id  File
───────  ────
0        data.vdb
1        vectors.vdb
2        graph.vdb
3        fulltext.vdb
4        undo.vdb
```

Page 0 of `data.vdb` is the database meta page containing:
- Database UUID
- Creation timestamp
- Page size
- Format version
- File layout version
- Engine feature flags

### Single-File Mode (Future)

For SQLite-like simplicity, a future option may pack all data into a single file with engine-tagged regions. This is NOT the initial implementation due to I/O contention concerns.

---

## 5. MVCC Tuple Metadata Format (28 bytes)

Every row/tuple in the database carries this metadata for multi-version concurrency control.

### Layout

```
Offset  Size  Field           Type    Description
──────  ────  ─────           ────    ───────────
0       8     txn_id_create   u64     Transaction that created this version
8       8     txn_id_expire   u64     Transaction that deleted/updated (0 = active)
16      8     undo_ptr        u64     Pointer to previous version in undo log
24      4     cmd_id          u32     Command sequence within transaction
```

**Total: 28 bytes per tuple**

### Field Details

#### txn_id_create
- Set when the tuple is first inserted
- Never changes after creation
- Used by snapshot visibility: `txn_id_create <= snapshot.txn_id && txn_id_create NOT IN snapshot.active_set`

#### txn_id_expire
- **0**: Tuple is current (not deleted or updated)
- **Non-zero**: Transaction ID that deleted or replaced this tuple
- On UPDATE: old tuple gets `txn_id_expire = current_txn`, new tuple is inserted with new `txn_id_create`
- On DELETE: tuple gets `txn_id_expire = current_txn`

#### undo_ptr
- Points to the previous version of this tuple in the undo log
- Format: `(file_id: u8, page_id: u32, offset: u16, padding: u16)` packed into u64
- **0**: No previous version (first version of this tuple)
- Enables traversing version chain for snapshot isolation

#### cmd_id
- Command sequence number within a single transaction
- Critical for correctness: `INSERT INTO t SELECT * FROM t` must see the table state BEFORE the INSERT started
- Visibility check includes: `cmd_id < current_cmd_id` for same-transaction reads
- Starts at 0 for each transaction, increments per statement

### Visibility Function (Pseudocode)

```
fn is_visible(tuple: &Tuple, snapshot: &Snapshot) -> bool {
    let created_by = tuple.txn_id_create;
    let expired_by = tuple.txn_id_expire;

    // Case 1: Created by current transaction
    if created_by == snapshot.current_txn {
        // Check cmd_id for same-transaction visibility
        if tuple.cmd_id >= snapshot.current_cmd_id {
            return false;  // Created by later command in same txn
        }
        // Not yet expired, or expired by later command
        if expired_by == 0 {
            return true;
        }
        if expired_by == snapshot.current_txn {
            return tuple.expire_cmd_id >= snapshot.current_cmd_id;
        }
        return true;
    }

    // Case 2: Created by committed transaction visible to this snapshot
    if !is_committed_before(created_by, snapshot) {
        return false;  // Creator not yet committed or aborted
    }

    // Case 3: Check expiration
    if expired_by == 0 {
        return true;  // Not expired
    }
    if expired_by == snapshot.current_txn {
        return false;  // Expired by current transaction
    }
    if !is_committed_before(expired_by, snapshot) {
        return true;  // Expirer not yet committed = still visible
    }
    return false;  // Expired by committed transaction
}

fn is_committed_before(txn_id: u64, snapshot: &Snapshot) -> bool {
    txn_id < snapshot.txn_id && txn_id NOT IN snapshot.active_txns
}
```

### MVCC Overhead Analysis

```
Per tuple: 28 bytes overhead
Example: 100-byte average row → 28% overhead
Example: 1KB average row → 2.7% overhead
Example: 6KB vector → 0.5% overhead

This is acceptable. InnoDB uses similar overhead (13 bytes hidden columns + undo pointer).
```

---

## Verification Checklist

| Item | Status | Notes |
|------|--------|-------|
| Page header format documented | Done | 48 bytes, all fields specified |
| format_version included | Done | Offset 45, u8 |
| All page types enumerated | Done | 16 types across 5 engines |
| Page size decision documented | Done | 8KB default, 16KB option |
| File layout documented | Done | Bundle directory with per-engine files |
| MVCC metadata format documented | Done | 28 bytes, visibility function specified |
| Visibility function pseudocode | Done | Handles same-txn cmd_id, snapshot isolation |
| Extension points identified | Done | Reserved page types, engine tags |
