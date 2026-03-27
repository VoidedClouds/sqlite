# Read Isolation for SQLite `BEGIN CONCURRENT`

## Overview

Read isolation adds a **read-committed** mode to SQLite's `BEGIN CONCURRENT` mechanism. In the default **snapshot** isolation, any page read by a concurrent transaction that was subsequently modified by another committed transaction causes `SQLITE_BUSY_SNAPSHOT` at commit time. Read-committed mode relaxes this: only **write-write** conflicts are detected, and stale reads are allowed.

This feature operates at **two levels** depending on which compile-time flags are enabled:

| Configuration | Conflict Granularity | Behaviour |
|---|---|---|
| `SQLITE_ENABLE_READ_ISOLATION` only | **Page-level** | Pages that were only read (not dirtied) by the current transaction are exempt from conflict detection. Dirty pages still conflict. |
| `SQLITE_ENABLE_READ_ISOLATION` + `SQLITE_ENABLE_ROW_LEVEL_LOCKING` | **Row-level** | `ROW_LOCK_READ` entries are skipped entirely during conflict detection. Only `ROW_LOCK_WRITE` vs `ROW_LOCK_WRITE` overlaps count. The P1 optimisation also skips recording read entries, saving memory and CPU. |

---

## Build Flag

### `SQLITE_ENABLE_READ_ISOLATION`

Enables the read-committed isolation mode:

- `PRAGMA isolation_level` pragma (get/set per connection)
- `SQLITE_ReadCommitted` connection flag on `db->flags`
- Page-level read-committed conflict bypass in `walScanConflicts()`
- WAL-level `bReadCommitted` flag propagation from pager to WAL

```bash
-DSQLITE_ENABLE_READ_ISOLATION
```

**Requires:** `BEGIN CONCURRENT` support (i.e., `SQLITE_OMIT_CONCURRENT` must **not** be defined).

**Independent of:** `SQLITE_ENABLE_ROW_LEVEL_LOCKING`. Read isolation works at the page level without RLL. When RLL is also enabled, it additionally operates at the row level.

---

## How It Works

### Page-Level Read-Committed (RI only, no RLL)

When `PRAGMA isolation_level = read_committed` is set and a page in `pAllRead` is found to have been modified by another transaction:

1. Look up the page in the pager cache
2. If the page is **NOT dirty** (not writeable by the current transaction): the page was only read → **skip the conflict**
3. If the page **IS dirty**: this is a write-write page conflict → **report conflict** as `SQLITE_BUSY_SNAPSHOT`
4. If the page is not in cache (evicted): conservatively treat as read-only → **skip the conflict**

This means page-level read-committed **only helps when reads and writes land on different physical pages**. If a transaction reads and writes the same leaf page (common with small tables), that page is dirty and the conflict will still be detected.

### Row-Level Read-Committed (RI + RLL)

When both flags are enabled:

1. **At row tracking time** (P1 optimisation): `ROW_LOCK_READ` entries are **not recorded** at all in `sqlite3BtreeIntegerKey()`. This saves memory and hash-table overhead since the entries would be ignored at conflict time anyway.

2. **At conflict detection time**: `rowLockSetHasConflict()` receives `bReadCommitted=1` and skips all `ROW_LOCK_READ` entries in the local transaction's `RowLockSet`. Only `ROW_LOCK_WRITE` vs `ROW_LOCK_WRITE` overlaps produce a conflict.

This provides finer-grained concurrency: two transactions can write different rows on the same page, and reads of the other transaction's rows are ignored.

---

## PRAGMA

```sql
-- Query current isolation level (default: 'snapshot')
PRAGMA isolation_level;

-- Set to read-committed mode
PRAGMA isolation_level = read_committed;

-- Revert to snapshot mode
PRAGMA isolation_level = snapshot;

-- Unknown values revert to snapshot
PRAGMA isolation_level = serializable;  -- → 'snapshot'
```

The pragma is per-connection and takes effect immediately. It applies to all subsequent `BEGIN CONCURRENT` transactions on that connection.

---

## Isolation Level Semantics

### Snapshot (default)

| Scenario | Result |
|----------|--------|
| T1 reads row A, T2 writes row A and commits, T1 commits | `SQLITE_BUSY_SNAPSHOT` |
| T1 writes row A, T2 writes row A and commits, T1 commits | `SQLITE_BUSY_SNAPSHOT` |
| T1 writes row A, T2 writes row B (same page) and commits, T1 commits | `SQLITE_BUSY_SNAPSHOT` (page-level) or OK (with RLL) |

### Read-Committed

| Scenario | Result |
|----------|--------|
| T1 reads row A, T2 writes row A and commits, T1 commits | **OK** (stale read allowed) |
| T1 writes row A, T2 writes row A and commits, T1 commits | `SQLITE_BUSY_SNAPSHOT` |
| T1 reads page P, T2 writes page P and commits, T1 commits | **OK** (if T1 didn't dirty P) |
| T1 writes page P, T2 writes page P and commits, T1 commits | `SQLITE_BUSY_SNAPSHOT` (page-level) or OK (with RLL, if rows disjoint) |

### Trade-offs

**Read-committed gains:**
- Higher commit success rate for read-heavy concurrent workloads
- Reduced false-positive conflicts when reads and writes are on different pages/rows

**Read-committed costs:**
- **Stale reads**: A transaction may read a value that was subsequently modified. The transaction's reads are NOT consistent with the committed state.
- **Phantoms**: New rows inserted by other transactions after the snapshot point are invisible, but modifications to existing rows that were read may be missed.
- **No serialisability guarantee**: The combination of reads and writes in a read-committed transaction may not correspond to any serial execution order.

---

## Implementation Details

### Connection Flag

`SQLITE_ReadCommitted` is defined as `HI(0x00100)` on `db->flags` when `SQLITE_ENABLE_READ_ISOLATION` is compiled in.

### WAL Integration

The `Wal` struct gains a `bReadCommitted` field (under `SQLITE_ENABLE_READ_ISOLATION`). This is set via `sqlite3WalSetReadCommitted()` at `BEGIN CONCURRENT` time and cleared at transaction end.

The `walScanConflicts()` function checks `pWal->bReadCommitted` **before** the RLL check:

```c
if( pAllRead && sqlite3BitvecTestNotNull(pAllRead, pgno) ){
  int bRowConflict = 1;

  // 1. Page-level RC check (SQLITE_ENABLE_READ_ISOLATION)
  if( pWal->bReadCommitted ){
    PgHdr *pChk = sqlite3PagerLookup(pPager, pgno);
    if( pChk && !sqlite3PagerIswriteable(pChk) ){
      bRowConflict = 0;  // read-only page → skip
    }
    // ... unref
  }

  // 2. Row-level RLL check (SQLITE_ENABLE_ROW_LEVEL_LOCKING)
  if( bRowConflict && pWal->pCurReadRows ){
    // ... row-level conflict detection
  }

  if( bRowConflict ){
    return SQLITE_BUSY_SNAPSHOT;
  }
}
```

### Call Chain

```
PRAGMA isolation_level = read_committed
  → db->flags |= SQLITE_ReadCommitted

BEGIN CONCURRENT
  → sqlite3BtreeBeginTrans()
    → sqlite3PagerSetReadCommitted(pPager, 1)
      → sqlite3WalSetReadCommitted(pWal, 1)
        → pWal->bReadCommitted = 1

COMMIT (conflict scan)
  → walScanConflicts()
    → page in pAllRead hit:
      → if bReadCommitted && page not dirty: skip
      → else if RLL: row-level check
      → else: BUSY_SNAPSHOT

Transaction end
  → btreeEndTransaction()
    → sqlite3PagerSetReadCommitted(pPager, 0)
      → pWal->bReadCommitted = 0
```

---

## Files Modified

| File | Changes |
|------|---------|
| `src/sqliteInt.h` | `SQLITE_ReadCommitted` flag definition (gated on `SQLITE_ENABLE_READ_ISOLATION`) |
| `src/wal.c` | `bReadCommitted` field on `Wal` struct; page-level RC check in `walScanConflicts()`; `sqlite3WalSetReadCommitted()` |
| `src/wal.h` | `sqlite3WalSetReadCommitted()` declaration |
| `src/pager.c` | `sqlite3PagerSetReadCommitted()` bridge function |
| `src/pager.h` | `sqlite3PagerSetReadCommitted()` declaration |
| `src/btree.c` | Flag propagation at `BEGIN CONCURRENT`; flag clear at transaction end; P1 read-skip optimisation (with RLL) |
| `src/pragma.c` | `PRAGMA isolation_level` handler |
| `tool/mkpragmatab.tcl` | `isolation_level` pragma definition |

---

## Test Files

### Standalone Read Isolation (no RLL required)

| File | Tests | Description |
|------|-------|-------------|
| `test/read_isolation.test` | 43 | Page-level RC: pragma round-trip, read-only page skip, write-write page conflict, snapshot regression, same-page read+write conflict |

### Combined RLL + Read Isolation

| File | Tests | Description |
|------|-------|-------------|
| `test/rowlock_read_isolation.test` | ~42 | Row-level RC with RLL: read-write row skip, write-write row conflict, snapshot + RLL regression, disjoint same-page writes |

---

## Build & Test

```bash
# Read isolation only (page-level RC)
make clean && ./configure --enable-all
LIBRARY_PATH=/opt/homebrew/lib \
  OPTS="-DSQLITE_ENABLE_READ_ISOLATION" \
  make testfixture
./testfixture test/read_isolation.test

# Read isolation + RLL (row-level RC)
make clean && ./configure --enable-all
LIBRARY_PATH=/opt/homebrew/lib \
  OPTS="-DSQLITE_ENABLE_ROW_LEVEL_LOCKING -DSQLITE_ENABLE_READ_ISOLATION" \
  make testfixture
./testfixture test/read_isolation.test test/rowlock.test test/pipeline.test
```

---

## Design Decisions

1. **Separate build flags**: `SQLITE_ENABLE_READ_ISOLATION` is independent of `SQLITE_ENABLE_ROW_LEVEL_LOCKING`. This allows deploying page-level read-committed without the complexity and memory overhead of row-level tracking. When both are enabled, they compose naturally.

2. **Page-level before row-level**: In `walScanConflicts()`, the page-level RC check runs first. If it clears the conflict (page was read-only), the more expensive RLL lookup is skipped entirely. This provides a fast path for the common case where reads and writes are on different pages.

3. **Conservative eviction handling**: If a page was in `pAllRead` but has been evicted from the pager cache, it's treated as read-only (skip the conflict). This is safe because eviction only happens for clean pages — dirty pages are pinned in cache.

4. **Per-transaction flag**: The `bReadCommitted` flag is set on the WAL at `BEGIN CONCURRENT` time and cleared at transaction end. This ensures the flag doesn't leak between transactions and correctly reflects the connection's isolation level at the time the transaction started.
