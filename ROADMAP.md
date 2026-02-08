# VaisDB - AI-Native Hybrid Database
## Project Roadmap

> **Version**: 0.1.0 (Design Phase)
> **Goal**: Vector + Graph + Relational + Full-Text search in a single DB, optimized for RAG
> **Language**: Pure Vais (with C FFI for system calls)
> **Last Updated**: 2026-02-08

---

## Overview

VaisDB solves the fundamental problem of RAG and AI agent systems: **4 databases for 1 use case**.

### Core Innovation
- Single query across vector similarity + graph traversal + SQL joins + full-text search
- ACID transactions spanning all engine types
- RAG-native features (semantic chunking, context preservation) at the DB level
- AI-native agent memory (episodic, semantic, procedural memory with hybrid retrieval)

### Prerequisites
- Vais standard library enhancements (tracked in [vais ROADMAP Phase 31](https://github.com/vaislang/vais))
  - `fsync`/`mmap`/`flock` for storage durability
  - Allocator state mutation fixes
  - String-keyed HashMap
  - Binary serialization
  - Directory operations

### Critical Design Principles (Throughout All Phases)
- **format_version in every on-disk structure** - enables online migration without dump/restore
- **engine_type tag in WAL records** - unified crash recovery across all 4 engines
- **MVCC visibility integrated from day 1** - not bolted on later
- **SIMD distance calculation** - 10x vector search performance difference
- **NULL 3-valued logic** - SQL correctness from the start

---

## Progress Summary

| Phase | Name | Status | Progress |
|-------|------|--------|----------|
| 0 | Architecture & Design Decisions | ✅ Complete | 43/43 (100%) |
| 1 | Storage Engine | ⏳ Planned | 0/38 (0%) |
| 2 | SQL Engine | ⏳ Planned | 0/42 (0%) |
| 3 | Vector Engine | ⏳ Planned | 0/24 (0%) |
| 4 | Graph Engine | ⏳ Planned | 0/22 (0%) |
| 5 | Full-Text Engine | ⏳ Planned | 0/16 (0%) |
| 6 | Hybrid Query Planner | ⏳ Planned | 0/20 (0%) |
| 7 | RAG & AI-Native Features | ⏳ Planned | 0/24 (0%) |
| 8 | Server & Client | ⏳ Planned | 0/20 (0%) |
| 9 | Production Operations | ⏳ Planned | 0/24 (0%) |
| 10 | Security & Multi-tenancy | ⏳ Planned | 0/16 (0%) |

---

## Phase 0: Architecture & Design Decisions

> **Status**: ✅ Complete
> **Dependency**: None
> **Goal**: Lock in all decisions that are impossible/painful to change after implementation begins
> **Model**: Opus (fixed - architecture decisions)
> **Design Documents**: `docs/architecture/stage1-7*.md`

These decisions affect ALL subsequent phases. Getting them wrong means rewriting from scratch.

### Stage 7 - Design Review Fixes (2026-02-08)
모드: 자동진행
- [x] 1. Stage 1 수정: Torn write 보호(FPI), expire_cmd_id(32B), 헤더 정렬, undo_ptr, IS_DIRTY 제거, 파일 레이아웃 (Opus 직접) ✅
  변경: docs/architecture/stage1-on-disk-format.md (FPI 섹션 추가, 32B MVCC 메타데이터, 자연 정렬 헤더, tmp/ 디렉토리)
- [x] 2. Stage 2 수정: CLR 등록(0x06), HNSW WAL page ref, checksum zeroing, PAGE_ALLOC/DEALLOC, 직렬화 포맷, 헤더 48B (Opus 직접) ✅
  변경: docs/architecture/stage2-wal-design.md (CLR 0x06, PAGE_ALLOC 0x07/DEALLOC 0x08, 48B 헤더, 직렬화 포맷 섹션)
- [x] 3. Stage 3 수정: CLOG+abort 가시성, eager undo, cmd_id(38B), fulltext MVCC, invisible_ratio, edge conflict (Opus 직접) ✅
  변경: docs/architecture/stage3-mvcc-strategy.md (CLOG 테이블, eager undo+CLR, 38B AdjEntry, fulltext posting MVCC)
- [x] 4. Stage 4 수정: HNSW/BP I/O 분리, idle/active 커넥션, undo 메모리, 에러코드 7자리, pressure 정규화 (Sonnet 위임) ✅
  변경: docs/architecture/stage4-memory-architecture.md (Layer 0 BP경유, idle 200KB/active 4.2MB, 에러코드 VAIS-0003001)
- [x] 5. Stage 5 수정: 카운트(67개), EE 구분, NOT_IMPLEMENTED/DATA_TOO_LONG/SERIALIZATION_FAILURE, severity, line/column (Sonnet 위임) ✅
  변경: docs/architecture/stage5-error-codes.md (67개 카탈로그, severity 필드, 4개 에러코드 추가, VaisError line+column)
- [x] 6. Stage 6 수정: ALTER SYSTEM, 포트 5433, max_connections auto, 5개 설정 추가, RELOAD CONFIG (Sonnet 위임) ✅
  변경: docs/architecture/stage6-configuration.md (ALTER SYSTEM→meta.vdb, 5433포트, 5개 설정, RELOAD/SIGHUP)
- [x] 7. ROADMAP 동기화: 파일 레이아웃, WAL 헤더, MVCC 32B, 에러코드, 설정 hierarchy (Opus 직접) ✅
  변경: ROADMAP.md (헤더 정렬, wal/ 디렉토리, 32B MVCC, 48B WAL, engine 06-07, category 06-10, SQL SET hierarchy)
진행률: 7/7 (100%)

### Stage 8 - Design Review Round 3 (2026-02-08)
모드: 자동진행
- [x] 1. Stage 3 AdjEntry expire_cmd_id 추가 + is_edge_visible cmd_id 로직 + CLOG 16KB/torn write (Opus 직접) ✅
  변경: docs/architecture/stage3-mvcc-strategy.md (AdjEntry 42B, is_edge_visible 3-case 구현, CLOG 16KB/crash safety)
- [x] 2. Stage 2 FPI WAL 레코드 등록(0x09) + Graph WAL page ref 추가 (Opus 직접) ✅
  변경: docs/architecture/stage2-wal-design.md (FPI 0x09, Graph WAL에 file_id/page_id 추가)
- [x] 3. Stage 5 에러코드 카운트 수정: SQL=17, Storage=8, Total=69 (Sonnet 위임) ✅
  변경: docs/architecture/stage5-error-codes.md (SQL 16→17, Storage 7→8, Total 67→69)
- [x] 4. Stage 6 scope 수정 + 누락 설정 4개 + cross-validation 공식 갱신 (Sonnet 위임) ✅
  변경: docs/architecture/stage6-configuration.md (max_concurrent_queries/deadlock GLOBAL, 4설정 추가, cross-validation)
- [x] 5. Stage 1+4 cross-doc 불일치 수정: WAL seg 32B, eviction priority, 벡터 테이블 주석 (Sonnet 위임) ✅
  변경: docs/architecture/stage1-on-disk-format.md (WAL seg 32B), stage4 (eviction priority, checklist label)
- [x] 6. Stage 7 직렬화 패턴 보강: MVCC tuple, AdjEntry, PostingEntry, CLOG, WAL seg header (Opus 직접) ✅
  변경: docs/architecture/stage7-implementation-mapping.md (5개 직렬화 패턴 + ClogPage helper)
- [x] 7. ROADMAP 동기화: Phase 0 카운트, 카테고리 코드, 메모리 범위 (Opus 직접) ✅
  변경: ROADMAP.md (Phase 0 43/43, 카테고리 09=type/10=notfound, 메모리 범위 정렬)
진행률: 7/7 (100%)

### Stage 1 - On-Disk Format Decisions (IMMUTABLE after first user)

- [x] **Unified page header format (48 bytes, naturally aligned)** - All 4 engines share this header:
  ```
  page_lsn: u64 (offset 0), txn_id: u64 (offset 8),
  page_id: u32 (offset 16), checksum: u32 (offset 20),
  prev_page: u32 (offset 24), next_page: u32 (offset 28),
  overflow_page: u32 (offset 32),
  free_space_offset: u16, item_count: u16, flags: u16, reserved: u16,
  page_type: u8, engine_tag: u8, compression_algo: u8, format_version: u8
  ```
- [x] **Page type registry** - Define all page types upfront: DATA, BTREE_INTERNAL, BTREE_LEAF, HNSW_NODE, HNSW_LAYER, GRAPH_ADJ, GRAPH_NODE, INVERTED_POSTING, INVERTED_DICT, FREELIST, OVERFLOW, META, WAL_SEGMENT, CATALOG
- [x] **Default page size decision** - 8KB default, 16KB for vector-heavy workloads. Document that this is **IMMUTABLE** after database creation
- [x] **File layout strategy** - Bundle directory (`.vaisdb/`): `data.vdb`, `wal/` (segment files), `vectors.vdb`, `graph.vdb`, `fulltext.vdb`, `meta.vdb`, `undo/undo.vdb`, `tmp/`. Appears as single unit, internal separation for I/O parallelism
- [x] **MVCC tuple metadata format (32 bytes)** - Every row carries:
  ```
  txn_id_create: u64, txn_id_expire: u64,
  undo_ptr: u64, cmd_id: u32, expire_cmd_id: u32
  ```
  `cmd_id`/`expire_cmd_id`: enables correct visibility for self-referencing operations within a transaction

### Stage 2 - WAL Design (Affects crash recovery of ALL engines)

- [x] **Unified WAL record header (48 bytes, naturally aligned)** - `lsn: u64, txn_id: u64, prev_lsn: u64, timestamp: u64, record_length: u32, checksum: u32, record_type: u8, engine_type: u8, reserved: [u8; 6]`
- [x] **Relational WAL records** - PAGE_WRITE, TUPLE_INSERT/DELETE/UPDATE, BTREE_SPLIT/MERGE, BTREE_INSERT/DELETE
- [x] **Vector WAL records** - HNSW_INSERT_NODE (all layer edges), HNSW_DELETE_NODE, HNSW_UPDATE_EDGES, HNSW_LAYER_PROMOTE, VECTOR_DATA_WRITE, QUANTIZATION_UPDATE
- [x] **Graph WAL records** - GRAPH_NODE_INSERT/DELETE, GRAPH_EDGE_INSERT/DELETE (both adjacency lists!), GRAPH_PROPERTY_UPDATE, ADJ_LIST_PAGE_SPLIT
- [x] **Full-text WAL records** - POSTING_LIST_APPEND/DELETE, DICTIONARY_INSERT/DELETE, TERM_FREQ_UPDATE
- [x] **Meta WAL records** - TXN_BEGIN/COMMIT/ABORT, CHECKPOINT_BEGIN/END, SCHEMA_CHANGE, CLR, PAGE_ALLOC/DEALLOC, FPI
- [x] **Physiological logging strategy** - Page-level physical + intra-page logical. Vector data uses logical logging (operation + params) since HNSW insertion is non-deterministic

### Stage 3 - MVCC Strategy Decisions

- [x] **In-place update + undo log** (InnoDB style) - Chosen over append-only because vector data is large (6KB/vector at 1536dim), bloat would be severe
- [x] **MVCC for vector indexes: post-filter strategy** - HNSW searches top_k * oversample_factor, then filters by MVCC visibility. Oversample ratio adapts based on uncommitted_ratio
- [x] **MVCC for graph adjacency lists** - Each AdjEntry carries `txn_id_create`/`txn_id_expire`. Traversal checks visibility per edge. Bitmap cache for hot snapshots
- [x] **Garbage collection design** - Low water mark from Active Transaction Table. Per-engine GC: undo log cleanup, HNSW soft-delete compaction, adjacency list compaction, posting list compaction. GC I/O throttling to avoid query impact
- [x] **Transaction timeout default** - 5 minutes. Long-running txns block GC and accumulate undo

### Stage 4 - Memory Architecture

- [x] **Memory budget allocation system** - Configurable split:
  - Buffer Pool: 30-60% (relational pages, B+Tree, graph adj, posting lists)
  - HNSW Cache: 10-30% (Layer 1+ pinned, Layer 0 via buffer pool)
  - Full-text Dictionary Cache: 2-10%
  - Query Execution Memory: 10-15% (sort buffers, hash join, intermediates)
  - System Overhead: 5% (connections, WAL buffer, metadata)
- [x] **Adaptive memory rebalancing** - Shift budget based on workload (no vector queries → shrink HNSW cache)
- [x] **Memory pressure handling** - Eviction priority: data pages < B+Tree internal < HNSW Layer 0 < HNSW Layer 1+ (never evict)
- [x] **Per-query memory limit** - Default 256MB. Spill to disk for large sorts/joins

### Stage 5 - Error Code System (IMMUTABLE after first client)

- [x] **Error code format** - `VAIS-EECCNNN` (EE=engine, CC=category, NNN=number)
- [x] **Engine codes** - 00=common, 01=SQL, 02=vector, 03=graph, 04=fulltext, 05=storage, 06=server, 07=client
- [x] **Category codes** - 01=syntax, 02=constraint, 03=resource, 04=concurrency, 05=internal, 06=auth, 07=config, 08=io, 09=type, 10=notfound
- [x] **Initial error catalog** - Define all error codes for Phase 1-2 before implementation

### Stage 6 - Configuration System Design

- [x] **Configuration hierarchy** - SQL SET > ALTER SYSTEM > CLI args > env vars > config file > defaults
- [x] **Session vs global scope** - `SET SESSION` vs `SET GLOBAL`
- [x] **Immutable settings documentation** - page_size, hnsw_m, hnsw_m_max_0 cannot change after creation
- [x] **Runtime-changeable settings** - buffer_pool_size, hnsw_ef_search, query_timeout, slow_query_threshold

### Verification

| Stage | Criteria |
|-------|----------|
| 1 | Page header format documented and reviewed, format_version included |
| 2 | All WAL record types enumerated, cross-engine crash recovery scenario walkthrough |
| 3 | MVCC visibility function pseudocode verified against 10 concurrency scenarios |
| 4 | Memory allocation spreadsheet: 4GB/8GB/16GB budgets, no component starved |
| 5 | Error code catalog: no conflicts, extensible for future engines |
| 6 | Config matrix: every parameter classified as immutable/restart/runtime |

---

## Phase 1: Storage Engine

> **Status**: ⏳ Planned
> **Dependency**: Phase 0 (Architecture Decisions) + Vais Phase 31 (fsync, mmap, flock)
> **Goal**: Unified storage layer shared by all 4 engines, WAL-based ACID

### Stage 1 - Page Manager

- [ ] **Unified page header implementation** - 48-byte header per Phase 0 spec
- [ ] **Page type dispatching** - Read page → check type → route to correct engine's deserializer
- [ ] **Page read/write via mmap** - Memory-mapped I/O with madvise hints
- [ ] **Buffered I/O fallback** - For systems without mmap support
- [ ] **Free page management** - Free list with page reuse, anti-fragmentation
- [ ] **CRC32C checksum** - Hardware-accelerated on every page read/write
- [ ] **Overflow page management** - For vectors > page_size and long text values
- [ ] **format_version migration hook** - Read old format → convert to new format on access (lazy migration)

### Stage 2 - Write-Ahead Log (WAL) - Unified

- [ ] **WAL segment files** - Configurable segment size (default 64MB), sequential writes
- [ ] **Unified WAL writer** - Accepts records from all 4 engines via engine_type tag
- [ ] **Group commit** - Batch multiple transactions' WAL records before fsync (critical: fsync = 100-200 TPS without batching)
- [ ] **WAL record serialization** - Binary format with varint encoding for compactness
- [ ] **Checkpoint** - Force dirty pages to disk, advance recovery point
- [ ] **Crash recovery** - Redo committed (all engines), undo uncommitted (all engines), using prev_lsn chain for undo
- [ ] **WAL truncation** - Reclaim segments after successful checkpoint
- [ ] **WAL archiving hook** - For point-in-time recovery (PITR), don't delete segments, move to archive

### Stage 3 - Buffer Pool

- [ ] **Clock replacement algorithm** - Lock-free approximate LRU (avoids global lock of true LRU)
- [ ] **Configurable cache size** - Dynamic resize without restart
- [ ] **Dirty page tracking** - Write-back on eviction or checkpoint
- [ ] **Pin/unpin mechanism** - Prevent eviction of actively used pages
- [ ] **Partitioned buffer pool** - Multiple hash partitions to reduce latch contention
- [ ] **Read-ahead** - Prefetch sequential pages for scan operations
- [ ] **Buffer pool statistics** - Hit rate, eviction count, dirty ratio for monitoring

### Stage 4 - B+Tree Index

- [ ] **B+Tree insert/search/delete** - Page-based nodes with unified page header
- [ ] **Leaf page chaining** - Doubly-linked list for bidirectional range scans
- [ ] **Node splitting/merging** - WAL record BEFORE new page allocation (prevent orphan pages on crash)
- [ ] **Prefix compression** - Reduce key storage in internal nodes
- [ ] **Bulk loading** - Sorted input → bottom-up construction (10-100x faster than serial insert)
- [ ] **Optimistic latch crabbing** - Read latches down, write-upgrade only at leaf (avoids root bottleneck)

### Stage 5 - Transaction Manager (Core)

- [ ] **Transaction ID allocator** - Monotonically increasing, crash-safe
- [ ] **Active Transaction Table (ATT)** - Track all active txns and their snapshot points
- [ ] **Undo log** - In-place update with undo chain via undo_ptr
- [ ] **Snapshot creation** - Copy ATT at BEGIN, use for visibility decisions
- [ ] **MVCC visibility function** - `is_visible(tuple, snapshot)` implementing 3VL (three-valued logic) for NULL txn_ids
- [ ] **Write-write conflict detection** - First-committer-wins for snapshot isolation
- [ ] **Deadlock detection** - Wait-for graph with 1-second timeout fallback
- [ ] **Transaction timeout** - Default 5 minutes, configurable

### Verification

| Stage | Criteria |
|-------|----------|
| 1 | Read/write 10K pages across all page types, checksum 100%, format_version round-trip |
| 2 | **Crash recovery test**: kill during WAL write → all committed data intact, all uncommitted rolled back. Test per engine_type |
| 3 | Buffer pool hit rate > 90% on repeated queries, no eviction of pinned pages |
| 4 | B+Tree 100K insert/search/delete, range scan correctness, concurrent read/write no corruption |
| 5 | 10 MVCC concurrency scenarios pass: read-committed, snapshot isolation, write-write conflict, self-referencing INSERT...SELECT |

---

## Phase 2: SQL Engine

> **Status**: ⏳ Planned
> **Dependency**: Phase 1 (Storage Engine)
> **Goal**: Core SQL with MariaDB-level commonly used features + NULL 3-valued logic from day 1

### Stage 1 - SQL Parser

- [ ] **Tokenizer** - SQL keywords, identifiers, literals (string, integer, float, NULL), operators
- [ ] **DDL** - CREATE TABLE, DROP TABLE, ALTER TABLE (ADD/DROP/RENAME COLUMN), CREATE INDEX, DROP INDEX
- [ ] **DML** - SELECT, INSERT (single + multi-row + INSERT...SELECT), UPDATE, DELETE
- [ ] **Expressions** - Arithmetic, comparison, logical (AND/OR/NOT with NULL propagation), BETWEEN, IN, LIKE, IS NULL/IS NOT NULL
- [ ] **JOINs** - INNER, LEFT, RIGHT, CROSS JOIN with ON/USING clause
- [ ] **Subqueries** - Uncorrelated (IN, EXISTS, scalar), correlated subqueries
- [ ] **CASE WHEN** - Simple and searched CASE expressions
- [ ] **Set operations** - UNION, UNION ALL, INTERSECT, EXCEPT
- [ ] **CTE** - WITH clause (non-recursive initially), WITH RECURSIVE for graph queries later
- [ ] **Prepared statements** - Parse once, execute many with parameter binding ($1, $2, ...)

### Stage 2 - Data Types & NULL Semantics

- [ ] **Core types** - INT (i64), FLOAT (f64), BOOL, VARCHAR(n), TEXT, BLOB, DATE, TIMESTAMP, VECTOR(dim)
- [ ] **NULL 3-valued logic** - `NULL = NULL → NULL`, `NULL AND TRUE → NULL`, `NULL OR TRUE → TRUE`, `NOT NULL → NULL`
- [ ] **NULL in aggregates** - COUNT(*) includes NULLs, COUNT(col) excludes, SUM/AVG skip NULLs
- [ ] **NULL in GROUP BY** - NULL values form one group
- [ ] **NULL in ORDER BY** - NULLS FIRST/NULLS LAST option
- [ ] **NULL in VECTOR** - NULL vector columns excluded from HNSW index, never returned by VECTOR_SEARCH
- [ ] **Type casting** - Implicit (INT→FLOAT) and explicit (CAST(x AS type))
- [ ] **Type checking** - Validate types in expressions, function arguments, comparisons

### Stage 3 - Schema & Catalog

- [ ] **System tables** - `vais_tables`, `vais_columns`, `vais_indexes`, `vais_embedding_models`, `vais_config`
- [ ] **Schema persistence** - Reserved pages in data file, loaded into memory at startup
- [ ] **Embedding model registry** - model_id, model_name, model_version, dimensions, distance_metric, is_active
- [ ] **Index-to-model binding** - Each vector index linked to a specific embedding model version
- [ ] **Constraint support** - PRIMARY KEY, NOT NULL, UNIQUE, DEFAULT, CHECK (basic)

### Stage 4 - Query Executor

- [ ] **Table scan** - Full scan with predicate pushdown, MVCC visibility filter
- [ ] **Index scan** - B+Tree lookup and range scan
- [ ] **Nested loop join** - With inner-side index lookup optimization
- [ ] **Hash join** - For equi-joins, spill to disk when exceeding memory limit
- [ ] **Sort** - In-memory quicksort + external merge sort for large results
- [ ] **Aggregation** - Hash-based GROUP BY with COUNT, SUM, AVG, MAX, MIN
- [ ] **DISTINCT** - Hash-based deduplication
- [ ] **LIMIT/OFFSET** - Early termination optimization
- [ ] **Window functions** - ROW_NUMBER(), RANK(), DENSE_RANK(), SUM/AVG OVER (PARTITION BY ... ORDER BY ...)

### Stage 5 - Query Optimizer (Basic)

- [ ] **Cost model** - Estimate I/O and CPU cost per operator
- [ ] **Index selection** - Choose index scan vs table scan based on selectivity
- [ ] **Join ordering** - Dynamic programming for small join count, greedy for large
- [ ] **Predicate pushdown** - Push WHERE conditions as close to scan as possible
- [ ] **EXPLAIN command** - Show estimated execution plan
- [ ] **EXPLAIN ANALYZE** - Show actual execution statistics (time, rows, memory per operator)

### Stage 6 - ALTER TABLE & Schema Migration

- [ ] **ADD COLUMN (NULL default)** - Metadata-only change, no data rewrite
- [ ] **ADD COLUMN (non-NULL default)** - Lazy migration: old rows return default on read, new rows store value
- [ ] **DROP COLUMN** - Logical deletion (mark invisible), physical cleanup in background
- [ ] **RENAME COLUMN** - Metadata-only change
- [ ] **ALTER COLUMN TYPE** - Background data rewrite with shadow column, atomic swap
- [ ] **Schema version per row** - Enable lazy migration without full table rewrite

### Verification

| Stage | Criteria |
|-------|----------|
| 1 | Parse 200+ SQL test queries including subqueries, CTE, set operations |
| 2 | NULL logic test suite: 50+ cases matching PostgreSQL behavior |
| 3 | Catalog round-trip: CREATE → restart → schema intact with model registry |
| 4 | TPC-B simplified benchmark pass, window function correctness |
| 5 | EXPLAIN shows index usage, optimizer picks index for selective queries |
| 6 | ALTER TABLE + restart + old data readable with new schema |

---

## Phase 3: Vector Engine

> **Status**: ⏳ Planned
> **Dependency**: Phase 1 (Storage Engine)
> **Goal**: Pinecone-level vector search with SIMD optimization and MVCC integration

### Stage 1 - HNSW Index

- [ ] **Multi-layer graph construction** - Navigable small world with exponential layer probability
- [ ] **HNSW parameters** - M=16, M_max_0=32, ef_construction=200 (configurable, IMMUTABLE per index)
- [ ] **Top-K ANN search** - Configurable ef_search for precision/speed trade-off
- [ ] **Distance functions with SIMD** - Cosine, L2, Dot product. **MUST use NEON (ARM) / SSE4.2+AVX2 (x86)** for 10x speedup on 1536-dim vectors
- [ ] **Incremental insert** - Add to graph without full rebuild, update WAL per Phase 0 spec
- [ ] **Soft delete** - Mark deleted, skip during search, compact during GC
- [ ] **MVCC post-filter** - Search with oversample_factor, filter by visibility, return top_k. Adaptive oversample based on uncommitted_ratio

### Stage 2 - Concurrency

- [ ] **Single-writer + multiple-reader** (Phase 1) - Write serialization, lock-free reads
- [ ] **Copy-on-write neighbor lists** (Phase 2) - Immutable arrays, atomic pointer swap, epoch-based reclamation for old arrays
- [ ] **HNSW Layer 1+ pinning** - Upper layers always in memory for consistent search entry points

### Stage 3 - Quantization

- [ ] **Scalar quantization (int8)** - 4x memory reduction, < 1% recall loss
- [ ] **Product quantization (PQ)** - Up to 64x compression for billion-scale
- [ ] **Adaptive selection** - Auto-choose based on vector count and memory budget
- [ ] **Oversampling for compressed search** - Configurable oversample factor, re-rank with full-precision vectors

### Stage 4 - Vector Storage & Types

- [ ] **VECTOR(dim) column type** - SQL DDL integration
- [ ] **Dimension-aware page layout** - Vectors > page_size use overflow pages
- [ ] **Batch insert path** - Bulk loading: insert all vectors, build HNSW bottom-up
- [ ] **Embedding model metadata** - Link index to model in catalog (Phase 2 Stage 3)
- [ ] **VECTOR_SEARCH() SQL function** - `VECTOR_SEARCH(column, query_vector, top_k=10)`
- [ ] **Pre/post filtering** - Apply SQL WHERE before or after ANN based on selectivity estimate

### Verification

| Stage | Criteria |
|-------|----------|
| 1 | 1M vectors: recall@10 > 0.95, < 10ms query. SIMD vs scalar: > 5x speedup |
| 2 | Concurrent read/write: no crash, no inconsistent results, oversample handles uncommitted |
| 3 | Quantized recall within 2% of full precision, memory < 50% of naive |
| 4 | VECTOR_SEARCH + WHERE filter end-to-end, NULL vectors excluded |

---

## Phase 4: Graph Engine

> **Status**: ⏳ Planned
> **Dependency**: Phase 1 (Storage Engine)
> **Goal**: Neo4j-level property graph with MVCC-aware multi-hop traversal

### Stage 1 - Property Graph Model

- [ ] **Node storage** - Node ID, labels (multi-label), properties (typed key-value pairs)
- [ ] **Edge storage** - Edge ID, source, target, type, direction, properties
- [ ] **Adjacency list with MVCC** - Each AdjEntry (42B packed): `target_node, edge_id, edge_type, txn_id_create, txn_id_expire, cmd_id, expire_cmd_id`. Visibility check per edge during traversal
- [ ] **Label index** - B+Tree index on node/edge labels for fast label-based lookup
- [ ] **Bidirectional WAL** - Edge insert/delete writes WAL for BOTH source and target adjacency lists

### Stage 2 - Graph Traversal

- [ ] **BFS/DFS** - Breadth-first and depth-first with MVCC snapshot consistency
- [ ] **Multi-hop query** - `GRAPH_TRAVERSE(node, depth=N)` with configurable max depth
- [ ] **Path finding** - Shortest path (Dijkstra/BFS) between two nodes
- [ ] **Edge type filtering** - Traverse only specific edge types
- [ ] **Cycle detection** - Visited-set to prevent infinite loops
- [ ] **Snapshot-consistent traversal** - Entire multi-hop traversal sees consistent graph state via snapshot

### Stage 3 - Graph Query Syntax

- [ ] **GRAPH_TRAVERSE() SQL function** - Returns table of (node_id, depth, path, edge_type)
- [ ] **Pattern matching** - `(a)-[r:REFERENCES]->(b)` syntax in WHERE clause
- [ ] **Variable-length paths** - `(a)-[:KNOWS*1..3]->(b)` for 1-to-3 hop paths
- [ ] **Graph aggregation** - Node degree, in/out degree

### Stage 4 - Integration

- [ ] **Graph + SQL joins** - Join graph traversal results with relational tables
- [ ] **Graph + Vector** - Find vector neighbors, then traverse their graph connections
- [ ] **Graph property indexes** - B+Tree on node/edge properties (e.g., index on node.name)
- [ ] **Graph statistics** - Node/edge count, degree distribution for optimizer cost model

### Verification

| Stage | Criteria |
|-------|----------|
| 1 | 1M nodes, 10M edges CRUD, adjacency list MVCC correctness |
| 2 | 3-hop traversal on 1M-node graph < 50ms, snapshot consistency during concurrent writes |
| 3 | Pattern matching query end-to-end, variable-length path correctness |
| 4 | Vector → Graph combined query correct, graph stats available to optimizer |

---

## Phase 5: Full-Text Engine

> **Status**: ⏳ Planned
> **Dependency**: Phase 1 (Storage Engine)
> **Goal**: Elasticsearch-level full-text search with WAL-integrated updates

### Stage 1 - Inverted Index

- [ ] **Tokenizer pipeline** - Unicode-aware word splitting, lowercasing, stop word removal
- [ ] **Inverted index** - Term → posting list (doc_id, position, term_frequency)
- [ ] **Posting list compression** - Variable-byte encoding, delta encoding for doc IDs
- [ ] **Incremental update** - Append to posting list + deletion bitmap (no rebuild)
- [ ] **Posting list compaction** - Background merge of deletion bitmap (remove deleted entries)
- [ ] **Dictionary B+Tree** - Term → posting list head page, stored in unified page format

### Stage 2 - Search & Ranking

- [ ] **BM25 scoring** - With configurable k1 (default 1.2) and b (default 0.75)
- [ ] **Document frequency tracking** - Updated via WAL, approximately correct under concurrency (documented)
- [ ] **Phrase search** - Position-aware multi-word matching
- [ ] **Boolean queries** - AND, OR, NOT operators with query parsing
- [ ] **FULLTEXT_MATCH() SQL function** - Returns (doc_id, score) pairs

### Stage 3 - Integration

- [ ] **Full-text + Vector hybrid** - BM25 + cosine similarity score fusion
- [ ] **Full-text + SQL** - Filter/sort text results with SQL predicates
- [ ] **Score fusion operators** - Weighted sum, reciprocal rank fusion (RRF)
- [ ] **CREATE FULLTEXT INDEX** - DDL integration: `CREATE FULLTEXT INDEX ON table(column)`

### Verification

| Stage | Criteria |
|-------|----------|
| 1 | 100K documents indexed, term lookup < 1ms, deletion bitmap works |
| 2 | BM25 ranking matches reference implementation (pyserini) |
| 3 | Hybrid vector+keyword search end-to-end, RRF produces reasonable rankings |

---

## Phase 6: Hybrid Query Planner

> **Status**: ⏳ Planned
> **Dependency**: Phase 2, 3, 4, 5
> **Goal**: Unified cost-based optimizer across all engine types

### Stage 1 - Unified Query AST

- [ ] **Extended SQL parser** - VECTOR_SEARCH, GRAPH_TRAVERSE, FULLTEXT_MATCH as first-class table-valued functions
- [ ] **Unified plan nodes** - VectorScan, GraphTraverse, FullTextScan alongside SeqScan, IndexScan, Join, Sort, Agg
- [ ] **Cross-engine cost model** - Estimate cost per engine operation (HNSW = f(ef_search, dimension), Graph = f(avg_degree, depth), BM25 = f(posting_length))
- [ ] **Plan enumeration** - Generate candidate plans combining engines

### Stage 2 - Cross-Engine Execution

- [ ] **Pipeline execution** - Stream results between engines without full materialization
- [ ] **Predicate pushdown into engines** - Push SQL WHERE into vector/graph/fulltext pre-filters
- [ ] **Result merging** - Merge and rank results from multiple engines
- [ ] **Score fusion** - Weighted sum and reciprocal rank fusion (RRF) as plan operators
- [ ] **Per-engine profiling** - Track time/rows/memory per engine in EXPLAIN ANALYZE

### Stage 3 - Optimization

- [ ] **Statistics collection** - Row count, index cardinality, value distribution histograms, vector count per index
- [ ] **Index selection** - Auto-choose best index per predicate (B+Tree vs HNSW vs fulltext)
- [ ] **Join ordering** - Dynamic programming for small join count
- [ ] **Query plan cache** - Cache plans for repeated query patterns (with prepared statement integration)

### Stage 4 - Verification & Diagnostics

- [ ] **EXPLAIN** - Show estimated plan with cost per operator
- [ ] **EXPLAIN ANALYZE** - Show actual execution: time, rows, memory, engine breakdown (e.g., "80% graph, 15% vector, 5% SQL")
- [ ] **Plan correctness** - Same results regardless of plan choice
- [ ] **Regression tests** - Plan stability across optimizer changes

### Verification

| Stage | Criteria |
|-------|----------|
| 1 | All hybrid query syntax parsed and planned |
| 2 | Vector+Graph+SQL combined query returns correct results, pipelined |
| 3 | Optimizer picks index scan for selective queries, table scan for non-selective |
| 4 | EXPLAIN ANALYZE shows per-engine cost breakdown |

---

## Phase 7: RAG & AI-Native Features

> **Status**: ⏳ Planned
> **Dependency**: Phase 6 (Hybrid Query Planner)
> **Goal**: RAG pipeline and AI agent memory built into the database

### Stage 1 - Embedding Integration

- [ ] **Pre-computed vector support** - `INSERT INTO docs (content, embedding) VALUES (...)` - MVP path, no external dependency
- [ ] **External embedding API** - `SET EMBEDDING_MODEL = 'openai:text-embedding-3-small'`, auto-embed on INSERT
- [ ] **Local model support** - `SET EMBEDDING_MODEL = 'local:model_path'` (future)
- [ ] **Model versioning** - Track which model generated each vector. Prevent mixing incompatible embeddings in same index
- [ ] **Model change + reindex** - `ALTER EMBEDDING MODEL ... REINDEX STRATEGY = BACKGROUND`: shadow index build → atomic swap. During reindex, new inserts dual-embed
- [ ] **Reindex progress tracking** - `SHOW REINDEX STATUS` shows percentage, ETA, estimated cost

### Stage 2 - Semantic Chunking

- [ ] **Document ingestion** - `INSERT INTO docs (content) VALUES (...)` with auto-chunking
- [ ] **Chunking strategies** - Fixed-size, sentence-boundary, paragraph-boundary (configurable)
- [ ] **Chunk metadata** - Parent document ID, position, overlap region
- [ ] **Chunk-to-chunk graph edges** - Automatic `NEXT_CHUNK`, `SAME_SECTION`, `SAME_DOCUMENT` relationships
- [ ] **TTL (Time-To-Live)** - `CREATE TABLE docs (..., TTL = '90 days')` for automatic expiration of stale documents

### Stage 3 - Context Preservation

- [ ] **Hierarchical document structure** - Document → Section → Paragraph → Chunk (graph hierarchy)
- [ ] **Context window builder** - Given a chunk, retrieve surrounding context via graph edges
- [ ] **Cross-reference tracking** - Auto-detect and link references between chunks
- [ ] **Temporal versioning** - Document versions, serve latest by default, query historical

### Stage 4 - RAG Query API

- [ ] **RAG_SEARCH() function** - Single call: embed query → vector search → graph expand → rank → return with context
- [ ] **Fact verification** - Cross-check vector results against relational data via automatic JOIN
- [ ] **Source attribution** - Return source document/chunk IDs with every result
- [ ] **Configurable pipeline** - Adjust retrieval depth, reranking weights, context window size

### Stage 5 - Agent Memory Engine

- [ ] **Memory type schema** - Built-in schemas for episodic (events/experiences), semantic (facts/knowledge), procedural (how-to/patterns), and working (active task context) memory types
- [ ] **Hybrid memory retrieval** - `MEMORY_SEARCH(query, memory_types, max_tokens)` — single function that searches across vector (semantic similarity) + graph (relational context) + SQL (metadata filters) + full-text (keyword match) and fuses results
- [ ] **Memory lifecycle management** - TTL-based expiration, importance decay (exponential decay with access-based refresh), memory consolidation (merge similar memories), and GC for expired memories
- [ ] **Token budget management** - `max_tokens` parameter on retrieval: rank and truncate results to fit within LLM context window budget. Prioritize by recency, importance, and relevance score
- [ ] **Memory importance scoring** - Automatic importance assignment based on access frequency, recency, explicit user marking, and cross-reference count. Importance decays over time unless refreshed
- [ ] **Agent session continuity** - `CREATE AGENT SESSION` / `RESUME AGENT SESSION` for persistent agent state across interactions. Session graph links working memory to episodic memories created during the session

### Verification

| Stage | Criteria |
|-------|----------|
| 1 | External API embed on INSERT, model change + reindex without downtime, no mixed-model vectors |
| 2 | Document insert → auto-chunked + embedded + graph-linked, TTL expiry works |
| 3 | Context window includes relevant surrounding chunks via graph |
| 4 | RAG_SEARCH returns attributed, fact-checked results with configurable pipeline |
| 5 | MEMORY_SEARCH returns fused results within token budget, importance decay works, session continuity across reconnect |

---

## Phase 8: Server & Client

> **Status**: ⏳ Planned
> **Dependency**: Phase 6 (Hybrid Query Planner)
> **Goal**: Client/server mode + embedded mode + wire protocol

### Stage 1 - Wire Protocol

- [ ] **TCP server** - Accept connections on configurable port (default 5433)
- [ ] **Binary protocol** - Length-prefixed messages: Query, Parse, Bind, Execute, Result, Error
- [ ] **Prepared statement protocol** - Parse → Bind (with parameters) → Execute cycle
- [ ] **Connection pooling** - Server-side connection management with max_connections limit
- [ ] **Authentication** - Username/password, API key, token-based
- [ ] **TLS support** - Encrypted connections (optional)

### Stage 2 - Client Libraries

- [ ] **Vais client** - Native Vais client library with connection pooling
- [ ] **Python client** - pip-installable, DB-API 2.0 compatible
- [ ] **REST API** - HTTP/JSON interface: `POST /query`, `GET /health`
- [ ] **Connection string** - `vaisdb://user:pass@host:port/dbname`

### Stage 3 - Embedded Mode

- [ ] **Library mode** - Link VaisDB directly into application (like SQLite)
- [ ] **Single-directory database** - One `.vaisdb/` directory per database
- [ ] **Zero-config** - Works out of the box without server setup
- [ ] **File locking** - flock to prevent multiple process access in embedded mode

### Stage 4 - Data Import/Export

- [ ] **COPY FROM** - Bulk import from CSV, JSON, JSONL (WAL-bypass option for speed)
- [ ] **COPY TO** - Export to CSV, JSON, JSONL
- [ ] **Vector binary format** - Binary import/export for vectors (text = 3x size, 10x slower)
- [ ] **Graph import ordering** - Enforce nodes-first → edges-second during import
- [ ] **Import with index disable** - Drop indexes → bulk import → rebuild indexes (10-100x faster)

### Verification

| Stage | Criteria |
|-------|----------|
| 1 | 100 concurrent connections, prepared statements, no data corruption |
| 2 | Python client: connect → query → results end-to-end |
| 3 | Embedded mode: open → query → close with single directory, flock works |
| 4 | COPY 1M rows < 30 seconds, vector binary import < 60 seconds for 100K vectors |

---

## Phase 9: Production Operations

> **Status**: ⏳ Planned
> **Dependency**: Phase 8 (Server & Client)
> **Goal**: Production-ready operations: backup, monitoring, profiling

### Stage 1 - Backup & Restore

- [ ] **Physical backup (online)** - Checkpoint → copy data files + WAL segments while serving queries
- [ ] **Logical backup** - SQL dump: DDL + INSERT statements (including vector serialization)
- [ ] **Point-in-time recovery (PITR)** - WAL archiving + recovery to specific timestamp
- [ ] **HNSW index backup** - Include graph structure (not just vectors) to avoid rebuild on restore
- [ ] **Restore verification** - Checksum validation after restore

### Stage 2 - Monitoring & Metrics

- [ ] **Health endpoint** - `GET /health` → `{"status": "healthy", "engines": {...}}`
- [ ] **System metrics** - uptime, connections, memory, disk usage
- [ ] **Buffer pool metrics** - hit_rate, dirty_pages, evictions_per_sec
- [ ] **WAL metrics** - size, write_rate, last_checkpoint_age, fsync_duration_avg
- [ ] **Transaction metrics** - active_count, longest_running_seconds, deadlocks, commit/rollback rate
- [ ] **Per-engine metrics** - Vector: hnsw_size, search_latency. Graph: nodes, edges, traversal_depth. Fulltext: documents, dictionary_size
- [ ] **Kubernetes probes** - `/health` (liveness), `/ready` (readiness)

### Stage 3 - Query Profiling

- [ ] **Slow query log** - Queries exceeding threshold with full execution stats
- [ ] **Slow query details** - Duration, plan, rows_scanned, rows_returned, engines_used, buffer hits/misses, memory_used, lock_wait_time
- [ ] **Per-engine breakdown** - "80% graph traversal, 15% vector search, 5% SQL join"
- [ ] **Query log rotation** - Size-based or time-based log rotation

### Stage 4 - Maintenance

- [ ] **VACUUM** - Reclaim space from deleted rows, compact undo log
- [ ] **REINDEX** - Rebuild specific index (B+Tree, HNSW, fulltext)
- [ ] **ANALYZE** - Update table statistics for optimizer
- [ ] **Database file compaction** - Defragment data files, reclaim free pages

### Verification

| Stage | Criteria |
|-------|----------|
| 1 | Online backup during writes → restore → data identical, PITR to 1-second granularity |
| 2 | All metrics exposed, Kubernetes probes pass under load |
| 3 | Slow query log captures all queries above threshold with per-engine breakdown |
| 4 | VACUUM reclaims space, REINDEX produces identical search results |

---

## Phase 10: Security & Multi-tenancy

> **Status**: ⏳ Planned
> **Dependency**: Phase 8 (Server & Client)
> **Goal**: Enterprise-grade security for multi-tenant RAG deployments

### Stage 1 - Access Control

- [ ] **User management** - CREATE/ALTER/DROP USER, password hashing (argon2)
- [ ] **Role-based access** - CREATE ROLE, GRANT/REVOKE on table/column level
- [ ] **SQL injection prevention** - Prepared statements enforced in protocol, string escaping for legacy
- [ ] **Error message sanitization** - No internal schema/path info in client-facing errors

### Stage 2 - Row-Level Security (RLS)

- [ ] **Policy definition** - `CREATE POLICY tenant_isolation ON docs USING (tenant_id = current_user_tenant())`
- [ ] **RLS in vector search** - Post-filter VECTOR_SEARCH results by policy (shared HNSW index, per-tenant visibility)
- [ ] **RLS in graph traversal** - Skip edges/nodes not visible to current user's policy
- [ ] **Tenant-isolated indexes (optional)** - `CREATE INDEX ... WITH (per_tenant = true)` for strict isolation

### Stage 3 - Encryption & Audit

- [ ] **TLS for connections** - Certificate-based, optional client cert auth
- [ ] **Encryption at rest (page-level)** - AES-256-CTR per page, key from external KMS or config
- [ ] **WAL encryption** - WAL records also encrypted (otherwise plaintext leaks through WAL)
- [ ] **Audit log** - DDL, auth attempts, privilege changes, configurable DML logging
- [ ] **Audit log integrity** - Append-only with checksum chain (tamper detection)
- [ ] **Key rotation** - Background re-encryption with new key, no downtime

### Verification

| Stage | Criteria |
|-------|----------|
| 1 | Unauthorized user cannot read/write, injection attempts rejected |
| 2 | Tenant A cannot see Tenant B's data in SQL, vector search, or graph traversal |
| 3 | Encrypted DB file unreadable without key, audit log detects tampering |

---

## Testing Strategy (Applies to ALL phases)

> Not a separate phase - integrated into every phase's verification

### Test Types Required

| Type | Purpose | When |
|------|---------|------|
| **Unit tests** | Per-function correctness | Every commit |
| **Integration tests** | Cross-engine queries | Every phase completion |
| **Crash recovery tests** | Kill during write → data intact | Phase 1+, every engine |
| **Fuzz tests** | SQL parser, protocol, vector input (NaN, Inf) | Phase 2+, continuous |
| **ACID correctness tests** | Atomicity, Consistency, Isolation, Durability | Phase 1+, Jepsen-style |
| **SQL correctness tests** | Compare results vs SQLite/PostgreSQL | Phase 2+ |
| **Vector correctness tests** | HNSW recall vs brute-force | Phase 3+ |
| **Performance regression** | Benchmark per commit, alert on >10% regression | Phase 1+ |
| **Concurrency stress** | N clients concurrent read/write | Phase 1+ |

### Crash Recovery Test Method
```
1. Start workload (mixed read/write across all engines)
2. At random point: SIGKILL the process
3. Restart and verify:
   - All committed transactions present
   - All uncommitted transactions absent
   - All checksums valid
   - HNSW index consistent with vector data
   - Graph adjacency lists consistent (both directions)
   - Posting lists consistent with documents
4. Repeat 100+ times with different kill points
```

---

## Milestone Summary

| Milestone | Phases | Deliverable |
|-----------|--------|-------------|
| **M0: Architecture** | Phase 0 | All design decisions documented and reviewed |
| **M1: Storage MVP** | Phase 0-1 | Page manager + WAL + Buffer Pool + B+Tree + MVCC |
| **M2: SQL MVP** | Phase 1-2 | CREATE, INSERT, SELECT, JOIN, WHERE, NULL logic |
| **M3: Vector MVP** | Phase 1, 3 | HNSW search + SIMD + MVCC post-filter + SQL integration |
| **M4: Graph MVP** | Phase 1, 4 | Property graph + multi-hop + MVCC-aware traversal |
| **M5: Hybrid MVP** | Phase 1-6 | All 4 engines + unified query planner |
| **M6: RAG MVP** | Phase 1-7 | Semantic chunking + embedding integration + RAG_SEARCH |
| **M7: Server MVP** | Phase 1-8 | Client/server + embedded mode + import/export |
| **M8: Production** | Phase 1-10 | Backup, monitoring, security, multi-tenancy |

---

## Benchmark Targets

| Category | Benchmark | Target |
|----------|-----------|--------|
| SQL | TPC-B (transactions) | Within 2x of SQLite |
| SQL | TPC-H (analytics, simplified) | Functional correctness |
| Vector | ann-benchmarks (SIFT-1M) | recall@10 > 0.95 at 10K QPS |
| Vector | OpenAI-1536 dim | < 10ms p99 query latency |
| Graph | LDBC Social Network | 3-hop < 50ms on 1M nodes |
| Full-text | MS MARCO (BM25) | Accuracy matches pyserini |
| Hybrid | Vector+Graph+SQL | < 2x slowest single-engine query |
| Durability | Crash recovery | 100% data integrity after 100 random kills |
| Concurrency | 64 clients mixed workload | No deadlocks, no data corruption |

---

**Maintainer**: Steve
