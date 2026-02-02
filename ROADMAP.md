# VaisDB - RAG-Native Hybrid Database
## Project Roadmap

> **Version**: 0.1.0 (Design Phase)
> **Goal**: Vector + Graph + Relational + Full-Text search in a single DB, optimized for RAG
> **Language**: Pure Vais (with C FFI for system calls)
> **Last Updated**: 2026-02-02

---

## Overview

VaisDB solves the fundamental problem of RAG systems: **4 databases for 1 use case**.

### Core Innovation
- Single query across vector similarity + graph traversal + SQL joins + full-text search
- ACID transactions spanning all engine types
- RAG-native features (semantic chunking, context preservation) at the DB level

### Prerequisites
- Vais standard library enhancements (tracked in [vais ROADMAP Phase 31](https://github.com/vaislang/vais))
  - `fsync`/`mmap`/`flock` for storage durability
  - Allocator state mutation fixes
  - String-keyed HashMap
  - Binary serialization
  - Directory operations

---

## Progress Summary

| Phase | Name | Status | Progress |
|-------|------|--------|----------|
| 1 | Storage Engine | ⏳ Planned | 0/20 (0%) |
| 2 | SQL Engine | ⏳ Planned | 0/20 (0%) |
| 3 | Vector Engine | ⏳ Planned | 0/16 (0%) |
| 4 | Graph Engine | ⏳ Planned | 0/16 (0%) |
| 5 | Full-Text Engine | ⏳ Planned | 0/12 (0%) |
| 6 | Hybrid Query Planner | ⏳ Planned | 0/16 (0%) |
| 7 | RAG-Native Features | ⏳ Planned | 0/12 (0%) |
| 8 | Server & Client | ⏳ Planned | 0/12 (0%) |

---

## Phase 1: Storage Engine

> **Status**: ⏳ Planned
> **Dependency**: Vais Phase 31 (fsync, mmap, flock)
> **Goal**: SQLite-style single-file storage with WAL-based ACID

### Stage 1 - Page Manager

- [ ] **Page format definition** - 4KB/8KB/16KB configurable page size, page header (id, type, checksum, LSN)
- [ ] **Page types** - Data page, index page, overflow page, free list page
- [ ] **Page read/write** - Disk I/O via mmap or buffered file I/O
- [ ] **Free page management** - Free list for page reuse after deletion
- [ ] **Checksum validation** - CRC32 on every page read/write

### Stage 2 - Write-Ahead Log (WAL)

- [ ] **WAL format** - Log sequence number (LSN), transaction ID, redo/undo records
- [ ] **WAL writer** - Append-only sequential writes with fsync
- [ ] **Checkpoint** - Periodic WAL-to-data-file flush
- [ ] **Crash recovery** - Redo committed, undo uncommitted on startup
- [ ] **WAL truncation** - Reclaim space after checkpoint

### Stage 3 - Buffer Pool

- [ ] **LRU page cache** - Configurable cache size (default: 256 pages)
- [ ] **Dirty page tracking** - Write-back on eviction or checkpoint
- [ ] **Pin/unpin mechanism** - Prevent eviction of actively used pages
- [ ] **Read-ahead** - Prefetch sequential pages for scan operations
- [ ] **Buffer pool statistics** - Hit rate, eviction count for tuning

### Stage 4 - B+Tree Index

- [ ] **B+Tree insert/search/delete** - Standard operations with page-based nodes
- [ ] **Leaf page chaining** - Linked list for range scans
- [ ] **Node splitting/merging** - Maintain balance on insert/delete
- [ ] **Bulk loading** - Sorted input → bottom-up B+Tree construction
- [ ] **Concurrent access** - Latch crabbing for thread-safe traversal

### Verification

| Stage | Criteria |
|-------|----------|
| 1 | Read/write 10K pages, checksum validation 100% |
| 2 | Crash recovery: kill during write → data intact after restart |
| 3 | Buffer pool hit rate > 90% on repeated queries |
| 4 | B+Tree 100K insert/search, range scan correctness |

---

## Phase 2: SQL Engine

> **Status**: ⏳ Planned
> **Dependency**: Phase 1 (Storage Engine)
> **Goal**: Core SQL support (MariaDB 80% coverage of commonly used features)

### Stage 1 - SQL Parser

- [ ] **Tokenizer** - SQL keywords, identifiers, literals, operators
- [ ] **DDL parsing** - CREATE TABLE, DROP TABLE, ALTER TABLE, CREATE INDEX
- [ ] **DML parsing** - SELECT, INSERT, UPDATE, DELETE
- [ ] **Expression parsing** - Arithmetic, comparison, logical, function calls
- [ ] **JOIN parsing** - INNER, LEFT, RIGHT, CROSS JOIN

### Stage 2 - Schema & Catalog

- [ ] **Table metadata** - Column names, types (INT, VARCHAR, TEXT, FLOAT, BOOL, DATE, BLOB), constraints
- [ ] **Index metadata** - Index name, columns, type (B+Tree, HNSW, etc.)
- [ ] **System tables** - `vais_tables`, `vais_columns`, `vais_indexes`
- [ ] **Schema persistence** - Store catalog in reserved pages
- [ ] **Type system** - Type checking, implicit/explicit casting

### Stage 3 - Query Executor

- [ ] **Table scan** - Full scan with predicate pushdown
- [ ] **Index scan** - B+Tree index lookup and range scan
- [ ] **Nested loop join** - Basic join implementation
- [ ] **Hash join** - For equi-joins on large tables
- [ ] **Sort** - External merge sort for ORDER BY
- [ ] **Aggregation** - GROUP BY with COUNT, SUM, AVG, MAX, MIN
- [ ] **LIMIT/OFFSET** - Result set pagination

### Stage 4 - Transaction Manager

- [ ] **BEGIN/COMMIT/ROLLBACK** - Transaction lifecycle
- [ ] **MVCC** - Multi-version concurrency control with read snapshots
- [ ] **Isolation levels** - READ COMMITTED, REPEATABLE READ
- [ ] **Deadlock detection** - Wait-for graph or timeout-based
- [ ] **Auto-commit mode** - Single statement transactions

### Verification

| Stage | Criteria |
|-------|----------|
| 1 | Parse all SQL in test suite (100+ queries) |
| 2 | CREATE/DROP/ALTER round-trip through catalog |
| 3 | TPC-B simplified benchmark pass |
| 4 | Concurrent read/write with no data corruption |

---

## Phase 3: Vector Engine

> **Status**: ⏳ Planned
> **Dependency**: Phase 1 (Storage Engine)
> **Goal**: Pinecone-level vector search with adaptive quantization

### Stage 1 - HNSW Index

- [ ] **HNSW graph construction** - Multi-layer navigable small world graph
- [ ] **Approximate nearest neighbor search** - Top-K query with configurable ef_search
- [ ] **Distance functions** - Cosine, Euclidean (L2), Dot product
- [ ] **Incremental insert/delete** - No full index rebuild needed

### Stage 2 - Quantization

- [ ] **Scalar quantization (int8)** - 4x memory reduction, < 1% recall loss
- [ ] **Product quantization** - Up to 64x compression for large-scale
- [ ] **Adaptive quantization** - Auto-select based on data distribution
- [ ] **Oversampling** - Configurable oversample factor for compressed search

### Stage 3 - Vector Storage

- [ ] **Dimension-aware page layout** - Optimize page utilization for vector data
- [ ] **VECTOR column type** - `VECTOR(1536)` in CREATE TABLE
- [ ] **Batch insert optimization** - Bulk vector loading path
- [ ] **Vector metadata** - Dimension, distance metric per index

### Stage 4 - Integration

- [ ] **VECTOR_SEARCH() function** - SQL-callable vector similarity search
- [ ] **Hybrid pre/post filtering** - Filter before or after ANN search
- [ ] **Score normalization** - Normalize similarity scores across distance metrics
- [ ] **Index statistics** - Recall estimation, memory usage reporting

### Verification

| Stage | Criteria |
|-------|----------|
| 1 | 1M vectors, recall@10 > 0.95, < 10ms query |
| 2 | Quantized recall within 2% of full precision |
| 3 | 10M vectors stored, < 50% memory vs naive |
| 4 | VECTOR_SEARCH + WHERE filter end-to-end |

---

## Phase 4: Graph Engine

> **Status**: ⏳ Planned
> **Dependency**: Phase 1 (Storage Engine)
> **Goal**: Neo4j-level property graph with multi-hop traversal

### Stage 1 - Property Graph Model

- [ ] **Node storage** - Node ID, labels, properties (key-value pairs)
- [ ] **Edge storage** - Edge ID, source, target, type, properties
- [ ] **Adjacency list** - Per-node edge list for fast traversal
- [ ] **Label index** - Fast lookup by node/edge label

### Stage 2 - Graph Traversal

- [ ] **BFS/DFS** - Breadth-first and depth-first traversal
- [ ] **Multi-hop query** - `GRAPH_TRAVERSE(node, depth=N)` with configurable depth
- [ ] **Path finding** - Shortest path between two nodes
- [ ] **Edge filtering** - Traverse only specific edge types
- [ ] **Cycle detection** - Prevent infinite loops in traversal

### Stage 3 - Graph Query Syntax

- [ ] **GRAPH_TRAVERSE() function** - SQL-callable graph traversal
- [ ] **Pattern matching** - `(a)-[r:REFERENCES]->(b)` style patterns
- [ ] **Path expressions** - Variable-length path queries
- [ ] **Graph aggregation** - Node degree, centrality metrics

### Stage 4 - Integration

- [ ] **Graph + SQL joins** - Join graph results with relational tables
- [ ] **Graph + Vector** - Traverse neighbors of vector search results
- [ ] **Graph indexes on B+Tree** - Index node/edge properties
- [ ] **Graph statistics** - Node/edge count, degree distribution

### Verification

| Stage | Criteria |
|-------|----------|
| 1 | 1M nodes, 10M edges stored and retrievable |
| 2 | 3-hop traversal on 1M-node graph < 50ms |
| 3 | Pattern matching query syntax end-to-end |
| 4 | Vector + Graph combined query correctness |

---

## Phase 5: Full-Text Engine

> **Status**: ⏳ Planned
> **Dependency**: Phase 1 (Storage Engine)
> **Goal**: Elasticsearch-level full-text search

### Stage 1 - Inverted Index

- [ ] **Tokenizer** - Whitespace, Unicode-aware word splitting
- [ ] **Inverted index** - Term → document list mapping with positions
- [ ] **Index compression** - Variable-byte encoding for posting lists
- [ ] **Incremental update** - Insert/delete without full rebuild

### Stage 2 - Search & Ranking

- [ ] **BM25 scoring** - Standard TF-IDF variant for relevance ranking
- [ ] **Phrase search** - Position-aware multi-word matching
- [ ] **Boolean queries** - AND, OR, NOT operators
- [ ] **FULLTEXT_MATCH() function** - SQL-callable full-text search

### Stage 3 - Integration

- [ ] **Full-text + Vector** - Hybrid keyword + semantic search
- [ ] **Full-text + SQL** - Filter/sort text results with SQL predicates
- [ ] **Score fusion** - Configurable weight blending across search types
- [ ] **Text column indexing** - `CREATE FULLTEXT INDEX ON table(column)`

### Verification

| Stage | Criteria |
|-------|----------|
| 1 | 100K documents indexed, term lookup < 1ms |
| 2 | BM25 ranking matches reference implementation |
| 3 | Hybrid vector+keyword search end-to-end |

---

## Phase 6: Hybrid Query Planner

> **Status**: ⏳ Planned
> **Dependency**: Phase 2, 3, 4, 5
> **Goal**: Unified optimizer across all engine types

### Stage 1 - Query Planning

- [ ] **Unified AST** - Parse VECTOR_SEARCH, GRAPH_TRAVERSE, FULLTEXT_MATCH as first-class functions
- [ ] **Cost model** - Estimate cost for each engine's operations
- [ ] **Plan enumeration** - Generate candidate plans combining engines
- [ ] **Plan selection** - Choose lowest-cost plan

### Stage 2 - Cross-Engine Execution

- [ ] **Pipeline execution** - Stream results between engines without materializing
- [ ] **Predicate pushdown** - Push SQL filters into vector/graph/text engines
- [ ] **Result merging** - Merge and rank results from multiple engines
- [ ] **Score fusion operators** - Weighted sum, reciprocal rank fusion

### Stage 3 - Optimization

- [ ] **Statistics collection** - Table row count, index cardinality, data distribution
- [ ] **Index selection** - Auto-choose best index per predicate
- [ ] **Join ordering** - Optimal join order for multi-table queries
- [ ] **Caching** - Query plan cache for repeated patterns

### Stage 4 - Verification

- [ ] **Explain plan** - `EXPLAIN` command showing execution plan
- [ ] **Plan correctness** - Same results regardless of plan choice
- [ ] **Benchmark** - Hybrid query < 2x slowest single-engine query
- [ ] **Regression tests** - Plan stability across optimizer changes

### Verification

| Stage | Criteria |
|-------|----------|
| 1 | All hybrid query syntax parsed and planned |
| 2 | Vector+Graph+SQL combined query returns correct results |
| 3 | Optimizer chooses index scan over table scan when appropriate |
| 4 | Explain plan matches actual execution |

---

## Phase 7: RAG-Native Features

> **Status**: ⏳ Planned
> **Dependency**: Phase 6 (Hybrid Query Planner)
> **Goal**: RAG pipeline built into the database

### Stage 1 - Semantic Chunking

- [ ] **Document ingestion** - `INSERT INTO docs (content) VALUES (...)` with auto-chunking
- [ ] **Chunking strategies** - Fixed-size, sentence-boundary, semantic-boundary
- [ ] **Chunk metadata** - Parent document ID, position, overlap region
- [ ] **Chunk-to-chunk graph edges** - Automatic `NEXT_CHUNK`, `SAME_SECTION` relationships

### Stage 2 - Context Preservation

- [ ] **Hierarchical document structure** - Document → Section → Paragraph → Chunk
- [ ] **Context window builder** - Given a chunk, retrieve surrounding context via graph
- [ ] **Cross-reference tracking** - Auto-detect and link references between chunks
- [ ] **Temporal versioning** - Track document versions, serve latest by default

### Stage 3 - RAG Query API

- [ ] **RAG_SEARCH() function** - Single call: embed query → vector search → graph expand → rank → return context
- [ ] **Fact verification** - Cross-check vector results against relational data
- [ ] **Source attribution** - Return source document/chunk IDs with every result
- [ ] **Configurable pipeline** - Adjust retrieval depth, reranking weights, context window size

### Verification

| Stage | Criteria |
|-------|----------|
| 1 | Document insert → auto-chunked + embedded + graph-linked |
| 2 | Context window includes relevant surrounding chunks |
| 3 | RAG_SEARCH returns attributed, fact-checked results |

---

## Phase 8: Server & Client

> **Status**: ⏳ Planned
> **Dependency**: Phase 6 (Hybrid Query Planner)
> **Goal**: Client/server mode + embedded mode

### Stage 1 - Wire Protocol

- [ ] **TCP server** - Accept client connections on configurable port
- [ ] **Protocol format** - Length-prefixed binary messages (query, result, error)
- [ ] **Connection pooling** - Server-side connection management
- [ ] **Authentication** - Username/password, API key

### Stage 2 - Client Library

- [ ] **Vais client** - Native Vais client library
- [ ] **Python client** - pip-installable Python client
- [ ] **REST API** - HTTP/JSON interface for any language
- [ ] **Connection string** - `vaisdb://host:port/dbname`

### Stage 3 - Embedded Mode

- [ ] **Library mode** - Link VaisDB directly into application (like SQLite)
- [ ] **Single-file database** - One `.vaisdb` file per database
- [ ] **Zero-config** - Works out of the box without server setup

### Verification

| Stage | Criteria |
|-------|----------|
| 1 | 100 concurrent connections, no data corruption |
| 2 | Python client: connect → query → results end-to-end |
| 3 | Embedded mode: open → query → close with single file |

---

## Milestone Summary

| Milestone | Phases | Target |
|-----------|--------|--------|
| **M1: Storage MVP** | Phase 1 | Page manager + WAL + B+Tree working |
| **M2: SQL MVP** | Phase 1-2 | Basic SQL (CREATE, INSERT, SELECT, JOIN) |
| **M3: Vector MVP** | Phase 1, 3 | HNSW search + SQL integration |
| **M4: Hybrid MVP** | Phase 1-6 | All 4 engines + unified query |
| **M5: RAG MVP** | Phase 1-7 | Semantic chunking + RAG_SEARCH |
| **M6: Production** | Phase 1-8 | Server + clients + embedded mode |

---

**Maintainer**: Steve
