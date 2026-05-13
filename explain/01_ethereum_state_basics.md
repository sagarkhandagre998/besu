# Ethereum State Basics: A Foundation for Understanding Bonsai Archive

> **Who is this for?**
> This guide is for engineers who know what a blockchain is but are new to Ethereum
> internals and to how Besu stores data. By the end you will have a solid mental model
> of Ethereum state, and you will be ready to understand why the Bonsai archive storage
> format exists and what problem it solves.

---

## Table of Contents

1. [What Is an Ethereum Node and What Does It Store?](#1-what-is-an-ethereum-node-and-what-does-it-store)
2. [What Is the World State?](#2-what-is-the-world-state)
3. [What Is an Ethereum Account?](#3-what-is-an-ethereum-account)
4. [What Is Account Storage?](#4-what-is-account-storage)
5. [What Is a Block?](#5-what-is-a-block)
6. [How Does the stateRoot Commit to the Entire World State?](#6-how-does-the-stateroot-commit-to-the-entire-world-state)
7. [Why Does a Node Need to Store Historic State?](#7-why-does-a-node-need-to-store-historic-state)
8. [Full Node vs Archive Node](#8-full-node-vs-archive-node)
9. [Summary and What Is Next](#9-summary-and-what-is-next)

---

## 1. What Is an Ethereum Node and What Does It Store?

An **Ethereum node** is a program that participates in the Ethereum peer-to-peer network.
Its three core duties are:

- **Download** every block from its peers as the chain grows.
- **Validate and re-execute** every transaction to confirm the chain is honest.
- **Serve queries** from wallets, dApps, and explorers via the JSON-RPC API.

### The librarian analogy

Think of a node as a **librarian** in a very special library. The library has two
sections:

1. **The archive shelves** — an immutable, append-only record of every block ever
   produced (analogous to a newspaper archive: yesterday's edition never changes).
2. **The live index** — a continuously updated lookup table that tells you the current
   state of every account, so a reader can instantly ask "what is Alice's balance right
   now?" without flipping through every newspaper since 2015.

To answer both current and historical queries, a node stores several distinct kinds
of data:

```
┌───────────────────────────────────────────────────────────────────────┐
│                     What a Node Stores                                │
├──────────────────────────┬────────────────────────────────────────────┤
│  Data                    │  Why it is needed                          │
├──────────────────────────┼────────────────────────────────────────────┤
│  Block headers           │  Proof of chain history; contains stateRoot│
│  Block bodies            │  The ordered list of transactions          │
│  Receipts                │  Outcome of each tx (logs, gas, status)    │
│  World state             │  Current balances, nonces, code            │
│  Account storage         │  Internal key-value data of contracts      │
│  Contract bytecode       │  The EVM code of every deployed contract   │
└──────────────────────────┴────────────────────────────────────────────┘
```

The exact *amount* of history kept depends on whether the node is a **full node** or
an **archive node** — a distinction covered in section 8.

---

## 2. What Is the World State?

### The giant shared spreadsheet

Imagine Ethereum as a **giant shared spreadsheet** visible to everyone on the internet.
Each row is one Ethereum account. Every time a new block is processed, a batch of rows
is updated: balances change, contract memory changes, new contracts are created.

The **world state** is a complete snapshot of that spreadsheet at a single moment in
time — specifically, after a particular block has been fully applied.

```
┌──────────────────────────────────────────────────────────────────────┐
│                 WORLD STATE  (snapshot after block 20,000,000)       │
├────────────────────────────┬─────────────────────────────────────────┤
│  Address                   │  Account data                           │
├────────────────────────────┼─────────────────────────────────────────┤
│  0xABCD...  (Alice)        │  balance: 5 ETH,  nonce: 12            │
│  0x1234...  (Bob)          │  balance: 2 ETH,  nonce: 3             │
│  0xUNI...   (Uniswap)      │  balance: 0,  code: <bytecode>         │
│                            │  storageRoot: 0x7f3a...                │
│  0xDAI...   (DAI token)    │  balance: 0,  code: <bytecode>         │
│                            │  storageRoot: 0x9c4b...                │
│  ... (millions more rows)  │  ...                                   │
└────────────────────────────┴─────────────────────────────────────────┘
```

### State transitions

Processing a block transforms one world state into the next:

```
World State          Block N             World State
at block N-1   ───► (transactions)  ───► at block N
  (old snap)         are applied          (new snap)
```

Each block therefore defines a **before** state and an **after** state. The after state
is committed to by a single 32-byte hash stored in the block header called the
`stateRoot` (see section 6).

### Key insight: it is a tree, not a flat file

The world state is **not** stored as a flat CSV. It is organised as a special tree
structure called a **Merkle Patricia Trie** so that the entire contents can be
summarised into a single 32-byte hash. More on that in section 6.

---

## 3. What Is an Ethereum Account?

Every participant — whether a person's wallet or a deployed smart contract — is an
**account**. An account is identified by a 20-byte **address** and contains exactly
four fields:

```
┌──────────────────────────────────────────────────────────────────────┐
│                        Ethereum Account                              │
│  address: 0xUNI...  (Uniswap Router)                                 │
├──────────────────────┬───────────────────────────────────────────────┤
│  Field               │  Value and meaning                            │
├──────────────────────┼───────────────────────────────────────────────┤
│  nonce               │  1  — number of txs sent (EOA) or contracts   │
│                      │       created (contract account)              │
│  balance             │  0 Wei  — ETH owned by this account           │
│  storageRoot         │  0x7f3a...  ──────────────────────────────►   │
│                      │       root hash of this contract's storage    │
│                      │       trie (empty hash for plain wallets)     │
│  codeHash            │  0x9c4b...  ──────────────────────────────►   │
│                      │       keccak-256 of this contract's EVM       │
│                      │       bytecode (empty hash for plain wallets) │
└──────────────────────┴───────────────────────────────────────────────┘
```

### Two flavours of account

**Externally Owned Account (EOA)** — a wallet controlled by a private key:

- `nonce` — increments with each outgoing transaction; prevents replay attacks.
- `balance` — ETH held.
- `storageRoot` — always the hash of an empty trie; EOAs have no storage.
- `codeHash` — always `keccak256("")`; EOAs have no code.

**Contract Account** — a deployed smart contract:

- `nonce` — increments each time the contract itself deploys another contract.
- `balance` — ETH held by the contract (e.g. a vault contract).
- `storageRoot` — root of the contract's own storage trie; changes as storage changes.
- `codeHash` — points to the stored bytecode; **immutable** after deployment.

```
  EOA (Alice's wallet)          Contract (Uniswap Router)
  ┌─────────────────────┐       ┌──────────────────────────┐
  │ nonce:       12     │       │ nonce:        1           │
  │ balance:  5 ETH     │       │ balance:      0 Wei       │
  │ storageRoot: empty  │       │ storageRoot: 0x7f3a... ──►│──► storage trie
  │ codeHash:    empty  │       │ codeHash:   0x9c4b... ──►│──► bytecode
  └─────────────────────┘       └──────────────────────────┘
```

---

## 4. What Is Account Storage?

A smart contract often needs to remember things between calls. A token contract must
remember how many tokens each address owns; a lending protocol must remember collateral
positions. This persistent memory lives in the contract's **account storage**.

### Filing cabinet analogy

Imagine the contract has a **cabinet of numbered drawers**. Each drawer holds one
256-bit value. When the contract's Solidity code declares `uint256 totalSupply`, the
compiler assigns it drawer number 0. The next state variable gets drawer 1, and so on.
For dynamic data like mappings, the drawer number is computed by hashing the mapping
key together with the base slot number.

### Structure

Account storage is a **key-value store** where:

- **Keys** are 256-bit (32-byte) slot indices.
- **Values** are 256-bit (32-byte) words.
- The entire store is itself a Merkle Patricia Trie, so it has a root hash —
  the `storageRoot` field in the parent account.

```
  Contract: 0xDAI...  (DAI Stablecoin)
  ┌────────────────────────────────────────────────────────────────┐
  │                      Account Storage                          │
  │           (a separate key-value trie per contract)            │
  ├──────────────────────────────┬─────────────────────────────────┤
  │  Slot (key, 32 bytes)        │  Value (32 bytes)               │
  ├──────────────────────────────┼─────────────────────────────────┤
  │  0x000...000  (slot 0)       │  0x000...0001  totalSupply      │
  │  0x000...001  (slot 1)       │  0xABCD...     owner address    │
  │  keccak256(Alice addr, 3)    │  0x000...2710  Alice's DAI bal  │
  │  keccak256(Bob   addr, 3)    │  0x000...03E8  Bob's   DAI bal  │
  │  ...                         │  ...                            │
  └──────────────────────────────┴─────────────────────────────────┘
                        │
                        │ root hash of this trie
                        ▼
              storageRoot stored in the DAI account
                        │
                        │ account stored in state trie
                        ▼
                    stateRoot  (in block header)
```

The cascading hash chain means: if even **one storage slot** changes, the
`storageRoot` changes, which changes the account's entry in the state trie, which
changes the `stateRoot` in the block header. Nothing can be silently altered.

---

## 5. What Is a Block?

A **block** is the fundamental unit of state transition in Ethereum. It bundles a set
of transactions and records, via the `stateRoot`, exactly what the world looked like
after those transactions were applied.

A block has three main parts: the **header**, the **transaction list**, and the
**receipts**.

### 5a. Block Header

The header is a compact, fixed-size summary of the block. Key fields:

```
┌────────────────────────────────────────────────────────────────────┐
│                        Block Header                                │
├──────────────────────────┬─────────────────────────────────────────┤
│  parentHash              │  Hash of the previous block's header.   │
│                          │  This is what forms the "chain".        │
├──────────────────────────┼─────────────────────────────────────────┤
│  number                  │  Block height (0, 1, 2, …, 20000000, …) │
├──────────────────────────┼─────────────────────────────────────────┤
│  timestamp               │  Unix seconds when the block was built  │
├──────────────────────────┼─────────────────────────────────────────┤
│  stateRoot        ◄────  │  Root hash of the world state AFTER     │
│                          │  all transactions are applied.          │
│                          │  This is the key field for state.       │
├──────────────────────────┼─────────────────────────────────────────┤
│  transactionsRoot        │  Root hash of the transactions trie     │
├──────────────────────────┼─────────────────────────────────────────┤
│  receiptsRoot            │  Root hash of the receipts trie         │
├──────────────────────────┼─────────────────────────────────────────┤
│  gasLimit / gasUsed      │  Block capacity and actual gas consumed  │
├──────────────────────────┼─────────────────────────────────────────┤
│  baseFeePerGas           │  EIP-1559 base fee (post-London)        │
└──────────────────────────┴─────────────────────────────────────────┘
```

### 5b. Transaction List

The body contains the ordered list of transactions. Order matters — it determines
which transaction gets to execute first and therefore which account states the second
transaction sees. Each transaction includes:

- `from` — sender, recovered from the ECDSA signature.
- `to` — recipient address, or null if this is a contract deployment.
- `value` — amount of ETH (in Wei) to transfer to `to`.
- `data` — calldata sent to the contract, or init bytecode for deployments.
- `nonce` — must equal the sender's current account nonce (prevents replays).
- Gas fee fields (`gasLimit`, `maxFeePerGas`, `maxPriorityFeePerGas`).

### 5c. Receipts

After the EVM executes a transaction it produces a **receipt**:

- `status` — 1 (success) or 0 (reverted).
- `gasUsed` — cumulative gas consumed up to and including this transaction.
- `logs` — the events emitted by contracts during execution (used by dApp frontends
  and indexers).

### Full picture: one block

```
┌──────────────────────────────────────────────────────────────────────────┐
│                              BLOCK  N                                    │
├──────────────────────────────────────────────────────────────────────────┤
│  HEADER                                                                  │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  parentHash       : 0xdeadbeef...  (hash of block N-1 header)     │  │
│  │  number           : N                                              │  │
│  │  timestamp        : 1_713_000_000                                  │  │
│  │  stateRoot        : 0xabcd1234...  ◄── commits to ALL state       │  │
│  │  transactionsRoot : 0x1111...                                      │  │
│  │  receiptsRoot     : 0x2222...                                      │  │
│  │  gasUsed          : 12_000_000                                     │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  TRANSACTIONS                           RECEIPTS                        │
│  ┌──────────────────────────┐           ┌───────────────────────────┐  │
│  │ [0] Alice → Bob          │           │ [0] status: ok            │  │
│  │     value: 1 ETH         │           │     gasUsed: 21_000        │  │
│  ├──────────────────────────┤           ├───────────────────────────┤  │
│  │ [1] Alice → Uniswap      │           │ [1] status: ok            │  │
│  │     data: swap(USDC→ETH) │           │     gasUsed: 150_000       │  │
│  │                          │           │     logs: [Swap(...)]      │  │
│  ├──────────────────────────┤           ├───────────────────────────┤  │
│  │ [2] Bob → null           │           │ [2] status: ok            │  │
│  │     data: <init bytecode>│           │     gasUsed: 800_000       │  │
│  │     (deploys DAI clone)  │           │     logs: []               │  │
│  └──────────────────────────┘           └───────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 6. How Does the `stateRoot` Commit to the Entire World State?

A single 32-byte hash — the `stateRoot` in the block header — cryptographically
**commits** to every account balance, every contract storage slot, and every byte of
contract code in the entire world state. This is one of the most elegant parts of
Ethereum's design.

### The tool: Merkle Patricia Trie (MPT)

A **trie** (rhymes with "try", short for re**trie**val) is a tree where the **path
from the root to a leaf encodes the lookup key**, and the **leaf holds the value**.

Ethereum's MPT adds two ideas on top:

1. **Patricia encoding** — long shared key prefixes are compressed into single nodes
   to avoid wasting space.
2. **Merkle hashing** — every node stores the **cryptographic hash of its children**.
   This means the root node's hash summarises the entire tree. Change any leaf and the
   hash ripples all the way back up to the root.

In the **state trie**:

- The **key** for each entry is `keccak256(account_address)` (32 bytes = 64 hex
  nibbles, used as the path).
- The **value** at each leaf is the RLP-encoded account (nonce, balance, storageRoot,
  codeHash).
- Every internal node's hash depends on its children.
- The **root node's hash** is the `stateRoot` published in the block header.

```
                        ┌─────────────────────┐
                        │      ROOT NODE       │
                        │   hash = stateRoot   │
                        └──────────┬──────────┘
                                   │
               ┌───────────────────┼───────────────────┐
               ▼                   ▼                   ▼
         ┌──────────┐        ┌──────────┐        ┌──────────┐
         │ Branch A │        │ Branch B │        │ Branch C │
         │ hash=H_A │        │ hash=H_B │        │ hash=H_C │
         └────┬─────┘        └────┬─────┘        └──────────┘
              │                   │
       ┌──────┴──────┐     ┌──────┴──────┐
       ▼             ▼     ▼             ▼
  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐
  │  Alice  │ │   Bob   │ │ Uniswap │ │   DAI   │
  │ account │ │ account │ │ account │ │ account │
  └─────────┘ └─────────┘ └─────────┘ └─────────┘

  Change Alice's balance → H_A changes → stateRoot changes.
  Change a DAI storage slot → DAI storageRoot changes →
    DAI's leaf changes → H_B changes → stateRoot changes.
```

### Why this matters

1. **Tamper evidence** — The `stateRoot` is stored in the block header, which is
   hashed into the next block's `parentHash`. Any silent alteration of historical
   state is detectable by anyone who checks the hash.

2. **Merkle proofs** — You can prove a single account's balance to someone who only
   knows the `stateRoot`, by providing the O(log n) nodes along the path. They do not
   need to download the whole state.

3. **Archive queries** — Because every block header permanently records its
   `stateRoot`, a node that stores the historical trie can reconstruct the full world
   state at any block number and answer queries about it.

### How accounts link to storage

Each account's `storageRoot` is itself the root of a separate MPT:

```
  stateRoot  (in block header)
      │
      │  state trie
      ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  leaf: 0xDAI...                                             │
  │  value: {                                                   │
  │    nonce:       1,                                          │
  │    balance:     0,                                          │
  │    storageRoot: 0x9c4b... ──────────────────────────────►   │
  │    codeHash:    0x7f3a... ──────────────────────────────►   │
  │  }                                                          │
  └─────────────────────────────────────────────────────────────┘
           │                               │
           │ storage trie for DAI          │ DAI bytecode
           ▼                               ▼
  ┌────────────────────────┐      ┌─────────────────────────┐
  │ slot 0: totalSupply    │      │  PUSH1 0x60             │
  │ slot 1: owner          │      │  PUSH1 0x40             │
  │ keccak(Alice, 3): bal  │      │  MSTORE                 │
  │ ...                    │      │  ...                    │
  └────────────────────────┘      └─────────────────────────┘
```

---

## 7. Why Does a Node Need to Store Historic State?

You might wonder: *"Once block N+1 arrives and updates the state, can't I throw away
the block N state? I only need the latest snapshot!"*

A pruning full node does exactly this, and it works perfectly for use cases that only
need the current chain tip. But many important use cases require historical state.

### Real-world queries that require historical state

**Historical balance lookup**

```
eth_getBalance("0xAlice...", blockNumber="0xE4E1C0")
```

A blockchain explorer, an auditor, or a tax tool needs to know what Alice's balance
was at block 15,000,000 — two years ago. Without the state at that block, this
question cannot be answered.

**Smart-contract simulation at a past block**

```
eth_call(
  { to: "0xUniswap...", data: "0x..." },
  blockNumber="0xE4E1BF"
)
```

A developer is debugging: *"Why did this trade fail last March?"* The EVM needs the
exact contract storage from that specific block to replay the call faithfully.
Current storage will give a wrong (or reverted) answer.

**Event and log reprocessing**

An indexer rebuilding its database from scratch replays every transaction from
genesis. Each replayed transaction needs the state from the block immediately
before it — otherwise the execution context is wrong and computed results
(e.g. which address received tokens) may be incorrect.

**Protocol research and security audits**

Security researchers examining a past exploit need to see the exact storage layout
at the moment of the attack. Protocol developers testing a proposed change need to
run it against historical state to verify it would have been safe.

### Visualising what is lost without history

```
  Block timeline:

  0 ──► 1 ──► ... ──► 14,999,999 ──► 15,000,000  (current tip)

  Full node (pruned):
  ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░████████████
                                       only recent
                                       state kept

  Archive node:
  ████████████████████████████████████████████████
  state at EVERY block is accessible

  ░ = state discarded    █ = state retained
```

A full node can answer `eth_getBalance(addr, "latest")` instantly but will return
an error for `eth_getBalance(addr, 15_000_000)` if that block is in the pruned
range. An archive node answers both.

---

## 8. Full Node vs Archive Node

The distinction comes down to **how much state history is retained**.

### Full Node

A full node downloads and validates every block since genesis. It re-executes every
transaction, verifying that the resulting `stateRoot` matches the one in the header.
However, once a block is buried deeply enough that a chain reorganisation cannot reach
it, the node **discards** the intermediate state. Only the current world state (and
perhaps the last ~128 blocks for reorg safety) is kept.

| Property | Value |
|---|---|
| Validates all blocks | Yes |
| Stores all block headers and bodies | Yes |
| Stores current world state | Yes |
| Stores state at arbitrary past blocks | No |
| Typical disk usage (2024) | ~1 TB |
| Can answer `eth_getBalance(addr, "latest")` | Yes |
| Can answer `eth_getBalance(addr, 15_000_000)` | No (if pruned) |

### Archive Node

An archive node does everything a full node does and additionally retains the world
state at **every single block** throughout history.

| Property | Value |
|---|---|
| Validates all blocks | Yes |
| Stores all block headers and bodies | Yes |
| Stores current world state | Yes |
| Stores state at arbitrary past blocks | Yes |
| Typical disk usage (2024, naive storage) | ~15–20 TB |
| Can answer `eth_getBalance(addr, "latest")` | Yes |
| Can answer `eth_getBalance(addr, 15_000_000)` | Yes |

### Side-by-side comparison

```
  ┌─────────────────────────────────────────────────────────────────────┐
  │            FULL NODE                  ARCHIVE NODE                  │
  │                                                                     │
  │  Block headers: █████████████    Block headers: █████████████       │
  │  Block bodies:  █████████████    Block bodies:  █████████████       │
  │  World state:                █   World state:   █████████████       │
  │               (latest only)                    (every block)        │
  │                                                                     │
  │  ✓  eth_getBalance(addr, "latest")                                  │
  │  ✗  eth_getBalance(addr, 15_000_000)   ✓  (archive only)           │
  │  ✓  eth_call(..., "latest")                                         │
  │  ✗  eth_call(..., 15_000_000)          ✓  (archive only)           │
  │  ✓  eth_getLogs(filter, recent)                                     │
  │  ✗  eth_getLogs(filter, full history)  ✓  (archive only)           │
  └─────────────────────────────────────────────────────────────────────┘
```

### The storage challenge for archive nodes

Storing the full state trie at every block naively is extremely expensive. A mature
Ethereum state trie contains tens of millions of nodes. If block N modifies 500
accounts and each account change touches several trie nodes, a naive copy-on-write
approach stores thousands of trie nodes per block — multiplied across 20+ million
blocks.

```
  Naive approach — full snapshot per block:

  Block 1:   [complete state trie copy]       ~  X GB
  Block 2:   [complete state trie copy]       ~  X GB
  Block 3:   [complete state trie copy]       ~  X GB
  ...
  Block N:   [complete state trie copy]       ~  X GB
  ─────────────────────────────────────────────────────
  Total: O(N × state_size)  →  enormous, impractical


  Bonsai approach — diff (changeset) per block:

  Block 1:   [genesis state]                  ~  X GB  (once)
  Block 2:   [changed: Alice balance, slot Y] ~  KB
  Block 3:   [changed: Uniswap storage slots] ~  KB
  ...
  Block N:   [changed: ...]                   ~  KB
           + [current flat state]             ~  X GB  (once)
  ─────────────────────────────────────────────────────
  Total: O(state_size + N × diff_size)  →  manageable
```

Storing only the *differences* (diffs) between consecutive states — rather than full
snapshots — is the core idea behind the **Bonsai** storage format in Besu. It makes
archive storage practical on commodity hardware and is covered in detail in the next
document.

---

## 9. Summary and What Is Next

### Quick-reference table

| Concept | One-sentence definition |
|---|---|
| **World State** | A snapshot of every Ethereum account at a given block |
| **Account** | An address with nonce, balance, storageRoot, and codeHash |
| **Account Storage** | Per-contract key-value slots that persist across calls |
| **Block Header** | Contains stateRoot, parentHash, number, and other metadata |
| **Block Body** | Ordered list of transactions included in the block |
| **Receipt** | Outcome of a transaction: status, gas used, emitted logs |
| **stateRoot** | 32-byte Merkle root that commits to the entire world state |
| **Merkle Patricia Trie** | The tree structure that makes stateRoot possible |
| **Full Node** | Validates everything; keeps only recent/current state |
| **Archive Node** | Validates everything; keeps state at every historical block |
| **Bonsai** | Besu's diff-based storage format that makes archive practical |

### Mental model to carry forward

```
  BLOCK HEADER  contains  stateRoot
                               │
                               │ is the root hash of
                               ▼
                         STATE TRIE
                        (one leaf per account)
                               │
                   ┌───────────┴────────────┐
                   ▼                        ▼
             EOA ACCOUNT             CONTRACT ACCOUNT
           nonce, balance            nonce, balance
           (no storage)              codeHash ──► bytecode
                                     storageRoot
                                          │
                                          │ is the root hash of
                                          ▼
                                    STORAGE TRIE
                                  (one leaf per slot)
```

### What is next?

- **`02_merkle_patricia_trie.md`** — A deep dive into how the Merkle Patricia Trie
  works: extension nodes, branch nodes, leaf nodes, and how Ethereum encodes and
  traverses them. You will come away able to hand-trace a trie lookup and understand
  why trie node sharing is both a feature and a storage challenge.

- **`03_state_storage_formats.md`** — How Besu (and other clients) historically stored
  the state trie on disk, why naive copy-on-write becomes unmanageable at archive
  scale, and what requirements a better solution must meet.

- **`04_bonsai_overview.md`** — The Bonsai design: flat world state, trie logs (diffs),
  and how they combine to give you fast current-state reads *and* efficient historical
  state reconstruction.