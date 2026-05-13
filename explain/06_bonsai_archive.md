# Bonsai Archive: Full Historic State Without the Forest

> How Besu adds a new key scheme and two new DB segments to give Bonsai unlimited
> historic state access — while keeping block-import performance intact.

---

## Table of Contents

1. [The Problem Bonsai Archive Solves](#1-the-problem-bonsai-archive-solves)
2. [The Core Idea: Block Number in the Key](#2-the-core-idea-block-number-in-the-key)
3. [The Four New DB Segments](#3-the-four-new-db-segments)
4. [How Keys Are Structured in the Archive](#4-how-keys-are-structured-in-the-archive)
5. [Finding the Nearest State Entry (getNearestBefore)](#5-finding-the-nearest-state-entry-getnearestbefore)
6. [How a Write Works in Archive Mode](#6-how-a-write-works-in-archive-mode)
7. [How a Read Works in Archive Mode](#7-how-a-read-works-in-archive-mode)
8. [The BonsaiArchiver: Moving Old State Asynchronously](#8-the-bonsaiarchiver-moving-old-state-asynchronously)
9. [Handling Deleted Accounts and Storage Slots](#9-handling-deleted-accounts-and-storage-slots)
10. [The BonsaiArchiveWorldStateProvider](#10-the-bonsaiarchiveworldstateprovider)
11. [Configuration and Data Storage Format](#11-configuration-and-data-storage-format)
12. [The LATEST_ARCHIVED_BLOCK Pointer](#12-the-latest_archived_block-pointer)
13. [Performance Characteristics](#13-performance-characteristics)
14. [Comparison: Forest vs Bonsai vs Bonsai Archive](#14-comparison-forest-vs-bonsai-vs-bonsai-archive)
15. [Limitations and Trade-offs](#15-limitations-and-trade-offs)
16. [Summary](#16-summary)

---

## 1. The Problem Bonsai Archive Solves

As you learned in the previous files, standard Bonsai keeps only the **current** world
state in its flat database:

```
  ACCOUNT_INFO_STATE[ keccak(address) ]  =  { current balance, nonce, ... }
```

One entry per account. Always up to date. Extremely fast to read — but completely blind
to history.

To serve historic state (e.g., `eth_getBalance(Alice, block=1000000)`), standard Bonsai
replays TrieLogs in reverse. But TrieLogs are pruned after ~512 blocks. So for any block
older than ~512 blocks from the current head, standard Bonsai simply cannot answer:

```
  Standard Bonsai node at block 20,000,000:

  eth_getBalance(Alice, "0xF4240")   ← block 1,000,000
      Error: historic state not available
      (19,999,488 blocks beyond the TrieLog window)
```

Before Bonsai Archive, the only solution was to run a **Forest** node, which keeps every
MPT node from every block — but that costs 1.5 TB+ of disk and is slow for current-state
reads.

Bonsai Archive is a third option: keep Bonsai's efficient current-state performance
**and** serve historic state for any block, by extending the flat DB with a smarter
key scheme.

---

## 2. The Core Idea: Block Number in the Key

The root insight is simple but powerful:

> **Append the block number to the key.**

In standard Bonsai, the flat account DB has one entry per account:

```
  Key:   keccak(address)               ← 32 bytes
  Value: RLP{ nonce, balance, ... }    ← current state
```

In Bonsai Archive, the key is extended with an 8-byte block number suffix:

```
  Key:   keccak(address) ++ block_number   ← 32 + 8 = 40 bytes
  Value: RLP{ nonce, balance, ... }        ← state at that block
```

Now multiple entries can exist for the same account — one for each block at which that
account's state **changed**:

```
  Archive key space for Alice's account (keccak = 0x4D95...):

  0x4D95...0000000000000001  →  { balance: 0 ETH }     ← Block 1 (genesis)
  0x4D95...000000000000000A  →  { balance: 5 ETH }     ← Block 10
  0x4D95...000000000000001F  →  { balance: 3 ETH }     ← Block 31
  0x4D95...0000000000013A2B  →  { balance: 7 ETH }     ← Block 80,427
```

RocksDB stores all entries in **lexicographic (sorted) order**. Because the address hash
is the leading prefix, all entries for a given account cluster together. Within that
cluster they are sorted by block number.

This layout enables the crucial `getNearestBefore` lookup described in section 5.

---

## 3. The Four New DB Segments

Bonsai Archive introduces four new RocksDB column families beyond the five standard Bonsai
segments. They are defined in `KeyValueSegmentIdentifier.java`:

```
  New segment                   Key format                       Role
  ─────────────────────────────────────────────────────────────────────────────────
  ACCOUNT_INFO_STATE_ARCHIVE    keccak(address) ++ block_num     Historic account state
  ACCOUNT_STORAGE_ARCHIVE       keccak(addr) ++ keccak(slot)     Historic storage slots
                                ++ block_num
  ACCOUNT_INFO_STATE_FREEZER    keccak(address) ++ block_num     Older historic accounts
  ACCOUNT_STORAGE_FREEZER       keccak(addr) ++ keccak(slot)     Older historic storage
                                ++ block_num
```

### Why Two Pairs? (Archive vs Freezer)

You might wonder: why have both `ACCOUNT_INFO_STATE_ARCHIVE` and
`ACCOUNT_INFO_STATE_FREEZER`? The naming reflects two layers of "depth":

- **ARCHIVE segments** — hold recently archived historic state (state that has been moved
  out of the primary flat DB segment but is still somewhat recent).
- **FREEZER segments** — hold the older tier of historic state (state that is very old
  and has been further migrated from the archive tier).

This two-tier approach follows the same pattern used in some database systems (hot/warm/cold
storage tiers). New writes go to the primary segments. After a configurable age they move
to the archive segments. Even older state may move to the freezer segments. Reads check
the primary segment first, then the archive, then the freezer.

### Key ID format: Full Name Instead of Byte

Notice that the four new segments use their **full string name** as the column family
identifier (e.g., `"ACCOUNT_INFO_STATE_ARCHIVE"` as bytes) rather than a single byte
(like `0x06` for `ACCOUNT_INFO_STATE`).

This is a deliberate improvement for operational debugging. When you use `ldb` (the
RocksDB command-line tool) to inspect the database, you can refer to the column by its
readable name:

```
  # Standard Bonsai segment (old style — must know byte mapping):
  ldb --db=. --column_family=$'\x06' scan

  # Bonsai Archive segment (new style — human readable):
  ldb --db=. --column_family=ACCOUNT_INFO_STATE_ARCHIVE scan

  # Query a specific account at a specific block:
  ldb --db=. get --key_hex --value_hex \
      --column_family=ACCOUNT_INFO_STATE_ARCHIVE \
      0x3e26253c5a8b02bb30c92798fa8083f0c966a954b019cad73aa12cdf3b344a7e00000000000113a1
```

The wiki's design document even includes an example of exactly this `ldb` query returning
`0xF84D0389...` — the RLP-encoded account state at block 0x113a1 (70,561).

---

## 4. How Keys Are Structured in the Archive

### Account Keys

```
  Standard Bonsai key:
  ┌────────────────────────────────────┐
  │  keccak256(address)    (32 bytes)  │
  └────────────────────────────────────┘

  Bonsai Archive key:
  ┌────────────────────────────────────┬──────────────────────┐
  │  keccak256(address)    (32 bytes)  │  block_number (8 bytes, big-endian uint64) │
  └────────────────────────────────────┴──────────────────────┘
  Total: 40 bytes
```

The block number is encoded as an **unsigned 64-bit big-endian** integer. Big-endian
encoding is essential: it makes lexicographic order (the order RocksDB uses) identical
to numerical order. If you stored block numbers in little-endian, the sorted order would
be 0x0100000000000000 (block 1) AFTER 0x0200000000000000 (block 2) — completely wrong.

Big-endian ensures:

```
  block 1   →  0x0000000000000001  ← smaller lexicographically
  block 10  →  0x000000000000000A
  block 31  →  0x000000000000001F
  block 256 →  0x0000000000000100  ← larger lexicographically
```

Sorted order = chronological order. This is what makes range scans and `getNearestBefore`
work correctly.

### Storage Keys

```
  Standard Bonsai storage key:
  ┌────────────────────────────────────┬────────────────────────────────────┐
  │  keccak256(address)    (32 bytes)  │  keccak256(slot_index)  (32 bytes) │
  └────────────────────────────────────┴────────────────────────────────────┘
  Total: 64 bytes

  Bonsai Archive storage key:
  ┌────────────────────────────────────┬────────────────────────────────────┬──────────────────────┐
  │  keccak256(address)    (32 bytes)  │  keccak256(slot_index)  (32 bytes) │  block_number (8 bytes)│
  └────────────────────────────────────┴────────────────────────────────────┴──────────────────────┘
  Total: 72 bytes
```

### Special Suffix Constants

`BonsaiArchiveFlatDbStrategy` defines two boundary suffixes used for range scans:

```java
  static final byte[] MAX_BLOCK_SUFFIX = Bytes.ofUnsignedLong(Long.MAX_VALUE).toArrayUnsafe();
  // = 0x7FFFFFFFFFFFFFFF  (the largest positive long in big-endian)

  static final byte[] MIN_BLOCK_SUFFIX = Bytes.ofUnsignedLong(0L).toArrayUnsafe();
  // = 0x0000000000000000  (block zero)
```

`MAX_BLOCK_SUFFIX` is used in `getNearestBefore` queries when you want to find the most
recent entry at or before a given block: you construct a query key with `MAX_BLOCK_SUFFIX`
and then ask for the nearest key that is ≤ that query key. This finds the latest version
of the account at or before the requested block. (More detail in section 5.)

---

## 5. Finding the Nearest State Entry (getNearestBefore)

This is the most important algorithmic concept in Bonsai Archive.

### The Problem

Accounts are only written to the archive DB on blocks where they **changed**. Most
blocks, most accounts don't change. So if you want Alice's balance at block 5,000 and
she last changed at block 4,993, there is no entry with key `keccak(Alice)++5000`. There
is an entry with key `keccak(Alice)++4993`.

How do you find it efficiently without scanning thousands of entries?

### The Solution: Lexicographic Nearest-Before Search

Because RocksDB stores entries in sorted key order, and because big-endian block numbers
make sorted order = chronological order, you can perform a **seek-to-nearest** operation:

```
  Query: "Alice's state at block 5,000"

  Construct query key:   keccak(Alice) ++ 0x0000000000001388   (5000 in hex)
  Call:                  storage.getNearestBefore(ACCOUNT_INFO_STATE_ARCHIVE, queryKey)

  RocksDB seeks to the largest key ≤ queryKey in the sorted key space:
  ─────────────────────────────────────────────────────────────────────
  keccak(Alice)++0x0000000000001379  →  { balance: 3 ETH }   ← block 4,985
  keccak(Alice)++0x0000000000001389  →  { balance: 5 ETH }   ← block 4,993  ← found!
  keccak(Alice)++0x0000000000001B58  →  { balance: 7 ETH }   ← block 7,000
  keccak(Alice)++0x00000000000030D4  →  { balance: 2 ETH }   ← block 12,500
  ─────────────────────────────────────────────────────────────────────
  Result: largest key ≤ keccak(Alice)++5000 is keccak(Alice)++4993
  Return: { balance: 5 ETH }  ✓
```

One disk seek. No scanning. O(log N) in the total number of archive entries (RocksDB's
B-tree index), but effectively O(1) in practice for a well-cached database.

### Verifying the Address Prefix

After `getNearestBefore` returns a result, the code must verify that the found key
**actually belongs to Alice** — not a completely different account that happens to be
lexicographically just before Alice's block 5,000 key.

```java
  Optional<SegmentedKeyValueStorage.NearestKeyValue> nearestAccount =
      storage
          .getNearestBefore(ACCOUNT_INFO_STATE_ARCHIVE, keyNearest)
          .filter(
              found ->
                  accountHash.getBytes().commonPrefixLength(found.key())
                      >= accountHash.getBytes().size());
```

The `.filter()` checks that the found key shares at least 32 bytes of common prefix with
`accountHash`. In other words: "the first 32 bytes of the found key must match Alice's
address hash exactly." If they don't match, the result is discarded (the account never
existed in the archive for that block range).

### getNearestAfter

The archive also supports `getNearestAfter` — finding the closest entry at or after a
given key. This is used in streaming/enumeration scenarios where you need to find the
next version of a piece of state after a given block.

---

## 6. How a Write Works in Archive Mode

When a new block is imported, `BonsaiArchiveFlatDbStrategy` handles writes differently
from standard Bonsai:

### Step 1: Determine the Block Number Context

Before writing, the strategy reads the current block number from `TRIE_BRANCH_STORAGE`:

```java
  private Optional<BonsaiContext> getStateArchiveContextForWrite(
          final SegmentedKeyValueStorage storage) {
      Optional<byte[]> archiveContext =
          storage.get(TRIE_BRANCH_STORAGE, WORLD_BLOCK_NUMBER_KEY);
      if (archiveContext.isPresent()) {
          // The current world state is at block N.
          // The block being imported is block N+1.
          // So we write archive entries with suffix = N+1.
          return Optional.of(
              new BonsaiContext(Bytes.wrap(archiveContext.get()).toLong() + 1));
      } else {
          // No block number recorded = genesis. Use suffix 0.
          return Optional.of(new BonsaiContext(0L));
      }
  }
```

`BonsaiContext` is a tiny wrapper that carries the block number to use as a key suffix.

### Step 2: Write the New State with the Block Number Suffix

Instead of writing:

```
  ACCOUNT_INFO_STATE[ keccak(Alice) ] = newValue    ← overwrites old value
```

Archive mode writes:

```
  ACCOUNT_INFO_STATE_ARCHIVE[ keccak(Alice) ++ blockNumber ] = newValue
```

The old value for Alice remains in the archive under its own block number key —
nothing is overwritten. The history is preserved.

```
  Block 31: Alice's balance changes from 5 ETH to 3 ETH

  Written to archive:
  ACCOUNT_INFO_STATE_ARCHIVE[ keccak(Alice) ++ 0x000000000000001F ] = { bal: 3 ETH }

  Previous entry (from block 10) remains untouched:
  ACCOUNT_INFO_STATE_ARCHIVE[ keccak(Alice) ++ 0x000000000000000A ] = { bal: 5 ETH }
```

### Step 3: Primary Segment Still Gets Updated

The primary `ACCOUNT_INFO_STATE` segment is also updated as in standard Bonsai —
this ensures that current-state reads (which don't need historic lookup) remain fast:

```
  ACCOUNT_INFO_STATE[ keccak(Alice) ] = { bal: 3 ETH }   ← current state (fast reads)
  ACCOUNT_INFO_STATE_ARCHIVE[ keccak(Alice) ++ 31 ] = { bal: 3 ETH }  ← archive copy
```

Wait — this means the current state is stored **twice**: once in the primary segment
(current-only, single entry per account) and once in the archive (with block suffix).
Yes, this is by design. The primary segment is for fast O(1) current-state reads; the
archive is for historic reads. They serve different access patterns.

---

## 7. How a Read Works in Archive Mode

Reading in archive mode goes through a layered lookup strategy. The implementation is in
`BonsaiArchiveFlatDbStrategy.getFlatAccount()`:

```
  getFlatAccount(accountHash, storage):

  Step 1: Determine the block context for reading
          Read WORLD_BLOCK_NUMBER_KEY from TRIE_BRANCH_STORAGE
          → blockNumber = the block number the world state is currently set to

  Step 2: Construct the "nearest before" query key
          queryKey = keccak(address) ++ blockNumber
          (if no block context, use MAX_BLOCK_SUFFIX to get the latest)

  Step 3: Search ACCOUNT_INFO_STATE_ARCHIVE
          result = storage.getNearestBefore(ACCOUNT_INFO_STATE_ARCHIVE, queryKey)
          Verify the prefix matches (see section 5).

  Step 4: If not found in ARCHIVE, search ACCOUNT_INFO_STATE_FREEZER
          result = storage.getNearestBefore(ACCOUNT_INFO_STATE_FREEZER, queryKey)
          Verify the prefix matches.

  Step 5: If still not found → account does not exist at this block
          Return Optional.empty()

  Step 6: Filter out "deleted" sentinel values
          (An empty byte array value means the account was deleted — see section 9)
          If the found value is DELETED_ACCOUNT_VALUE → return Optional.empty()
          Otherwise → return the account bytes
```

Visualised:

```
  Read request: "Alice at block 5,000"

  ┌──────────────────────────────────────────────────────────────────────┐
  │  ACCOUNT_INFO_STATE_ARCHIVE                                          │
  │                                                                      │
  │  getNearestBefore( keccak(Alice) ++ 5000 )                           │
  │           │                                                          │
  │           ├── Found keccak(Alice)++4993 → { bal: 5 ETH }   ✓ return │
  │           └── Not found / wrong prefix → go to Freezer              │
  └──────────────────────────────────────────────────────────────────────┘
           │
           ▼ (if not found)
  ┌──────────────────────────────────────────────────────────────────────┐
  │  ACCOUNT_INFO_STATE_FREEZER                                          │
  │                                                                      │
  │  getNearestBefore( keccak(Alice) ++ 5000 )                           │
  │           │                                                          │
  │           ├── Found →   ✓ return (increment archive counter metric)  │
  │           └── Not found → account did not exist at block 5000        │
  └──────────────────────────────────────────────────────────────────────┘
```

### Metrics

`BonsaiArchiveFlatDbStrategy` tracks two counters:

```java
  protected final Counter getAccountFromArchiveCounter;
  protected final Counter getStorageFromArchiveCounter;
```

These count how often account/storage reads had to fall through to the freezer tier.
If these counters are high, it means the archive tier is not serving enough data and
more state may need to be in the ARCHIVE segment rather than FREEZER.

---

## 8. The BonsaiArchiver: Moving Old State Asynchronously

The `BonsaiArchiver` class is the background worker that moves account and storage state
from the **primary flat DB segments** to the **archive segments** as blocks grow old.

### Why Move at All?

The primary segments (`ACCOUNT_INFO_STATE`, `ACCOUNT_STORAGE_STORAGE`) in standard Bonsai
hold only the **current** value per account. In archive mode, new block imports write new
values to both the primary segments AND the archive segments.

Over time, the primary segments would still hold a single (current) value per account —
that is fine. But we also want the **previous** (superseded) values to be accessible in
the archive. The `BonsaiArchiver` is what moves those superseded values.

### The Archiver Flow

```
  New block N is added to the chain
       │
       ▼
  BonsaiArchiver.onBlockAdded() is called
       │
       ▼
  executeAsync() schedules the work on a background thread
       │
       ▼
  archiveMutex.tryLock()  ← ensures only one archiver run at a time
       │
       ▼
  moveBlockStateToArchive()
       │
       ├─ Calculate: retainAboveThisBlock = chainHead - 10
       │             (don't archive the 10 most recent blocks yet)
       │
       ├─ Build list of blocks to archive:
       │     from latestArchivedBlock+1 to retainAboveThisBlock
       │     (up to 1000 blocks per run — CATCHUP_LIMIT)
       │
       └─ For each block to archive:
              │
              ├─ Load the TrieLog for that block
              │     trieLogManager.getTrieLogLayer(blockHash)
              │
              ├─ For each account in the TrieLog's account changes:
              │     rootWorldStateStorage.archivePreviousAccountState(
              │         parentBlockHeader,
              │         address.addressHash()
              │     )
              │     ← moves the PREVIOUS state for this account
              │       (the state before this block changed it)
              │       from ACCOUNT_INFO_STATE to ACCOUNT_INFO_STATE_ARCHIVE
              │
              ├─ For each storage slot in the TrieLog's storage changes:
              │     rootWorldStateStorage.archivePreviousStorageState(
              │         parentBlockHeader,
              │         concatenate(address.addressHash(), slotKey.getSlotHash())
              │     )
              │
              └─ Update latestArchivedBlock = this block's number
                 Persist to DB: TRIE_LOG_STORAGE[ LATEST_ARCHIVED_BLOCK_KEY ]
```

### The CATCHUP_LIMIT

```java
  private static final int CATCHUP_LIMIT = 1000;
```

If the node starts up after being offline for a long time, it may be thousands of blocks
behind on archiving. The CATCHUP_LIMIT caps each archiver run at 1,000 blocks, ensuring
that archiving does not hog all I/O. Multiple runs will catch up over time.

Progress is logged every 100 archived blocks:

```
  INFO archive progress: state up to block 18500000 archived (250 behind chain head 18500250)
  INFO archive progress: state up to block 18500100 archived (150 behind chain head 18500250)
  INFO archive progress: state up to block 18500200 archived (50 behind chain head 18500250)
```

### The 10-Block Safety Buffer

```java
  private static final int DISTANCE_FROM_HEAD_BEFORE_ARCHIVING_OLD_STATE = 10;
  final long retainAboveThisBlock =
      blockchain.getChainHeadBlockNumber() - DISTANCE_FROM_HEAD_BEFORE_ARCHIVING_OLD_STATE;
```

The archiver never archives the 10 most recent blocks. This safety buffer accounts for:

1. **Short reorgs**: A reorg 1–2 blocks deep is common on Ethereum. If block N has already
   been archived (moved from primary to archive), and then a reorg replaces block N, the
   archiver would have moved state that needs to be undone. The 10-block buffer makes this
   very unlikely.
2. **Processing latency**: The archiver runs asynchronously. The primary segments must
   remain accurate for current-state reads while archiving is in progress.

### Metrics

The archiver exposes a gauge metric:

```java
  metricsSystem.createLongGauge(
      BesuMetricCategory.BLOCKCHAIN,
      "archived_blocks_state",
      "Total number of blocks for which state has been archived",
      () -> latestArchivedBlock.get());
```

This allows monitoring systems (Prometheus/Grafana) to track how far behind the archiver
is and alert if it falls significantly behind the chain head.

---

## 9. Handling Deleted Accounts and Storage Slots

What happens when an account is deleted (self-destruct) or a storage slot is set to zero?

In standard Bonsai, deletion is straightforward: `removeFlatAccount()` simply calls
`transaction.remove(ACCOUNT_INFO_STATE, key)`.

In Bonsai Archive, you cannot just delete the entry — future reads for an older block
might still need to know "this account existed at block X but was deleted at block Y."
Simply removing the entry would make it look like the account never existed.

The solution is a **sentinel value** ("tombstone"):

```java
  public static final byte[] DELETED_ACCOUNT_VALUE = new byte[0];
  public static final byte[] DELETED_STORAGE_VALUE = new byte[0];
```

When an account is deleted in archive mode, instead of removing the key, a new archive
entry is written with an **empty byte array** as the value:

```
  Block 500: Alice self-destructs

  Written to archive:
  ACCOUNT_INFO_STATE_ARCHIVE[ keccak(Alice) ++ 500 ] = []   ← empty = "deleted"

  Previous entries remain:
  ACCOUNT_INFO_STATE_ARCHIVE[ keccak(Alice) ++ 10  ] = { bal: 5 ETH }
  ACCOUNT_INFO_STATE_ARCHIVE[ keccak(Alice) ++ 31  ] = { bal: 3 ETH }
```

When a read for `keccak(Alice)` at any block ≥ 500 finds the empty value via
`getNearestBefore`, the code recognizes it as a deletion sentinel:

```java
  return accountFound
      .filter(
          found ->
              !Arrays.areEqual(
                  DELETED_ACCOUNT_VALUE, found.value().orElse(DELETED_ACCOUNT_VALUE)))
      .flatMap(SegmentedKeyValueStorage.NearestKeyValue::wrapBytes);
```

`Arrays.areEqual(DELETED_ACCOUNT_VALUE, ...)` — if the found value is an empty array
(the tombstone), the filter rejects it and the method returns `Optional.empty()`, meaning
"the account does not exist at this block." This correctly communicates that Alice's
account was deleted.

For a query at block 499 (before the deletion), `getNearestBefore` finds the block 31
entry instead (since block 31 < 499 < 500), and returns the real account data. ✓

---

## 10. The BonsaiArchiveWorldStateProvider

`BonsaiArchiveWorldStateProvider` extends `BonsaiWorldStateProvider` and overrides
`getWorldState()` to handle the two very different cases for historic state queries.

```java
  @Override
  public Optional<MutableWorldState> getWorldState(final WorldStateQueryParams queryParams) {
      if (queryParams.shouldWorldStateUpdateHead()) {
          return getFullWorldState(queryParams);   // Normal: importing new block
      } else {
          // Historic query path — two strategies depending on age:
          final BlockHeader chainHeadBlockHeader = blockchain.getChainHeadHeader();
          if (chainHeadBlockHeader.getNumber() - queryParams.getBlockHeader().getNumber()
                  >= trieLogManager.getMaxLayersToLoad()) {
              // Block is beyond the TrieLog window → use archive flat DB
              return cachedWorldStorageManager
                  .getWorldState(chainHeadBlockHeader.getHash())
                  .map(MutableWorldState::disableTrie)    // ← don't verify stateRoot
                  .flatMap(worldState ->
                      rollMutableArchiveStateToBlockHash(
                          (PathBasedWorldState) worldState,
                          queryParams.getBlockHeader().getBlockHash()))
                  .map(MutableWorldState::freezeStorage);
          }
          // Block is within the TrieLog window → standard Bonsai rollback
          return super.getWorldState(queryParams);
      }
  }
```

### Decision Tree

```
  getWorldState(queryParams):

  Is this query for the current head (new block import)?
      YES → getFullWorldState()  [standard path]
      NO  → historic query:

  Is the requested block within the TrieLog window (≤ 512 blocks old)?
      YES → super.getWorldState()  [standard Bonsai TrieLog rollback]
      NO  → archive path:
            1. Get the chain-head world state from cache
            2. disableTrie()           ← do NOT verify stateRoot (we can't, trie not present)
            3. rollMutableArchiveStateToBlockHash()  ← set block hash/number context
            4. freezeStorage()         ← make it read-only
```

### disableTrie() and Why It Matters

In standard Bonsai, after rolling back, the resulting world state verifies its root hash
against the block header's `stateRoot`. This catches corruption: if the flat DB is wrong,
the hash won't match the block header and an error is raised.

For archive queries, this verification is **not possible** because:
- Only one copy of the MPT trie exists (the current head state).
- Rolling the MPT back 19 million blocks is not feasible.
- The flat DB for historic blocks is correct by construction (it was written during block
  import and verified at that time).

`disableTrie()` tells the world state to skip trie operations entirely and rely solely on
the flat DB. This is safe for archive reads because:

1. Each entry in the archive was written at the time the block was processed.
2. The stateRoot was verified at import time against the MPT.
3. Therefore, the data in the archive is trustworthy even without re-verification.

### rollMutableArchiveStateToBlockHash()

This method performs a minimal "roll" for archive state — unlike the full TrieLog-based
rollback, it just updates the world state's internal block hash and number pointers:

```java
  protected Optional<MutableWorldState> rollMutableArchiveStateToBlockHash(
          final PathBasedWorldState mutableState, final Hash blockHash) {
      try {
          // Persist the block hash and number (and stateRoot) for this archive query.
          // This sets WORLD_BLOCK_HASH_KEY and WORLD_BLOCK_NUMBER_KEY so that
          // subsequent flat DB reads use the correct block number as the key suffix.
          mutableState.persist(blockchain.getBlockHeader(blockHash).get());
          return Optional.of(mutableState);
      } catch (final MerkleTrieException re) {
          throw re;   // re-throw to trigger state healing if needed
      } catch (final Exception e) {
          LOG.atInfo()...
          return Optional.empty();
      }
  }
```

The key insight: by calling `persist()` with the target block's header, Besu updates
`WORLD_BLOCK_NUMBER_KEY` to the target block number. All subsequent flat DB reads from
this world state will construct archive keys with that block number as the suffix — and
`getNearestBefore` will find the correct historic values. ✓

### freezeStorage()

After rolling, the world state is frozen — made read-only. This prevents any accidental
writes that could corrupt the current-state flat DB while a historic query is in progress.

---

## 11. Configuration and Data Storage Format

### Enabling Bonsai Archive

Bonsai Archive is a new data storage format, distinct from standard Bonsai:

```
  besu --data-storage-format=X_BONSAI_ARCHIVE [other flags]
```

The `X_` prefix indicates this is experimental. In `KeyValueSegmentIdentifier`, the format
is declared as `X_BONSAI_ARCHIVE` alongside `BONSAI` and `FOREST`:

```java
  ACCOUNT_INFO_STATE(
      new byte[] {6},
      EnumSet.of(BONSAI, X_BONSAI_ARCHIVE),   // ← used by both
      false, true, false),
  ...
  ACCOUNT_INFO_STATE_ARCHIVE(
      "ACCOUNT_INFO_STATE_ARCHIVE".getBytes(StandardCharsets.UTF_8),
      EnumSet.of(X_BONSAI_ARCHIVE),            // ← only for archive
      true, false, true),
```

This means `X_BONSAI_ARCHIVE` nodes use all five standard Bonsai segments **plus** the
four new archive/freezer segments — nine segments in total.

### Flat DB Mode Version

The flat DB mode is persisted as a version byte in `TRIE_BRANCH_STORAGE`. For archive mode:

```
  TRIE_BRANCH_STORAGE[ FLAT_DB_MODE_KEY ] = Bytes.of(0x02)   ← archive mode
```

When Besu starts, it reads this byte to automatically select the correct flat DB strategy:
`BonsaiArchiveFlatDbStrategy` for `0x02`, `BonsaiFullFlatDbStrategy` for `0x01`, etc.

### Starting from Scratch vs Migrating

**New node**: Start with `--data-storage-format=X_BONSAI_ARCHIVE` from genesis or snap sync.
The node will write archive entries from the very first block.

**Migrating from standard Bonsai**: Not directly supported. The archive segments would be
empty (no historic state was captured). You would need to resync from scratch.

**Migrating from Forest**: Not directly supported. Forest and Bonsai use fundamentally
different storage layouts.

---

## 12. The LATEST_ARCHIVED_BLOCK Pointer

The `BonsaiArchiver` needs to persist its progress — if the node restarts, it should
resume archiving from where it left off, not re-process thousands of already-archived blocks.

```
  ACCOUNT_INFO_STATE_ARCHIVE segment also stores one special metadata entry:

  Key:   LATEST_ARCHIVED_BLOCK_KEY  (a special well-known key)
  Value: block_number (big-endian uint64)
```

On startup, `BonsaiArchiver.initialize()` reads this value:

```java
  public void initialize() {
      latestArchivedBlock.set(rootWorldStateStorage.getLatestArchivedBlock().orElse(0L));
  }
```

After each block is archived, it updates this value:

```java
  rootWorldStateStorage.setLatestArchivedBlock(block.getKey());
  latestArchivedBlock.set(block.getKey());
```

This makes archiving **crash-safe**: if the node crashes during archiving, it will resume
from the last successfully archived block number on the next startup.

---

## 13. Performance Characteristics

### Block Import Performance

Block import in archive mode is slightly slower than standard Bonsai because:

1. Each block import writes archive entries in addition to the primary flat DB entries.
2. The archive segments are large (they grow indefinitely with history).
3. Large RocksDB column families can have higher write amplification due to compaction.

However, the design mitigates this:

- The archiving of **old** state (moving from primary to archive) is done **asynchronously**
  by `BonsaiArchiver`. The critical import path only writes new state.
- The 10-block safety buffer means the archiver never races with the importer on the same
  blocks.
- The archive segments are configured with `ordered = true` (sorted keys), enabling
  efficient sequential reads but accepting slightly higher write cost.

### Historic State Query Performance

For blocks within the TrieLog window (< 512 blocks old):
- Performance is identical to standard Bonsai: TrieLog rollback.

For blocks beyond the TrieLog window:
- Account read: **O(log N)** where N is total archive entries — effectively one seek.
- Contrast with Forest: **O(depth)** trie traversal (~8–12 random disk reads).
- Contrast with standard Bonsai: **❌ not available**.

Archive reads are thus **faster than Forest** for historic state because:
1. One seek vs 8–12 random reads.
2. The archive segments are column families; RocksDB caches hot entries in the block cache.
3. No RLP node decoding at each trie level — just decode the final account value.

### Disk Usage

```
  Approximate disk usage comparison (Ethereum mainnet, 2024):

  Format              Disk
  ─────────────────────────────────────────────────────────
  Forest              ~1.5 TB+ (all trie nodes, all history)
  Standard Bonsai     ~700 GB  (one trie + flat DB)
  Bonsai Archive      ~900 GB–1.2 TB  (estimated, grows with chain history)
```

Bonsai Archive sits between standard Bonsai and Forest. It is larger than standard Bonsai
because it stores all historic flat DB entries. It is smaller than Forest because:

- It stores account/storage state directly (not MPT nodes).
- MPT nodes have significant overhead (hashes at every level, encoding overhead).
- The flat DB entries are compact: 40 bytes key + ~100 bytes value per account version.
- Forest stores many redundant trie nodes (every account shares a path through the MPT,
  so each update rewrites many nodes, all of which are stored forever).

---

## 14. Comparison: Forest vs Bonsai vs Bonsai Archive

```
  ┌────────────────────────────────┬──────────────┬──────────────┬─────────────────┐
  │ Property                       │ Forest       │ Bonsai       │ Bonsai Archive  │
  ├────────────────────────────────┼──────────────┼──────────────┼─────────────────┤
  │ Disk usage (mainnet 2024)      │ ~1.5 TB+     │ ~700 GB      │ ~900 GB–1.2 TB  │
  │ Current state reads            │ Slow (8-12   │ Fast (1 DB   │ Fast (1 DB      │
  │                                │ trie hops)   │ lookup)      │ lookup)         │
  │ Historic reads (< 512 blocks)  │ ✅ Instant   │ ✅ TrieLog  │ ✅ TrieLog      │
  │ Historic reads (any block)     │ ✅ Instant   │ ❌ Error     │ ✅ Archive seek │
  │ Block import speed             │ Slow         │ Fast         │ Slightly slower │
  │                                │              │              │ than Bonsai     │
  │ State verification             │ ✅ Full trie │ ✅ Full trie │ ✅ Current head  │
  │                                │ at any block │ at head      │ only (archive   │
  │                                │              │              │ unverified)     │
  │ Data storage format            │ FOREST       │ BONSAI       │ X_BONSAI_ARCHIVE│
  │ Architecture complexity        │ Low          │ Medium       │ High            │
  │ Use case                       │ Archive      │ Validator /  │ Archive +       │
  │                                │ nodes (old)  │ RPC (current)│ RPC (all hist.) │
  └────────────────────────────────┴──────────────┴──────────────┴─────────────────┘
```

### When to Choose Each

| Need | Use |
|------|-----|
| Running a validator / staker | Bonsai |
| DApp backend (current state only) | Bonsai |
| DApp backend (some recent history) | Bonsai (tune TrieLog window) |
| Block explorer / analytics (full history) | Bonsai Archive |
| `eth_getBalance` at block 1,000,000 | Bonsai Archive or Forest |
| `eth_call` on an old block | Bonsai Archive or Forest |
| Smallest possible disk footprint | Bonsai |
| Best read performance for historic state | Bonsai Archive (faster than Forest) |

---

## 15. Limitations and Trade-offs

### 1. No Per-Transaction Historic State

Bonsai Archive stores state at the **block level** — it records the state of each account
after all transactions in that block have executed. It does not record intermediate states
mid-block (i.e., after transaction 3 but before transaction 4).

For `eth_getBalance` or `eth_getStorageAt` at a block boundary, this is fine. For `debug_traceTransaction` (tracing a single transaction within a block), the implementation
must re-execute the block up to that transaction. Bonsai Archive provides the starting
state for that replay but does not short-circuit it.

### 2. Archive State is Unverified

When serving historic state beyond the TrieLog window, the `BonsaiArchiveWorldStateProvider`
calls `disableTrie()` — skipping stateRoot verification. The data is correct by
construction (it was verified when the block was imported), but the client cannot
re-verify it on demand. A malicious or corrupted flat DB could serve incorrect historic
state without being caught. (Forest, by contrast, can always re-verify any historic
stateRoot because it retains all historic trie nodes.)

### 3. Only One Canonical State Per Block

The archive stores state for canonical blocks. If a block is later orphaned by a reorg,
its archive entries may persist (they are keyed by block number, not block hash). For
very short reorgs within the TrieLog window, the TrieLog rollback mechanism handles
canonical chain updates. For deeper reorgs (beyond the window), archive entries for
orphaned blocks would be unreachable but would still occupy disk space.

### 4. Cannot Migrate from Existing Bonsai or Forest

As mentioned in section 11, there is no migration path. You must resync from scratch to
use Bonsai Archive. For a node on Ethereum mainnet, this is a significant undertaking
(days of sync time with snap sync).

### 5. Experimental Status

The `X_` prefix on `X_BONSAI_ARCHIVE` explicitly marks this as experimental. The API,
configuration, and internal format may change in future versions of Besu without a
migration path. Production use should be evaluated carefully.

---

## 16. Summary

Bonsai Archive extends standard Bonsai with a simple but powerful idea:

> **Append the block number to flat DB keys to store one entry per block-change,
> not one entry per account.**

This allows historic state to be retrieved with a single `getNearestBefore` seek — as
fast as a current-state lookup — without requiring the bloated MPT node storage of
Forest mode.

### The full picture in one diagram

```
  ┌──────────────────────────────────────────────────────────────────────────────────────┐
  │                         Bonsai Archive on-disk layout                                │
  │                                                                                      │
  │  Standard Bonsai segments (current state):                                           │
  │  ┌──────────────────────────────────────────────────────────────────────────────┐    │
  │  │  ACCOUNT_INFO_STATE      [ keccak(addr) ]          = current account RLP    │    │
  │  │  ACCOUNT_STORAGE_STORAGE [ keccak(addr)++keccak(slot) ] = current slot RLP │    │
  │  │  CODE_STORAGE            [ key ]                   = bytecode               │    │
  │  │  TRIE_BRANCH_STORAGE     [ path ]                  = current MPT node       │    │
  │  │  TRIE_LOG_STORAGE        [ blockHash ]             = recent block diffs     │    │
  │  └──────────────────────────────────────────────────────────────────────────────┘    │
  │                                                                                      │
  │  Archive segments (all historic state):                                              │
  │  ┌──────────────────────────────────────────────────────────────────────────────┐    │
  │  │  ACCOUNT_INFO_STATE_ARCHIVE   [ keccak(addr)++blockNum ]  = historic RLP   │    │
  │  │  ACCOUNT_STORAGE_ARCHIVE      [ keccak(addr)++keccak(slot)++blockNum ] = RLP│   │
  │  │  ACCOUNT_INFO_STATE_FREEZER   [ keccak(addr)++blockNum ]  = older historic │    │
  │  │  ACCOUNT_STORAGE_FREEZER      [ keccak(addr)++keccak(slot)++blockNum ] = RLP│   │
  │  └──────────────────────────────────────────────────────────────────────────────┘    │
  └──────────────────────────────────────────────────────────────────────────────────────┘

  Query: eth_getBalance(Alice, blockNumber=4993)
  Answer: ACCOUNT_INFO_STATE_ARCHIVE.getNearestBefore( keccak(Alice) ++ 4993 )
          → returns the entry at block 4993 (or nearest earlier block) in O(log N)
```

### Key Classes to Remember

| Class | Role |
|-------|------|
| `BonsaiArchiveFlatDbStrategy` | Read/write logic with block-numbered keys and `getNearestBefore` |
| `BonsaiArchiver` | Background worker: moves old state from primary to archive segments |
| `BonsaiArchiveWorldStateProvider` | Routes historic queries to archive flat DB or TrieLog rollback |
| `KeyValueSegmentIdentifier` | Defines all DB segments including the 4 new archive/freezer ones |
| `BonsaiContext` | Carries the block number context for archive key construction |

### Next File

**`07_code_walkthrough.md`** — A step-by-step walk through the actual Besu source code,
tracing a single `eth_getBalance` call at a historic block from the JSON-RPC handler all
the way down to the `getNearestBefore` RocksDB call.