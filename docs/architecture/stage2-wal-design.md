# Stage 2: WAL Design

> **Status**: Design Complete
> **Impact**: Affects crash recovery of ALL engines
> **Last Updated**: 2026-02-02

---

## 1. Unified WAL Record Header (42 bytes)

Every WAL record across all engines begins with this header.

### Layout

```
Offset  Size  Field           Type    Description
──────  ────  ─────           ────    ───────────
0       8     lsn             u64     Log Sequence Number (monotonically increasing)
8       8     txn_id          u64     Transaction that generated this record
16      8     prev_lsn        u64     Previous LSN for this transaction (undo chain)
24      1     record_type     u8      Type of WAL record (see registry)
25      1     engine_type     u8      Engine that generated this record
26      4     record_length   u32     Total record length including header and payload
30      8     timestamp       u64     Unix timestamp in microseconds
38      4     checksum        u32     CRC32C of entire record (header + payload)
```

**Total header: 42 bytes**

### LSN Design

- LSN = `(segment_number: u32) << 32 | offset_in_segment: u32`
- Monotonically increasing across all engines
- Single WAL writer (all engines serialize through one writer)
- LSN is stored in page headers (`page_lsn`) to determine if redo is needed

### prev_lsn Chain

- Each transaction's WAL records form a linked list via `prev_lsn`
- Used during ABORT: walk the chain backwards, undo each operation
- `prev_lsn = 0` for the first record of a transaction

---

## 2. WAL Record Type Registry

### Meta Records (0x00-0x0F)

```
Value  Name                Payload
─────  ────                ───────
0x00   TXN_BEGIN           { isolation_level: u8 }
0x01   TXN_COMMIT          { }
0x02   TXN_ABORT           { }
0x03   CHECKPOINT_BEGIN    { active_txns: Vec<u64> }
0x04   CHECKPOINT_END      { }
0x05   SCHEMA_CHANGE       { change_type: u8, schema_data: bytes }
```

### Relational Records (0x10-0x1F)

```
Value  Name                Payload
─────  ────                ───────
0x10   PAGE_WRITE          { file_id: u8, page_id: u32, offset: u16, old_data: bytes, new_data: bytes }
0x11   TUPLE_INSERT        { file_id: u8, page_id: u32, slot: u16, tuple_data: bytes }
0x12   TUPLE_DELETE        { file_id: u8, page_id: u32, slot: u16, old_tuple: bytes }
0x13   TUPLE_UPDATE        { file_id: u8, page_id: u32, slot: u16, old_tuple: bytes, new_tuple: bytes }
0x14   BTREE_SPLIT         { file_id: u8, page_id: u32, new_page_id: u32, split_key: bytes, direction: u8 }
0x15   BTREE_MERGE         { file_id: u8, page_id: u32, sibling_id: u32, parent_id: u32 }
0x16   BTREE_INSERT        { file_id: u8, page_id: u32, key: bytes, value: bytes }
0x17   BTREE_DELETE        { file_id: u8, page_id: u32, key: bytes, old_value: bytes }
```

### Vector Records (0x20-0x2F)

```
Value  Name                Payload
─────  ────                ───────
0x20   HNSW_INSERT_NODE    { node_id: u64, layer: u8, neighbors: Vec<(u64, f32)> }
0x21   HNSW_DELETE_NODE    { node_id: u64, layers_affected: Vec<u8> }
0x22   HNSW_UPDATE_EDGES   { node_id: u64, layer: u8, old_neighbors: Vec<u64>, new_neighbors: Vec<u64> }
0x23   HNSW_LAYER_PROMOTE  { node_id: u64, new_max_layer: u8 }
0x24   VECTOR_DATA_WRITE   { file_id: u8, page_id: u32, offset: u16, vector_data: bytes }
0x25   QUANTIZATION_UPDATE { index_id: u32, codebook: bytes }
```

### Graph Records (0x30-0x3F)

```
Value  Name                  Payload
─────  ────                  ───────
0x30   GRAPH_NODE_INSERT     { node_id: u64, labels: Vec<String>, properties: bytes }
0x31   GRAPH_NODE_DELETE     { node_id: u64, old_labels: Vec<String>, old_properties: bytes }
0x32   GRAPH_EDGE_INSERT     { edge_id: u64, src: u64, dst: u64, edge_type: String,
                               properties: bytes, src_adj_page: u32, dst_adj_page: u32 }
0x33   GRAPH_EDGE_DELETE     { edge_id: u64, src: u64, dst: u64,
                               src_adj_page: u32, dst_adj_page: u32, old_data: bytes }
0x34   GRAPH_PROPERTY_UPDATE { entity_type: u8, entity_id: u64,
                               old_properties: bytes, new_properties: bytes }
0x35   ADJ_LIST_PAGE_SPLIT   { node_id: u64, old_page: u32, new_page: u32,
                               direction: u8, split_data: bytes }
```

**Key: Edge operations (0x32, 0x33) record BOTH adjacency list pages** (source and destination). Crash recovery must update both or neither.

### Full-Text Records (0x40-0x4F)

```
Value  Name                  Payload
─────  ────                  ───────
0x40   POSTING_LIST_APPEND   { term_hash: u64, page_id: u32, entry: PostingEntry }
0x41   POSTING_LIST_DELETE   { term_hash: u64, page_id: u32, doc_id: u64 }
0x42   DICTIONARY_INSERT     { term: String, posting_head_page: u32 }
0x43   DICTIONARY_DELETE     { term: String, old_posting_head: u32 }
0x44   TERM_FREQ_UPDATE      { term_hash: u64, doc_id: u64, old_freq: u32, new_freq: u32 }
```

---

## 3. WAL Segment Files

### Segment Format

```
mydb.vaisdb/wal/
├── 000001.wal    # Segment 1 (up to 64MB)
├── 000002.wal    # Segment 2
└── ...
```

### Segment Header (32 bytes, at start of each segment file)

```
Offset  Size  Field             Type    Description
──────  ────  ─────             ────    ───────────
0       4     magic             u32     0x56414C57 ("VALW")
4       4     segment_number    u32     Monotonically increasing segment ID
8       8     first_lsn         u64     LSN of first record in this segment
16      8     last_lsn          u64     LSN of last record (updated on close)
24      4     format_version    u32     WAL format version
28      4     checksum          u32     CRC32C of segment header
```

### Segment Size

- Default: 64MB per segment
- New segment when current exceeds size limit
- Smaller segments = more granular archiving for PITR
- Configurable: `wal_segment_size` (immutable after creation)

---

## 4. Group Commit

### Problem

`fsync()` is the bottleneck: 100-200 TPS without batching (SSD latency ~0.1-0.5ms per fsync).

### Solution: Group Commit

```
Writer Thread:
  1. Append WAL records to buffer (no fsync)
  2. When COMMIT record received:
     a. Add to "pending commit" queue
     b. If queue has records OR timeout (default 1ms):
        - Write all pending records to WAL file
        - Single fsync()
        - Notify all waiting transactions: "committed"
```

### Configuration

```
wal_sync_mode:
  - "fsync"       : fsync per group (default, durable)
  - "fdatasync"   : fdatasync per group (faster, metadata not synced)
  - "async"       : no fsync (fastest, risk of data loss on crash)

wal_group_commit_timeout_us: 1000  # Max wait before forced fsync (microseconds)
wal_buffer_size: 16MB              # In-memory WAL buffer before forced flush
```

### Throughput Impact

```
Without group commit: ~200 TPS (1 fsync per transaction)
With group commit (1ms window): ~10,000+ TPS (1 fsync per batch)
```

---

## 5. Physiological Logging Strategy

### General Principle

VaisDB uses **physiological logging**: physical to the page, logical within the page.

```
WAL record identifies:
  - WHICH page (physical: file_id + page_id)
  - WHAT operation within that page (logical: insert tuple at slot X)
```

### Why Not Pure Physical?

Pure physical (full page images) wastes WAL space. An 8KB page write for a 100-byte tuple update is 80x overhead.

### Why Not Pure Logical?

Pure logical requires deterministic replay. HNSW insertion is non-deterministic (random layer selection), so logical replay would produce different graph structures.

### Per-Engine Strategy

| Engine | Strategy | Reason |
|--------|----------|--------|
| **Relational** | Physiological | Standard: page + intra-page operation |
| **B+Tree** | Physiological + structure modification (split/merge as compound records) | Split must be atomic even if it spans pages |
| **Vector (HNSW)** | Logical (operation + result) | Record the INSERT result (which layer, which neighbors) not the search process. Replay produces identical graph. |
| **Graph** | Physiological + bidirectional compound | Edge insert = compound record with both adjacency pages |
| **Full-text** | Physiological | Posting list append is a page-level operation |

### HNSW Logical Logging Detail

HNSW insertion involves a random layer selection and a greedy search for neighbors. This is non-deterministic. Therefore:

```
Log AFTER the operation completes:
  HNSW_INSERT_NODE {
    node_id: 42,
    max_layer: 2,
    neighbors_layer_0: [(10, 0.95), (23, 0.87), ...],
    neighbors_layer_1: [(5, 0.91), (17, 0.83), ...],
    neighbors_layer_2: [(1, 0.88)]
  }

Replay: directly insert node with these exact neighbors.
No search needed during recovery → deterministic + fast.
```

---

## 6. Checkpoint

### Purpose

- Force all dirty pages to disk
- Advance the recovery starting point
- Allow truncation of old WAL segments

### Checkpoint Process

```
1. Write CHECKPOINT_BEGIN { active_txns } to WAL
2. Flush all dirty pages in buffer pool to disk (can be gradual/fuzzy)
3. fsync all data files
4. Write CHECKPOINT_END to WAL
5. fsync WAL
6. Update meta page with checkpoint LSN
7. WAL segments before checkpoint LSN are safe to truncate/archive
```

### Fuzzy Checkpoint

Pages are flushed while the database continues to serve queries. No global lock required. The `page_lsn` on each flushed page records the state at flush time.

### Checkpoint Trigger

```
wal_checkpoint_interval: 300       # seconds (default 5 minutes)
wal_checkpoint_wal_size: 256MB     # trigger when WAL exceeds this size
```

---

## 7. Crash Recovery

### ARIES-style Recovery (3 phases)

```
Phase 1: ANALYSIS
  - Scan WAL from last checkpoint forward
  - Rebuild Active Transaction Table (ATT)
  - Identify dirty pages

Phase 2: REDO (all engines)
  - Replay WAL records from checkpoint forward
  - For each record: if page_lsn < record_lsn → apply redo
  - This restores ALL changes (committed and uncommitted)
  - Engine tag routes record to correct engine's redo handler

Phase 3: UNDO (uncommitted transactions)
  - For each transaction in ATT that was not committed:
    - Walk prev_lsn chain backwards
    - Apply compensating operations (undo)
    - Write CLR (Compensation Log Record) to WAL
  - After all uncommitted transactions are undone, recovery complete
```

### Cross-Engine Recovery Example

```
Scenario: Crash during a transaction that did:
  1. INSERT INTO docs (content, embedding) VALUES (...)    [SQL + Vector]
  2. Graph edges auto-created for chunking                 [Graph]
  3. Full-text index updated                               [Full-text]
  4. COMMIT (not reached - crash)

Recovery:
  REDO: Replay all 4 engine operations (pages are restored)
  UNDO: Transaction was not committed
    - Undo full-text posting list append
    - Undo graph edge inserts (both adjacency lists)
    - Undo HNSW node insert (mark as soft-deleted)
    - Undo SQL tuple insert
  Result: All 4 engines consistent, as if transaction never happened
```

### Compensation Log Records (CLR)

During UNDO, write CLR records to WAL. This ensures:
- If crash occurs DURING recovery, undo is not re-done (CLR has `undo_next_lsn` pointing past the undone record)
- Recovery is idempotent

---

## 8. WAL Truncation & Archiving

### Truncation

After a successful checkpoint, WAL segments whose last LSN < checkpoint LSN can be deleted.

```
# Safe to delete segments where last_lsn < checkpoint_lsn
# Also ensure no active transaction references records in those segments
```

### Archiving (for PITR)

```
wal_archive_mode: off | on
wal_archive_path: "/backups/wal/"

When on:
  - Completed WAL segments are copied to archive path before deletion
  - PITR: restore base backup + replay archived WAL segments up to target timestamp
```

---

## Verification Checklist

| Item | Status | Notes |
|------|--------|-------|
| Unified WAL record header | Done | 42 bytes, all fields specified |
| Relational WAL records | Done | PAGE_WRITE, TUPLE_*, BTREE_* |
| Vector WAL records | Done | HNSW_*, VECTOR_*, QUANTIZATION_* |
| Graph WAL records | Done | Bidirectional edge records, ADJ_LIST_SPLIT |
| Full-text WAL records | Done | POSTING_*, DICTIONARY_*, TERM_FREQ_* |
| Meta WAL records | Done | TXN_*, CHECKPOINT_*, SCHEMA_CHANGE |
| Physiological logging strategy | Done | Per-engine strategy documented |
| HNSW non-determinism handled | Done | Log result, not process |
| Cross-engine crash recovery | Done | ARIES 3-phase with engine routing |
| Group commit | Done | 1ms batching for throughput |
| Checkpoint design | Done | Fuzzy checkpoint, no global lock |
| WAL archiving hook | Done | For PITR support |
