# A2 — How Does Besu Solve These Problems?

> **Series:** Besu Explained from First Principles
> **File:** `A2_how_besu_solves_it.md`
> **Prerequisites:** [A1 — What is the Business Requirement?](./A1_business_requirement.md)
> **Next:** [A3 — Deep Architecture Walkthrough](./A3_deep_architecture.md)

---

## Introduction

In [A1](./A1_business_requirement.md) we established *what* Besu must accomplish: serve two distinct customer types (public Ethereum participants and enterprise/consortium operators) with a single, modular, enterprise-grade Ethereum execution client written in Java.

This document answers the *how*. For every major problem category identified in A1 — EVM execution, storage, sync, APIs, networking, enterprise features, modularity, and performance — we walk through the specific design decisions, architectural patterns, key classes, and engineering trade-offs that Besu's engineers chose. Where possible, we trace the code path so you can navigate the real source tree with confidence.

Think of this document as a guided tour through the engine room of a very large ship. After reading it, you will understand not just what each component does, but *why* it was built that way.

---

## Section 1: Solving EVM Execution

### 1.1 The Core Challenge

The Ethereum Virtual Machine is a deterministic, stack-based, 256-bit virtual machine. Every full node in the world must execute every transaction and arrive at the *exact same* resulting state — down to the last byte of every storage slot. Any divergence means the node is on a different chain. This demands:

- Bit-for-bit correctness across thousands of opcodes and precompiles
- Hard-fork awareness: the rules change at precise block numbers (or timestamps, post-Merge), and old blocks must be re-executable under old rules
- Gas metering: every instruction costs gas, and execution must halt at exactly the right point
- Reversion semantics: if a transaction runs out of gas or hits a `REVERT`, *all* state changes from that transaction must be rolled back, but the gas is still consumed

### 1.2 The Standalone EVM Module

Besu's answer is to treat the EVM as a **first-class, independently deployable library**, housed in the `evm/` subproject. This is not merely a code organisation choice — it means:

- The EVM can be imported and used without any storage, networking, or RPC infrastructure
- External tools (fuzzers, test harnesses, academic research tools) can use Besu's EVM directly
- The Ethereum Foundation's `ethereum/tests` reference test suite can be run against just the `evm/` module, catching bugs before they ever interact with storage or P2P code
- The `evmtool` CLI utility is built on top of this module, letting developers run raw EVM bytecode from the command line for debugging

Key classes in `evm/`:
- `EVM` — the main execution loop; reads opcodes from bytecode and dispatches to `Operation` implementations
- `Operation` — interface implemented by every opcode (`AddOperation`, `SloadOperation`, `CallOperation`, etc.)
- `MessageFrame` — a single call frame on the EVM call stack; holds the program counter, stack, memory, remaining gas, and a reference to the world state view
- `GasCalculator` — computes the gas cost of each operation; *each hard fork has its own `GasCalculator` subclass* because gas schedules change across upgrades
- `PrecompileContractRegistry` — maps well-known addresses (0x01–0x0a and beyond) to native Java implementations of precompiled contracts (ECRECOVER, SHA256, BN128 pairing, BLAKE2, KZG point evaluation, etc.)

### 1.3 ProtocolSpec: Bundling Rules Per Hard Fork

A `ProtocolSpec` is the central configuration object for a *single Ethereum upgrade epoch*. It bundles together everything that can differ between hard forks:

```
ProtocolSpec {
  EVM                        // opcode set + gas calculator for this fork
  TransactionValidator       // what makes a transaction valid at this fork
  TransactionProcessor       // how to execute transactions
  BlockHeaderValidator       // block header validation rules
  BlockBodyValidator         // block body (uncle/withdrawal) validation rules
  BlockImporter              // orchestrates full block import
  BlockProcessor             // applies all txs in a block to the world state
  BlockReward                // mining/staking reward schedule
  PrecompileContractRegistry // active precompiles for this fork
  WithdrawalsValidator       // post-Shanghai: validates EIP-4895 withdrawals
  ... and many more
}
```

The elegance here is that **the same code path handles every block ever produced on Ethereum**, from block 0 (Frontier rules) to the latest Cancun/Prague block, because the caller always first resolves the correct `ProtocolSpec` for the block's number/timestamp, then delegates all validation and execution to it.

### 1.4 ProtocolSchedule: A Timeline of ProtocolSpecs

`ProtocolSchedule` is a sorted map from activation criteria (block number pre-Merge, timestamp post-Merge) to `ProtocolSpec`. Its primary API is:

```
ProtocolSpec getByBlockHeader(BlockHeader header)
```

The implementation walks the schedule and returns the most recent `ProtocolSpec` whose activation point is ≤ the header's block number/timestamp.

`MainnetProtocolSchedule` is the concrete implementation for Ethereum Mainnet. It hardcodes every hard fork activation:

| Fork         | Activation Point              |
|--------------|-------------------------------|
| Frontier     | Block 0                       |
| Homestead    | Block 1,150,000               |
| DAO Fork     | Block 1,920,000               |
| Tangerine Whistle | Block 2,463,000          |
| Spurious Dragon   | Block 2,675,000          |
| Byzantium    | Block 4,370,000               |
| Constantinople | Block 7,280,000             |
| Istanbul     | Block 9,069,000               |
| Berlin       | Block 12,244,000              |
| London (EIP-1559) | Block 12,965,000         |
| Paris (Merge) | Terminal Total Difficulty     |
| Shanghai     | Timestamp 1,681,338,455       |
| Cancun       | Timestamp 1,710,338,135       |
| Prague       | Timestamp (TBD)               |

For private/consortium networks, operators define their own fork schedule in the genesis file using `config` fields like `"berlinBlock": 0` — meaning "activate Berlin rules from genesis". This is how Besu supports private networks running modern EVM features without any Proof-of-Work history.

### 1.5 Transaction Execution: The Full Code Path

When a block arrives, `BlockProcessor.processBlock()` iterates over transactions and calls `TransactionProcessor.processTransaction()` for each. The high-level steps are:

1. **Pre-validation**: nonce check, sender balance ≥ gas limit × gas price + value, intrinsic gas check
2. **Gas purchase**: deduct `gasLimit × effectiveGasPrice` from sender's balance upfront
3. **Create or call**: if `to` is null, it's a contract creation (runs initcode, stores resulting bytecode); otherwise it's a message call
4. **EVM execution**: `EVM.runToHalt()` iterates opcodes until STOP, RETURN, REVERT, or out-of-gas
5. **Gas refund**: unused gas is returned to the sender; EIP-3529 (London) caps refunds at 20% of gas used
6. **Fee distribution**: base fee is burned (EIP-1559); priority fee goes to the block coinbase/validator
7. **State commit or revert**: on success, changes are committed to the pending world state; on REVERT or OOG, they are discarded (but gas is not refunded)
8. **Receipt generation**: a `TransactionReceipt` is produced containing logs, gas used, and status (1 = success, 0 = failure)

The `WorldUpdater` abstraction is key here. It wraps the actual world state and provides a *copy-on-write* view: changes accumulate in memory, and only on explicit `commit()` are they pushed to the parent updater or ultimately to the backing storage. This makes reversion cheap — just discard the updater.

### 1.6 Parallel Transaction Execution

Introduced in 2024, parallel transaction execution is one of Besu's most significant performance innovations.

**The idea**: Most transactions in a block touch *different* accounts and storage slots. If we can identify which transactions conflict (i.e., two transactions read or write the same storage location), we can run non-conflicting transactions on multiple threads simultaneously.

**The approach — optimistic execution with conflict detection**:
1. Speculatively execute all transactions in parallel using multiple worker threads, each with its own `WorldUpdater` (an isolated state snapshot)
2. After all workers finish, a *conflict detector* inspects the read/write sets of each transaction. A conflict exists when transaction B reads a storage slot that transaction A (earlier in the block) wrote to, *and* both ran in parallel
3. Conflicting transactions are re-executed *sequentially* in the correct order, using the committed state from all prior transactions as their starting point
4. Non-conflicting results are merged into the final block state

**Trade-off**: In the best case (lots of independent DeFi transactions), parallel execution provides near-linear speedup with core count. In the worst case (a block full of transactions to the same popular contract like Uniswap), almost every transaction conflicts and you pay the overhead of the speculative execution plus the re-execution. Real-world blocks tend toward the optimistic case for most block types.

---

## Section 2: Solving Storage

### 2.1 The Scale Problem

Ethereum's world state is a key-value mapping from 20-byte account addresses to account objects (nonce, balance, code hash, storage root). The storage of every smart contract is itself a sub-trie. By 2024, the full archive state exceeded 15 TB; even a pruned "recent state only" node requires hundreds of GB.

Besu must:
- Store this data durably and efficiently
- Retrieve any account or storage slot in milliseconds (RPC calls and block processing depend on it)
- Support rolling back the state to a previous block (needed for chain reorganisations and historical queries)
- Ideally, keep storage footprint manageable for operators running on commodity hardware

### 2.2 The Storage Abstraction Layer

Before discussing specific storage formats, it's important to understand that Besu insulates almost all business logic from storage details via a clean interface hierarchy:

```
KeyValueStorage          (raw bytes key → bytes value, put/get/delete)
  └── RocksDBKeyValueStorage    (default production backend)
  └── InMemoryKeyValueStorage   (tests, ephemeral nodes)
  └── SegmentedKeyValueStorage  (multiple logical namespaces in one DB)

WorldStateStorage        (semantic wrapper: storeAccount, getAccount, etc.)
  └── BonsaiWorldStateKeyValueStorage
  └── ForestWorldStateKeyValueStorage
```

This means:
- Switching storage backends (e.g., a custom plugin providing a distributed KV store) requires only implementing `KeyValueStorage` — no changes to EVM or sync code
- Tests run against `InMemoryKeyValueStorage` without needing RocksDB on the CI machine
- The `StorageService` plugin API exposes `KeyValueStorageFactory` so plugins can register entirely new storage backends

### 2.3 RocksDB: The Default Backend

RocksDB (developed at Meta, open-sourced) is an LSM-tree-based (Log-Structured Merge-tree) key-value store written in C++, wrapped for Java via JNI. Besu chose it because:

- **Write throughput**: LSM trees batch random writes into sequential writes, which are dramatically faster on SSDs
- **Read performance**: column families allow logically separate namespaces (blockchain data, state data, receipts) in one DB instance, with independent compaction tuning
- **Maturity**: used by Ethereum clients, databases, and distributed systems worldwide
- **Bloom filters**: RocksDB's bloom filters make point lookups fast even for large datasets
- **Compression**: LZ4/Snappy/Zstd compression at the block level reduces disk I/O

Besu's `RocksDBConfiguration` exposes tuning knobs (block cache size, max open files, background compaction threads) that expert operators can adjust for their hardware.

**Trade-off**: RocksDB uses JNI, which adds complexity (native library bundling, potential JVM crash risks from native code bugs). The alternative would be a pure-Java store, but nothing available at Besu's inception offered comparable performance. The JNI boundary is carefully managed — all RocksDB operations are done through a thin, well-tested wrapper.

### 2.4 Forest of Tries (Archive Mode)

The traditional Ethereum storage model — which Besu calls "Forest of Tries" — stores every trie node by its Keccak-256 hash (content-addressed storage). This is the canonical representation described in the Yellow Paper.

Properties:
- Every unique trie node ever computed is stored permanently (content-addressed, so if the same subtree appears in two blocks, it's stored only once)
- You can retrieve the complete state at *any* historical block by walking the trie from the block's state root
- Required for archive nodes that must answer `eth_getStorageAt` at arbitrary historical blocks
- **Downside**: Ethereum's state trie has undergone hundreds of millions of state transitions; the raw trie node storage for a full archive node is enormous (15+ TB)

Use case: archive nodes, block explorers, analytics tools.

### 2.5 Bonsai Tries (Default for Most Nodes)

Bonsai Tries is Besu's proprietary storage innovation, designed to serve the needs of validator nodes and RPC providers that need *current* state quickly but don't need every historical state.

**Core insight**: For a node that only needs recent state (say, the last 512 blocks), we don't need to store every historical trie node. Instead, we can store:
1. The **flat database**: a direct mapping from account address → account value, and from `(address, storageKey)` → storage value. This allows O(1) lookups without traversing any trie.
2. The **trie structure** for the *current* state root only — needed for generating Merkle proofs
3. **Trie Logs**: a per-block journal recording exactly which accounts and storage slots changed in that block, and what their old and new values were

**How queries work in Bonsai**:
- `eth_getBalance(address)` for the current head: direct flat DB lookup, extremely fast
- `eth_getBalance(address, blockNumber)` for a recent block: start from the flat DB (current state), then *replay* trie logs backwards (or forwards) to reconstruct state at the target block

**How reorgs work in Bonsai**:
- When a chain reorg occurs (a different fork becomes canonical), Besu replays the trie logs for the reverted blocks in reverse to undo their changes, then applies the trie logs for the new canonical blocks forward
- This is far cheaper than re-executing all the transactions from scratch

**Trade-offs**:
- Bonsai cannot serve arbitrary historical state queries beyond its trie log retention window (configurable, default ~512 blocks)
- Archive mode with Bonsai is possible (retain all trie logs) but the storage advantage over Forest diminishes for very deep history
- Bonsai's flat DB layout is a Besu-specific format; migrating to another client requires a full re-sync

**Storage savings**: In practice, Bonsai reduces state storage from ~1 TB (Forest pruned) to roughly 650–750 GB for a full Mainnet node, with continued improvement as the implementation matures.

### 2.6 Trie Log: The Change Journal

The Trie Log is arguably the most important data structure in Bonsai that most developers overlook. For every imported block, Besu writes a `TrieLog` entry containing:

- The block hash (key)
- A list of `(accountKey, oldValue, newValue)` tuples for every account that changed
- A list of `(accountKey, storageKey, oldValue, newValue)` tuples for every storage slot that changed
- The old and new code for any accounts whose code changed (contract deployments)

This journal serves double duty:
1. **State rollback** for reorgs (replay logs in reverse)
2. **State fast-forward** for skipping block re-execution when rolling forward (replay logs in forward order)

---

## Section 3: Solving Sync

### 3.1 The Bootstrap Problem

A new node joining the Ethereum network has zero knowledge of the chain. It must somehow download and verify ~19 million blocks' worth of transactions and arrive at exactly the same 32-byte state root as every other node — without trusting any single peer. This is the bootstrap problem, and it has become harder over time as the chain grows.

Pre-2021 approaches (full sync from genesis, old fast sync) became impractically slow. A new approach was needed.

### 3.2 Snap Sync: Download State as Data, Not as a Trie

Snap Sync (EIP-style specification introduced by Ethereum Foundation and implemented across clients) is the dominant sync strategy for Besu today.

**The problem with old fast sync**: Fast sync downloaded trie nodes — the internal nodes of the Merkle Patricia Trie — by hash. But trie nodes are small and there are billions of them. Downloading them one at a time (even in batches) generates enormous numbers of small requests, and the trie structure means you can't easily parallelise because you need parent nodes to discover child node hashes.

**Snap Sync's key insight**: Account leaves (the actual account values) can be enumerated in sorted order. You don't need the internal trie structure to get the data — you just need it to *verify* the data. So:

1. **Download account ranges**: Request all accounts between address hash X and address hash Y from peers. Receive a flat list of (address hash, account RLP) pairs. Verify the batch against the known state root using a range proof (a Merkle proof covering the boundaries).
2. **Download storage ranges**: For each contract account, similarly download all its storage slots as a flat range.
3. **Heal the trie**: Once all leaves are downloaded, the node *reconstructs* the trie locally. Some internal trie nodes may be missing or inconsistent (because the chain moved during download); a "healing" phase requests missing trie nodes from peers.
4. **Pivot management**: The target state root (the "pivot") may become stale as new blocks arrive. Besu manages pivot advancement — periodically moving the target forward and re-downloading changed ranges.

**Result**: Snap Sync reduces initial sync time from days to hours on typical hardware (the exact time depends on network speed, peer availability, and hardware).

Besu both *consumes* snap sync (as a client, downloading from peers) and *serves* snap sync (as a server, answering range requests from other nodes). The snap sync server is important for network health — without servers, new nodes can't sync.

### 3.3 Checkpoint Sync: Trust a Known-Good Starting Point

Checkpoint Sync pushes the trust model even further. Instead of syncing from genesis, the operator specifies a recent, known-good checkpoint (block hash + number + difficulty) in the genesis configuration or CLI. Besu then:

1. Downloads just the block headers from the checkpoint to the current head (to understand the canonical chain)
2. Snap syncs the state at the checkpoint
3. Processes new blocks normally from there

**Trust trade-off**: The operator is trusting that the checkpoint hash is correct. This is usually acceptable because:
- Checkpoints are published by the Ethereum Foundation and major clients
- You can verify a checkpoint against multiple independent sources
- Once synced, all future blocks are cryptographically verified normally

**Result**: Checkpoint sync can get a node from zero to fully synced in under an hour on fast hardware — a dramatic improvement for validator operators who need to recover quickly.

### 3.4 The Pipeline Architecture

Besu's sync subsystem is not a monolithic "download everything then validate" loop. It's a **staged pipeline** where each stage runs concurrently and passes work items to the next stage via queues. This is analogous to a CPU instruction pipeline or an industrial assembly line.

Stages in the sync pipeline:

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  Header Download │────▶│  Body Download   │────▶│  Block Validate  │
│  (from peers)    │     │  (txs, uncles)   │     │  (header rules)  │
└──────────────────┘     └──────────────────┘     └────────┬─────────┘
                                                            │
                                                   ┌────────▼─────────┐
                                                   │  Block Import    │
                                                   │  (EVM execution) │
                                                   └──────────────────┘
```

Each stage has a bounded queue. If the "Block Import" stage is the bottleneck (because EVM execution is slow), back-pressure flows upstream and the download stages slow down automatically. This prevents unbounded memory growth and allows the pipeline to run at the speed of the slowest stage without wasting resources.

The pipeline design also makes it easy to add new stages (e.g., a "receipt download" stage for receipt sync) without touching existing stage logic.

---

## Section 4: Solving APIs

### 4.1 The JSON-RPC Server

Besu's JSON-RPC server is built on **Vert.x**, an event-driven, non-blocking application framework for the JVM. This choice was deliberate:

- Vert.x uses an event loop model (similar to Node.js) for I/O, meaning a small number of threads can handle thousands of simultaneous connections
- Long-running operations (like `eth_getLogs` scanning millions of blocks) are dispatched to a worker thread pool, keeping the event loop responsive
- Vert.x's HTTP server handles keep-alive, chunked encoding, and connection limits natively

The JSON-RPC server exposes multiple *method namespaces*, each of which must be explicitly enabled by the operator:

| Namespace  | Purpose                                             |
|------------|-----------------------------------------------------|
| `eth_`     | Core Ethereum: balances, transactions, blocks, logs |
| `net_`     | Network info: peer count, network ID                |
| `web3_`    | Client version, sha3 utility                        |
| `txpool_`  | Transaction pool inspection                         |
| `debug_`   | Block/tx tracing, state inspection                  |
| `trace_`   | Parity-style transaction tracing                    |
| `admin_`   | Node management: add peers, node info               |
| `miner_`   | Mining config (relevant for private PoA networks)   |
| `clique_`  | Clique PoA management                               |
| `ibft_`    | IBFT2 management                                    |
| `qbft_`    | QBFT management                                     |
| `perm_`    | Permissioning management                            |
| `plugins_` | Plugin management                                   |
| `engine_`  | Consensus client communication (separate port)      |

By default, only `eth_`, `net_`, and `web3_` are enabled. Operators enabling `admin_` or `debug_` on a public-facing node is a security risk, so the allowlist approach forces conscious opt-in.

### 4.2 WebSocket Transport

In addition to HTTP, Besu serves the same JSON-RPC methods over WebSocket. WebSocket enables:

- **Subscriptions** (`eth_subscribe`): clients register for events (new blocks, new pending transactions, log events matching a filter) and receive push notifications without polling
- **Long-lived connections**: useful for dApp backends that need real-time block data
- **Reduced latency**: eliminates HTTP connection setup overhead for high-frequency callers

Besu's WebSocket server is also built on Vert.x.

### 4.3 GraphQL

Besu includes a GraphQL server (also on Vert.x) implementing the Ethereum GraphQL schema. GraphQL allows callers to request exactly the fields they need in a single query, reducing over-fetching. For example, a block explorer might request `hash`, `number`, `gasUsed`, and `transactionCount` for 100 blocks in a single GraphQL query, whereas JSON-RPC would require 100 separate `eth_getBlockByNumber` calls.

GraphQL is particularly useful for:
- Block explorer backends
- Analytics tools
- Applications that query many related entities (blocks → transactions → receipts → logs) in a single round trip

### 4.4 Engine API: The Consensus Client Interface

Post-Merge Ethereum separates the **execution layer** (Besu) from the **consensus layer** (Teku, Lighthouse, Prysm, etc.). They communicate via the **Engine API**, a JSON-RPC subset served on a *separate, authenticated HTTP endpoint*.

The authentication mechanism is **JWT (JSON Web Tokens)**: the execution and consensus clients share a secret file (`jwtsecret`), and the consensus client includes a JWT in every Engine API request. Besu validates the JWT before processing any Engine API call. This prevents any non-consensus-client process from instructing Besu to set a new head.

Key Engine API methods:

| Method                        | Caller           | Purpose                                              |
|-------------------------------|------------------|------------------------------------------------------|
| `engine_newPayloadV3`         | Consensus client | "Here is a new execution payload; validate it"       |
| `engine_forkchoiceUpdatedV3`  | Consensus client | "The canonical head is now X; optionally start building a block" |
| `engine_getPayloadV3`         | Consensus client | "Give me the best payload you've built so far"       |
| `engine_exchangeCapabilities` | Both             | Negotiate supported versions                         |

The separation of concerns here is important: Besu knows nothing about attestations, validator duties, or consensus. The consensus client knows nothing about EVM execution or storage. The Engine API is the thin, well-defined boundary between them.

---

## Section 5: Solving P2P Networking

### 5.1 DevP2P Stack Overview

Besu implements Ethereum's **DevP2P** networking stack — the peer-to-peer communication protocol used by all Ethereum execution-layer clients. DevP2P has two layers:

1. **Discovery layer**: Finding other Ethereum nodes on the internet
2. **RLPx transport layer**: Establishing secure, encrypted connections to known peers and exchanging application data

### 5.2 Node Discovery: Finding Peers

Ethereum uses a UDP-based distributed hash table (DHT) loosely inspired by Kademlia for peer discovery.

**Discovery v4** (original):
- Each node has a 64-byte public key (the node ID)
- Nodes are arranged in a logical keyspace by XOR distance between node IDs
- When looking for new peers, a node asks its known peers "who do you know that is close to this target ID?"
- Bootstrap is seeded via hardcoded bootnodes (well-known, long-running nodes)
- Messages: `PING/PONG` (liveness), `FINDNODE/NEIGHBORS` (routing table discovery), `ENRRequest/ENRResponse` (node metadata)

**Discovery v5** (EIP-8085, used for Portal Network and other overlay protocols):
- More flexible protocol supporting arbitrary ENR (Ethereum Node Records) content
- Better support for encrypted and authenticated communication
- Enables topic advertisement (nodes can advertise what protocols they support)
- Besu supports v5 for compatibility with newer network infrastructure

### 5.3 RLPx: Encrypted Peer Communication

Once a peer is discovered, Besu establishes an **RLPx** TCP connection:

1. **Handshake**: An ECIES (Elliptic Curve Integrated Encryption Scheme) encrypted handshake negotiates a shared session secret using ECDH key exchange. After this, all communication is encrypted.
2. **Hello message**: Each side identifies itself: client version, protocol capabilities, listen port, node ID
3. **Capability negotiation**: Both sides advertise the sub-protocols they support (`eth/68`, `snap/1`, etc.). Only mutually supported protocols are used.
4. **Framing**: After handshake, messages are sent as RLP-encoded frames, each tagged with the sub-protocol and message type

This means Besu's P2P layer supports **multiplexing**: a single TCP connection to a peer can carry `eth` (blockchain sync), `snap` (snap sync), and potentially other sub-protocol messages simultaneously.

### 5.4 ETH Sub-Protocol: Block and Transaction Gossip

The `eth` sub-protocol (currently `eth/68`) is the primary protocol for:

- **Block announcements**: When Besu imports a new block, it announces it to peers with `NewBlockHashes` (just the hash, letting peers request if interested)
- **Transaction gossip**: New transactions are broadcast to a subset of peers with `Transactions` or `PooledTransactions` (EIP-2464); this propagates transactions across the network
- **Block/header/body requests**: `GetBlockHeaders`, `BlockHeaders`, `GetBlockBodies`, `BlockBodies` — used during sync to fetch historical data
- **Receipts**: `GetReceipts`, `Receipts` for fetching transaction receipts

**EIP-2464 (Transaction announcement)**: To reduce bandwidth, Besu first announces transaction hashes to peers (`NewPooledTransactionHashes`). Peers respond with `GetPooledTransactions` only for transactions they don't already have. This avoids sending full transaction data to peers who already received it from another path.

### 5.5 Peer Management

Besu's peer manager maintains a target peer count (default 25, configurable). It continuously:

- Discovers new candidate peers via the discovery layer
- Establishes connections to candidates if below target
- Drops connections to peers that are slow, unresponsive, or misbehaving (via a reputation scoring system)
- Prefers peers that are on the canonical chain (by comparing TD/head block)

**Static nodes**: Operators can configure specific node enodes that Besu always tries to maintain connections to. This is used in private networks where the topology is controlled.

**Bootnodes**: A list of long-running, well-connected nodes used for initial peer discovery. Besu hardcodes the official Ethereum bootnodes, but operators can override this (essential for private networks that have no connection to Mainnet).

**Max peers**: On validator nodes or RPC providers with limited resources, operators reduce max peers to free up resources for transaction processing. On nodes that are primarily serving snap sync data to others (contributing to network health), max peers can be increased.

---

## Section 6: Solving Enterprise / Private Network Needs

### 6.1 The Genesis File: Configuration as Code

In public Ethereum, genesis is fixed (everyone uses the same genesis block). In private/consortium networks, operators define their own genesis, specified in a JSON file passed to Besu at startup.

The genesis file controls:
- **Chain ID**: unique identifier preventing transaction replay across chains
- **Initial account allocations**: pre-fund accounts without mining
- **Consensus configuration**: which protocol (`qbft`, `ibft2`, `clique`, `ethash`, `merge`), block period, validator set, epoch length
- **Fork activation points**: activate EVM features from block 0 (skipping all PoW history)
- **Extra data**: for QBFT/IBFT2, encodes the initial validator set in the genesis block's `extraData` field
- **Gas limit**: sets the initial block gas limit (can be adjusted by validators over time)
- **EIP-1559 base fee parameters**: initial base fee, elasticity multiplier

This approach means a private network is fully defined by a single file under version control. New nodes joining the consortium just need this file and the bootnode addresses — they don't need out-of-band configuration.

### 6.2 QBFT: Byzantine Fault Tolerant Consensus for Private Networks

QBFT (Quorum Byzantine Fault Tolerance) is Besu's recommended consensus protocol for private/consortium networks. It's a Proof-of-Authority variant based on the Istanbul BFT (IBFT) algorithm, with improvements for correctness and liveness.

**How QBFT works**:

```
Round structure:
  1. PROPOSE: The round's proposer broadcasts a PREPARE message containing the proposed block
  2. PREPARE: Validators broadcast PREPARE messages (endorsing the proposal)
              Once 2/3+1 PREPARE messages received → PREPARED state
  3. COMMIT:  Validators broadcast COMMIT messages
              Once 2/3+1 COMMIT messages received → block is COMMITTED (finalized)
  4. ROUND-CHANGE: If a proposer is unresponsive (timeout), validators broadcast
                   ROUND-CHANGE to elect a new proposer
```

**Key properties**:
- **Immediate finality**: Once committed, a block is final. There are no forks, no reorgs. This is ideal for financial applications (a payment confirmed in block N is *always* in block N).
- **Known validators**: Only nodes in the validator set can propose blocks. This is configurable at network setup and can be changed via governance transactions.
- **BFT tolerance**: The network can tolerate up to `(n-1)/3` faulty (malicious or crashed) validators. For a 4-validator network, 1 can be faulty. For 7 validators, 2 can be faulty.
- **Block period**: Configurable (e.g., 2 seconds, 5 seconds). Blocks are produced on a timer, not when full.
- **No mining**: No PoW computation. Block production is pure message passing between validators.

**Validator set management**: The current validator set is encoded in each block's `extraData` field. Validators can be added or removed via on-chain governance (special transaction types that modify the validator set), with the new set taking effect at the next block.

### 6.3 Node Permissioning

In a consortium network, you don't want random internet nodes to connect and consume resources (or worse, attempt to sync and observe private transaction data). Besu supports two levels of node permissioning:

**Static allowlist (file-based)**:
- Operators specify a list of allowed enode URLs in a configuration file
- Any node not on the list is rejected at the RLPx connection stage
- Simple to set up, requires all operators to agree on the list and update it manually

**Smart contract-based permissioning (EEA Permissioning)**:
- A permissioning contract is deployed on the private network
- The contract's `connectionAllowed(sourceEnode, destinationEnode)` function is called for each connection attempt
- The validator set (via governance) controls who can update the contract
- Much more flexible: add/remove nodes without touching configuration files, auditability via transaction history, supports complex rules (e.g., "allow all nodes from organisation A to connect to nodes from organisation B")

### 6.4 Account Permissioning

Beyond node-level access, QBFT networks often need to restrict which *accounts* can submit transactions. Besu supports:

**Account allowlist (file-based)**: Only listed Ethereum addresses can submit transactions to the network.

**Smart contract-based account permissioning**: Similar to node permissioning, a contract's `transactionAllowed(sender, target, value, gasPrice, gasLimit, data)` function is called before accepting any transaction into the pool or including it in a block. This allows very fine-grained rules: "account X can only interact with contract Y," "account X cannot send more than Z ETH per transaction," etc.

### 6.5 Plugin System: Extending Without Forking

The plugin system is the architectural mechanism that allows Besu to serve the "long tail" of enterprise requirements without bloating the core codebase. Covered in depth in Section 7.

---

## Section 7: Solving Modularity with the Plugin API

### 7.1 The Extensibility Problem

Every enterprise has unique requirements:
- A bank might need custom transaction validation (KYC checks before transactions are accepted into the pool)
- A government network might need custom storage backends that integrate with approved HSM (Hardware Security Module) infrastructure
- A DeFi protocol operator might want to customise block building (MEV management, transaction ordering rules)
- An analytics firm might want to stream every block event to Kafka without writing custom sync code

If Besu tried to build all of these into core, it would become unmaintainable. If enterprises had to fork Besu and maintain their own fork, they'd fall behind upstream updates and miss security fixes.

The plugin system solves this via a **stable API boundary**: Besu publishes interfaces in `plugin-api/`, and anyone can write code against those interfaces without touching core. Plugins ship as JAR files dropped into a `plugins/` directory.

### 7.2 Module Structure

```
plugin-api/                          ← Public API (stable, versioned)
  src/main/java/
    org/hyperledger/besu/plugin/
      BesuPlugin.java                ← Interface every plugin implements
      BesuContext.java               ← Registry of services plugins can access
      services/
        BesuEvents.java              ← Subscribe to blockchain events
        BlockchainService.java       ← Read chain head, blocks, transactions
        TransactionPoolService.java  ← Inspect and manage tx pool
        StorageService.java          ← Register custom storage backends
        RpcEndpointService.java      ← Register custom JSON-RPC methods
        MiningService.java           ← Control block building (for private nets)
        MetricsSystem.java           ← Register custom Prometheus metrics
        PermissioningService.java    ← Register custom permissioning rules
        SecurityModuleService.java   ← Register custom signing backends
        TransactionSelectionService.java ← Customise tx selection for blocks
        ... and more
```

### 7.3 Plugin Lifecycle

Plugins follow a well-defined lifecycle that integrates with Besu's startup/shutdown sequence:

```
1. DISCOVERY:    Besu scans plugins/ directory for JARs
                 Uses Java ServiceLoader to find BesuPlugin implementations
                 Instantiates each plugin

2. REGISTER:     plugin.register(BesuContext) is called
                 Plugin uses BesuContext to look up and configure services
                 Plugin registers its capabilities (e.g., "I provide storage")
                 No services are fully started yet

3. BEFORE_EXTERNAL_SERVICES:
                 Called before RPC/P2P servers start
                 Plugin can configure things that need to be in place before
                 external connections are accepted

4. START:        plugin.start() is called
                 Plugin starts any background threads, connections to external
                 systems, etc.

5. AFTER_EXTERNAL_SERVICES_MAIN_LOOP:
                 Full node is running
                 Plugin can begin subscribing to events and processing data

6. STOP:         On shutdown, plugin.stop() is called
                 Plugin cleans up resources, flushes buffers, closes connections
```

### 7.4 Event Subscriptions via BesuEvents

The `BesuEvents` service is perhaps the most commonly used by plugins. It allows plugins to react to things happening inside Besu without polling:

```
BesuEvents events = context.getService(BesuEvents.class);

// Called every time a new block is added to the canonical chain
events.addBlockAddedListener(blockAddedEvent -> {
    Block block = blockAddedEvent.getBlock();
    // stream to Kafka, update a database, etc.
});

// Called when a transaction is added to the transaction pool
events.addTransactionAddedListener(tx -> {
    // custom compliance check, alert system, etc.
});

// Called when a block is removed (reorg)
events.addBlockReorgListener(reorgEvent -> {
    // compensate for reverted transactions
});
```

### 7.5 Custom Transaction Selection

The `TransactionSelectionService` is used by block-building plugins to customise which transactions get included in blocks, in what order, and with what constraints. The interface provides:

- `evaluateTransactionPreProcessing`: called before EVM execution of a candidate transaction; can reject the transaction immediately based on metadata (gas price, sender, etc.)
- `evaluateTransactionPostProcessing`: called after EVM execution; can reject the transaction based on execution results (e.g., "don't include transactions that emit specific event types")
- `onTransactionSelected`: notification that a transaction was selected; allows stateful logic ("no more than 10 transactions from sender X per block")
- `onTransactionNotSelected`: notification that a transaction was rejected

This is how MEV management, custom ordering rules, and compliance-based transaction filtering are implemented in Besu without modifying the core transaction pool.

### 7.6 Custom RPC Methods

Plugins can register entirely new JSON-RPC namespaces and methods:

```java
RpcEndpointService rpc = context.getService(RpcEndpointService.class);
rpc.registerRpcEndpoint("myPlugin_getCustomData", this::handleGetCustomData);
```

This allows enterprise operators to expose custom APIs (e.g., "get the current compliance status of an address") through the same JSON-RPC interface their applications already use, without standing up a separate service.

### 7.7 Hot Config Reload

The `plugins_reloadPluginConfig` JSON-RPC method triggers a configuration reload across all plugins without restarting the node. This is critical for production networks where downtime is costly. For example, updating a permissioning list or changing a compliance rule can be applied live.

---

## Section 8: Solving Performance (2024–2025 Achievements)

### 8.1 Context: The Performance Pressure

As Ethereum's adoption grew and the L1 block gas limit increased (from 15M → 30M → 45M → 60M+ gas per block), execution clients faced pressure to process larger blocks within the fixed 12-second slot time. A client that falls behind can't serve validators, can't respond to RPC calls, and eventually falls off the canonical chain.

Besu's performance engineering from 2023 onwards was driven by a systematic programme of profiling, targeted optimisation, and architectural improvements.

### 8.2 Gas Limit Journey

The block gas limit on Mainnet has been progressively increased:

| Period    | Gas Limit | Notes                                                |
|-----------|-----------|------------------------------------------------------|
| 2021–2022 | 15M gas   | Post-EIP-1559 baseline                               |
| 2023      | 30M gas   | Doubled after client performance improvements        |
| 2024 H1   | 36M gas   | Gradual increase as clients optimised                |
| 2024 H2   | 45M gas   | Further increase post-parallel-tx work               |
| 2025      | 60M+ gas  | Targeted by Ethereum roadmap with client support     |

Besu's ability to process these larger blocks was a prerequisite for the Ethereum community to feel safe raising the limit.

### 8.3 Profiling Methodology

Besu's performance team uses **Async Profiler** — a low-overhead sampling profiler that can profile both Java code and native (JNI/C++) code simultaneously. This is crucial because RocksDB's JNI boundary means some performance bottlenecks are in native code, invisible to Java-only profilers.

Standard profiling workflow:
1. Replay historical Mainnet blocks (particularly "heavy" blocks with complex DeFi transactions) in a test harness
2. Profile with Async Profiler to produce flamegraphs
3. Identify hot spots (functions consuming disproportionate CPU or allocation)
4. Optimise, re-profile to confirm improvement
5. Run the full Ethereum test suite to confirm correctness is preserved

### 8.4 ModExp Precompile Optimisation

The `modexp` precompile (EIP-198, address 0x05) computes modular exponentiation: `base^exp mod modulus`. It's used heavily in RSA verification and certain ZK proving systems. Its performance dramatically affects blocks that use these operations.

Besu's optimisation involved:
- Replacing the naive Java BigInteger implementation with a native library call (using the GMP library via JNI, or alternatively the Java `BigInteger.modPow` with careful argument ordering)
- Implementing the EIP-2565 gas repricing rules correctly (Berlin upgrade changed gas cost calculation to reflect actual compute cost better)
- Caching results for repeated calls with the same inputs (some block patterns call modexp with identical arguments multiple times)

### 8.5 Parallel Transaction Execution Performance

As described in Section 1.6, parallel transaction execution introduces both speedup and overhead. The performance engineering work focused on:

- **Conflict detection overhead**: The read/write set tracking must be lightweight. Besu uses a per-`WorldUpdater` tracking map with minimal locking.
- **Thread pool sizing**: Too few threads and parallelism is limited; too many and context-switching overhead hurts. The default is tuned to available CPU cores.
- **Re-execution cost**: The worst case is when many transactions conflict. Besu tracks conflict rates in metrics, allowing operators to tune or disable parallel execution for workloads that are heavily conflicting.
- **Memory pressure**: Running N transactions in parallel means N copies of partial world state in memory. Besu uses lazy initialisation to minimise the per-thread memory footprint.

### 8.6 RPC Optimisations

High-traffic RPC nodes (e.g., Infura, Alchemy equivalents) make millions of `eth_call`, `eth_getLogs`, and `eth_getBalance` requests per day. Besu's RPC optimisations include:

- **Result caching**: Frequently requested values (e.g., contract code, recent block data) are cached in memory to avoid redundant storage lookups
- **Parallel log scanning**: `eth_getLogs` with a broad block range can be parallelised across the block range, with results merged and sorted
- **Bloom filter pre-screening**: Before loading full block receipts for log scanning, Besu checks the block's Bloom filter. If the filter doesn't match the query topics/addresses, the block is skipped entirely — avoiding expensive storage reads
- **Efficient JSON serialisation**: Switching to faster JSON libraries and reducing object allocation during serialisation

### 8.7 Bonsai Performance Optimisations

Bonsai Tries provide storage savings, but the trie log replaying for historical queries introduced its own performance considerations:

- **Trie log compaction**: Old trie logs beyond the retention window are periodically purged to keep the database size bounded
- **State cache**: The most recently accessed accounts and storage slots are held in an LRU cache, dramatically reducing RocksDB reads for "hot" accounts (validators, popular DeFi contracts)
- **Flat DB read path**: For current-state queries, Besu's Bonsai implementation bypasses trie traversal entirely and reads directly from the flat database, giving O(1) account lookups

---

## Section 9: Architecture Decisions Summary Table

The following table summarises the key design problems, the solutions Besu chose, the primary code location, and the trade-offs accepted.

| Problem | Solution | Key Class / Module | Trade-off Accepted |
|---|---|---|---|
| Execute EVM bytecode correctly across all hard forks | Standalone `evm/` module + `ProtocolSpec` per fork | `EVM`, `ProtocolSpec`, `MainnetProtocolSchedule` | Complexity of maintaining per-fork configs; justified by hard fork cadence |
| Roll back state on revert or reorg | `WorldUpdater` copy-on-write + Bonsai TrieLog | `WorldUpdater`, `TrieLog` | Memory overhead of tracking changes; necessary for correctness |
| Store enormous Ethereum state efficiently | Bonsai Tries with flat DB + TrieLog journal | `BonsaiWorldStateKeyValueStorage`, `TrieLog` | Cannot serve arbitrary historical state beyond retention window |
| Serve any historical state (archive mode) | Forest of Tries (content-addressed trie nodes) | `ForestWorldStateKeyValueStorage` | Very large storage footprint (TB scale) |
| Fast initial sync | Snap Sync: download leaf data, reconstruct trie | `SnapSyncDownloader`, `SnapWorldStateDownloader` | Requires peers serving snap protocol; complex healing phase |
| Fastest possible startup | Checkpoint Sync | `CheckpointSyncConfiguration` | Requires operator to trust a checkpoint hash |
| High-throughput API serving | Vert.x async HTTP/WebSocket server | `JsonRpcHttpService`, `WebSocketService` | Vert.x learning curve; async programming complexity |
| CL/EL separation post-Merge | Engine API with JWT auth on separate port | `EngineJsonRpcService`, `JwtAuthenticationService` | Must maintain two processes per validator node |
| Find peers on the internet | DevP2P Discovery v4/v5 (Kademlia-style DHT) | `DiscoveryPeer`, `PeerDiscoveryAgent` | DHT-based discovery has high churn; mitigated by peer reputation |
| Encrypt peer communications | RLPx (ECIES handshake + framed messages) | `RLPxConnection`, `HandshakeHandler` | CPU overhead of encryption on every message |
| Immediate finality for private networks | QBFT BFT consensus | `QbftBlockCreator`, `QbftController` | Requires known, permissioned validator set; not suitable for public chain |
| Restrict network access | Node permissioning (allowlist or smart contract) | `NodeSmartContractPermissioningController` | Smart contract approach requires on-chain governance overhead |
| Customise without forking | Plugin API + Java ServiceLoader | `BesuPlugin`, `BesuContext`, `plugin-api/` | Plugin API must be kept stable across releases; constrains refactoring |
| Custom transaction ordering (MEV, compliance) | `TransactionSelectionService` plugin interface | `TransactionSelectionService` | Plugin bugs can stall block production; careful sandboxing needed |
| Increase throughput on multi-core hardware | Optimistic parallel transaction execution | `ParallelTransactionExecutor` | Overhead on highly conflicting workloads; must detect conflicts correctly |
| Minimise storage reads for hot accounts | LRU cache on Bonsai flat DB | `BonsaiCachedWorldStorageManager` | Cache memory pressure; must be bounded and evict correctly |
| Observe node internals | Prometheus metrics + OpenTelemetry tracing | `MetricsSystem`, `BesuMetricCategory` | Metric cardinality must be controlled to avoid Prometheus overload |

---

## Section 10: The Layered Architecture — Putting It All Together

Having examined each subsystem in isolation, it's worth stepping back to see how they compose. Besu's architecture is best understood as a set of concentric layers:

```
┌─────────────────────────────────────────────────────────────────────┐
│                         OPERATOR INTERFACE                          │
│         CLI (Picocli)  ·  Config Files  ·  Genesis JSON             │
├─────────────────────────────────────────────────────────────────────┤
│                          EXTERNAL APIS                               │
│    JSON-RPC (HTTP/WS)  ·  GraphQL  ·  Engine API  ·  Metrics        │
├─────────────────────────────────────────────────────────────────────┤
│                        PLUGIN LAYER                                  │
│   Custom RPC  ·  Custom TX Selection  ·  Custom Storage  ·  Events  │
├─────────────────────────────────────────────────────────────────────┤
│                       BLOCKCHAIN LAYER                               │
│    Block Import  ·  Transaction Pool  ·  Sync Pipeline              │
├─────────────────────────────────────────────────────────────────────┤
│                       CONSENSUS LAYER                                │
│    QBFT  ·  IBFT2  ·  Clique  ·  Merge PoS (via Engine API)         │
├─────────────────────────────────────────────────────────────────────┤
│                       EXECUTION LAYER                                │
│    EVM  ·  TransactionProcessor  ·  ProtocolSchedule                │
├─────────────────────────────────────────────────────────────────────┤
│                        STORAGE LAYER                                 │
│    Bonsai Tries  ·  Forest Tries  ·  RocksDB  ·  KeyValueStorage    │
├─────────────────────────────────────────────────────────────────────┤
│                       NETWORKING LAYER                               │
│    DevP2P Discovery  ·  RLPx  ·  ETH subprotocol  ·  Snap protocol  │
└─────────────────────────────────────────────────────────────────────┘
```

Each layer depends only on layers below it (or on the plugin API for cross-cutting concerns). This clean dependency direction means:

- The EVM module has no dependency on networking
- The JSON-RPC server has no dependency on consensus details
- Storage implementations have no dependency on EVM logic
- Plugins can reach across layers via the service interfaces exposed through `BesuContext`

This architecture is what makes Besu simultaneously suitable for:
- A Mainnet validator (enable networking + Engine API + Bonsai + full JSON-RPC)
- A private QBFT network (enable QBFT consensus + permissioning + custom genesis, disable Mainnet networking)
- An archive node for analytics (enable Forest tries + full historical JSON-RPC)
- An embedded EVM for testing (use just the `evm/` module, nothing else)

---

## Summary

Besu solves its business requirements through a consistent set of architectural principles:

1. **Abstractions at every boundary**: `KeyValueStorage`, `WorldUpdater`, `ProtocolSpec`, `BesuPlugin` — every major interface is defined as a Java interface, enabling multiple implementations and easy testing
2. **Composition over inheritance**: `ProtocolSchedule` composes `ProtocolSpec` objects; the plugin system composes arbitrary extensions; the sync pipeline composes stages
3. **Explicit hard-fork handling**: Rather than littering the codebase with `if (blockNumber > LONDON_BLOCK)` conditionals, all fork-specific logic is encapsulated in the appropriate `ProtocolSpec`
4. **Async-first for I/O**: Vert.x for RPC, pipeline architecture for sync — I/O operations never block the main execution thread
5. **Performance via data structure choice**: Bonsai's flat DB for O(1) state lookups; Bloom filters for log scanning; LRU caches for hot state; parallel execution for throughput

Each of these choices comes with trade-offs (complexity, memory overhead, correctness requirements), but the trade-offs were made deliberately with the dual customer requirement in mind: Besu must be good enough for Ethereum Mainnet *and* flexible enough for enterprise networks.

---

## What's Next?

In **[A3 — Deep Architecture Walkthrough](./A3_deep_architecture.md)**, we will:
- Walk through the full lifecycle of a single transaction, from submission to finality
- Examine the `BesuController` startup sequence and dependency injection graph
- Deep-dive into the Bonsai trie data structures with diagrams
- Trace a full Snap Sync session from first peer contact to sync complete
- Explore the QBFT consensus state machine in detail

> *"A good architecture is one where every decision has a reason, and every reason has a constraint."*
> — paraphrased from Rich Hickey

---

*Document version: 2025 | Besu main branch | Written for the Besu Explained series*