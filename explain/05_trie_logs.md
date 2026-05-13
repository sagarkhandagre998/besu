# Trie Logs: The Time Machine Inside Bonsai

> A complete guide to how Bonsai records block-by-block state changes, uses them to
> reconstruct historic state, and eventually discards them.

---

## Table of Contents

1. [Why TrieLogs Exist](#1-why-trielogs-exist)
2. [The Anatomy of a TrieLog](#2-the-anatomy-of-a-trielog)
3. [PathBasedValue: Recording Before and After](#3-pathbasedvalue-recording-before-and-after)
4. [How a TrieLog is Built During Block Import](#4-how-a-trielog-is-built-during-block-import)
5. [How TrieLogs are Stored on Disk](#5-how-trielogs-are-stored-on-disk)
6. [Rolling Forward and Rolling Backward](#6-rolling-forward-and-rolling-backward)
7. [The Trie Log Retention Window](#7-the-trie-log-retention-window)
8. [The TrieLogPruner](#8-the-trielogpruner)
9. [The TrieLogManager](#9-the-trielogmanager)
10. [Chain Reorganisations and TrieLogs](#10-chain-reorganisations-and-trielogs)
11. [TrieLog Serialization Format](#11-trielog-serialization-format)
12. [TrieLog as a Plugin Extension Point](#12-trielog-as-a-plugin-extension-point)
13. [Limits and What Bonsai Archive Fixes](#13-limits-and-what-bonsai-archive-fixes)
14. [Summary](#14-summary)

---

## 1. Why TrieLogs Exist

Standard Bonsai keeps **only one copy of the world state** on disk — the current chain-head
state. This is what makes it disk-efficient compared to Forest mode. But this immediately
raises a question: if you only keep the latest state, how do you answer a question like:

> "What was Alice's ETH balance 200 blocks ago?"

The answer is the **TrieLog** — a compact, per-block record of exactly what changed in that
block, storing both the **old value (before the block)** and the **new value (after the block)**
for every account, storage slot, and contract that was touched.

Think of a TrieLog as a database transaction log or a git diff:

```
  Block 18,500,000 TrieLog:

  Account changes:
    Alice:  { before: 5.0 ETH,  after: 3.0 ETH  }  ← sent 2 ETH
    Bob:    { before: 1.0 ETH,  after: 3.0 ETH  }  ← received 2 ETH
    Miner:  { before: 0.0 ETH,  after: 0.002 ETH }  ← gas reward

  Storage changes:
    UniswapV3 Pool, slot 0x5a: { before: 1000000, after: 999800 }
    USDC Token,    slot 0x2b: { before: 500,      after: 300    }

  Code changes:
    NewContract: { before: null, after: 0x6080... }  ← new deployment
```

Given this log, you can move the world state in either direction:

- **Roll backward**: Start from the current state. Undo Block 18,500,000 by replacing each
  "after" value with its "before" value. Repeat for each block between the current head and
  your target. You arrive at the state as it was before that block.

- **Roll forward**: Start from an older state. Apply Block 18,500,000 by replacing each
  "before" value with its "after" value.

This is the core mechanic that allows Bonsai to serve historic state queries without
storing complete historic snapshots.

---

## 2. The Anatomy of a TrieLog

The main class is `TrieLogLayer` (in `common/trielog/TrieLogLayer.java`). It implements
the `TrieLog` plugin interface, which allows external plugins to read TrieLog data without
depending on internal Besu classes.

```
  TrieLogLayer
  ┌──────────────────────────────────────────────────────────────────────┐
  │  blockHash    : Hash                                                 │
  │  blockNumber  : Optional<Long>                                       │
  │  frozen       : boolean                                              │
  │                                                                      │
  │  accounts     : Map<Address, PathBasedValue<AccountValue>>           │
  │  code         : Map<Address, PathBasedValue<Bytes>>                  │
  │  storage      : Map<Address,                                         │
  │                      Map<StorageSlotKey,                             │
  │                           PathBasedValue<UInt256>>>                  │
  └──────────────────────────────────────────────────────────────────────┘
```

### The Three Change Maps

**accounts map**
Records every account whose state changed in this block. The key is the account's `Address`
(the full 20-byte address, not its hash — hashing happens later at the storage layer).
The value is a `PathBasedValue<AccountValue>` holding the before and after account state.

```
  Example entry:
  Address("0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045")  →
      PathBasedValue {
          prior:   AccountValue { nonce=5, balance=5 ETH, storageRoot=0x..., codeHash=0x... }
          updated: AccountValue { nonce=6, balance=3 ETH, storageRoot=0x..., codeHash=0x... }
      }
```

**code map**
Records any contract whose deployed bytecode changed in this block. This almost always
means a new contract deployment (code goes from `null` to actual bytecode). Self-destructed
contracts go from bytecode to `null`.

```
  Example entry (new deployment):
  Address("0xNewContract...")  →
      PathBasedValue {
          prior:   null       ← did not exist before
          updated: Bytes(0x6080604052...)  ← deployed bytecode
      }
```

**storage map**
A two-level map. The outer key is the contract `Address`. The inner key is a `StorageSlotKey`
(which wraps the slot index and its hash). The value records the before and after 256-bit
storage values.

```
  Example entry:
  Address("0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48")  →   ← USDC
      {
          StorageSlotKey(slot=0x2b) →
              PathBasedValue { prior: UInt256(500), updated: UInt256(300) }
          StorageSlotKey(slot=0x3f) →
              PathBasedValue { prior: UInt256(0),   updated: UInt256(1)   }
      }
```

### The `frozen` Flag

Once a TrieLog has been built and is ready to persist, it is **frozen** — the `frozen`
boolean is set to `true` and any attempt to add further changes throws an exception. This
prevents accidental mutation after the fact (a defensive programming pattern for immutable
data objects).

```
  trieLog.freeze();
  trieLog.addAccountChange(...)  // throws IllegalStateException: "Layer is Frozen"
```

---

## 3. PathBasedValue: Recording Before and After

`PathBasedValue<T>` is a tiny generic wrapper that holds two references:

```
  PathBasedValue<T>
  ┌──────────────────┐
  │  prior:   T      │  ← state BEFORE the block executed
  │  updated: T      │  ← state AFTER the block executed
  │  isCleared: bool │  ← true if the entry was deleted (e.g., self-destruct)
  └──────────────────┘
```

The `isCleared` flag is important because there is a difference between:

- A value that was set to zero/empty (it still exists, just with a zero value)
- A value that was completely removed from state (deleted — should not exist at all)

For example, when a contract self-destructs, its account entry is **deleted** from the
state trie. The TrieLog marks this with `isCleared = true` so that a rollback can
properly re-delete the entry rather than restoring it to an empty account state.

```
  Regular zero value:  PathBasedValue { prior: 5, updated: 0, isCleared: false }
  Deleted entry:       PathBasedValue { prior: 5, updated: null, isCleared: true }
```

During a backward roll (undo), the logic checks `isCleared`:
- If `isCleared` is true on the "updated" side → the forward direction deleted this entry,
  so going backward we need to **restore** the prior value.
- If `isCleared` is true on the "prior" side → this entry was created in the block,
  so going backward we need to **delete** it.

---

## 4. How a TrieLog is Built During Block Import

The TrieLog is constructed by the `WorldStateUpdateAccumulator` as it processes each
transaction in the block. Here is the full flow:

```
  Block arrives
       │
       ▼
  BonsaiWorldState.processTransactions()
       │
       ├─ Transaction 1 executes
       │       │
       │       └─ EVM reads/writes accounts and storage
       │          Changes are buffered in the UpdateAccumulator (in RAM)
       │
       ├─ Transaction 2 executes  (same accumulator, continues accumulating)
       │
       ├─ ... (all N transactions)
       │
       ▼
  Block execution complete.
  UpdateAccumulator holds ALL net changes for the block.
       │
       ▼
  TrieLogFactoryImpl.buildFrom(accumulator)
       │
       ├─ For each changed account in accumulator:
       │       accumulatedAccount = accumulator.getAccount(address)
       │       trieLog.addAccountChange(
       │           address,
       │           accumulatedAccount.getPrior(),    ← original value from disk
       │           accumulatedAccount.getUpdated()   ← new value after execution
       │       )
       │
       ├─ For each changed storage slot in accumulator:
       │       trieLog.addStorageChange(address, slot, prior, updated)
       │
       ├─ For each changed code entry in accumulator:
       │       trieLog.addCodeChange(address, priorCode, updatedCode, blockHash)
       │
       ▼
  trieLog.setBlockHash(blockHash)
  trieLog.setBlockNumber(blockNumber)
  trieLog.freeze()
       │
       ▼
  Write trieLog to TRIE_LOG_STORAGE[ blockHash ]   ← WAL: log before committing state
       │
       ▼
  Write updated state to ACCOUNT_INFO_STATE, ACCOUNT_STORAGE_STORAGE, TRIE_BRANCH_STORAGE
```

The "prior" values in the TrieLog come from what the accumulator read from disk at the
beginning of transaction execution — the state **before** the block modified it. The
"updated" values are the net result after all transactions in the block have run.

Note that the TrieLog captures **net** changes per block, not per transaction. If Alice
sends ETH in transaction 1 and receives ETH back in transaction 3, the TrieLog only
records her starting balance (prior) and her final balance (updated). The intermediate
values are not preserved in the TrieLog.

---

## 5. How TrieLogs are Stored on Disk

TrieLogs are stored in the `TRIE_LOG_STORAGE` segment (byte ID `0x0a`):

```
  TRIE_LOG_STORAGE
  ┌─────────────────────────────────────────────────────────────────────────┐
  │  Key:   block_hash (32 bytes)                                           │
  │  Value: RLP-serialized TrieLog                                          │
  └─────────────────────────────────────────────────────────────────────────┘
```

The key is the **block hash**, not the block number. This is important for two reasons:

1. **Chain reorganisations**: Two different blocks at the same number (a canonical block
   and an uncle/rejected block) have different hashes. Both can have TrieLogs in the
   database without collision.

2. **Direct access**: When you want to roll to a specific block, you know its hash from the
   block header. You look up `TRIE_LOG_STORAGE[ block_hash ]` directly — O(1).

The number of TrieLogs in the database at any time is approximately equal to the
retention window size (default 512), because the pruner deletes old ones.

---

## 6. Rolling Forward and Rolling Backward

This is the heart of historic state queries. Let us trace through exactly what happens.

### Scenario Setup

```
  Current chain head: Block 1000
  Query: "What was Alice's balance at Block 997?"
  Trie logs available: Blocks 489 through 1000 (512-block window)
```

### Step 1: Obtain a "Mutable" World State Snapshot

Besu starts from the chain-head world state (which reflects Block 1000). It wraps this in
a **snapshot layer** — a layered storage object where writes go to an in-memory overlay
and reads first check the overlay, then fall back to the underlying persistent storage.

```
  ┌────────────────────────────────────┐
  │  SnapshotLayer (in RAM, empty)     │  ← writes go here
  │                                    │
  │  Falls through to:                 │
  │  ┌──────────────────────────────┐  │
  │  │  Persistent Flat DB (Block   │  │  ← reads come from here if not in snapshot
  │  │  1000 state)                 │  │
  │  └──────────────────────────────┘  │
  └────────────────────────────────────┘
```

### Step 2: Apply TrieLog 1000 in Reverse

Read `TRIE_LOG_STORAGE[ block_1000_hash ]`. For every entry in the TrieLog:

```
  For each account change { address, prior, updated }:
      snapshotLayer.put(keccak(address), RLP.encode(prior))
      ↑  We write the BEFORE value into the snapshot,
         overriding the current (Block 1000) value.

  For each storage change { address, slotKey, prior, updated }:
      snapshotLayer.put(keccak(address) ++ keccak(slotKey), RLP.encode(prior))

  For deleted entries (isCleared on updated):
      snapshotLayer.delete(key)
      ↑  The entry was created in Block 1000, so going backward we remove it.

  For created entries (isCleared on prior):
      snapshotLayer.put(key, RLP.encode(updated))
      ↑  Wait — this is backward roll, so if prior is null,
         the entry didn't exist before Block 1000.
         We need to mark it as deleted in the snapshot layer.
```

After applying TrieLog 1000 in reverse, the snapshot layer now represents Block 999's
state for every account/slot that changed in Block 1000.

### Step 3: Repeat for Blocks 999 and 998

Apply TrieLog 999 in reverse → snapshot now represents Block 998's state.
Apply TrieLog 998 in reverse → snapshot now represents Block 997's state.

### Step 4: Read Alice's Balance

```
  snapshotLayer.get(keccak(Alice's address))
       │
       ├─ Hit in snapshot → return this value (the Block 997 value)  ✓
       │
       └─ Miss in snapshot → fall through to persistent flat DB
              (Alice didn't change in blocks 998, 999, or 1000,
               so the current flat DB value IS her Block 997 value)  ✓
```

Either way the answer is correct. The snapshot layer only contains entries that
**changed** in the rolled-back blocks. Accounts that didn't change are served directly
from the persistent flat DB (which already holds their unchanged value).

### Rolling Forward (Less Common)

Rolling forward applies TrieLog entries in the "forward" direction — replacing "prior"
values with "updated" values. This is used when moving from an older cached state to
a newer one (e.g., when the block cache needs to advance).

---

## 7. The Trie Log Retention Window

The retention window is controlled by the `--bonsai-historical-block-limit` flag
(default: 512).

```
  Chain head: Block N

  Retention window (512 blocks):
  ┌──────────────────────────────────────────────────────────────┐
  │  [N-511] [N-510] ... [N-2] [N-1] [N]  ← TrieLogs exist      │
  └──────────────────────────────────────────────────────────────┘
        │                               │
     oldest                          newest

  [N-512] and older  ←  TrieLogs DELETED, historic state UNAVAILABLE
```

Why 512? It is a balance between:

- **Storage cost**: Each TrieLog can be hundreds of kilobytes for busy blocks. 512 logs
  may use several gigabytes on a busy chain (like Ethereum mainnet).
- **Utility**: Most applications that need "recent" history (e.g., watching for confirmed
  transactions, reading state at "safe" or "finalized" tags) need fewer than 512 blocks.
- **EIP-4444**: The Ethereum roadmap envisions nodes not being required to serve very old
  history. 512 blocks covers roughly 1.7 hours on Ethereum mainnet — enough for practical
  "recent" queries.

The value 512 is also not arbitrary: it aligns with Ethereum's finality horizon
(roughly 2 epochs = 64 blocks) plus a comfortable margin.

### Configuring the Window

```
  # Keep 1000 blocks of history (uses more disk, serves more historic queries)
  besu --bonsai-historical-block-limit=1000

  # Keep only 128 blocks (saves disk, limits historic access further)
  besu --bonsai-historical-block-limit=128

  # Unlimited (keep all trie logs — essentially Forest-like disk growth)
  # Not recommended for standard Bonsai; this is what Bonsai Archive replaces
```

---

## 8. The TrieLogPruner

`TrieLogPruner` is a component that runs asynchronously after new blocks are added. Its
job is simple: delete TrieLogs that are outside the retention window.

```
  TrieLogPruner lifecycle:
  ┌──────────────────────────────────────────────────────────────────────┐
  │                                                                      │
  │  On startup:                                                         │
  │    Read all block numbers from TRIE_LOG_STORAGE                      │
  │    Build a sorted list of (blockNumber, blockHash) pairs             │
  │    Delete any entries outside the retention window immediately       │
  │                                                                      │
  │  After each new block is added:                                      │
  │    Pruner is triggered asynchronously                                │
  │    It calculates the oldest block to retain:                         │
  │        retain_from = chainHead - retentionWindow                     │
  │    It deletes TRIE_LOG_STORAGE[ hash ] for all blocks < retain_from  │
  │                                                                      │
  └──────────────────────────────────────────────────────────────────────┘
```

The pruner is deliberately asynchronous — it does not block the block import critical
path. A short delay in pruning old TrieLogs is acceptable; they just take up a bit more
disk space temporarily.

```
  Example pruner run (chain head at Block 10,512):

  Retention window: 512 blocks
  Retain from: 10,512 - 512 = Block 10,000

  Prune: TRIE_LOG_STORAGE[ hash of block 9,999 ]   ← deleted
         TRIE_LOG_STORAGE[ hash of block 9,998 ]   ← deleted
         TRIE_LOG_STORAGE[ hash of block 9,997 ]   ← deleted
         ...

  Keep:  TRIE_LOG_STORAGE[ hash of block 10,000 ]  ← retained (oldest)
         TRIE_LOG_STORAGE[ hash of block 10,001 ]  ← retained
         ...
         TRIE_LOG_STORAGE[ hash of block 10,512 ]  ← retained (newest)
```

### NoOpTrieLogManager

For scenarios where historic state access is entirely disabled (e.g., certain light-client
configurations or testing environments), Besu provides `NoOpTrieLogManager` — an
implementation that discards all TrieLogs immediately and never serves historic state.
This further reduces disk usage at the cost of zero historic state support.

---

## 9. The TrieLogManager

`TrieLogManager` (the interface, implemented by the concrete manager class) coordinates
all TrieLog operations. Its responsibilities:

```
  TrieLogManager responsibilities:
  ┌────────────────────────────────────────────────────────────────────┐
  │                                                                    │
  │  saveTrieLog(accumulator, newWorldState, blockHash)                │
  │    → Build TrieLog from accumulator                                │
  │    → Serialize and write to TRIE_LOG_STORAGE[ blockHash ]          │
  │    → Emit TrieLogAddedEvent for any plugin listeners               │
  │                                                                    │
  │  getTrieLogLayer(blockHash) → Optional<TrieLog>                    │
  │    → Read from TRIE_LOG_STORAGE[ blockHash ]                       │
  │    → Deserialize and return                                        │
  │                                                                    │
  │  getMaxLayersToLoad() → long                                       │
  │    → Returns the configured retention window size (e.g. 512)       │
  │                                                                    │
  │  addCachedLayer(blockHash, worldState)                             │
  │    → Notify the cached world storage manager                       │
  │                                                                    │
  └────────────────────────────────────────────────────────────────────┘
```

### TrieLogAddedEvent

After each TrieLog is saved, a `TrieLogAddedEvent` is published on Besu's event bus.
External plugins can subscribe to this event to receive TrieLogs in real time — for
example, a plugin that replicates TrieLogs to an external database or a message queue.

```
  Block imported
       │
       ▼
  TrieLog saved to disk
       │
       ▼
  TrieLogAddedEvent published
       │
       ├─ Plugin A: replicate to external DB
       ├─ Plugin B: forward to message queue
       └─ Plugin C: compute derived analytics
```

---

## 10. Chain Reorganisations and TrieLogs

A chain reorganisation ("reorg") happens when the node discovers a longer chain that
diverges from the current canonical chain. Bonsai must handle reorgs using TrieLogs.

```
  Before reorg:
  ... ← [Block 100] ← [Block 101_A] ← [Block 102_A]   ← canonical

  After reorg (102_B and 103_B are longer/heavier):
  ... ← [Block 100] ← [Block 101_A] ← [Block 102_A]   ← orphaned
                    ↖
                     [Block 101_B] ← [Block 102_B] ← [Block 103_B]  ← new canonical
```

To switch from the old chain to the new chain:

```
  Step 1: Roll BACKWARD through orphaned blocks
          Apply TrieLog[102_A] in reverse → state at Block 101_A
          Apply TrieLog[101_A] in reverse → state at Block 100   ← common ancestor

  Step 2: Roll FORWARD through new canonical blocks
          Apply TrieLog[101_B] forward → state at Block 101_B
          Apply TrieLog[102_B] forward → state at Block 102_B
          Apply TrieLog[103_B] forward → state at Block 103_B   ← new head

  Step 3: Persist the new head state to the flat DB
```

This works cleanly because TrieLogs exist for all branches within the retention window —
even non-canonical blocks have their TrieLogs stored (keyed by block hash). The pruner
only deletes TrieLogs by age (block number), not by chain membership.

For deep reorgs (more than the retention window depth), Bonsai cannot perform the
rollback via TrieLogs. In that case the node would need to resync — deep reorgs beyond
the retention window are considered catastrophic network events in practice.

---

## 11. TrieLog Serialization Format

TrieLogs are serialized using **RLP** (Recursive Length Prefix), the same encoding
Ethereum uses for transactions, blocks, and receipts.

The serialization is handled by `TrieLogFactoryImpl`. The format encodes:

```
  RLP-encoded TrieLog:
  [
    blockHash,                        ← 32 bytes
    blockNumber,                      ← 8 bytes (uint64)
    [                                 ← account changes list
      [
        address,                      ← 20 bytes
        prior_account_RLP_or_null,    ← RLP(nonce, balance, storageRoot, codeHash) or null
        updated_account_RLP_or_null,
        is_cleared                    ← boolean
      ],
      ...
    ],
    [                                 ← code changes list
      [
        address,
        prior_code_or_null,
        updated_code_or_null,
        is_cleared
      ],
      ...
    ],
    [                                 ← storage changes list
      [
        address,
        [                             ← slot changes for this address
          [
            slot_key_hash,            ← 32 bytes (keccak of slot index)
            prior_value_or_null,      ← UInt256 or null
            updated_value_or_null,
            is_cleared
          ],
          ...
        ]
      ],
      ...
    ]
  ]
```

The encoding is deterministic — the same TrieLog always produces the same bytes. This
matters because TrieLogs can be exported and shared between nodes for replay verification.

### Size Characteristics

```
  Typical TrieLog sizes (Ethereum mainnet):

  Empty block (no transactions):   ~  100 bytes
  Light block (few txns):          ~   5 KB
  Average block:                   ~ 50–200 KB
  Busy DeFi block (many txns):     ~ 500 KB – 2 MB

  512 blocks × average 100 KB = ~50 MB of TrieLog data on disk
  512 blocks × busy 1 MB       = ~500 MB of TrieLog data on disk
```

This is a significant but manageable amount of storage compared to the gigabytes of trie
node data that Forest mode accumulates.

---

## 12. TrieLog as a Plugin Extension Point

The `TrieLog` interface (in the plugin API) is a stable public API that external plugins
can use without depending on Besu internals:

```
  plugin-api/TrieLog interface:
  ┌──────────────────────────────────────────────────────────────────────┐
  │  getBlockHash()    → Hash                                            │
  │  getBlockNumber()  → Optional<Long>                                  │
  │                                                                      │
  │  getAccountChanges() → Map<Address, PathBasedValue<AccountValue>>    │
  │  getCodeChanges()    → Map<Address, PathBasedValue<Bytes>>           │
  │  getStorageChanges() → Map<Address,                                  │
  │                              Map<StorageSlotKey,                     │
  │                                   PathBasedValue<UInt256>>>          │
  │                                                                      │
  │  getAccount(address)           → Optional<AccountValue>              │
  │  getPriorAccount(address)      → Optional<AccountValue>              │
  │  getCode(address)              → Optional<Bytes>                     │
  │  getStorageByStorageSlotKey(address, slot) → Optional<UInt256>       │
  └──────────────────────────────────────────────────────────────────────┘
```

This means a plugin can:

- Subscribe to `TrieLogAddedEvent`
- Receive the `TrieLog` object
- Read all state changes without parsing raw RLP or touching the database directly
- Forward changes to external systems (Kafka, PostgreSQL, Elasticsearch, etc.)

---

## 13. Limits and What Bonsai Archive Fixes

Here is a precise summary of what TrieLogs can and cannot do:

```
  ┌──────────────────────────────────────────────────────────────────────────┐
  │  What TrieLogs CAN do                                                    │
  ├──────────────────────────────────────────────────────────────────────────┤
  │  ✅ Roll state backward to any block within the retention window         │
  │  ✅ Roll state forward for chain reorganisations                         │
  │  ✅ Record net changes per block (not per transaction)                   │
  │  ✅ Record deletions (isCleared flag)                                    │
  │  ✅ Serve as crash-recovery WAL (log before state write)                 │
  │  ✅ Be exported to plugins via the public TrieLog API                    │
  └──────────────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────────────┐
  │  What TrieLogs CANNOT do                                                 │
  ├──────────────────────────────────────────────────────────────────────────┤
  │  ❌ Serve historic state older than the retention window                 │
  │  ❌ Reconstruct per-transaction state (only per-block net changes)       │
  │  ❌ Be kept indefinitely without growing unboundedly (pruner deletes)    │
  │  ❌ Serve state for blocks where a TrieLog was never saved               │
  │     (e.g., blocks imported before Bonsai was enabled)                   │
  └──────────────────────────────────────────────────────────────────────────┘
```

The critical gap is the first item in the "cannot do" list. You cannot set the retention
window to infinity because:

1. The TrieLogs themselves would consume enormous disk space (hundreds of GB over months).
2. Rolling back thousands of blocks is slow — each step requires one TrieLog read and
   many write operations on the snapshot layer.
3. For accounts that change frequently (e.g., a DEX pool), you would still need to
   replay every block individually, unlike a direct key lookup.

**Bonsai Archive** solves this by storing historic state **directly in the flat DB**
with a new key scheme — so that historic state can be retrieved with a single database
lookup (like current state), instead of by replaying TrieLogs.

---

## 14. Summary

```
  TrieLog in one picture:

       Block N-2          Block N-1           Block N
  ──────────────────────────────────────────────────────►  time
       │                   │                   │
       ▼                   ▼                   ▼
  TrieLog[N-2]        TrieLog[N-1]        TrieLog[N]
  { before, after }   { before, after }   { before, after }
       │                   │                   │
       └───────────────────┴───────────────────┘
                           │
                    Used to roll back
                    from Block N to
                    any block ≥ N - window

  State at Block N (current) is always in the flat DB.
  State at Block N-k (for k ≤ window) is reconstructed by rolling.
  State at Block N-k (for k > window) is UNAVAILABLE on standard Bonsai.
```

| Concept | One-line summary |
|---------|-----------------|
| TrieLog | Per-block diff storing before+after for every changed account/slot/code |
| PathBasedValue | Wrapper holding `prior` and `updated` with an `isCleared` deletion flag |
| Retention window | Number of recent TrieLogs kept on disk (default 512) |
| Rolling backward | Undo blocks one at a time by swapping "updated" → "prior" in a snapshot layer |
| Rolling forward | Apply blocks by swapping "prior" → "updated" (used in reorgs) |
| TrieLogPruner | Asynchronous job that deletes TrieLogs older than the retention window |
| TrieLogManager | Orchestrates save, load, and cache notification for TrieLogs |
| Plugin API | `TrieLog` interface lets external plugins read diffs without internal dependencies |

### Next File

**`06_bonsai_archive.md`** — How Bonsai Archive extends the flat DB with a new key scheme
that encodes block numbers into keys, allowing historic state to be looked up in O(1)
without replaying any TrieLogs. This is the feature that makes Bonsai a full archive node.