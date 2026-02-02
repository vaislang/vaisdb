# Stage 4: Memory Architecture

> **Status**: Design Complete
> **Impact**: Determines performance characteristics and workload adaptability
> **Last Updated**: 2026-02-02

---

## 1. Memory Budget Allocation System

### Default Allocation

Total memory budget is configurable at startup (`memory_budget`). Default: auto-detect (75% of system RAM, max 16GB).

```
Component                    Default %   Min %   Description
─────────                    ─────────   ─────   ───────────
Buffer Pool                  50%         30%     Relational pages, B+Tree, graph adj, posting lists
HNSW Cache                   25%         10%     Vector index nodes (Layer 1+ pinned)
Full-text Dictionary Cache   5%          2%      Term dictionary for fast lookup
Query Execution Memory       15%         10%     Sort buffers, hash join, intermediates
System Overhead              5%          5%      Connections, WAL buffer, metadata, GC
```

### Budget Spreadsheets

#### 4GB Total Memory

```
Component                    Allocation   Notes
─────────                    ──────────   ─────
Buffer Pool                  2.0 GB       ~250K 8KB pages cached
HNSW Cache                   1.0 GB       ~163K 1536-dim vectors (6KB each) cached
Full-text Dictionary Cache   200 MB       ~2M unique terms
Query Execution Memory       600 MB       2-3 concurrent complex queries
System Overhead              200 MB       ~100 connections
```

**Suitable for**: Small deployments, < 10M rows, < 1M vectors.

#### 8GB Total Memory

```
Component                    Allocation   Notes
─────────                    ──────────   ─────
Buffer Pool                  4.0 GB       ~500K 8KB pages cached
HNSW Cache                   2.0 GB       ~327K 1536-dim vectors cached
Full-text Dictionary Cache   400 MB       ~4M unique terms
Query Execution Memory       1.2 GB       5-10 concurrent complex queries
System Overhead              400 MB       ~200 connections
```

**Suitable for**: Medium deployments, < 100M rows, < 10M vectors.

#### 16GB Total Memory

```
Component                    Allocation   Notes
─────────                    ──────────   ─────
Buffer Pool                  8.0 GB       ~1M 8KB pages cached
HNSW Cache                   4.0 GB       ~654K 1536-dim vectors cached
Full-text Dictionary Cache   800 MB       ~8M unique terms
Query Execution Memory       2.4 GB       10-20 concurrent complex queries
System Overhead              800 MB       ~500 connections
```

**Suitable for**: Large deployments, < 1B rows, < 100M vectors.

### Configuration

```sql
SET GLOBAL memory_budget = '8GB';
SET GLOBAL buffer_pool_percent = 50;
SET GLOBAL hnsw_cache_percent = 25;
SET GLOBAL dict_cache_percent = 5;
SET GLOBAL query_memory_percent = 15;
-- System overhead is always the remainder
```

---

## 2. Adaptive Memory Rebalancing

### Problem

Static allocation wastes memory. A SQL-heavy workload doesn't need 25% for HNSW cache.

### Solution: Workload-Based Rebalancing

```
Every 60 seconds, the memory manager evaluates:

  sql_pressure   = buffer_pool.miss_rate * buffer_pool.eviction_rate
  vector_pressure = hnsw_cache.miss_rate * vector_query_count
  text_pressure   = dict_cache.miss_rate * fulltext_query_count

  If sql_pressure > threshold AND vector_pressure < threshold:
    shift 5% from HNSW Cache → Buffer Pool

  If vector_pressure > threshold AND sql_pressure < threshold:
    shift 5% from Buffer Pool → HNSW Cache

  etc.
```

### Constraints

- No component below its minimum %
- Shift increments: 5% of total budget per rebalancing cycle
- Maximum 2 shifts per cycle (avoid oscillation)
- Rebalancing pauses during checkpoint (too much I/O)

### Monitoring

```sql
SHOW MEMORY STATUS;

-- Returns:
-- component          | allocated_mb | used_mb | hit_rate | miss_rate | pressure
-- buffer_pool        | 4096         | 3800    | 0.94     | 0.06      | low
-- hnsw_cache         | 2048         | 2040    | 0.88     | 0.12      | medium
-- dict_cache         | 400          | 120     | 0.99     | 0.01      | low
-- query_memory       | 1200         | 450     | N/A      | N/A       | low
-- system             | 400          | 280     | N/A      | N/A       | low
```

---

## 3. Memory Pressure Handling

### Eviction Priority (lowest to highest)

When memory is scarce, components evict in this order:

```
Priority  Component                 Reason
────────  ─────────                 ──────
1 (first) Data pages (clean)        Easy to re-read from disk
2         B+Tree leaf pages         Can be re-read, less frequent than data
3         Posting list pages        Full-text lookups less latency-sensitive
4         Graph adjacency pages     Graph traversal can tolerate latency
5         HNSW Layer 0 nodes        Large but re-loadable from disk
6         B+Tree internal pages     Small, high fan-out, hot
7         Dictionary cache          Small, very frequently accessed
8 (never) HNSW Layer 1+ nodes      Upper layers MUST stay in memory
          (and entry point)         for consistent search performance
```

### Why HNSW Layer 1+ is Pinned

HNSW search starts from the top layer and navigates down. If upper layers are evicted:
- Every vector query starts with disk I/O
- Latency becomes unpredictable (cache miss at the start of every search)
- Upper layers are small (exponentially fewer nodes per layer) → low memory cost

```
Layer distribution for M=16, 1M vectors:
  Layer 0: ~1,000,000 nodes (all vectors)
  Layer 1: ~62,500 nodes (1/16)
  Layer 2: ~3,906 nodes
  Layer 3: ~244 nodes
  Layer 4: ~15 nodes
  Layer 5: ~1 node (entry point)

Layers 1-5 total: ~66,666 nodes × 6KB ≈ 400MB for 1M 1536-dim vectors
This is acceptable and pinned.
```

### Out-of-Memory Handling

```
If all eviction fails and memory is still insufficient:
  1. Reject new queries with VAIS-000301 "Insufficient memory"
  2. Allow running queries to complete
  3. Trigger aggressive GC
  4. Log warning for operator attention

NEVER crash or corrupt data due to memory pressure.
```

---

## 4. Per-Query Memory Limit

### Default: 256MB per query

```
query_memory_limit: 256MB    # Maximum memory for a single query execution
```

### Why 256MB?

- Prevents a single runaway query from consuming all execution memory
- Sort of 10M rows × 100 bytes = 1GB → exceeds limit → spill to disk
- Hash join build side of 5M rows × 200 bytes = 1GB → exceeds limit → grace hash join

### Spill-to-Disk Strategy

When a query operator exceeds its memory allocation:

```
Sort:
  1. Sort current in-memory chunk
  2. Write sorted run to temp file
  3. Continue sorting next chunk
  4. Merge sorted runs at the end (external merge sort)

Hash Join:
  1. Build side exceeds memory
  2. Partition both sides by hash into temp files
  3. Process each partition that fits in memory
  4. (Grace hash join algorithm)

Aggregation:
  1. Hash table exceeds memory
  2. Spill partition to disk
  3. Re-process spilled partitions one at a time
```

### Temp File Location

```
Temp files stored in: mydb.vaisdb/tmp/
Cleaned up after query completes (or on startup for crash recovery)
```

### Configuration

```sql
SET SESSION query_memory_limit = '512MB';  -- For heavy analytical queries
SET SESSION query_memory_limit = '64MB';   -- For OLTP connections
```

---

## 5. Connection Memory

### Per-Connection Overhead

```
Component              Size        Notes
─────────              ────        ─────
Connection state       ~2 KB       Socket, auth, session vars
Parse cache            ~64 KB      Cached prepared statement plans
Result buffer          ~128 KB     Streaming result set buffer
Transaction state      ~4 KB       Snapshot, active locks
Sort buffer (default)  ~4 MB       Initial sort buffer per query
Total per connection   ~4.2 MB     Before query execution
```

### Max Connections

```
max_connections = (system_overhead_memory - 100MB) / 4.2MB

4GB budget:  (200MB - 100MB) / 4.2MB ≈ 23 connections
8GB budget:  (400MB - 100MB) / 4.2MB ≈ 71 connections
16GB budget: (800MB - 100MB) / 4.2MB ≈ 166 connections
```

These are conservative estimates. Actual per-connection memory varies with query complexity.

---

## 6. WAL Buffer

### Size

```
wal_buffer_size: 16MB (default)
```

- In-memory buffer for WAL records before flush
- Sized to hold ~1ms of WAL writes at peak throughput
- Flushed on group commit or when buffer is full

---

## Verification Checklist

| Item | Status | Notes |
|------|--------|-------|
| Memory allocation system | Done | Default percentages with min bounds |
| 4GB budget spreadsheet | Done | No component starved |
| 8GB budget spreadsheet | Done | No component starved |
| 16GB budget spreadsheet | Done | No component starved |
| Adaptive rebalancing | Done | Workload-based with oscillation protection |
| Memory pressure handling | Done | Eviction priority defined, HNSW L1+ pinned |
| Per-query memory limit | Done | 256MB default, spill-to-disk |
| Connection memory model | Done | ~4.2MB per connection |
| OOM handling | Done | Reject queries, never crash |
