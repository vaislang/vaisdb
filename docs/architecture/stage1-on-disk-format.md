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
0       8     page_lsn            u64     LSN of last WAL record that modified this page
8       8     txn_id              u64     Transaction that last modified this page
16      4     page_id             u32     Unique page identifier (0 = meta page)
20      4     checksum            u32     CRC32C of entire page (excluding this field)
24      4     prev_page           u32     Previous page in chain (0 = none)
28      4     next_page           u32     Next page in chain (0 = none)
32      4     overflow_page       u32     First overflow page (0 = none)
36      2     free_space_offset   u16     Offset to start of free space within page
38      2     item_count          u16     Number of items/tuples/entries in this page
40      2     flags               u16     Bit flags (see below)
42      2     reserved            u16     Reserved for future use (must be 0)
44      1     page_type           u8      Page type (see registry below)
45      1     engine_tag          u8      Engine that owns this page
46      1     compression_algo    u8      0=none, 1=lz4, 2=zstd
47      1     format_version      u8      Page format version (starts at 1)
```

**Total: 48 bytes (naturally aligned)**

> **Alignment Note**: Fields are ordered for natural alignment: u64 at 8-byte boundaries, u32 at 4-byte boundaries, u16 at 2-byte boundaries. This eliminates padding and allows the CPU to access each field with a single aligned load instruction.

### Design Rationale

- **page_id (u32)**: Supports up to 4 billion pages. At 8KB page size = 32TB max database. At 16KB = 64TB. Sufficient for initial release.
- **page_type + engine_tag**: Separated to allow the same page type concept across engines (e.g., "leaf node" exists in B+Tree and HNSW) while still knowing which engine owns it.
- **page_lsn (u64)**: Critical for crash recovery. Buffer pool compares page_lsn to WAL to determine if page needs redo.
- **checksum (u32)**: CRC32C chosen for hardware acceleration (Intel CRC32 instruction, ARM CRC extension). Computed over entire page with checksum field zeroed. Verified on every page read.
- **format_version (u8)**: Enables lazy online migration. Reader checks version, converts old format on access, writes back new format. No dump/restore needed.
- **free_space_offset**: Points to the boundary between used and free space within the page body. Grows from top (items) and bottom (data), meeting in the middle.

### Flags (u16 bit field)

```
Bit 0:  (reserved)         Reserved, must be 0 (see note below)
Bit 1:  IS_LEAF            Leaf node (B+Tree, HNSW layer 0)
Bit 2:  IS_ROOT            Root node
Bit 3:  IS_OVERFLOW        This page is an overflow continuation
Bit 4:  IS_COMPRESSED      Page body is compressed
Bit 5:  NEEDS_COMPACTION   Page has excessive dead space
Bit 6:  IS_PINNED          Do not evict from buffer pool
Bit 7:  HAS_TOMBSTONES     Page contains soft-deleted entries
Bits 8-15: Reserved (must be 0)
```

> **Note on IS_DIRTY**: Dirty tracking is maintained only in the buffer pool's in-memory page descriptor, not persisted on disk. Bit 0 was originally designated IS_DIRTY but is now reserved. A page's dirty state is transient — it indicates the in-memory copy diverges from the on-disk copy — and has no meaning after being flushed to disk.

### Checksum Calculation

```
fn calculate_checksum(page: &[u8; PAGE_SIZE]) -> u32 {
    // Zero out checksum field (offset 20, 4 bytes)
    let mut buf = page.clone();
    buf[20..24] = [0, 0, 0, 0];
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

0x50   WAL_SEGMENT         common    WAL segment (reserved, see note below)
0x51   UNDO_LOG            common    Undo log pages

0xFE   OVERFLOW            common    Overflow continuation page
0xFF   UNUSED              common    Unallocated page
```

> **Note on WAL_SEGMENT (0x50)**: WAL segment files use their own 32-byte segment header format (see Stage 2), not the unified page header. This page type is reserved for future use if WAL data needs to be stored in the unified page format.

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

> **Note:** The table above shows raw vector data sizes only. Per-tuple MVCC metadata (32 bytes) and HNSW neighbor list pointers are stored separately and do not reduce the available space for vector data within the page body.

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
├── tmp/            # Temporary files (spill-to-disk for sorts/joins)
├── meta.vdb        # Database metadata, configuration
└── lock            # flock file for embedded mode
```

### Design Rationale

- **Separate files per engine**: Enables I/O parallelism. Vector queries don't contend with SQL writes at the filesystem level.
- **Unified page format**: All `.vdb` files use the same page format with the same 48-byte header. Only the page types differ.
- **WAL directory**: Separate WAL segments for easy archiving (PITR). Sequential writes are isolated from random data access.
- **Undo directory**: Undo log separated from data for write pattern optimization.
- **tmp/ directory**: Temporary files for query execution spill-to-disk (sorts, hash joins, materialized intermediate results). Cleaned on database open. Files in this directory are ephemeral and never included in backups.
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

**`meta.vdb`** is the authoritative metadata store for the database, containing all configuration and schema information.

Page 0 of `data.vdb` is a bootstrap page that contains a pointer/reference to `meta.vdb` for bootstrap purposes only. This allows the database open sequence to locate `meta.vdb` even if the directory structure is not yet fully read. The bootstrap page contains:
- Magic number (to identify valid VaisDB data files)
- Page size
- Format version
- Reference to `meta.vdb` (relative path within the bundle directory)

The full metadata in `meta.vdb` includes:
- Database UUID
- Creation timestamp
- Page size
- Format version
- File layout version
- Engine feature flags

### Single-File Mode (Future)

For SQLite-like simplicity, a future option may pack all data into a single file with engine-tagged regions. This is NOT the initial implementation due to I/O contention concerns.

---

## 5. MVCC Tuple Metadata Format (32 bytes)

Every row/tuple in the database carries this metadata for multi-version concurrency control.

### Layout

```
Offset  Size  Field            Type    Description
──────  ────  ─────            ────    ───────────
0       8     txn_id_create    u64     Transaction that created this version
8       8     txn_id_expire    u64     Transaction that deleted/updated (0 = active)
16      8     undo_ptr         u64     Pointer to previous version in undo log
24      4     cmd_id           u32     Command sequence within creating transaction
28      4     expire_cmd_id    u32     Command sequence within deleting/updating transaction
```

**Total: 32 bytes per tuple**

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
- Format: `(file_id: u8, page_id: u32, offset: u16, padding: u8)` packed into u64 (8 + 32 + 16 + 8 = 64 bits)
- **0**: No previous version (first version of this tuple)
- Enables traversing version chain for snapshot isolation

#### cmd_id
- Command sequence number within the creating transaction
- Critical for correctness: `INSERT INTO t SELECT * FROM t` must see the table state BEFORE the INSERT started
- Visibility check includes: `cmd_id < current_cmd_id` for same-transaction reads
- Starts at 0 for each transaction, increments per statement

#### expire_cmd_id
- Command sequence number within the deleting/updating transaction
- Tracks which command within a transaction deleted or updated this tuple
- Works in tandem with `cmd_id`: `cmd_id` tracks the creation command sequence, while `expire_cmd_id` tracks the deletion/update command sequence within the same (or different) transaction
- When `txn_id_expire == current_txn`, `expire_cmd_id` determines whether the expiration is visible to the current command: if `expire_cmd_id >= current_cmd_id`, the tuple is still visible (the deletion/update hasn't happened yet from this command's perspective)
- Set to 0 when `txn_id_expire` is 0 (tuple not expired)

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
        return tuple.expire_cmd_id >= snapshot.current_cmd_id;
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
Per tuple: 32 bytes overhead
Example: 100-byte average row → 32% overhead
Example: 1KB average row → 3.1% overhead
Example: 6KB vector → 0.5% overhead

This is acceptable. InnoDB uses similar overhead (13 bytes hidden columns + undo pointer).
The additional 4 bytes (expire_cmd_id) provide correct same-transaction visibility semantics.
```

---

## 6. Torn Write Protection (Full Page Image)

### The Torn Write Problem

An 8KB (or 16KB) database page write is **not atomic** at the filesystem level. Most filesystems and storage devices guarantee atomicity only at the hardware sector level (typically 512 bytes or 4KB). If VaisDB writes an 8KB page and the system crashes mid-write, the on-disk page may be **half-updated**: the first 4KB contains new data while the second 4KB still contains old data (or vice versa). This is a **torn write** (also called a partial write or fractured write).

A torn page is internally inconsistent — the checksum will fail, but more critically, the data cannot be trusted. Simply discarding the page is not an option since it may contain committed data. Standard WAL redo cannot fix this either, because redo applies a logical delta that assumes the base page is intact.

### Solution: Full Page Image (FPI)

VaisDB adopts the **Full Page Image** approach, following PostgreSQL's design:

1. **After each checkpoint**, a per-page flag (`needs_fpi`) is set for every page in the buffer pool.
2. **On the first modification** to any page after a checkpoint, the complete **old page image** (the full 8KB/16KB page as it existed on disk) is written to the WAL **before** the modification's redo record.
3. This FPI WAL record is written only **once per page per checkpoint cycle**. After the FPI is written, subsequent modifications to the same page within the same checkpoint cycle write only the normal (compact) redo records.
4. The `needs_fpi` flag is cleared after the FPI is written.

### Recovery with FPI

During crash recovery, if a page's checksum fails (torn write detected):

1. Scan the WAL backward to find the most recent FPI record for that page.
2. **Restore** the page from the FPI (overwrite the torn page with the complete page image from WAL).
3. Then **replay** all subsequent redo records for that page to bring it to the latest committed state.

This guarantees that recovery always has a consistent base page to work from, even if the on-disk copy is torn.

### Trade-offs

| Aspect | FPI (VaisDB/PostgreSQL) | Doublewrite Buffer (InnoDB) |
|--------|------------------------|---------------------------|
| WAL size | Larger (full page images in WAL) | Smaller WAL, but extra doublewrite file |
| Write amplification | Higher right after checkpoint | Constant 2x for all dirty pages |
| Implementation complexity | Simpler (WAL-only mechanism) | More complex (separate doublewrite file) |
| Recovery speed | Fast (FPI directly in WAL stream) | Extra step to check doublewrite buffer |

**Design decision**: FPI is chosen over InnoDB's doublewrite buffer for its simplicity. The WAL size increase is transient (only right after each checkpoint) and is bounded by the number of distinct pages modified in the first checkpoint cycle.

### FPI WAL Record Format

```
FPI WAL record:
  - record_type:  FPI (dedicated WAL record type)
  - file_id:      u8    Which data file
  - page_id:      u32   Which page
  - page_data:    [u8; PAGE_SIZE]  Complete page image
```

The FPI record is a self-contained snapshot of the page. No dependency on other WAL records for correctness.

---

## Verification Checklist

| Item | Status | Notes |
|------|--------|-------|
| Page header format documented | Done | 48 bytes, naturally aligned, all fields specified |
| format_version included | Done | Offset 47, u8 |
| All page types enumerated | Done | 16 types across 5 engines |
| Page size decision documented | Done | 8KB default, 16KB option |
| File layout documented | Done | Bundle directory with per-engine files + tmp/ |
| meta.vdb vs page 0 clarified | Done | meta.vdb authoritative, page 0 bootstrap only |
| MVCC metadata format documented | Done | 32 bytes, visibility function specified |
| expire_cmd_id field added | Done | Offset 28, u32, same-txn expiration tracking |
| undo_ptr packing corrected | Done | u8 + u32 + u16 + u8 = 64 bits |
| Visibility function pseudocode | Done | Handles same-txn cmd_id/expire_cmd_id, snapshot isolation |
| Torn write protection documented | Done | FPI approach (PostgreSQL style) |
| IS_DIRTY flag clarified | Done | In-memory only, not persisted on disk |
| WAL_SEGMENT page type clarified | Done | Reserved; WAL uses own segment header format |
| Extension points identified | Done | Reserved page types, engine tags |
