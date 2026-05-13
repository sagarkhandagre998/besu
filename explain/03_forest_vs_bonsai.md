# Forest vs Bonsai: Besu's Two Storage Strategies

> Understanding why Bonsai was invented by first understanding what came before it.

---

## Table of Contents

1. [The Storage Problem Every Ethereum Node Must Solve](#1-the-storage-problem-every-ethereum-node-must-solve)
2. [Forest Mode: The Original Approach](#2-forest-mode-the-original-approach)
3. [The Problems with Forest Mode](#3-the-problems-with-forest-mode)
4. [Bonsai Mode: A Smarter Approach](#4-bonsai-mode-a-smarter-approach)
5. [How Bonsai Stores the Flat Database](#5-how-bonsai-stores-the-flat-database)
6. [How Bonsai Handles Historic State (Rollback)](#6-how-bonsai-handles-historic-state-rollback)
7. [The Tradeoff: What Bonsai Gives Up](#7-the-tradeoff-what-bonsai-gives-up)
8. [Side-by-Side Comparison](#8-side-by-side-comparison)
9. [Summary and What Is Next](#9-summary-and-what-is-next)

---

## 1. The Storage Problem Every Ethereum Node Must Solve

When Besu processes a new block it must:

1. **Execute** every transaction in the block (which changes account balances, storage slots, contract code, etc.)
2. **Verify** that the resulting world state matches the `stateRoot` declared in the block header.
3. **Persist** the new world state to disk so it survives a restart.
4. **Answer queries** such as `eth_getBalance`, `eth_call`, `eth_getStorageAt` — sometimes for historic blocks, not just the current head.

The challenge is that Ethereum's world state is stored as a Merkle Patricia Trie (MPT) — a powerful structure for hashing but not designed for efficient disk storage. Besu solves this problem differently in each of its two storage modes: **Forest** and **Bonsai**.

---

## 2. Forest Mode: The Original Approach

Forest mode is the simpler, more straightforward approach. It stores **every version of every trie node that has ever been written**.

### How Forest Stores State

Think of the MPT as a tree. Every time a block is processed:

- Leaf nodes that changed get new hashes.
- Every ancestor node from that leaf up to the root gets a new hash too (write amplification — see file 02).
- **All of these new nodes are written to the database.**
- The old nodes from the previous block are **not deleted**.

The result is that after N blocks, the database contains N complete "snapshots" of the trie (or more precisely, all the individual nodes across all N versions, with massive sharing of unchanged sub-trees).

```
  Block 1 trie:             Block 2 trie:             Block 3 trie:
  Root_1                    Root_2                    Root_3
   |                         |                         |
  Branch_1A                 Branch_2A                 Branch_3A
  /        \                /        \                /        \
Leaf_X1  Leaf_Y1         Leaf_X2  Leaf_Y1         Leaf_X2  Leaf_Y2
(acct X  (acct Y         (acct X  (acct Y         (acct X  (acct Y
 bal=5)   bal=3)          bal=7)   bal=3)          bal=7)   bal=9)

  All nodes for all blocks exist in the DB at the same time.
  Unchanged nodes (Leaf_Y1, Leaf_X2) are shared between block versions.
```

In the actual database, each node is stored like this:

```
  Key:   hash(node_bytes)       <-- the Keccak-256 hash of the RLP-encoded node
  Value: node_bytes             <-- the raw node content
```

The "key is the hash of the value" pattern means that if two different blocks share an unchanged subtree, they both simply reference the same key — no duplication. Storage is naturally deduplicated.

### How Forest Answers Historic Queries

To get the state at block N, you need the `stateRoot` of block N. You then traverse the trie starting from that root. Since all nodes from all blocks are still in the database (keyed by hash), the traversal just works — you follow the node hashes from the block N root, and all the nodes you need are present.

```
  Query: "What was Alice's balance at Block 2?"

  1. Look up Block 2's header  -->  stateRoot = Root_2_hash
  2. DB[ Root_2_hash ]         -->  Root_2 node
  3. Follow nibbles of keccak(Alice's address) ...
  4. Arrive at Leaf for Alice  -->  balance = 7 ETH  ✓
```

This is elegant. Historic queries work automatically because nothing is ever deleted.

---

## 3. The Problems with Forest Mode

### Problem 1: Enormous Disk Usage

Because Forest keeps every node from every version of the trie, disk usage grows continuously. Each block may touch thousands of accounts, each touching several nodes on the path to the root. Over millions of blocks, this adds up to hundreds of gigabytes just for the trie node data.

```
  Ethereum mainnet as of 2024 (approximate):
  Forest node storage: ~1.5 TB+ and growing with every block
  Bonsai node storage: ~700 GB  (roughly half)
```

### Problem 2: Slow State Access (Deep Trie Traversals)

Reading the balance of a single account requires 8–12 random disk reads (one per trie node on the path from root to leaf). Random reads on spinning disks or even SSDs have latency. A busy RPC node serving many requests in parallel is often bottlenecked here.

```
  eth_getBalance(address) with Forest:
  ┌─────────────────────────────────────────────────────┐
  │  Read root node        (1 DB lookup)                │
  │  Read branch node      (1 DB lookup)                │
  │  Read extension node   (1 DB lookup)                │
  │  Read branch node      (1 DB lookup)                │
  │  ...                   (several more)               │
  │  Read leaf node        (1 DB lookup)                │
  │  Decode RLP, extract balance                        │
  └─────────────────────────────────────────────────────┘
  Total: ~8–12 random DB reads per account lookup
```

### Problem 3: Slow Block Import

Every block import triggers write amplification: changing N accounts forces rewriting N × D nodes (where D is the trie depth ≈ 8–9). On mainnet with hundreds of transactions per block, this means thousands of writes per block — all of which must be flushed to RocksDB.

### Problem 4: No Natural Pruning

If you want to run a "pruned" full node (one that only keeps recent state to save disk space), Forest makes this hard. To prune old nodes you need to walk through and determine which nodes are no longer referenced by any retained stateRoot — a complex and expensive garbage-collection operation.

---

## 4. Bonsai Mode: A Smarter Approach

Bonsai (named after the small, carefully maintained bonsai tree — a contrast to the sprawling "forest") was introduced to solve the disk space and performance problems of Forest mode.

### The Core Idea

Instead of storing every historical trie node, Bonsai stores:

1. **Only the current (head) state trie** — one trie, not a whole forest of historical tries.
2. **A flat database** — a direct key→value lookup for current account states and storage, bypassing trie traversal for common reads.
3. **Trie Logs** — compact diff records that describe what changed in each block, used to reconstruct any recent historical state by "rolling back" the flat DB.

```
  Forest approach:
  ┌────────────────────────────────────┐
  │ DB: ALL trie nodes from ALL blocks │
  │ (huge, but any block queryable)    │
  └────────────────────────────────────┘

  Bonsai approach:
  ┌──────────────────────┐  ┌───────────────────┐  ┌──────────────────────┐
  │ Current state trie   │  │ Flat DB           │  │ Trie Logs            │
  │ (one snapshot only)  │  │ (fast lookups for │  │ (diffs for recent    │
  │                      │  │  current state)   │  │  N blocks only)      │
  └──────────────────────┘  └───────────────────┘  └──────────────────────┘
```

---

## 5. How Bonsai Stores the Flat Database

The Flat Database is the most important new concept Bonsai introduces. It is a set of key-value tables (called **segments** or **column families** in RocksDB terminology) that store the current state directly — without needing to traverse the trie.

Bonsai uses three flat segments:

### ACCOUNT_INFO_STATE

Stores the current state of every account:

```
  Key:   hash(account_address)          -- 32 bytes (Keccak hash of the address)
  Value: RLP-encoded account state      -- { nonce, balance, storageRoot, codeHash }

  Example:
  Key:   0x4D95FBAF35Fc5A815983F9df94821C1c089DC02f  (hashed)
  Value: RLP{ nonce=5, balance=1.2eth, storageRoot=0x..., codeHash=0x... }
```

### ACCOUNT_STORAGE_STORAGE

Stores every storage slot of every contract:

```
  Key:   hash(account_address) ++ hash(storage_slot_key)   -- 64 bytes total
  Value: RLP-encoded storage value

  Example:
  Key:   0x4D95...c02f  (account hash)  ++  0x0000...0001  (slot 1 hash)
  Value: RLP{ 0x2A }   (value 42 at slot 1)
```

### CODE_STORAGE

Stores contract bytecode:

```
  Key:   hash(code)   OR   hash(account_address)    (configurable)
  Value: raw bytecode bytes
```

### Why is the flat DB so much faster?

With the flat DB, answering `eth_getBalance(address)` is just **one** database lookup:

```
  eth_getBalance(address) with Bonsai flat DB:
  ┌────────────────────────────────────────────────────┐
  │  key = keccak256(address)                          │
  │  value = DB[ ACCOUNT_INFO_STATE ][ key ]           │
  │  return RLP.decode(value).balance                  │
  └────────────────────────────────────────────────────┘
  Total: 1 DB lookup. No trie traversal needed.
```

Compare that to 8–12 lookups in Forest mode. This is why Bonsai is dramatically faster for current-state queries.

### What about the trie then?

Bonsai still maintains the full MPT — it is needed to compute the `stateRoot` hash and verify it against the block header. But for the vast majority of read queries, the flat DB is used instead. The trie is only traversed when you need to:

- Prove state membership (Merkle proofs)
- Verify the world state root
- Reconstruct state at a block beyond the trie log window

---

## 6. How Bonsai Handles Historic State (Rollback)

Bonsai only keeps one copy of the trie (current state). So how does it answer queries for old blocks?

The answer is **Trie Logs** — a log of what changed in each block, stored in the `TRIE_LOG_STORAGE` segment.

```
  Block N trie log contains:
  ┌────────────────────────────────────────────┐
  │ blockHash: 0xabc...                        │
  │ accountChanges:                            │
  │   Alice: { before: bal=5, after: bal=7 }   │
  │   Bob:   { before: bal=3, after: bal=1 }   │
  │ storageChanges:                            │
  │   ContractX, slot1: { before: 0, after: 99}│
  └────────────────────────────────────────────┘
```

To query state at Block N-3 (four blocks ago):

```
  Current state (head = Block N)
        |
        |  Apply trie log N   backwards  (undo block N changes)
        v
  State at Block N-1
        |
        |  Apply trie log N-1 backwards  (undo block N-1 changes)
        v
  State at Block N-2
        |
        |  Apply trie log N-2 backwards  (undo block N-2 changes)
        v
  State at Block N-3  <-- answer your query here
```

Each step is cheap because trie logs are compact diffs, not full snapshots.

### The Trie Log Window

Besu keeps trie logs for only a limited number of recent blocks (default: 512). This means:

- **Blocks within 512 of the head**: Bonsai can roll back using trie logs. Fast and accurate.
- **Blocks older than 512**: Bonsai cannot roll back — those trie logs have been pruned.

This is the fundamental limitation that **Bonsai Archive** was built to overcome.

---

## 7. The Tradeoff: What Bonsai Gives Up

Bonsai is not strictly better than Forest in every dimension. Here is what it sacrifices:

### Historic State Access

A standard Bonsai node can only answer historic queries for the last ~512 blocks. Queries beyond that window fail. An application that needs to call `eth_getBalance(address, blockNumber)` for a block from last year will get an error on a standard Bonsai node.

```
  eth_getBalance(Alice, blockNumber=1000000) on a standard Bonsai node
  at current head = 20000000:

  Gap = 20000000 - 1000000 = 19000000 blocks
  Trie log window = 512 blocks

  Result: ERROR - historic state not available
```

### More Complex Code

Bonsai is considerably more complex to implement and maintain than Forest. The flat DB, trie log management, rollback logic, and state healing all add complexity that Forest mode simply does not need.

### State Healing Requirement

If Bonsai's flat DB becomes inconsistent (e.g., due to a crash or a snap sync that didn't complete), a "healing" process is needed to scan and repair the flat DB by comparing it against the trie. Forest mode does not have this concept because the trie is always the source of truth.

---

## 8. Side-by-Side Comparison

```
  ┌──────────────────────────┬──────────────────────────┬──────────────────────────┐
  │ Property                 │ Forest Mode              │ Bonsai Mode              │
  ├──────────────────────────┼──────────────────────────┼──────────────────────────┤
  │ Disk usage               │ Very high (~1.5 TB+)     │ Lower (~700 GB)          │
  │ Current state reads      │ Slow (8-12 DB hops)      │ Fast (1 DB lookup)       │
  │ Block import speed       │ Slower (many trie writes)│ Faster (flat DB writes)  │
  │ Historic state access    │ Any block, always        │ Last ~512 blocks only    │
  │ Implementation complexity│ Simple                   │ Complex                  │
  │ State pruning            │ Difficult                │ Built-in (trie log prune)│
  │ Snap sync compatibility  │ Yes                      │ Yes                      │
  │ Archive node support     │ Yes (full history)       │ Only with Bonsai Archive │
  └──────────────────────────┴──────────────────────────┴──────────────────────────┘
```

### When to Choose Which

| Scenario | Recommended Mode |
|----------|-----------------|
| Running a validator / staker | Bonsai (fast, small disk) |
| Running an RPC endpoint for dApps (current state only) | Bonsai |
| Running a block explorer or analytics tool needing all history | Bonsai Archive or Forest |
| Running a full archive node | Bonsai Archive (new!) or Forest |

---

## 9. Summary and What Is Next

### Key Takeaways

- **Forest** is simple but expensive: it stores every trie node from every block, making historic queries trivial but disk usage enormous.
- **Bonsai** is efficient for current state: a flat DB gives O(1) account lookups, and only the current trie is kept. Trie logs allow rolling back to recent blocks.
- The critical limitation of Bonsai is that it cannot serve historic state older than the trie log window (~512 blocks by default).
- This limitation is exactly what **Bonsai Archive** solves.

### The Gap Bonsai Archive Fills

```
  Forest:        [block 0] .... [block N-512] .... [block N]
                  ✓ historic                          ✓ current
                  (huge disk, slow reads)

  Bonsai:        [block 0] .... [block N-512] .... [block N]
                  ✗ too old                 ✓ trie  ✓ flat DB
                                            log
                                            window

  Bonsai Archive:[block 0] .... [block N-512] .... [block N]
                  ✓ archive DB              ✓ trie  ✓ flat DB
                  (efficient key scheme)    log
                                            window
```

### Next Files in This Series

| File | Topic |
|------|-------|
| `04_bonsai_deep_dive.md` | Bonsai internals: DB segments, flat DB modes, trie log lifecycle |
| `05_trie_logs.md` | TrieLog in detail: structure, writing, reading, pruning |
| `06_bonsai_archive.md` | Bonsai Archive: the problem it solves and exactly how it works |
| `07_code_walkthrough.md` | Walking through the actual Besu source code for Bonsai Archive |