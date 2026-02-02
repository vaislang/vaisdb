# Stage 6: Configuration System Design

> **Status**: Design Complete
> **Impact**: Configuration hierarchy and scoping affect operational flexibility
> **Last Updated**: 2026-02-02

---

## 1. Configuration Hierarchy

Settings are resolved in priority order (highest first):

```
Priority  Source              Example
────────  ──────              ───────
1 (high)  SQL SET command     SET SESSION query_timeout = 30;
2         CLI arguments       vaisdb --port 5433 --memory-budget 8GB
3         Environment vars    VAISDB_PORT=5433
4         Config file         vaisdb.conf (in database directory)
5 (low)   Built-in defaults   Hardcoded in source
```

### Config File Format

```toml
# vaisdb.conf - VaisDB Configuration
# Located at: mydb.vaisdb/vaisdb.conf

[server]
port = 5432
max_connections = 100
bind_address = "0.0.0.0"

[storage]
page_size = 8192          # IMMUTABLE after creation
wal_segment_size = 67108864  # 64MB, IMMUTABLE after creation

[memory]
memory_budget = "8GB"
buffer_pool_percent = 50
hnsw_cache_percent = 25
dict_cache_percent = 5
query_memory_percent = 15

[wal]
sync_mode = "fsync"       # fsync | fdatasync | async
group_commit_timeout_us = 1000
checkpoint_interval_sec = 300

[vector]
hnsw_m = 16               # IMMUTABLE per index
hnsw_m_max_0 = 32         # IMMUTABLE per index
hnsw_ef_construction = 200 # IMMUTABLE per index
hnsw_ef_search = 64       # Runtime-changeable

[query]
query_memory_limit = "256MB"
query_timeout_sec = 30
slow_query_threshold_ms = 1000

[transaction]
default_isolation = "snapshot"  # snapshot | read_committed
transaction_timeout_sec = 300

[gc]
gc_io_limit_mbps = 50
gc_cpu_limit_percent = 10
gc_min_interval_sec = 60

[logging]
log_level = "info"        # debug | info | warn | error
slow_query_log = true
log_file = "vaisdb.log"
```

### Environment Variable Mapping

Config keys map to env vars with `VAISDB_` prefix and `_` separators:

```
server.port           → VAISDB_SERVER_PORT
memory.memory_budget  → VAISDB_MEMORY_BUDGET
wal.sync_mode         → VAISDB_WAL_SYNC_MODE
vector.hnsw_ef_search → VAISDB_VECTOR_HNSW_EF_SEARCH
```

---

## 2. Session vs Global Scope

### Scope Definitions

| Scope | Description | Lifetime | Visibility |
|-------|-------------|----------|------------|
| **GLOBAL** | Server-wide default | Until restart or `SET GLOBAL` | All new sessions |
| **SESSION** | Per-connection override | Until connection closes | This connection only |

### SQL Commands

```sql
-- Set for current session only
SET SESSION query_timeout = 60;
SET query_timeout = 60;          -- SESSION is default

-- Set server-wide default (affects new sessions)
SET GLOBAL query_timeout = 60;

-- View current effective value
SHOW query_timeout;

-- View all settings with scope
SHOW ALL SETTINGS;

-- Reset to default
RESET query_timeout;             -- Reset session to global
RESET ALL;                       -- Reset all session overrides
```

### Scope Resolution

```
Effective value = SESSION override ?? GLOBAL override ?? config file ?? default
```

---

## 3. Setting Classification: Immutable / Restart / Runtime

### Immutable Settings (Cannot change after creation)

These are stored in the database meta page and are part of the on-disk format.

```
Setting                   Default     Notes
───────                   ───────     ─────
page_size                 8192        Set at CREATE DATABASE
wal_segment_size          64MB        Set at CREATE DATABASE
hnsw_m (per index)        16          Set at CREATE INDEX
hnsw_m_max_0 (per index)  32          Set at CREATE INDEX
hnsw_ef_construction      200         Set at CREATE INDEX
```

Attempting to change these returns `VAIS-0007002 IMMUTABLE_CONFIG`.

### Restart-Required Settings

These require server restart to take effect (stored in config file).

```
Setting                   Default     Notes
───────                   ───────     ─────
port                      5432        Bind port
bind_address              0.0.0.0     Listen address
max_connections           100         Connection limit (affects memory allocation)
wal_sync_mode             fsync       WAL durability mode
wal_archive_mode          off         WAL archiving for PITR
log_file                  vaisdb.log  Log file path
tls_cert_file             (none)      TLS certificate path
tls_key_file              (none)      TLS private key path
```

### Runtime-Changeable Settings (via SET GLOBAL or SET SESSION)

```
Setting                         Default     Scope    Notes
───────                         ───────     ─────    ─────
memory_budget                   auto        GLOBAL   Dynamic resize
buffer_pool_percent             50          GLOBAL   Triggers rebalance
hnsw_cache_percent              25          GLOBAL   Triggers rebalance
dict_cache_percent              5           GLOBAL   Triggers rebalance
query_memory_percent            15          GLOBAL   Triggers rebalance
hnsw_ef_search                  64          SESSION  Higher = better recall, slower
query_memory_limit              256MB       SESSION  Per-query memory cap
query_timeout_sec               30          SESSION  Query execution timeout
transaction_timeout_sec         300         SESSION  Transaction timeout
slow_query_threshold_ms         1000        GLOBAL   Slow query log threshold
default_isolation               snapshot    SESSION  Isolation level
gc_io_limit_mbps                50          GLOBAL   GC I/O throttle
gc_cpu_limit_percent            10          GLOBAL   GC CPU throttle
gc_min_interval_sec             60          GLOBAL   GC frequency
log_level                       info        GLOBAL   Log verbosity
slow_query_log                  true        GLOBAL   Enable/disable slow query log
group_commit_timeout_us         1000        GLOBAL   Group commit window
checkpoint_interval_sec         300         GLOBAL   Checkpoint frequency
```

---

## 4. Configuration Matrix Summary

```
┌──────────────────────────────┬───────────┬──────────┬─────────┬─────────┐
│ Setting                      │ Immutable │ Restart  │ Runtime │ Session │
│                              │           │ Required │ Global  │ Override│
├──────────────────────────────┼───────────┼──────────┼─────────┼─────────┤
│ page_size                    │     ✓     │          │         │         │
│ wal_segment_size             │     ✓     │          │         │         │
│ hnsw_m / hnsw_m_max_0       │     ✓     │          │         │         │
│ hnsw_ef_construction         │     ✓     │          │         │         │
├──────────────────────────────┼───────────┼──────────┼─────────┼─────────┤
│ port                         │           │    ✓     │         │         │
│ bind_address                 │           │    ✓     │         │         │
│ max_connections              │           │    ✓     │         │         │
│ wal_sync_mode                │           │    ✓     │         │         │
│ wal_archive_mode             │           │    ✓     │         │         │
│ log_file                     │           │    ✓     │         │         │
│ tls_cert_file / tls_key_file │           │    ✓     │         │         │
├──────────────────────────────┼───────────┼──────────┼─────────┼─────────┤
│ memory_budget                │           │          │    ✓    │         │
│ *_percent (memory split)     │           │          │    ✓    │         │
│ slow_query_threshold_ms      │           │          │    ✓    │         │
│ gc_* settings                │           │          │    ✓    │         │
│ log_level                    │           │          │    ✓    │         │
│ slow_query_log               │           │          │    ✓    │         │
│ group_commit_timeout_us      │           │          │    ✓    │         │
│ checkpoint_interval_sec      │           │          │    ✓    │         │
├──────────────────────────────┼───────────┼──────────┼─────────┼─────────┤
│ hnsw_ef_search               │           │          │    ✓    │    ✓    │
│ query_memory_limit           │           │          │    ✓    │    ✓    │
│ query_timeout_sec            │           │          │    ✓    │    ✓    │
│ transaction_timeout_sec      │           │          │    ✓    │    ✓    │
│ default_isolation            │           │          │    ✓    │    ✓    │
└──────────────────────────────┴───────────┴──────────┴─────────┴─────────┘
```

---

## 5. SHOW Commands

```sql
-- Show one setting
SHOW page_size;
-- page_size | 8192 | immutable | Set at database creation

-- Show all settings
SHOW ALL SETTINGS;
-- Returns: name, value, scope, changeable, description

-- Show only runtime-changeable settings
SHOW RUNTIME SETTINGS;

-- Show session overrides
SHOW SESSION OVERRIDES;

-- Show effective config with source
SHOW CONFIG SOURCE;
-- query_timeout | 60 | session (SET SESSION)
-- port          | 5432 | config_file (vaisdb.conf)
-- page_size     | 8192 | database_meta (immutable)
```

---

## 6. Validation Rules

### On Startup

```
1. Read config file
2. Apply environment variable overrides
3. Apply CLI argument overrides
4. Validate ALL settings:
   - Type check (integer, string, boolean, size)
   - Range check (port 1-65535, percentages 0-100, etc.)
   - Cross-validation (memory percentages sum ≤ 95%)
   - Immutable check (compare with database meta page)
5. If validation fails: abort startup with clear error message
```

### On SET Command

```
1. Parse value
2. Type check
3. Range check
4. Scope check (is this setting session-changeable?)
5. If GLOBAL: persist to config state, apply to defaults
6. If SESSION: store in session state
7. Trigger side effects (e.g., memory rebalance)
```

### Cross-Validation Rules

```
buffer_pool_percent + hnsw_cache_percent + dict_cache_percent + query_memory_percent ≤ 95
  (remaining ≥ 5% for system overhead)

max_connections * 4.2MB < system_overhead_allocation
  (enough memory for all connections)

query_memory_limit ≤ query_memory_percent * memory_budget
  (single query can't exceed pool)
```

---

## Verification Checklist

| Item | Status | Notes |
|------|--------|-------|
| Configuration hierarchy | Done | CLI > env > file > defaults |
| Session vs global scope | Done | SET SESSION / SET GLOBAL |
| Immutable settings | Done | page_size, HNSW params |
| Restart-required settings | Done | port, bind_address, etc. |
| Runtime-changeable settings | Done | Memory, timeouts, etc. |
| Config matrix | Done | Every param classified |
| SHOW commands | Done | All, source, overrides |
| Validation rules | Done | Type, range, cross-validation |
| Config file format | Done | TOML with sections |
| Env var mapping | Done | VAISDB_ prefix convention |
