# Row-Level Locking for SQLite `BEGIN CONCURRENT`

## Overview

Row-level locking (RLL) adds fine-grained conflict detection to SQLite's `BEGIN CONCURRENT` mechanism. Instead of detecting conflicts at the page level (which produces false positives when two transactions touch different rows on the same B-tree leaf page), RLL tracks individual `(root-page, key)` pairs and resolves conflicts at the row level.

### Problem Statement

SQLite's existing `BEGIN CONCURRENT` implementation uses a page-level bitvec (`pAllRead`) to track which database pages a transaction has read. At commit time, if any page in `pAllRead` was also written by another committed transaction, the commit fails with `SQLITE_BUSY_SNAPSHOT` — even if the actual rows accessed by the two transactions were entirely disjoint.

For workloads where multiple concurrent writers touch different rows on the **same** B-tree leaf page, this produces frequent false conflicts and unnecessary rollbacks.

### Solution

RLL adds a per-transaction `RowLockSet` that tracks individual `(root-page, key)` pairs for both reads and writes. At commit time, when a page-level conflict is detected, the system falls back to row-level comparison: if the rows actually modified by the committed transaction don't overlap with the rows read/written by the committing transaction, the commit succeeds and a 3-way cell-level merge is performed on the shared page.

---

## Build Flag

### `SQLITE_ENABLE_ROW_LEVEL_LOCKING`

Enables the entire row-level locking subsystem:

- `RowLockSet` / `RowLockEntry` data structures
- Row tracking in B-tree cursor operations (read/insert/delete)
- Row-level conflict detection bypass in WAL commit
- 3-way cell merge for pages modified by both transactions
- `PRAGMA row_level_locking` (on/off per connection)
- `PRAGMA row_lock_threshold` (per-connection entry limit)
- `sqlite3_limit(SQLITE_LIMIT_ROW_LOCK_ENTRIES, N)` API
- WAL sidecar row-log for cross-process conflict detection (WAL1 and WAL2)

```bash
-DSQLITE_ENABLE_ROW_LEVEL_LOCKING
```

**Requires:** `BEGIN CONCURRENT` support (i.e., `SQLITE_OMIT_CONCURRENT` must **not** be defined).

### Combined with Read Isolation

When combined with `SQLITE_ENABLE_READ_ISOLATION` (see [read-isolation.md](read-isolation.md)):

- `ROW_LOCK_READ` entries are skipped entirely in read-committed mode (P1 optimisation)
- At conflict detection, only `ROW_LOCK_WRITE` vs `ROW_LOCK_WRITE` overlaps count as conflicts
- `PRAGMA isolation_level` becomes functional (without RI it always returns `"snapshot"`)

```bash
-DSQLITE_ENABLE_ROW_LEVEL_LOCKING -DSQLITE_ENABLE_READ_ISOLATION
```

---

## Architecture

### Data Structures

#### `RowLockEntry` (`btreeInt.h`)

Represents a single tracked row in a transaction's lock set:

| Field       | Type    | Description |
|-------------|---------|-------------|
| `iRoot`     | `Pgno`  | B-tree root page (table/index identifier) |
| `iRowid`    | `i64`   | Integer key (INTKEY tables); 0 for BLOBKEY |
| `pKey`      | `u8*`   | Serialised key blob (BLOBKEY / WITHOUT ROWID); NULL for INTKEY |
| `nKey`      | `int`   | Byte length of `pKey` |
| `nKeyField` | `int`   | Number of PK fields for comparison (0 = all fields) |
| `iGen`      | `u32`   | Generation counter for savepoint rollback |
| `eType`     | `u8`    | `ROW_LOCK_READ` (1) or `ROW_LOCK_WRITE` (2) |
| `pNext`     | `RowLockEntry*` | Next entry in hash bucket |

#### `RowLockSet` (`btreeInt.h`)

Hash table of `RowLockEntry` objects per B-tree shared structure:

| Field        | Type    | Description |
|--------------|---------|-------------|
| `aHash`      | `RowLockEntry**` | Hash bucket array |
| `nAlloc`     | `int`   | Number of buckets allocated |
| `nEntry`     | `int`   | Total entries inserted |
| `nThreshold` | `int`   | Max entries before page-level fallback (spill) |
| `bSpilled`   | `u8`    | True if threshold exceeded |
| `db`         | `sqlite3*` | Owning connection (NULL after detach) |
| `iGenCur`    | `u32`   | Current generation counter |
| `nSvpt`      | `int`   | Active savepoint levels |
| `aSvpt`      | `u32*`  | Generation at start of each savepoint |

#### `WalCommitRowSet` (`wal.c`)

Per-commit record stored in the shared `WalRowLog`:

| Field       | Type    | Description |
|-------------|---------|-------------|
| `iMinFrame` | `u32`   | First WAL frame of the commit (file-local) |
| `iMaxFrame` | `u32`   | Last WAL frame of the commit (file-local) |
| `pRows`     | `RowLockSet*` | Rows written (owned by this node) |
| `pNext`     | `WalCommitRowSet*` | Linked list (newest first) |

#### `WalRowLog` (`wal.c`)

Per-WAL-file singleton shared by all connections in a process:

| Field                | Type    | Description |
|----------------------|---------|-------------|
| `pKey`               | `void*` | `apWiData[0]` — unique per WAL file |
| `nRef`               | `int`   | Reference count |
| `apCommits[2]`       | `WalCommitRowSet*` | Per-WAL-file committed row sets (newest first) |
| `abSidecarLoaded[2]` | `int`   | Whether sidecar file has been loaded (per WAL file) |
| `aiSidecarReadOff[2]`| `i64`   | Sidecar file read watermark (per WAL file) |

In WAL1 mode only index `[0]` is used. In WAL2 mode both indices are active.

#### Sidecar Fields on `struct Wal` (`wal.c`)

Each `Wal` connection holds per-file sidecar state:

| Field             | Type    | Description |
|-------------------|---------|-------------|
| `apSidecarFd[2]`  | `sqlite3_file*` | Open fds for sidecar files (ptrs into `Wal` alloc) |
| `azSidecar[2]`    | `char*` | Malloc'd paths; NULL = disabled |
| `abSidecarOk[2]`  | `int`   | 1 = sidecar file is open and writable |
| `aiSidecarOff[2]` | `i64`   | Next write offset within sidecar files |

### Row Tracking Points

Rows are tracked at the B-tree cursor level:

| Operation | Type | Location |
|-----------|------|----------|
| `sqlite3BtreeIntegerKey()` | `ROW_LOCK_READ` | When cursor reads an INTKEY row |
| `sqlite3BtreeInsert()` (INTKEY) | `ROW_LOCK_WRITE` | Integer key insert/update |
| `sqlite3BtreeInsert()` (BLOBKEY) | `ROW_LOCK_WRITE` | Index / WITHOUT ROWID insert |
| `sqlite3BtreeDelete()` (INTKEY) | `ROW_LOCK_WRITE` | Integer key delete |
| `sqlite3BtreeDelete()` (BLOBKEY) | `ROW_LOCK_WRITE` | Index / WITHOUT ROWID delete |

### Conflict Detection Flow

```
BEGIN CONCURRENT
  ├── Allocate pRowLocks on BtShared
  ├── Track ROW_LOCK_READ / ROW_LOCK_WRITE as cursor ops execute
  └── COMMIT
       ├── Phase 1: walPreCheck() [no write lock]
       │   ├── Scan WAL frames since snapshot
       │   ├── For each page in pAllRead that was modified:
       │   │   ├── [RI] If bReadCommitted and page is read-only: skip
       │   │   ├── [RLL] Decode external frame → (iWal, localFrame)  [WAL2]
       │   │   ├── [RLL] Look up WalCommitRowSet for that frame
       │   │   ├── [RLL] Call rowLockSetHasConflict(pCommit, pMine, bRC)
       │   │   │   ├── Returns 0: no row overlap → skip conflict
       │   │   │   ├── Returns 1: genuine conflict → BUSY_SNAPSHOT
       │   │   │   └── Returns -1: spilled/unknown → page-level fallback
       │   │   └── Record page for 3-way merge if both txns dirtied it
       │   └── Capture WAL header state in WalPreCheckCtx
       │
       ├── Phase 2: walDeltaCheck() [acquire write lock]
       │   ├── Load sidecar row-log(s) for cross-process commits
       │   │   ├── WAL1: walSidecarLoad(pWal, &head, 0)
       │   │   └── WAL2: walSidecarLoad(pWal, &head, 0) + (pWal, &head, 1)
       │   ├── Scan only frames committed since pre-check
       │   └── Same conflict logic as Phase 1 for delta frames
       │
       └── Phase 3: 3-way cell merge
            ├── For each WalMergePage:
            │   ├── Read committed version from WAL frame
            │   ├── Binary-search both pages for conflicting cells
            │   └── Insert/replace cells from committed version
            └── Write merged pages to WAL
```

### Spill / Threshold Mechanism

When a transaction tracks more rows than `nThreshold` (configurable via `PRAGMA row_lock_threshold` or `sqlite3_limit(SQLITE_LIMIT_ROW_LOCK_ENTRIES)`), the `RowLockSet` sets `bSpilled=1` and stops inserting new entries. At conflict detection time, `rowLockSetHasConflict()` returns `-1` (unknown), and the system falls back to page-level conflict detection — the same behaviour as without RLL.

Default threshold: `SQLITE_MAX_ROW_LOCK_ENTRIES` = `0x7fffffff` (effectively unlimited).

### Savepoint Integration

The `RowLockSet` supports nested savepoints via a generation counter:

- `rowLockSetBegin(pSet, nSvpt)`: Opens savepoint level, increments generation
- `SAVEPOINT_ROLLBACK`: Removes all entries with `iGen >= aSvpt[iSvpt]`
- `SAVEPOINT_RELEASE`: Entries adopted by parent scope (no removal)

This integrates with both user `SAVEPOINT` statements and implicit statement-level transactions.

### DDL Fallback

When DDL is detected during a `CONCURRENT` transaction (promotion to `CONCURRENT_SCHEMA`), all row-level lock sets are discarded via `sqlite3BtreeDiscardRowLocks()`. The transaction falls back to page-level conflict detection, which is the correct conservative behaviour after a schema change.

---

## WAL Sidecar Row-Log (Cross-Process)

For cross-process conflict detection, committed row sets are persisted to sidecar files alongside the WAL. This is the "Option C" design.

### WAL1 Mode — Single Sidecar

In WAL1 mode a single sidecar file is used:

```
<database>-rowlog       ← row log for <database>-wal
```

### WAL2 Mode — Dual Sidecar

In WAL2 mode each WAL file has its own sidecar:

```
<database>-rowlog       ← row log for <database>-wal   (file 0)
<database>-wal2-rowlog  ← row log for <database>-wal2  (file 1)
```

Each sidecar's 16-byte header stores the salt pair of its corresponding WAL file. Frame numbers inside each sidecar are **file-local** (1, 2, 3 …), not externally encoded.

### Sidecar File Format

```
Header (16 bytes):
  u32  magic      0x524C4F47 ("RLOG")
  u32  version    1
  u32  salt1      (from WAL file header — generation check)
  u32  salt2      (from WAL file header)

Per-record:
  u32  mxFrame      Last frame of this commit (file-local)
  u32  iMinFrame    First frame of this commit (file-local)
  <body>            Output of rowLockSetSerialize():
    u32  nEntry
    u8   bSpilled
    u8[3] reserved
    per-entry:
      INTKEY:  u8 eType, u8 flags=0x01, u16 pad, u32 iRoot, i64 iKey  (16 bytes)
      BLOBKEY: u8 eType, u8 flags=0x02, u16 nKeyField, u32 iRoot, u32 nKey, u8[nKey]
  u32  checksum     Additive over all preceding bytes of this record
```

### WAL2 Lifecycle Mapping

| WAL2 Event | Sidecar Action |
|---|---|
| Writer appends frames to WAL file X | Append commit record to sidecar X |
| Checkpoint backfills file X up to `mxSafeFrame` | Trim sidecar X entries where `iMaxFrame <= mxSafeFrame` |
| Checkpoint completes file X (all frames backfilled) | Delete sidecar X |
| Writer switches active file X → Y | Future appends go to sidecar Y; sidecar X remains for readers |
| `sqlite3WalClose()` with `isDelete` | Delete both sidecar files |
| Crash recovery / salt mismatch on load | Delete the mismatched sidecar file |

### WAL2 Conflict Scan

`walScanConflicts()` iterates both WAL files via `iLoop`. The hash entries yield external frame numbers (`sLoc.iZero + i`). To look up the matching commit row set, external frames are decoded to file-local form:

```c
u32 localFrame;
int iWalFile;
if( bWal2 ){
  iWalFile = walExternalDecode(sLoc.iZero + i, &localFrame);
}else{
  iWalFile = 0;
  localFrame = sLoc.iZero + i;
}
pCRS = walFindCommitRowSet(pWal, iWalFile, localFrame);
```

### WAL2 Salt Validation

In WAL2 mode, `pHead->aSalt` reflects only the currently active WAL file. When loading the sidecar for the non-active file, the salt is read from that WAL file's header on disk (offset 16, 8 bytes) rather than from shared memory.

### Sidecar Lifecycle (Common)

- Created lazily on first `CONCURRENT` commit with RLL enabled
- Loaded incrementally by `walSidecarLoad()` when the WAL write lock is held
- Deleted when the WAL is checkpointed and reset (WAL1: TRUNCATE checkpoint; WAL2: per-file checkpoint completion)
- On any I/O error, permanently disabled via `walSidecarFail()` — falls back to page-level

---

## Two-Phase Commit Protocol

The commit is split into two phases to reduce write-lock contention:

### Phase 1: `sqlite3WalPreCheck()` — No Write Lock

- Runs under the transaction's existing read lock
- Scans all WAL frames since the snapshot for page-level conflicts
- For each hit, attempts row-level resolution using in-memory `WalCommitRowSet`
- If a genuine conflict is found, returns `SQLITE_BUSY_SNAPSHOT` immediately
- Captures the WAL header state in `WalPreCheckCtx`

### Phase 2: `sqlite3WalDeltaCheck()` — Write Lock Held

- Acquires the WAL write lock (retries via busy-handler on `SQLITE_BUSY`)
- Loads the sidecar row-log(s) for any cross-process commits (both files in WAL2)
- Scans only the **delta** frames committed since Phase 1
- If clean, the write lock is retained for frame writing

This design allows most of the conflict-scan work to happen in parallel with other writers.

---

## API Surface

### Compile-Time Limits

| Macro | Default | Description |
|-------|---------|-------------|
| `SQLITE_MAX_ROW_LOCK_ENTRIES` | `0x7fffffff` | Hard limit for row lock entries per connection |

### Runtime Limits

```c
// Get/set row lock entry limit
sqlite3_limit(db, SQLITE_LIMIT_ROW_LOCK_ENTRIES, newVal);
```

### PRAGMAs

| PRAGMA | Values | Description |
|--------|--------|-------------|
| `row_level_locking` | `0` / `1` | Enable/disable RLL per connection |
| `row_lock_threshold` | integer | Max entries before spill to page-level |

### Test Controls

```c
// Get number of WalCommitRowSet entries in WAL row log (sum of both files)
sqlite3_test_control(SQLITE_TESTCTRL_WAL_ROWLOG_COUNT, db, &count);
```

---

## Test Files

### RLL Tests (via `test/rowlock.test` driver)

| File | Description |
|------|-------------|
| `test/rowlock.test` | Driver — sources all sub-test files |
| `test/rowlock_basic.test` | Basic INTKEY row-level conflict detection (§1–2) |
| `test/rowlock_savepoint.test` | Savepoint rollback/release integration (§3–5) |
| `test/rowlock_threshold.test` | Spill threshold and page-level fallback (§6,10,11,13–15) |
| `test/rowlock_blobkey.test` | BLOBKEY / WITHOUT ROWID / index tracking (§8) |
| `test/rowlock_ddl.test` | DDL fallback to page-level (§9) |
| `test/rowlock_wal.test` | WAL row-log count via TESTCTRL (§12) |
| `test/rowlock_sidecar.test` | WAL1 sidecar lifecycle and cross-process (§14) |
| `test/rowlock_wal2.test` | WAL2 dual-sidecar: basic conflicts, file lifecycle, file-switch, checkpoint cleanup, incremental load, generation reset (§20–28) |

### Combined RLL + Read Isolation Tests (via `test/rowlock_read_isolation.test` driver)

| File | Description |
|------|-------------|
| `test/rowlock_read_isolation.test` | Driver — sources WAL1 and WAL2 sub-files |
| `test/rowlock_read_isolation_wal.test` | RC/snapshot + RLL under WAL1 (§1–5) |
| `test/rowlock_read_isolation_wal2.test` | RC/snapshot + RLL under WAL2: read-write overlap ignored in RC, write-write detected, generation reset after checkpoint, cross-process sidecar, file-switch, read-first snapshot detection (§1–8) |

### Other Test Files

| File | Description |
|------|-------------|
| `test/pipeline.test` | Two-phase commit pipeline tests |
| `test/read_isolation.test` | Standalone page-level read isolation tests (no RLL) |

See [read-isolation.md](read-isolation.md) for standalone read isolation documentation.

---

## Files Modified

### Core Source Files

| File | Changes |
|------|---------|
| `src/btree.c` | RowLockSet implementation (hash table, insert, conflict check, serialize/deserialize, savepoint, 3-way merge), row tracking in cursor ops |
| `src/btreeInt.h` | `RowLockEntry`, `RowLockSet` struct definitions, `pRowLocks` on `BtShared` |
| `src/btree.h` | Public API for row lock set operations |
| `src/wal.c` | `WalCommitRowSet`, `WalRowLog` (with per-file arrays), dual-sidecar I/O, two-phase commit, `WalMergePage`, WAL2 external frame decode for row-log lookup |
| `src/wal.h` | Two-phase commit types (`WalPreCheckCtx`), `WalMergePage`, row-log functions (including `iWal` parameter for WAL2) |
| `src/pager.c` | Bridge functions (`sqlite3PagerSetReadRowSet`, `sqlite3PagerRecordCommitRows`), `sqlite3WalGetMxFrame()` returns file-local mxFrame in WAL2 |
| `src/pager.h` | Row-lock pager bridge declarations |
| `src/sqliteInt.h` | Forward declarations, `SQLITE_RowLevelLocking` flag |
| `src/sqliteLimit.h` | `SQLITE_MAX_ROW_LOCK_ENTRIES` |
| `src/sqlite.h.in` | `SQLITE_LIMIT_ROW_LOCK_ENTRIES`, `SQLITE_TESTCTRL_WAL_ROWLOG_COUNT` |
| `src/main.c` | Limit registration, test control handler |
| `src/pragma.c` | `row_lock_threshold` pragma handler |
| `src/vdbe.c` | Savepoint integration, DDL detection/row-lock discard |
| `src/test1.c` | Tcl test bindings for limits and test controls |
| `tool/mkpragmatab.tcl` | Pragma table definitions |

---

## Build & Test

```bash
# RLL only (snapshot isolation)
make clean && ./configure --enable-all
LIBRARY_PATH=/opt/homebrew/lib \
  OPTS="-DSQLITE_ENABLE_ROW_LEVEL_LOCKING" \
  make testfixture
./testfixture test/rowlock.test test/pipeline.test

# RLL + Read Isolation (full suite)
make clean && ./configure --enable-all
LIBRARY_PATH=/opt/homebrew/lib \
  OPTS="-DSQLITE_ENABLE_ROW_LEVEL_LOCKING -DSQLITE_ENABLE_READ_ISOLATION" \
  make testfixture
./testfixture test/pipeline.test test/read_isolation.test \
              test/rowlock.test test/rowlock_read_isolation.test
```

---

## Design Decisions

1. **Graceful degradation**: When the row lock set exceeds the threshold (`bSpilled`), the system transparently falls back to page-level conflict detection. No errors, no behaviour change — just the same false-positive rate as without RLL.

2. **Two-phase commit**: Pre-check without the write lock allows conflict scanning to happen concurrently with other committers, reducing lock contention for read-heavy CONCURRENT workloads.

3. **3-way cell merge**: When RLL determines two transactions wrote different rows on the same leaf page, a cell-level merge is performed in-place rather than aborting. This is the key innovation that allows true row-level concurrency.

4. **Dual-sidecar row-log (WAL2)**: WAL2 rotates between two WAL files, each with its own salt and frame numbering. A single sidecar would be invalidated on every file switch. The dual-sidecar design gives each WAL file its own sidecar with file-local frame numbers and per-file salt validation, ensuring row-log data survives file switches and is checkpointed independently.

5. **Silent sidecar degradation**: Any I/O error permanently disables the sidecar (`walSidecarFail()`). The system falls back to page-level conflict detection with no user-visible error — cross-process row-level precision is a best-effort optimisation.

6. **DDL safety**: Schema changes automatically discard all row-level tracking and fall back to page-level. This is conservative but correct — DDL operations can restructure B-tree pages in ways that invalidate row-level tracking.
