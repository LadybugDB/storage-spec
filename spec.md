# LadybugDB Local Storage Specification

This document specifies the **single-file local storage** of LadybugDB — everything
that is persisted to the on-disk database file (`<dbpath>`) plus its companion
sidecar files (WAL, shadow file). It is derived from the implementation under
`src/storage` and `src/include/storage`.

> Note on terminology: the `local_storage/` subdirectory implements *per-transaction
> uncommitted (MVCC) data* and is **out of scope** for this spec. "Local storage"
> here means the durable, single-database-file storage format.

---

## 1. Overview

LadybugDB stores an entire database — catalog, table data, indexes, free-space
bookkeeping — in **one main database file**, referenced as `dataFH` / `databasePath`.
Two sidecar files provide durability and atomic checkpointing:

| File                | Suffix / name                | Purpose                                          |
|---------------------|------------------------------|--------------------------------------------------|
| Main data file      | `<dbpath>`                   | Catalog, metadata, tables, indexes, page headers |
| WAL                 | `<dbpath>.wal`               | Write-ahead log of committed transactions        |
| Checkpoint WAL      | `<dbpath>.wal.checkpoint`    | Frozen WAL during checkpoint rotation            |
| Shadow file         | `<dbpath>.shadow`            | Copy-on-write shadow pages for atomic writes     |

All on-disk I/O is page-based and goes through the **Buffer Manager**, which pins
pages into a fixed-size buffer pool and evicts them back to disk on pressure.

The storage layer is organized into these subsystems (matching the directory
structure under `src/storage/`):

- **Storage Manager** — top-level owner of the data file, WAL, shadow file, tables,
  and indexes; drives recovery and checkpointing.
- **Buffer Manager** — page cache, eviction, spilling; owns `FileHandle`s.
- **File Handle** — per-file abstraction over pages, page states, page manager.
- **Page Manager / Free Space Manager / Page Allocator** — page allocation and
  free-space tracking inside the data file.
- **Shadow File / Shadow Utils** — copy-on-write shadow pages for atomic updates.
- **Database Header** — on-disk header (page 0) with version, UUID, page ranges.
- **Disk Array / Disk Array Collection** — append-only page-resident arrays
  used by metadata and indexes.
- **WAL** — write-ahead log with checksummed, TypeSpec-generated records.
- **Checkpointer** — coordinates checkpoints: serialize catalog, flush tables,
  apply shadow pages, rotate WAL, update header.
- **Tables** (`table/`) — node tables, rel tables, columnar storage, CSR,
  node groups, column chunks, compression.
- **Indexes** (`index/`) — hash index (primary key) and ART index.
- **Compression** (`compression/`) — constant, uncompressed, integer bitpacking,
  ALP float compression.
- **Undo Buffer** — in-memory MVCC undo/rollback records (not on-disk format).
- **Stats / Predicates** — table/column statistics and scan predicates.

---

## 2. Constants and Configuration

Build-time tunables come from `cmake/templates/system_config.h.in` and
`src/include/common/constants.h`. Defaults:

| Constant                          | Default          | Notes                                                |
|-----------------------------------|------------------|------------------------------------------------------|
| `LBUG_PAGE_SIZE`                  | 4096 (1<<12)     | Page size for the data file and most structures.     |
| `TEMP_PAGE_SIZE`                  | 262144 (1<<18)   | Large page size for temp/operator files.             |
| `StorageConstants::PAGE_GROUP_SIZE` | 1024 (1<<10)   | Pages grown at a time; page-state group granularity. |
| `StorageConfig::NODE_GROUP_SIZE`  | build-tunable    | Rows per node group.                                 |
| `StorageConfig::CHUNKED_NODE_GROUP_CAPACITY` | min(2048, NODE_GROUP_SIZE) | Rows per chunked node group.              |
| `StorageConfig::MAX_SEGMENT_SIZE` | build-tunable    | Max bytes for a column segment (== `LBUG_PAGE_SIZE`).|
| `StorageConstants::MAX_NUM_ROWS_IN_TABLE` | 1<<62   | Hard row-count cap; offsets ≥ this are "local".      |
| `StorageConstants::DB_HEADER_PAGE_IDX` | 0          | Database header lives on page 0.                     |
| `HashIndexConstants::NUM_HASH_INDEXES` | 256 (1<<8) | Number of hashed sub-indexes.                       |

### File / page sizing constraints

- `maxDBSize` must be a power of two and at least
  `2 * LBUG_PAGE_SIZE * PAGE_GROUP_SIZE` (one page group for the data file and
  one for the shadow file).
- `bufferPoolSize` must be at least `LBUG_PAGE_SIZE`.

### Storage version

`StorageVersionInfo` (`src/include/storage/storage_version_info.h`) gates on-disk
compatibility. The magic bytes are `"LBUG"`. Readable versions:

- `STORAGE_VERSION_40` — releases 0.12.0 … 0.16.1.
- `STORAGE_VERSION_41` — 0.17.x; adds table storage `FORMAT` field to catalog entries.
- `STORAGE_VERSION_42` — 0.18.0; adds per-FROM/TO rel multiplicity.

`canReadStorageVersion(v)` accepts 40, 41, and the current build version.

---

## 3. File Layout

### 3.1 Main data file (`<dbpath>`)

A flat sequence of fixed-size pages (`LBUG_PAGE_SIZE` each). Page 0 is reserved
for the **Database Header**. All other structures allocate page ranges through
the `PageManager` and address data by `page_idx_t`.

Logical regions, tracked in the header and page manager:

- **Database header** — page 0 (see §5).
- **Catalog pages** — serialized catalog (schema, table entries). Range recorded
  in `DatabaseHeader::catalogPageRange`.
- **Metadata pages** — serialized storage metadata (page manager state, table
  metadata). Range recorded in `DatabaseHeader::metadataPageRange`.
- **Table data pages** — column chunks, node groups, CSR headers.
- **Index pages** — hash index slots/headers, ART index nodes.
- **Disk-array pages** — append-only arrays for metadata/index bookkeeping.
- **Free pages** — tracked by the Free Space Manager; reclaimable on checkpoint.

The file grows in **page groups** of `PAGE_GROUP_SIZE` pages. Each page group has
a `PageState` entry in the `FileHandle` for buffer-manager tracking.

### 3.2 WAL (`<dbpath>.wal`)

Append-only log of WAL records (see §7). Prefixed with a `WALHeader` carrying the
`databaseID` UUID and an `enableChecksums` flag. Records are length-prefixed and
(optionally) checksummed. The WAL is rotated during checkpoint: the live WAL is
frozen into `<dbpath>.wal.checkpoint`, a new empty WAL is opened, and after the
checkpoint lands the frozen file is cleared.

### 3.3 Shadow file (`<dbpath>.shadow`)

A copy-on-write spill file used so that checkpoint-bound writes do not mutate the
live data file in place until the checkpoint commits. The shadow file holds:

- A `ShadowFileHeader` (`{databaseID, numShadowPages}`).
- A list of `ShadowPageRecord` (`{originalFileIdx, originalPageIdx}`) mapping each
  shadowed original page to its shadow page.
- The shadow page contents themselves.

`ShadowFile::applyShadowPages()` replays shadow pages back onto the original data
file at checkpoint finalize; `replayShadowPageRecords()` does the same during
crash recovery. The shadow file is verified against the data file's `databaseID`
UUID before replay (`FileDBIDUtils`).

---

## 4. Buffer Manager

The Buffer Manager (`src/storage/buffer_manager/`) is the sole path to disk pages.
It owns a fixed-size pool of page-sized frames backed by `mmap`'d virtual memory
regions (`VMRegion`) and serves `pinPage` / `optimisticReadPage` requests against
registered `FileHandle`s.

### 4.1 Page state machine

Each page has a `PageState` — a 64-bit packed word (top 8 bits = state, next bit =
dirty, low 55 bits = version) atomically updated:

```
UNLOCKED (0) -> LOCKED (1) -> UNLOCKED      (normal read/write)
UNLOCKED      -> MARKED (2)                   (enqueued as eviction candidate)
MARKED        -> EVICTED (3)                  (frame reclaimed)
EVICTED       -> UNLOCKED                     (page re-loaded on next pin)
```

- **`LOCKED`** — page is being read or written; no eviction.
- **`UNLOCKED`** — resident, recently usable; eligible for "second chance" eviction.
- **`MARKED`** — queued in the eviction queue as a candidate.
- **`EVICTED`** — frame released via `MADV_DONTNEED`; contents must be re-read on pin.

`EvictionCandidate` checks `isEvictable` (MARKED) and `isSecondChanceEvictable`
(UNLOCKED). A claimed candidate is locked, and if dirty it is flushed to disk
before the frame is released.

### 4.2 Memory Manager and Spiller

`MemoryManager` (`memory_manager.h`) allocates `MemoryBuffer`s (TEMP_PAGE_SIZE
default) for query-execution scratch and node-group in-memory data. When the pool
is under pressure the **Spiller** (`spiller.cpp`) writes `ChunkedNodeGroup`s to a
temporary spill file and reloads them on demand, keyed by file position. Spilled
buffers are tracked via `SpillResult`.

### 4.3 Frame ↔ page mapping

Frames are grouped in `VMRegion` frame groups of `PAGE_GROUP_SIZE` frames. Each
file page group allocated by a `FileHandle` claims one frame group, so a page
uniquely maps to a frame via `getFrameIdx(pageIdx)`. Two page-size classes exist:
`LBUG_PAGE_SIZE` (data file) and `TEMP_PAGE_SIZE` (temp/operator files).

### 4.4 FileHandle flags

`FileHandle` opens files with a bitmask of flags:

| Flag                                  | Meaning                                          |
|---------------------------------------|--------------------------------------------------|
| `O_PERSISTENT_FILE_READ_ONLY`         | Read-only existing persistent file.              |
| `O_PERSISTENT_FILE_CREATE_NOT_EXISTS` | Create the data file if it doesn't exist.        |
| `O_IN_MEM_TEMP_FILE`                  | In-memory temporary file (large pages).          |
| `O_PERSISTENT_FILE_IN_MEM`            | In-memory database (`:memory:`).                 |
| `O_LOCKED_PERSISTENT_FILE`            | Acquire an OS file lock.                          |

`isInMemoryMode()` is true when the file is a new tmp in-memory file.

---

## 5. Database Header

The database header lives on **page 0** of the data file
(`StorageConstants::DB_HEADER_PAGE_IDX`). It is serialized with debug-keyed fields
and validated on open.

### 5.1 Serialized fields (in order)

| Debug key        | Type                | Meaning                                        |
|------------------|---------------------|------------------------------------------------|
| `magic`          | 4 bytes `"LBUG"`    | File magic.                                    |
| `storage_version`| `uint64_t`          | On-disk format version (see §2.3).             |
| `catalog`        | `page_idx_t`, `page_idx_t` | Start page + num pages of catalog region. |
| `metadata`       | `page_idx_t`, `page_idx_t` | Start page + num pages of metadata region.|
| `databaseID`     | `uint64_t` (UUID)   | Unique DB id; matched against WAL/shadow.      |
| *(no key)*       | `uint8_t`           | Header format version.                          |
| *(no key)*       | `uint64_t`          | `dataFileNumPages` (only if format version ≥ 2).|

Old format (pre-0.14.x) stops after `databaseID`; the trailing `dataFileNumPages`
is read only when the format-version byte equals
`HEADER_FORMAT_VERSION_WITH_DATAFILE_NUM_PAGES` (2). This keeps older files
readable.

### 5.2 `DatabaseHeader` struct (`database_header.h`)

```cpp
struct DatabaseHeader {
    PageRange catalogPageRange;
    PageRange metadataPageRange;
    page_idx_t dataFileNumPages{0};
    uuid databaseID{0};
    storage_version_t storageVersion{StorageVersionInfo::getStorageVersion()};
    ...
};
```

- `createInitialHeader()` — generates a random `databaseID` and the current
  storage version; used when a new database file is created.
- `readDatabaseHeader(FileInfo&)` — returns `nullopt` if the file is smaller than
  one page (empty/new file).
- `updateCatalogPageRange` / `freeMetadataPageRange` — release the old ranges back
  to the `PageManager` before recording new ones.

The `databaseID` UUID is written to the WAL and shadow file headers and verified
on replay (`FileDBIDUtils::verifyDatabaseID`) so stale sidecar files from a
previous database at the same path are rejected.

---

## 6. Page Management and Free Space

### 6.1 Allocation hierarchy

```
PageAllocator (interface)
   └── PageManager          (data file: alloc + free + FSM + versioning)
          └── FreeSpaceManager   (free-list bookkeeping)
```

- **`PageAllocator`** (`page_allocator.h`) — minimal interface: `allocatePageRange(n)`
  and `freePageRange(range)`. Used by code that needs pages but not full FSM
  semantics (e.g. indexes, in-memory builders).
- **`PageManager`** (`page_manager.h`) — the data file's allocator. Owns a
  `FreeSpaceManager`, a `version` counter, and `lastCheckpointVersion`. Exposed to
  tables/indexes via `dataFH->getPageManager()`.

### 6.2 Page ranges

`PageRange { page_idx_t startPageIdx; page_idx_t numPages; }` is the unit of
allocation. `allocatePageRange(n)` first consults the FSM for a fitting free range;
if none, it appends a new page group to the file (`FileHandle::addNewPages`).

### 6.3 Free Space Manager

`FreeSpaceManager` keeps free pages in **leveled sorted free lists** — a vector of
`std::set<PageRange>` indexed by `getLevel(numPages)` (log-scaled size class). This
gives fast best-fit lookup. Operations:

- `addFreePages(range)` — immediately reusable free pages.
- `addUncheckpointedFreePages(range)` — freed but **not reusable until the next
  checkpoint finalizes** (prevents reclaiming a page still referenced by the
  on-disk checkpoint image). Tracked in `uncheckpointedFreePageRanges`.
- `popFreePages(n)` — best-fit pop; splits the range if larger than requested.
- `mergeFreePages(fileHandle)` — coalesces adjacent uncheckpointed ranges so the
  file tail can be truncated.
- `finalizeCheckpoint(fileHandle)` — promotes uncheckpointed ranges into the main
  free lists; called from `PageManager::finalizeCheckpoint`.
- `rollbackCheckpoint()` — discards uncheckpointed ranges (abort).
- `clearEvictedBufferManagerEntriesIfNeeded` — de-dupes eviction-queue entries for
  repeatedly freed/reused pages.

### 6.4 Versioning

`PageManager::version` is a monotonic atomic bumped on every allocation/free.
`changedSinceLastCheckpoint()` compares it against `lastCheckpointVersion`; the
checkpointer uses this to skip work for idle storage managers and to decide
whether to bump the on-disk metadata.

---

## 7. Disk Arrays

`DiskArray<T>` (`src/include/storage/disk_array.h`) is the workhorse for
append-only, page-resident arrays used by metadata structures and indexes. It is
a three-level indirection:

```
DiskArrayHeader (stable, pre-allocated page)
      │ firstPIPPageIdx
      ▼
PIP (Page Indices Page)  ──nextPipPageIdx──►  PIP ──► ...
      │ pageIdxs[]
      ▼
Array Pages (actual elements)
```

- **Header** (`DiskArrayHeader`): `{ numElements; firstPIPPageIdx; _padding }`.
  Lives at a stable, pre-allocated page so its location never moves.
- **PIP** (`Page Index Page`): a full `LBUG_PAGE_SIZE` page containing
  `NUM_PAGE_IDXS_PER_PIP = (PAGE_SIZE - sizeof(page_idx_t)) / sizeof(page_idx_t)`
  page indices, plus a `nextPipPageIdx` link (or `NULL_PAGE_IDX`).
- **Array Pages**: hold `numElementsPerPage = PAGE_SIZE / alignedElementSize`
  elements each.

### Read/write transaction isolation

Each `DiskArray` carries **two headers**: `headerForReadTrx` (committed/on-disk
view) and `headerForWriteTrx` (in-flight updates). `getNumElements(trxType)`
returns the appropriate count. Writes go through shadow pages (unless
`bypassShadowing` is set, e.g. for hash-index slot arrays) so the on-disk image is
not mutated until checkpoint. `checkpointInMemoryIfNecessary()` /
`rollbackInMemoryIfNecessary()` promote or discard the write header.

### `DiskArrayCollection`

`DiskArrayCollection` (`disk_array_collection.h`) manages many `DiskArray`s in
one file via a chain of `HeaderPage`s. Each `HeaderPage` holds
`NUM_HEADERS_PER_PAGE = (PAGE_SIZE - sizeof(page_idx_t) - sizeof(uint32_t)) /
sizeof(DiskArrayHeader)` headers plus a `nextHeaderPage` link and `numHeaders`.
`getDiskArray<T>(idx)` returns a typed view bound to the read/write header pair.

### `WriteIterator`

Bulk sequential writes use `DiskArrayInternal::WriteIterator`, which caches the
pinned (shadowed) array page and only re-seeks when crossing a page boundary.
`pushBack` grows the array by allocating new array pages via the `PageAllocator`.

---

## 8. Tables and Node Groups

Tables (`src/storage/table/`) store user data in a **columnar, node-grouped**
layout. Two top-level table kinds:

- **`NodeTable`** — nodes keyed by internal node ID (`offset_t`). Rows addressed
  by `(nodeGroupIdx, rowIdxInGroup)`.
- **`RelTable`** — relationships, stored directionally in `RelTableData` for each
  direction (`FWD`/`BWD`/`BOTH`), using CSR for the adjacency.

Variants exist for non-native sources (`ArrowNodeTable`, `ArrowRelTable`,
`IceDiskNodeTable`, `IceDiskRelTable`, `ForeignRelTable`) that scan from Arrow /
Parquet / foreign sources rather than the local data file; those are read paths
and not part of the durable format.

### 8.1 Node groups and chunked node groups

- **`NodeGroup`** — a horizontal slice of a table covering `NODE_GROUP_SIZE` rows.
  Has a `residency_state` (in-memory / on-disk / spilled) and owns version info.
- **`ChunkedNodeGroup`** — the persistent unit; holds one `ColumnChunk` per column
  over `CHUNKED_NODE_GROUP_CAPACITY` rows. `NodeGroupDataFormat` is `REGULAR` or
  `CSR`.
- **`NodeGroupCollection`** — a table's ordered set of node groups; scans iterate
  `nodeGroupIdx`.

### 8.2 Columns and column chunks

`Column` (`column.h`) is the persistent column reader/writer. A column is a
sequence of **segments** (≤ `MAX_SEGMENT_SIZE` bytes, == `LBUG_PAGE_SIZE`),
each described by `ColumnChunkMetadata` and read via `ColumnChunkData`.
Specialized columns:

- `StringColumn` — dictionary + overflow pages (`string_t` offsets + inline data).
- `ListColumn` / `StructColumn` — nested columns with child columns.
- `NullColumn` — separate null-bitmask column (one bit per value).
- `DictionaryColumn` — dictionary-encoded column.

Each `ColumnChunk` is flushed through `Column::flushChunkData`, which applies the
selected `CompressionType` (see §10) and writes compressed segments to pages
allocated by the `PageAllocator`, returning `ColumnChunkMetadata` (page range,
compression metadata, min/max stats).

### 8.3 CSR for relationships

`RelTableData` stores each direction as a **CSR (Compressed Sparse Row)**
structure with `CSRHeaderColumns { offset, length }`. CSR is organized into
leaf regions of `CSR_LEAF_REGION_SIZE` and supports two densities:
`PACKED_CSR_DENSITY` (0.8) for compact regions and `LEAF_HIGH_CSR_DENSITY` (1.0)
for dense leaf regions. `CSRNodeGroup` and `CSRChunkedNodeGroup` implement the
CSR node-group format; inserts/updates go through `VersionRecordHandler`
(`PersistentVersionRecordHandler` / `InMemoryVersionRecordHandler`).

### 8.4 Versioning (MVCC) within a node group

`VersionInfo` + `UpdateInfo` + `VectorUpdateInfo` track in-place updates and
deletes per node group so readers see a transaction-consistent snapshot while
uncommitted/committed-but-uncheckpointed changes live in memory. The
`VersionRecordHandler` bridges these to the persistent chunked groups
(`commitInsert`, `commitDelete`, `rollbackInsert`). The `UndoBuffer` (see §13)
holds the rollback records.

---

## 9. Indexes

Indexes (`src/storage/index/`) provide primary-key and secondary lookups.

### 9.1 Hash index (primary key)

`PrimaryKeyIndex` (`hash_index.cpp`) is a disk-resident chained hash index.
- **256 sub-indexes** (`NUM_HASH_INDEXES = 1<<8`); the top 8 bits of the hash
  select the sub-index.
- Two header pages (`INDEX_HEADER_PAGES = 2`) hold all 256 `HashIndexHeaderOnDisk`
  entries (each <32 bytes).
- Each sub-index has a **primary slot array** and an **overflow slot array**,
  stored as `DiskArray`s in a `DiskArrayCollection` (`bypassShadowing = true`).
- Slot capacity: `SLOT_CAPACITY_BYTES = 256`; entries are `Slot<T>` with a
  `SlotHeader` + up to `PERSISTENT_SLOT_CAPACITY` key/offset pairs.
- `SlotType { PRIMARY, OVF }`; overflow chains via `nextOvfSlotId`.

`InMemHashIndex` is the in-memory builder used during `COPY`/bulk load; it is
flushed to the on-disk hash index at commit.

### 9.2 ART index (both primary and secondary)

`ARTIndex` (`art_index.cpp`, `art_index_disk.cpp`) is an [adaptive radix tree](https://db.in.tum.de/~leis/papers/ART.pdf) for
indexes. Nodes are written to the data file in `LBUG_PAGE_SIZE`-aligned
chunks via `ARTIndexDiskUtils`; leaf/prefix layout follows the standard ART
design. `IndexStorageInfo` carries the ART's root page and layout.

### 9.3 Index plumbing

`IndexType` registers an index kind (`typeName`, constraint type, definition
type, load function). `IndexInfo`/`IndexStorageInfo` are serialized into the
metadata region. `IndexConstraintType { PRIMARY, SECONDARY_NON_UNIQUE }` and
`IndexDefinitionType { BUILTIN, EXTENSION }` classify indexes. Extensions can
register custom index types via `StorageManager::registerIndexType`.

---

## 10. Compression

`CompressionType` (`compression.h`):

| Value                  | Applicable types                  | Notes |
|------------------------|-----------------------------------|------|
| `UNCOMPRESSED` (0)     | all                               | Raw fixed-width values. |
| `INTEGER_BITPACKING` (1) | int8…int64, uint8…uint64, int128 | Bit-packed using min-width bit count. |
| `BOOLEAN_BITPACKING` (2) | bool                             | One bit per value. |
| `CONSTANT` (3)         | all                               | Single repeated value; stores min==max. |
| `ALP` (4)              | float, double                    | Adaptive Lossless floating-Point compression. |

`CompressionMetadata` (`{ min, max, compression }`, plus `ExtraMetadata` for
ALP) is computed per segment at flush time. `CompressionMetadata::numValues()`
returns how many values of a given physical type fit in a page under the chosen
compression — used by `ColumnChunkData` to size segments.

Compression is selected per segment: if all values equal → `CONSTANT`; booleans →
`BOOLEAN_BITPACKING`; integers → `INTEGER_BITPACKING` (with INT128 fast path);
floats/doubles → `ALP` when beneficial, else `UNCOMPRESSED`. Compression can be
disabled globally (`StorageManager::compressionEnabled()`).

---

## 11. Write-Ahead Log

The WAL (`src/storage/wal/`) durably records committed changes so the database
can be recovered after a crash. It is append-only, optionally checksummed
(`ChecksumWriter`/`ChecksumReader`), and prefixed with a `WALHeader`
`{ databaseID, enableChecksums }`.

### 11.1 Record types (`WALRecordType`)

| Type | Value | Meaning |
|------|-------|---------|
| `BEGIN_TRANSACTION_RECORD` | 1 | Start of a transaction's WAL entries. |
| `COMMIT_RECORD`            | 2 | Transaction commit marker. |
| `COPY_TABLE_RECORD`        | 13 | Bulk-loaded a table. |
| `CREATE_CATALOG_ENTRY_RECORD` | 14 | New catalog entry (table/index/…). |
| `CREATE_INDEX_RECORD`      | 15 | New index. |
| `DROP_CATALOG_ENTRY_RECORD` | 16 | Dropped catalog entry. |
| `ALTER_TABLE_ENTRY_RECORD` | 17 | Table schema change. |
| `UPDATE_SEQUENCE_RECORD`   | 18 | Sequence value update. |
| `TABLE_INSERTION_RECORD`   | 30 | Row insertions (node or rel). |
| `NODE_DELETION_RECORD`     | 31 | Node deletions. |
| `NODE_UPDATE_RECORD`       | 32 | Node updates. |
| `REL_DELETION_RECORD`      | 33 | Rel deletions. |
| `REL_DETACH_DELETE_RECORD` | 34 | DETACH DELETE of rels. |
| `REL_UPDATE_RECORD`        | 35 | Rel updates. |
| `LOAD_EXTENSION_RECORD`    | 100 | Extension load. |
| `CHECKPOINT_RECORD`        | 254 | Marks end of replayable log at a checkpoint. |

`INVALID_RECORD` (0) is reserved to detect reads from empty buffers.

### 11.2 Record generation

WAL record C++ is **generated from TypeSpec** (`wal/typespec/records/*.tsp`) via
`scripts/generate_wal_typespec.py`. Normal builds compile the checked-in
`.h`/`.cpp`; regeneration is opt-in via the `generate_wal_typespec` CMake target.
Each record has a write-side (`*_record.cpp`) and a replay-side
(`*_record_replay.cpp`) implementation. TypeSpec metadata comments preserve wire
compatibility (`@defaults`, `@wire_names`, `@owned_names`, `@debug_fields`).

### 11.3 LocalWAL and group commit

`LocalWAL` buffers per-transaction records in memory. On commit,
`WAL::logCommittedWAL` appends them to the shared WAL and assigns a
`commitSequence`. Durability is gated by a `groupCommitCV`; `durableCommitSequence`
tracks fsync progress. `WAL::logAndFlushCheckpoint` writes a `CHECKPOINT_RECORD`
and flushes.

### 11.4 WAL rotation

During checkpoint the live WAL is **rotated**: it's frozen as
`<dbpath>.wal.checkpoint` and a new empty WAL is opened (`rotateForCheckpoint`).
After the checkpoint lands, `clearFrozenWAL()` removes the frozen file. If the
WAL was *not* rotated (no concurrent writers), `WAL::clear()` truncates it to 0.
A WAL can be **poisoned** (`poisonNoLock`) on an unrecoverable error;
`throwIfPoisoned` is checked on access.

---

## 12. Checkpointer

`Checkpointer` (`src/storage/checkpointer.cpp`) atomically persists a
transaction-consistent snapshot. Phases (driven by `writeCheckpoint`):

1. **`beginCheckpoint(snapshotTS)`** — acquire the write gate, capture the
   snapshot timestamp, copy the current `DatabaseHeader` into `checkpointHeader`,
   and capture per-table change-epoch watermarks under the lock.
2. **`checkpointStoragePhase()` / `checkpointStorage()`** — for each
   `CheckpointTarget { catalog, storageManager }`, serialize tables/indexes that
   changed since the last checkpoint into new pages; record `hasStorageChanges`.
   Safe to run after the write gate is released when WAL rotation occurred,
   because node-data reads are bounded by the frozen WAL + `snapshotTS`.
3. **`serializeCatalogAndMetadata(header, hasStorageChanges)`** — serialize the
   catalog and storage metadata into fresh page ranges; update
   `header.catalogPageRange` / `header.metadataPageRange`.
4. **`writeDatabaseHeader(header)`** — persist the new header to page 0.
5. **`logCheckpointAndApplyShadowPages(walRotated)`** — append a
   `CHECKPOINT_RECORD` to the WAL, apply shadow pages onto the data file
   (`ShadowFile::applyShadowPages`), then clear the WAL (if not rotated) and the
   shadow file.
6. **`finishCheckpoint()`** — runs after the write gate is released; finalizes
   the durable checkpoint. May encounter new writers with timestamps > snapshotTS.
7. **`postCheckpointCleanup(canResetPageManagerToCurrent)`** —
   `StorageManager::finalizeCheckpoint()` + `PageManager::finalizeCheckpoint()`
   (promotes uncheckpointed free pages, resets version watermarks), and clears BM
   eviction-queue dups. Exceptions here propagate (the DB is already durable).

`rollback()` / `rollbackCheckpoint()` undo an in-progress checkpoint: restore
the header, `PageManager::rollbackCheckpoint` → `FreeSpaceManager::rollbackCheckpoint`,
and discard shadow pages.

`canAutoCheckpoint` decides whether an automatic checkpoint may run for the
given transaction. Multiple attached databases are checkpointed together via
`collectCheckpointTargets()`.

---

## 13. Recovery and Transaction Interaction

### 13.1 Recovery (`StorageManager::recover`)

On startup, `WALReplayer::replay` runs in two passes:

1. **Frozen WAL** (`replayFrozenWAL`) — the `<dbpath>.wal.checkpoint` file, if
   present, is replayed first. It begins by `ShadowFile::replayShadowPageRecords`
   to redo any shadow-page applies that didn't land before the crash, then
   replays records until it hits a `CHECKPOINT_RECORD` (stop marker).
2. **Active WAL** (`replayActiveWAL`) — `<dbpath>.wal` is replayed the same way
   (shadow replay first, then records), skipping the `databaseID` header and
   stopping at a `CHECKPOINT_RECORD`.

`replayWALRecord` dispatches each record type to its `*_replay.cpp` handler.
`throwOnWalReplayFailure` controls whether corruption raises or is best-effort.
The `databaseID` in the WAL/shadow headers is verified against the data file's
UUID before replay so mismatched sidecars are rejected.

### 13.2 Per-transaction local storage (MVCC, out of scope)

`LocalStorage` / `LocalNodeTable` / `LocalRelTable` (`local_storage/`) hold
**uncommitted** inserts/updates/deletes for a transaction. Row offsets ≥
`MAX_NUM_ROWS_IN_TABLE` are interpreted as local (transaction-private) rows. On
commit these are merged into the persistent tables and recorded in the WAL; on
rollback they're discarded. This is the MVCC write buffer, **not** the durable
file format described in this spec.

### 13.3 Undo buffer

`UndoBuffer` (`undo_buffer.h`) holds in-memory rollback records per transaction:
`CATALOG_ENTRY`, `SEQUENCE_ENTRY`, `UPDATE_INFO`, `INSERT_INFO`, `DELETE_INFO`.
`UndoBuffer::commitRecord` applies the commit timestamp to version info and
advances catalog entries; rollback walks the buffer in reverse. Undo records are
in-memory only — durability is the WAL's job.

---

## 14. Reference: Source Layout

```
src/storage/
  storage_manager.cpp        # top-level owner; recover, checkpoint, table/index registry
  checkpointer.cpp           # checkpoint phases
  database_header.cpp        # page-0 header serialize/validate
  file_handle.cpp            # per-file page abstraction + page manager hook
  page_manager.cpp           # data-file allocator + versioning
  free_space_manager.cpp     # leveled free lists
  optimistic_allocator.cpp   # lock-free allocation helper
  overflow_file.cpp          # string/list overflow pages
  shadow_file.cpp            # copy-on-write shadow pages
  shadow_utils.cpp           # shadow-version pin helpers
  disk_array.cpp             # append-only page-resident arrays
  disk_array_collection.cpp  # many DiskArrays via header-page chain
  undo_buffer.cpp            # in-memory MVCC rollback records
  storage_utils.cpp          # page/offset utilities
  storage_version_info.cpp   # on-disk version compatibility
  file_db_id_utils.cpp       # UUID verification for sidecar files
  buffer_manager/            # page cache, eviction, spiller, VM regions
  compression/               # constant/bitpacking/ALP compression
  index/                     # hash index (PK) + ART index (secondary)
  local_storage/             # per-transaction MVCC buffers (out of scope)
  predicate/                 # scan predicates (null/constant/column)
  stats/                     # table/column/planner statistics
  table/                     # node/rel tables, node groups, columns, CSR
  wal/                       # WAL, checksums, replayer, TypeSpec records
```

Header mirrors live under `src/include/storage/` with the same structure.
