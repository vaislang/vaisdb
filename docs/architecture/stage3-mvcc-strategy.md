# Stage 3: MVCC Strategy Decisions

> **Status**: Design Complete
> **Impact**: Affects correctness of ALL concurrent operations across all engines
> **Last Updated**: 2026-02-02

---

## 1. Update Strategy: In-Place Update + Undo Log

### Decision

**In-place update** with undo log (InnoDB-style), NOT append-only (PostgreSQL-style).

### Rationale

| Factor | In-Place + Undo | Append-Only |
|--------|----------------|-------------|
| Vector data bloat | Minimal (1 copy) | Severe (6KB * N versions) |
| Read amplification | Follow undo chain | Scan HOT chain |
| Write amplification | Low (update + undo) | Low (append) |
| VACUUM necessity | Undo log cleanup only | Full table VACUUM (HOT chains) |
| Graph adjacency bloat | Minimal | Each edge update duplicates list |

For VaisDB, vector data at 1536-dim is ~6KB per vector. Append-only would create massive bloat with even moderate update rates. In-place update keeps the current version on the data page and pushes old versions to undo log.

### How It Works

```
UPDATE row:
  1. Copy current row to undo log (including MVCC metadata)
  2. Modify row in-place on data page
  3. Set new row's undo_ptr → undo log entry
  4. Old snapshot readers follow undo_ptr to find their visible version

DELETE row:
  1. Set txn_id_expire = current_txn_id
  2. Row remains on page until GC reclaims it

INSERT row:
  1. Write new row to data page
  2. Set txn_id_create = current_txn_id, txn_id_expire = 0
  3. undo_ptr = 0 (no previous version)
```

---

## 2. MVCC for Vector Indexes: Post-Filter Strategy

### Problem

HNSW index is a shared structure. It cannot maintain per-transaction versions of the graph. But different transactions may see different sets of valid vectors.

### Decision: Post-Filter with Adaptive Oversampling

```
Vector Search Process:
  1. Search HNSW for top_k * oversample_factor candidates
  2. For each candidate, check MVCC visibility against current snapshot
  3. Filter out invisible candidates
  4. Return top_k visible results
```

### Oversample Factor

The oversample factor compensates for candidates that are invisible to the current snapshot (uncommitted inserts by other transactions, or soft-deleted vectors).

```
Base oversample factor: 2.0 (search 2x candidates, filter to k)

Adaptive formula:
  uncommitted_ratio = active_uncommitted_vectors / total_indexed_vectors
  oversample = max(2.0, 1.0 / (1.0 - uncommitted_ratio) * 1.5)

Example:
  1% uncommitted → oversample = 2.0 (base is fine)
  10% uncommitted → oversample = max(2.0, 1.67) = 2.0
  30% uncommitted → oversample = max(2.0, 2.14) = 2.14
  50% uncommitted → oversample = max(2.0, 3.0) = 3.0
```

### When Oversample Is Insufficient

If after filtering there are fewer than `top_k` results:
1. Double the oversample factor
2. Re-search (with higher `ef_search` if needed)
3. If still insufficient after 3 retries, return available results with warning

### Soft-Deleted Vectors in HNSW

Deleted vectors are NOT immediately removed from the HNSW graph (would require expensive graph repair). Instead:
- Marked as soft-deleted (IS_TOMBSTONE flag in HNSW node)
- Skipped during search result collection
- Physically removed during GC compaction (graph repair phase)

---

## 3. MVCC for Graph Adjacency Lists

### Design

Each adjacency list entry carries MVCC metadata:

```
AdjEntry {
    target_node:    u64,    // Destination node ID
    edge_id:        u64,    // Edge ID for property lookup
    edge_type:      u16,    // Edge type (index into type table)
    txn_id_create:  u64,    // Transaction that created this edge
    txn_id_expire:  u64,    // Transaction that deleted this edge (0 = active)
}

Total: 34 bytes per adjacency entry
```

### Traversal Visibility

During graph traversal, each edge is checked for visibility:

```
fn traverse_neighbors(node_id: u64, snapshot: &Snapshot) -> Vec<AdjEntry> {
    let adj_page = load_adjacency_page(node_id);
    adj_page.entries
        .filter(|e| is_edge_visible(e, snapshot))
        .collect()
}

fn is_edge_visible(entry: &AdjEntry, snapshot: &Snapshot) -> bool {
    // Created by committed txn visible to this snapshot
    let created_visible = is_committed_before(entry.txn_id_create, snapshot)
                          || entry.txn_id_create == snapshot.current_txn;

    // Not yet expired, or expired by uncommitted txn
    let not_expired = entry.txn_id_expire == 0
                      || (!is_committed_before(entry.txn_id_expire, snapshot)
                          && entry.txn_id_expire != snapshot.current_txn);

    created_visible && not_expired
}
```

### Snapshot-Consistent Multi-Hop Traversal

Entire multi-hop traversals use a single snapshot. This guarantees:
- No phantom edges appearing mid-traversal
- No disappearing nodes between hops
- Consistent view of the graph at a single point in time

### Bitmap Cache for Hot Snapshots

For frequently accessed snapshots (e.g., long-running analytical queries):
- Build a visibility bitmap per adjacency page
- Cache bitmap keyed by `(page_id, snapshot_id)`
- Avoids per-entry visibility check on repeated page access
- Cache invalidated when page is modified

---

## 4. Garbage Collection Design

### Overview

GC reclaims space from:
1. Undo log entries for committed transactions with no active readers
2. Soft-deleted HNSW nodes
3. Expired adjacency list entries
4. Deleted posting list entries (via deletion bitmap merge)

### Low Water Mark

```
low_water_mark = min(txn_id for all active transactions)

Any version with txn_id_expire < low_water_mark can be safely removed:
  no active transaction can see it.
```

### Per-Engine GC

#### Relational / Undo Log

```
GC Process:
  1. Find undo entries where txn_id < low_water_mark
  2. Undo entries only needed by transactions that are still visible
  3. Remove undo entries that no active snapshot references
  4. Reclaim undo log pages
```

#### Vector / HNSW

```
HNSW GC (Compaction):
  1. Identify soft-deleted nodes where txn_id_expire < low_water_mark
  2. For each deleted node:
     a. Remove from neighbor lists of connected nodes
     b. Repair graph: connect neighbors to each other if needed
     c. Reclaim node page space
  3. This is expensive: batch and run during low-traffic periods
```

#### Graph / Adjacency Lists

```
Adjacency GC:
  1. Scan adjacency pages for entries where txn_id_expire < low_water_mark
  2. Remove expired entries
  3. Compact adjacency pages (move entries to fill gaps)
  4. Reclaim empty adjacency pages
```

#### Full-Text / Posting Lists

```
Posting List GC:
  1. Merge deletion bitmap into posting list
  2. Remove entries marked as deleted
  3. Re-compress posting list (delta encoding)
  4. Reclaim freed pages
```

### I/O Throttling

```
gc_io_limit_mbps: 50           # Max MB/s of I/O for GC operations
gc_cpu_limit_percent: 10       # Max CPU% for GC threads
gc_schedule: "02:00-06:00"     # Preferred GC window (optional)
gc_min_interval_sec: 60        # Minimum interval between GC runs
```

GC yields to active queries. If query latency increases during GC, GC backs off automatically.

### GC Trigger Conditions

```
Trigger when ANY of:
  - Undo log size > undo_log_max_size (default 1GB)
  - Dead tuples ratio > 20% of total tuples
  - HNSW soft-deleted nodes > 10% of total nodes
  - Adjacency dead entries > 20% of total entries
  - Posting list deletion bitmap > 30% of posting entries
```

---

## 5. Transaction Timeout

### Default: 5 minutes

```
transaction_timeout_sec: 300
```

### Why 5 Minutes?

- Long transactions block GC (low_water_mark stays old)
- Accumulate undo log entries
- Hold write locks preventing other writers
- Most OLTP transactions complete in < 1 second
- 5 minutes is generous for batch operations while preventing runaway transactions

### Behavior on Timeout

```
1. Transaction is marked as ABORTED
2. All locks released immediately
3. Undo is applied lazily (undo records remain, cleaned up by GC)
4. Client receives error: VAIS-000301 "Transaction timeout after 300 seconds"
5. Client must BEGIN new transaction
```

### Configurable per Session

```sql
SET SESSION transaction_timeout = 600;  -- 10 minutes for this session
SET SESSION transaction_timeout = 0;    -- No timeout (dangerous)
```

---

## 6. Isolation Levels

### Supported Levels

| Level | Description | Default |
|-------|-------------|---------|
| READ COMMITTED | See committed data at statement start | No |
| SNAPSHOT ISOLATION (REPEATABLE READ) | See committed data at transaction start | **Yes** |

### Not Supported Initially

- READ UNCOMMITTED: Not useful, easy to add later
- SERIALIZABLE: Requires SSI (serializable snapshot isolation), significant complexity. Planned for future.

### Snapshot Isolation Implementation

```
BEGIN:
  snapshot = {
    txn_id: next_txn_id(),
    active_txns: copy of Active Transaction Table,
    current_cmd_id: 0
  }

Each statement:
  snapshot.current_cmd_id += 1

COMMIT:
  Check for write-write conflicts (first-committer-wins)
  If conflict: ABORT and return error
  If no conflict: Write TXN_COMMIT to WAL, release locks

ABORT:
  Write TXN_ABORT to WAL, release locks
```

### Write-Write Conflict Detection

```
On UPDATE/DELETE:
  1. Check if the target tuple's txn_id_expire has been set
     by another committed transaction since our snapshot
  2. If yes: write-write conflict → abort this transaction
  3. If set by an active (uncommitted) transaction: wait for it to commit/abort
     (with deadlock detection timeout)
```

---

## 7. MVCC Concurrency Scenarios (Verification)

### Scenario 1: Basic Snapshot Isolation
```
T1: BEGIN
T1: SELECT * FROM t WHERE id = 1  → sees row version V1
T2: BEGIN
T2: UPDATE t SET x = 2 WHERE id = 1
T2: COMMIT
T1: SELECT * FROM t WHERE id = 1  → still sees V1 (snapshot)
T1: COMMIT
```
**Expected**: T1 sees consistent snapshot throughout. **Verified in visibility function**.

### Scenario 2: Write-Write Conflict
```
T1: BEGIN
T2: BEGIN
T1: UPDATE t SET x = 1 WHERE id = 1
T2: UPDATE t SET x = 2 WHERE id = 1  → blocks (T1 holds write)
T1: COMMIT
T2: detects conflict → ABORT with error
```
**Expected**: First-committer wins. T2 must retry. **Verified**.

### Scenario 3: INSERT...SELECT Self-Reference
```
T1: BEGIN
T1: INSERT INTO t SELECT * FROM t
```
**Expected**: SELECT sees table state BEFORE the INSERT started (cmd_id check). **Verified via cmd_id in visibility function**.

### Scenario 4: Phantom Reads (Allowed in SI)
```
T1: BEGIN
T1: SELECT COUNT(*) FROM t WHERE x > 5  → 10 rows
T2: INSERT INTO t (x) VALUES (100)
T2: COMMIT
T1: SELECT COUNT(*) FROM t WHERE x > 5  → still 10 rows (snapshot)
```
**Expected**: No phantom in snapshot isolation. **Verified**.

### Scenario 5: Vector Search Visibility
```
T1: BEGIN
T2: INSERT INTO docs (embedding) VALUES ([...])  -- uncommitted
T1: VECTOR_SEARCH(embedding, query, top_k=5)
```
**Expected**: T2's vector is in HNSW but filtered by MVCC post-filter. T1 does not see it. **Verified via post-filter strategy**.

### Scenario 6: Graph Traversal Consistency
```
T1: BEGIN, start 3-hop traversal
T2: DELETE edge (A→B), COMMIT (during T1's traversal)
T1: continues traversal
```
**Expected**: T1 still sees edge A→B (snapshot). Entire traversal is consistent. **Verified via adjacency entry visibility**.

### Scenario 7: Concurrent HNSW Insert
```
T1: INSERT vector V1 into HNSW
T2: INSERT vector V2 into HNSW (concurrent)
Both COMMIT
```
**Expected**: Both vectors in HNSW. Graph structure may differ from serial execution (HNSW is non-deterministic) but both are valid. **Verified: single-writer serializes HNSW writes**.

### Scenario 8: Deadlock
```
T1: UPDATE row A, then UPDATE row B
T2: UPDATE row B, then UPDATE row A
```
**Expected**: Deadlock detected. One transaction aborted. **Handled by wait-for graph with 1-second timeout**.

### Scenario 9: Long Transaction GC Blocking
```
T1: BEGIN (holds snapshot at txn_id 100)
... many other transactions commit ...
GC runs: cannot clean anything after txn_id 100
```
**Expected**: Low water mark stuck at 100. Transaction timeout (5 min) eventually aborts T1, allowing GC. **Verified**.

### Scenario 10: Cross-Engine Transaction
```
T1: BEGIN
T1: INSERT INTO docs (content, embedding) VALUES (...)  -- SQL + Vector
T1: Auto graph edges created                             -- Graph
T1: Full-text index updated                              -- Full-text
T1: COMMIT
```
**Expected**: All-or-nothing across all 4 engines. If crash before COMMIT, all undone. **Verified via unified WAL and ARIES recovery**.

---

## Verification Checklist

| Item | Status | Notes |
|------|--------|-------|
| In-place update + undo strategy | Done | Vector bloat prevented |
| Vector MVCC post-filter | Done | Adaptive oversampling |
| Graph adjacency MVCC | Done | Per-entry visibility, bitmap cache |
| GC design (all engines) | Done | Low water mark, per-engine strategy |
| GC I/O throttling | Done | Configurable limits |
| Transaction timeout | Done | 5 min default, configurable |
| Isolation levels | Done | Snapshot isolation default |
| Write-write conflict | Done | First-committer-wins |
| 10 concurrency scenarios | Done | All verified against visibility function |
