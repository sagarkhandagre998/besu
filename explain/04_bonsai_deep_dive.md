# Bonsai Deep Dive: Internals, DB Segments, and Flat DB Modes

> A detailed look at how Bonsai works under the hood — from RocksDB column families to flat DB
> healing — before we introduce the Archive extension.

---

## Table of Contents

1. [Bonsai's Physical Database Layout](#1-bonsais-physical-database-layout)
2. [The Five Core DB Segments](#2-the-five-core-db-segments)
3. [Flat DB Modes: Partial vs Full](#3-flat-db-modes-partial-vs-full)
4. [How a Block Import Works in Bonsai](#4-how-a-block-import-works-in-bonsai)
5. [The World State Update Accumulator](#5-the-world-state-update-accumulator)
6. [The Trie Log Lifecycle](#6-the-trie-log-lifecycle)
7. [How Historic State Queries Work](#7-how-historic-state-queries-work)
8. [The Cached World Storage Manager](#8-the-cached-world-storage-manager)
9. [State Healing: When the Flat DB Goes Wrong](#9-state-healing-when-the-flat-db-goes-wrong)
10. [Code Storage Strategy](#10-code-storage-strategy)
11. [Summary and What Is Next](#11-summary-and-what-is-next)

---

## 1. Bonsai's Physical Database Layout

Besu uses **RocksDB** as its underlying key-value store. RocksDB organises data into separate
namespaces called **column families** (Besu calls them **segments**). Each segment is an
independent sorted key-value store — think of it as a separate table in a database.

Bonsai uses the following segments (defined in `KeyValueSegmentIdentifier.java`):

```
  RocksDB on disk
  ┌─────────────────────────────────────────────────────────────────────────────┐
  │  Segment name             │  Byte ID  │  Used by           │  Purpose       │
  │───────────────────────────┼───────────┼────────────────────┼────────────────│
  │  ACCOUNT_INFO_STATE       │  0x06     │  BONSAI            │  Current acct  │
  │  CODE_STORAGE             │  0x07     │  BONSAI            │  Contract code │
  │  ACCOUNT_STORAGE_STORAGE  │  0x08     │  BONSAI            │  Current slots │
  │  TRIE_BRANCH_STORAGE      │  0x09     │  BONSAI            │  MPT nodes     │
  │  TRIE_LOG_STORAGE         │  0x0a     │  BONSAI            │  Block diffs   │
  └─────────────────────────────────────────────────────────────────────────────┘
```

The byte IDs are how RocksDB column families were originally referenced in Besu. When
diagnosing issues with `ldb` (the RocksDB command-line tool) you would write something like:

```
  ldb --db=/path/to/db --column_family=$'\x09' scan
```

to inspect `TRIE_BRANCH_STORAGE`. These single-byte IDs were chosen to keep the column-family
descriptor small, but they make direct debugging harder because you must know the mapping.
(The Bonsai Archive feature improves on this — more in file 06.)

---

## 2. The Five Core DB Segments

### 2a. ACCOUNT_INFO_STATE (Flat Accounts)

This is the **flat account table** — the heart of Bonsai's read performance advantage.

```
  Key format:   keccak256(account_address)   — 32 bytes
  Value format: RLP-encoded account state    — { nonce, balance, storageRoot, codeHash }

  ┌────────────────────────────┬──────────────────────────────────────────────┐
  │ Key (32 bytes)             │ Value (RLP bytes)                            │
  ├────────────────────────────┼──────────────────────────────────────────────┤
  │ 0x3e2625...b019...         │ F8 4D 03 89 17 1F A6 91 DC ... (RLP stream) │
  │ 0x4D95FB...DC02f (hashed)  │ F8 4D 03 89 ...                             │
  └────────────────────────────┴──────────────────────────────────────────────┘
```

Since the key is a hash (uniform, random-looking), entries are spread evenly across the
key space — RocksDB's sorted structure means a range scan will touch all accounts in
lexicographic order of their hashed addresses.

**Critically: only ONE entry per account exists here.** It always reflects the current (chain-head)
state. This is what makes Bonsai disk-efficient compared to Forest, but also what limits its
ability to serve historic state.

### 2b. ACCOUNT_STORAGE_STORAGE (Flat Storage Slots)

Every storage slot of every contract lives here.

```
  Key format:   keccak256(account_address) ++ keccak256(slot_index)   — 64 bytes
  Value format: RLP-encoded 256-bit value

  Example — ERC-20 token contract, balances mapping:
  ┌──────────────────────────────────────────────────────────────────┐
  │ Key:  hash(contract) ++ hash(slot_for_Alice)                     │
  │ Value: RLP{ 0x000...2A }   (Alice holds 42 tokens)               │
  └──────────────────────────────────────────────────────────────────┘
```

The layout means all storage slots for a given contract are **adjacent** in the sorted key
space (because they all start with the same 32-byte account hash prefix). A range scan from
`hash(contract) ++ 0x000...000` to `hash(contract) ++ 0xfff...fff` yields all storage slots
for that contract in one sequential read — much faster than following individual trie pointers.

### 2c. CODE_STORAGE (Contract Bytecode)

```
  Key format:   keccak256(code)              (default: code-hash keying)
              OR keccak256(account_address)  (account-hash keying, via --Xbonsai-code-using-code-hash-enabled)
  Value format: raw bytecode bytes
```

Code is immutable once deployed, so the same bytecode hash is shared across all instances
of the same contract factory pattern. Code-hash keying naturally deduplicates bytecode (two
contracts with identical code share one entry). Account-hash keying avoids the hash
computation during writes.

### 2d. TRIE_BRANCH_STORAGE (MPT Nodes)

This segment stores the actual Merkle Patricia Trie nodes — the same type of data that
Forest mode stores, but only for the **current head state** (not historic versions).

```
  Key format:   node_location_in_trie   — variable length (path from root)
  Value format: RLP-encoded MPT node

  The location encodes the path: e.g. location = [3, f, 7, a, ...] for an account
  whose keccak hash starts with nibbles 3, f, 7, a, ...
```

Special entries in `TRIE_BRANCH_STORAGE` also track meta-information:

```
  WORLD_ROOT_HASH_KEY  -->  current stateRoot (32 bytes)
  WORLD_BLOCK_HASH_KEY -->  current block hash (32 bytes)
  WORLD_BLOCK_NUMBER_KEY --> current block number (8 bytes, big-endian long)
```

These three keys are the "bookmark" that tells Bonsai which block the current state
corresponds to, without having to look at the blockchain data structures.

### 2e. TRIE_LOG_STORAGE (Block Diffs)

Each entry is the serialised diff (TrieLog) for one block.

```
  Key format:   block_hash   — 32 bytes
  Value format: RLP-encoded TrieLog object

  TrieLog structure:
  ┌──────────────────────────────────────────────────────────────────┐
  │ blockHash: 0xabc...                                              │
  │ blockNumber: 18500000                                            │
  │ accountChanges: {                                                │
  │     Alice: { prior: {nonce:4, bal:5eth}, updated: {nonce:5, bal:3eth} } │
  │     Bob:   { prior: {nonce:1, bal:2eth}, updated: {nonce:1, bal:4eth} } │
  │ }                                                                │
  │ storageChanges: {                                                │
  │     ContractX: {                                                 │
  │         slot1: { prior: 0, updated: 99 }                         │
  │     }                                                            │
  │ }                                                                │
  │ codeChanges: { ... }                                             │
  └──────────────────────────────────────────────────────────────────┘
```

Trie logs are the Bonsai equivalent of the WAL (Write-Ahead Log) pattern: they record
what changed so you can undo it. They are written **before** the world state is updated,
ensuring crash safety.

---

## 3. Flat DB Modes: Partial vs Full

The flat DB does not always contain every account. Besu supports two sub-modes for the
flat account and storage tables:

### Partial Flat DB Mode

After a **snap sync** (the default fast-sync strategy), the flat DB may not yet contain
all accounts. Snap sync downloads a point-in-time snapshot of accounts and storage, but
during the initial sync it is written in batches that may leave gaps.

In **partial mode**:
- Bonsai attempts to read from the flat DB first.
- On a miss (entry not found), it falls back to traversing the trie.
- This fallback is slower but guarantees correctness.

```
  Partial flat DB read path:

  getFlatAccount(accountHash)
       │
       ▼
  ACCOUNT_INFO_STATE[ accountHash ]  ──found──▶  return value  ✓
       │
      miss
       │
       ▼
  Traverse state MPT from stateRoot
  following nibbles of accountHash   ──found──▶  return value  ✓
       │
      miss
       │
       ▼
  return empty (account does not exist)
```

### Full Flat DB Mode

Once the flat DB has been fully healed/populated (either during snap sync completion or
after a full trie scan), Bonsai switches to **full mode**:

- Reads come entirely from the flat DB.
- No trie traversal fallback.
- A miss in the flat DB definitively means the account does not exist.
- Much faster and simpler.

```
  Full flat DB read path:

  getFlatAccount(accountHash)
       │
       ▼
  ACCOUNT_INFO_STATE[ accountHash ]  ──found──▶  return value  ✓
       │
      miss
       │
       ▼
  return empty (account does not exist)   — no trie fallback
```

The mode is stored as a version byte inside the `TRIE_BRANCH_STORAGE` segment. When Besu
starts, it reads this byte to determine which strategy to use.

```
  TRIE_BRANCH_STORAGE[ FLAT_DB_MODE_KEY ] = 0x00  -->  Partial mode
  TRIE_BRANCH_STORAGE[ FLAT_DB_MODE_KEY ] = 0x01  -->  Full mode
  TRIE_BRANCH_STORAGE[ FLAT_DB_MODE_KEY ] = 0x02  -->  Archive mode (X_BONSAI_ARCHIVE)
```

---

## 4. How a Block Import Works in Bonsai

When a new block arrives from the network and passes validation, Bonsai processes it as follows:

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │                     Block Import Pipeline                            │
  │                                                                      │
  │  1. Receive Block (header + transactions + uncles)                   │
  │          │                                                           │
  │  2. Execute each transaction against current world state             │
  │          │  (EVM runs each transaction, reading from flat DB)        │
  │          │                                                           │
  │  3. Collect all state changes in the WorldStateUpdateAccumulator     │
  │          │  (an in-memory buffer: "who changed, what changed")       │
  │          │                                                           │
  │  4. Compute the new stateRoot from the accumulated changes           │
  │          │  (apply changes to the in-memory trie, hash upward)       │
  │          │                                                           │
  │  5. Verify the computed stateRoot == block header's stateRoot        │
  │          │  (if mismatch → reject block)                             │
  │          │                                                           │
  │  6. Build the TrieLog for this block                                 │
  │          │  (record before+after for every changed account/slot)     │
  │          │                                                           │
  │  7. Write to DB (atomic transaction):                                │
  │          │  a. TRIE_LOG_STORAGE[ blockHash ]  = trieLog              │
  │          │  b. ACCOUNT_INFO_STATE[ accountHash ] = newValue          │
  │          │  c. ACCOUNT_STORAGE_STORAGE[ key ]   = newValue           │
  │          │  d. TRIE_BRANCH_STORAGE[ location ]   = newTrieNode       │
  │          │  e. TRIE_BRANCH_STORAGE[ WORLD_ROOT_HASH_KEY ] = stateRoot│
  │          │  f. TRIE_BRANCH_STORAGE[ WORLD_BLOCK_HASH_KEY ] = hash    │
  │          │  g. TRIE_BRANCH_STORAGE[ WORLD_BLOCK_NUMBER_KEY ] = num   │
  │          │                                                           │
  │  8. The old values in ACCOUNT_INFO_STATE are silently overwritten    │
  │          │  (unlike Forest, which keeps all old nodes)               │
  │          │                                                           │
  │  9. Prune old TrieLogs beyond the retention window (async)           │
  └──────────────────────────────────────────────────────────────────────┘
```

**Step 7a comes before 7b–g** — this is the write-ahead log pattern. If the node crashes
between 7a and 7b, the next startup can recover: the trie log is present, the flat DB
still reflects the state before the crash, and the crash is effectively rolled back.

---

## 5. The World State Update Accumulator

The `BonsaiWorldStateUpdateAccumulator` (and its parent `PathBasedWorldStateUpdateAccumulator`)
is an **in-memory scratch pad** that collects all state changes while a block is being
executed.

Think of it as a local "diff" that sits on top of the persisted world state:

```
  Persisted State (disk)
  ┌──────────────────────────┐
  │ Alice balance = 5 ETH    │  <── unchanged, read from flat DB
  │ Bob   balance = 3 ETH    │
  │ ContractX slot1 = 99     │
  └──────────────────────────┘
           ↑  reads fall through
  ┌──────────────────────────┐
  │ UpdateAccumulator (RAM)  │  <── live during block execution
  │ Alice balance = 3 ETH    │  (Alice sent 2 ETH → pending update)
  │ ContractX slot1 = 200    │  (contract wrote to slot1)
  └──────────────────────────┘
           ↑  EVM reads from here first, then falls through to disk
```

During transaction execution the EVM reads accounts through this layered view. If
`Alice` has already been modified in the accumulator (e.g., by a previous transaction in
the same block), the EVM sees the updated value, not the stale disk value.

At the end of block execution, the accumulator is "committed" — its changes are
persisted to the flat DB and the trie, and it feeds the TrieLog builder.

---

## 6. The Trie Log Lifecycle

```
  Block N arrives
       │
       ▼
  TrieLog for Block N is built from the accumulator
  and written to TRIE_LOG_STORAGE[ block_N_hash ]
       │
       ▼
  The flat DB and trie are updated
       │
       ▼
  Many blocks later... Block N+512 arrives
       │
       ▼
  TrieLogPruner runs (async)
  It deletes TRIE_LOG_STORAGE[ block_N_hash ]
  (Block N is now too old to roll back to anyway)
       │
       ▼
  State at Block N is now GONE from a standard Bonsai node
```

The `TrieLogPruner` keeps a configurable number of recent trie logs (default 512). The
pruner runs asynchronously after new blocks are added, so it does not slow down the
critical path of block import.

```
  TRIE_LOG_STORAGE contents (sliding window):
  ┌────────────────────────────────────────────────────────────────────┐
  │ [block N-511] [block N-510] ... [block N-1] [block N]             │
  │   newest ◄─────────────────────────────────────── oldest           │
  │                                 ↑                                  │
  │                          window of 512                             │
  │                                                                    │
  │ [block N-512] ← DELETED by pruner when block N+1 arrives           │
  └────────────────────────────────────────────────────────────────────┘
```

---

## 7. How Historic State Queries Work

When an application calls `eth_getBalance(address, "0x11A5B30")` (some old block), Besu must
serve the state at that block. Here is what happens step-by-step:

```
  Query: eth_getBalance(Alice, block = N-5)
  Current head: block N
  Trie log window: 512 blocks (N-5 is within window)

  Step 1: Request a world state at block N-5
          BonsaiWorldStateProvider.getWorldState(queryParams{blockN-5})

  Step 2: Get the chain-head world state from the cache
          (a PathBasedWorldState object pointing at the head flat DB)

  Step 3: Create a layered (snapshot) world state
          BonsaiSnapshotWorldStateKeyValueStorage wraps the head storage
          with a diff layer on top

  Step 4: Roll back 5 blocks by applying trie logs in reverse

          Apply TrieLog[N]   backwards → diff layer shows state at N-1
          Apply TrieLog[N-1] backwards → diff layer shows state at N-2
          Apply TrieLog[N-2] backwards → diff layer shows state at N-3
          Apply TrieLog[N-3] backwards → diff layer shows state at N-4
          Apply TrieLog[N-4] backwards → diff layer shows state at N-5

  Step 5: Read Alice's balance from the layered world state
          → returns the balance as it was at block N-5

  Step 6: The diff layer is discarded (nothing was written to disk)
```

The layered world state uses **copy-on-write**: reads go through the diff layer first,
then fall back to the persistent flat DB. Old values extracted from trie logs are stored
in the diff layer. This means rolling back never touches the on-disk state.

### What happens beyond 512 blocks?

```
  Query: eth_getBalance(Alice, block = N-5000)
  Trie log window: 512 blocks

  Block N-5000 is 4488 blocks outside the window.
  Trie logs for blocks N-5000 through N-512 have been pruned.

  Standard Bonsai result: ERROR
  "State not available for block N-5000"
```

This is the hard wall that Bonsai Archive tears down.

---

## 8. The Cached World Storage Manager

`BonsaiCachedWorldStorageManager` (extending `PathBasedCachedWorldStorageManager`) keeps a
small in-memory cache of recently accessed world states.

```
  Cache structure:
  ┌──────────────────────────────────────────────────────────────┐
  │  blockHash → PathBasedCachedWorldView                        │
  │                                                              │
  │  head_block_hash   → mutable head world state (always kept) │
  │  head-1 block_hash → cached snapshot world state            │
  │  head-2 block_hash → cached snapshot world state            │
  │  ...                                                         │
  └──────────────────────────────────────────────────────────────┘
```

The cache avoids the cost of replaying trie logs repeatedly for the same block.
For example, if 10 RPC calls all ask for state at block N-3, the trie logs are
only replayed once; subsequent calls get the cached layered world state.

Cache eviction is based on a reference-counting mechanism: a world state is evicted
when no active query holds a reference to it.

---

## 9. State Healing: When the Flat DB Goes Wrong

The flat DB can become inconsistent in certain scenarios:

- **Snap sync interruption**: The node downloaded part of the state snapshot but crashed.
- **Invalid MPT nodes**: Corruption in the trie causes a mismatch between the trie and flat DB.
- **Reorg during sync**: A chain reorganisation happened while the flat DB was being populated.

Besu detects these situations when a read from the flat DB returns a value whose hash does
not match the corresponding MPT leaf. The healing process then:

```
  1. Identify the bad account (address and storage location)
  2. Remove the corrupted MPT nodes for that account from TRIE_BRANCH_STORAGE
  3. Mark the flat DB as "partial" mode (fall back to trie for misses)
  4. Trigger a snap-heal: re-download the missing trie nodes from peers
  5. Once healing is complete, upgrade back to "full" mode
```

The `BonsaiWorldStateProvider.prepareStateHealing()` method implements steps 1–3:

```
  prepareStateHealing(address, location):
    1. Build a StoredMerklePatriciaTrie from the current stateRoot
    2. Walk the trie to the account leaf and its storage trie
    3. Collect all node locations that are invalid
    4. Delete those nodes from TRIE_BRANCH_STORAGE
    5. Call downgradeToPartialFlatDbMode()
```

This allows the node to continue operating during healing (with slower performance)
rather than halting entirely.

---

## 10. Code Storage Strategy

Besu supports two strategies for storing contract bytecode, controlled by the
`--Xbonsai-code-using-code-hash-enabled` flag:

### Strategy A: Account-Hash Keying (default)

```
  Key:   keccak256(account_address)
  Value: bytecode bytes

  Pro: No need to hash the bytecode at write time (saves CPU)
  Con: Two contracts with identical bytecode store it twice
```

### Strategy B: Code-Hash Keying (with flag)

```
  Key:   keccak256(bytecode)
  Value: bytecode bytes

  Pro: Natural deduplication — identical contracts share one entry
  Con: Must hash the bytecode at write time
       Must look up by code hash at read time (requires account entry first)
```

In practice, code duplication is not a major issue (there are fewer unique contracts
than accounts), so the default account-hash keying is fine for most nodes.

---

## 11. Summary and What Is Next

### Bonsai's Three Pillars

```
  ┌─────────────────────────────────────────────────────────────────────────────┐
  │  Pillar 1: Flat DB              │  Pillar 2: Single Trie    │  Pillar 3: TrieLog  │
  │                                  │                            │                     │
  │  ACCOUNT_INFO_STATE              │  Only the head state trie  │  Per-block diffs    │
  │  ACCOUNT_STORAGE_STORAGE         │  is kept on disk           │  enable rollback    │
  │  CODE_STORAGE                    │                            │  to recent blocks   │
  │                                  │  No history = small disk   │                     │
  │  O(1) reads for current state    │  footprint                 │  Pruned beyond      │
  │  (vs O(depth) for trie traversal)│                            │  512 blocks         │
  └─────────────────────────────────────────────────────────────────────────────┘
```

### What remains unsolved by standard Bonsai

| Need | Standard Bonsai | Solution |
|------|----------------|----------|
| Fast current-state reads | ✅ Flat DB | — |
| Small disk footprint | ✅ Single trie | — |
| Recent historic queries (< 512 blocks) | ✅ TrieLog rollback | — |
| Historic queries (any block, any time) | ❌ Not possible | Bonsai Archive |

The next file explores exactly how Bonsai Archive addresses the "any block, any time"
historic query requirement by adding a new key scheme and two new DB segments.

### Next File

**`05_trie_logs.md`** — A deep dive into TrieLog: its internal structure, serialization
format, how forward and backward rolling works, and how the pruner manages the retention window.