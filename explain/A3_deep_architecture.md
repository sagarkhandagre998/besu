# A3 — Deep Architecture of Hyperledger Besu

> 🔴 **Level:** Advanced | ⏱️ **Read Time:** ~60 minutes

---

## Introduction

Now that you understand *why* Besu exists (A1) and *how* its design decisions solve the problems (A2), this document goes deep into the actual architecture — every module, every major class, every subsystem, and how they all wire together.

By the end of this file you will be able to open the Besu GitHub repository and immediately navigate to any component you need.

---

## Section 1: Repository Structure — Gradle Multi-Project Build

Besu is a **Gradle multi-project build**. The repo root contains a `settings.gradle` that declares ~30 subprojects. Each subproject is an independent JAR with its own `build.gradle`, source tree, and tests.

```
besu/                                  ← repo root
│
├── app/                               ← Entry point: Besu.java + Dagger DI root
├── config/                            ← Genesis file parsing, network config, chain config
│
├── consensus/
│   ├── common/                        ← Shared BFT types, validator voting, round logic
│   ├── merge/                         ← Post-Merge PoS: MergeCoordinator, PayloadAttributes
│   ├── qbft/                          ← QBFT implementation (Besu-specific block types)
│   ├── qbft-core/                     ← Reusable QBFT state machine (no Besu block deps)
│   ├── ibft/                          ← IBFT 2.0 implementation
│   ├── ibftlegacy/                    ← IBFT 1.0 (read-only legacy support)
│   └── clique/                        ← Clique PoA (block production deprecated in 25.12.0)
│
├── crypto/
│   ├── algorithms/                    ← Pure crypto: secp256k1, ECDSA, Keccak256 (no plugins)
│   └── services/                      ← Crypto with plugin-api integration (SignatureAlgorithmFactory)
│
├── datatypes/                         ← Core Ethereum types: Address, Hash, Wei, Difficulty,
│                                        BlockHeader, Transaction, Log, TransactionReceipt
│
├── ethereum/
│   ├── api/                           ← All API servers: JSON-RPC, GraphQL, WS, Engine API
│   ├── blockcreation/                 ← Block proposal and sealing for validators/miners
│   ├── core/                          ← Transaction processing, block import, world state mgmt
│   ├── eth/                           ← ETH wire protocol: sync manager, block fetcher, tx pool
│   ├── evmtool/                       ← Standalone EVM CLI tool (for testing/debugging)
│   ├── mock-p2p/                      ← Mock P2P for testing without real networking
│   ├── p2p/                           ← DevP2P: RLPx transport, peer discovery, peer management
│   ├── permissioning/                 ← Node/account permissioning rules and smart contract
│   ├── referencetests/                ← Ethereum Foundation EVM reference test runner
│   ├── rlp/                           ← RLP (Recursive Length Prefix) encode/decode
│   ├── trie/                          ← Merkle Patricia Trie, Bonsai, Forest implementations
│   └── verkletrie/                    ← Experimental Verkle Trie (future Ethereum state format)
│
├── evm/                               ← STANDALONE EVM: opcodes, gas calculators, precompiles
│                                        (can be used as a library outside Besu)
│
├── metrics/
│   ├── core/                          ← MetricsSystem interface, Prometheus implementation
│   └── rocksdb/                       ← RocksDB-specific metrics collector
│
├── nat/                               ← NAT traversal: UPnP, Kubernetes, Docker IP detection
├── platform/                          ← OS utilities: native lib loading, memory detection
│
├── plugin-api/                        ← PUBLIC plugin interfaces:
│                                        BesuPlugin, BesuContext, all Service interfaces
│
├── plugins/
│   └── rocksdb/                       ← RocksDB storage plugin (implements StoragePlugin)
│
├── services/
│   ├── kvstore/                       ← KeyValueStorage abstraction + RocksDB implementation
│   ├── pipeline/                      ← Async pipeline stages used by sync
│   └── tasks/                         ← Task queue abstractions (work queues for sync)
│
├── testfuzz/                          ← Property-based fuzz tests for EVM and RLP
├── testutil/                          ← Shared test utilities and fixtures
└── util/                              ← General utilities: bytes, collections, subscribers
```

### Module Dependency Direction (simplified)

```
app
 ├──► ethereum/api
 ├──► ethereum/core ──► evm ──► datatypes
 ├──► ethereum/eth  ──► ethereum/p2p
 ├──► consensus/*   ──► consensus/common
 ├──► metrics/core
 └──► plugin-api
          ▲
    plugins/* (implement plugin-api)
    services/kvstore ──► plugins/rocksdb
```

**Golden Rule**: dependencies only flow *downward*. Lower modules (`datatypes`, `evm`, `crypto`) never import upper modules (`ethereum/api`, `consensus/*`). This is enforced by Gradle.

---

## Section 2: Dependency Injection with Dagger 2

Besu uses **Dagger 2** for compile-time dependency injection — not Spring, not Guice.

### Why Dagger?

| Concern | Spring / Guice | Dagger 2 |
|---------|---------------|----------|
| DI resolution | Runtime (reflection) | Compile-time (annotation processing) |
| Startup speed | Slower | Fast |
| Error detection | Runtime | Compile-time |
| JVM footprint | Heavier | Lighter |

### Key Files

```
app/src/main/java/org/hyperledger/besu/
├── Besu.java                          ← main() entry point
└── components/
    ├── BesuComponent.java             ← Dagger @Component interface
    ├── BesuPluginContextModule.java   ← @Module: provides BesuPluginContextImpl
    └── BesuCommandModule.java         ← @Module: provides BesuCommand
```

### How it Works

```java
// BesuComponent.java — the top-level DI container
@Singleton
@Component(modules = {BesuCommandModule.class, BesuPluginContextModule.class})
public interface BesuComponent {
    BesuCommand getBesuCommand();
}
```

At compile time, Dagger generates `DaggerBesuComponent.java`. At runtime:

```java
// Besu.java — main()
BesuComponent besuComponent = DaggerBesuComponent.create();  // generated class
BesuCommand besuCommand = besuComponent.getBesuCommand();    // fully wired instance
```

All dependencies of `BesuCommand` are automatically injected because Dagger reads the `@Inject` constructor annotations and builds the entire object graph at compile time.

---

## Section 3: The CLI Layer — BesuCommand

`BesuCommand` is the **largest single class** in the codebase — it is the command-line interface.

### Framework: PicoCLI

PicoCLI is a Java CLI framework that maps command-line arguments to annotated fields:

```java
@Command(name = "besu", ...)
public class BesuCommand implements Runnable {

    @Option(names = "--network", ...)
    private NetworkName network = NetworkName.MAINNET;

    @Option(names = "--sync-mode", ...)
    private SyncMode syncMode = SyncMode.SNAP;

    @Option(names = "--data-path", ...)
    private Path dataDir;

    @Option(names = "--rpc-http-enabled", ...)
    private boolean rpcHttpEnabled = false;

    // ... hundreds more options
}
```

### Subcommands

`BesuCommand` has nested subcommands for utility operations:

```
besu
 ├── blocks
 │    ├── import   ← import blocks from RLP file
 │    └── export   ← export blocks to RLP file
 ├── operator
 │    ├── generate-blockchain-config   ← generate QBFT/IBFT genesis + keys
 │    └── generate-log-bloom-cache     ← pre-build log bloom indexes
 ├── rlp
 │    ├── encode   ← encode validator list → RLP extraData
 │    └── decode   ← decode RLP extraData → validator list
 └── public-key
      ├── export   ← print node's public key
      └── export-address   ← print node's Ethereum address
```

### Configuration File

All options can also be provided via a TOML config file:

```toml
# besu.toml
network="mainnet"
sync-mode="SNAP"
data-path="/var/lib/besu"
rpc-http-enabled=true
rpc-http-port=8545
metrics-enabled=true
```

**Precedence order**: `CLI args` > `config file` > `defaults`

### After Parsing: `run()`

Once PicoCLI finishes parsing, it calls `BesuCommand.run()`, which:
1. Validates all option combinations
2. Configures logging level
3. Prints configuration overview
4. Calls `buildController()` → `buildRunner()` → `startRunner()`

---

## Section 4: BesuController — The Central Wiring Point

`BesuController` is the object that holds references to all the major subsystems. It is built by a `BesuControllerBuilder`.

### Builder Selection by Consensus Type

```
BesuCommand.buildController()
     │
     ├── reads genesis file → detects consensus type
     │
     ▼
BesuControllerBuilderFactory.fromEthNetworkConfig(config)
     │
     ├── genesis has "ibft2" config?   → IbftBesuControllerBuilder
     ├── genesis has "qbft" config?    → QbftBesuControllerBuilder
     ├── genesis has "clique" config?  → CliqueBesuControllerBuilder
     └── default (Mainnet / PoS)       → MainnetBesuControllerBuilder
```

### What BesuController Contains

```
BesuController
 ├── ProtocolSchedule          ← which EVM rules apply at which block number
 ├── ProtocolContext           ← blockchain + worldStateArchive + consensusContext
 │    ├── Blockchain           ← the canonical chain (read/write)
 │    ├── WorldStateArchive    ← access to world state (Bonsai or Forest)
 │    └── ConsensusContext     ← consensus-specific state (e.g., validator set for QBFT)
 ├── TransactionPool           ← pending transactions waiting to be included
 ├── EthProtocolManager        ← manages ETH wire protocol peers + messages
 ├── SynchronizerConfiguration ← sync strategy settings
 ├── MiningCoordinator         ← (PoA/PoW) or MergeCoordinator (PoS)
 └── BesuPluginContextImpl     ← plugin lifecycle manager
```

---

## Section 5: The Protocol Layer

This is the most important conceptual layer in Besu. It defines *which Ethereum rules apply when*.

### ProtocolSpec

A `ProtocolSpec` is an **immutable bundle of rules for one Ethereum hard fork**:

```java
public class ProtocolSpec {
    private final EVM evm;                           // EVM with correct opcodes for this fork
    private final GasCalculator gasCalculator;       // gas costs for this fork
    private final BlockHeaderValidator blockHeaderValidator;
    private final BlockBodyValidator blockBodyValidator;
    private final BlockImporter blockImporter;
    private final TransactionValidator transactionValidator;
    private final PrecompileContractRegistry precompiles; // 0x01-0x0a + EIP-4844 KZG
    private final BlockReward blockReward;
    // ... more fork-specific rules
}
```

### ProtocolSchedule

A `ProtocolSchedule` is a **timeline of ProtocolSpecs**:

```
Block 0         → Frontier ProtocolSpec
Block 1,150,000 → Homestead ProtocolSpec
Block 2,463,000 → Tangerine Whistle ProtocolSpec
Block 2,675,000 → Spurious Dragon ProtocolSpec
Block 4,370,000 → Byzantium ProtocolSpec
Block 7,280,000 → Constantinople ProtocolSpec
Block 9,069,000 → Istanbul ProtocolSpec
Block 12,244,000→ Berlin ProtocolSpec
Block 12,965,000→ London ProtocolSpec  ← EIP-1559 activated
Block 15,537,394→ Paris ProtocolSpec   ← The Merge (PoW → PoS)
Block 17,034,870→ Shanghai ProtocolSpec ← withdrawals enabled
Block 19,426,587→ Cancun ProtocolSpec  ← EIP-4844 blobs activated
(upcoming)      → Prague ProtocolSpec
```

At every block import, Besu calls `protocolSchedule.getByBlockHeader(header)` to get the correct `ProtocolSpec` for that block's rules.

### GasCalculator Hierarchy

```
AbstractGasCalculator
 └── FrontierGasCalculator
      └── HomesteadGasCalculator
           └── TangerineWhistleGasCalculator
                └── SpuriousDragonGasCalculator
                     └── IstanbulGasCalculator
                          └── BerlinGasCalculator
                               └── LondonGasCalculator
                                    └── CancunGasCalculator
                                         └── PragueGasCalculator
```

Each subclass overrides only the gas costs that changed in that fork. This avoids duplicating unchanged rules.

---

## Section 6: The EVM Module Deep Dive

The `evm/` module is a **self-contained, standalone EVM** that can run outside of Besu.

### Key Classes

```
evm/
 ├── EVM.java                    ← main execution loop
 ├── frame/
 │    └── MessageFrame.java      ← execution context (stack, memory, storage, gas, PC)
 ├── operation/
 │    ├── Operation.java         ← interface: execute(frame, evm) → OperationResult
 │    ├── OperationRegistry.java ← map of opcode byte → Operation implementation
 │    ├── AddOperation.java      ← ADD opcode
 │    ├── SloadOperation.java    ← SLOAD opcode
 │    ├── CallOperation.java     ← CALL opcode (creates sub-frame)
 │    └── ... (one class per opcode)
 ├── precompile/
 │    ├── PrecompileContractRegistry.java
 │    ├── ECRECOVERPrecompiledContract.java   ← 0x01
 │    ├── SHA256PrecompiledContract.java      ← 0x02
 │    ├── ModExpPrecompiledContract.java      ← 0x05 (heavily optimized in 2025)
 │    └── KZGPointEvalPrecompiledContract.java ← 0x0a (EIP-4844 blobs)
 ├── processor/
 │    ├── MessageCallProcessor.java    ← handles CALL to existing contracts
 │    └── ContractCreationProcessor.java ← handles CREATE/CREATE2
 └── gascalculator/
      └── (all GasCalculator subclasses)
```

### MessageFrame — The Execution Context

`MessageFrame` is the **heart of EVM execution**. It holds everything needed to execute one message call:

```java
public class MessageFrame {
    // EVM execution state
    private final Deque<Bytes32> stack;      // operand stack (max 1024 items)
    private final MutableBytes memory;       // temporary memory (cleared after call)
    private long remainingGas;               // gas counter
    private int pc;                          // program counter

    // Call context
    private final Address recipient;         // contract being called
    private final Address sender;            // caller
    private final Bytes inputData;           // calldata
    private final Wei value;                 // ETH sent with call

    // World state access
    private final WorldUpdater worldUpdater; // read/write contract storage + balances

    // Result
    private State state;                     // RUNNING, COMPLETED_SUCCESS, REVERT, EXCEPTIONAL_HALT
    private Bytes returnData;
    private List<Log> logs;
}
```

### EVM Main Loop

```java
// EVM.java (simplified)
public void runToHalt(MessageFrame frame) {
    while (frame.getState() == RUNNING) {
        int opcode = frame.readByte(frame.getPC());  // fetch opcode
        Operation operation = operationRegistry.get(opcode);
        OperationResult result = operation.execute(frame, this);

        if (result.getHaltReason().isPresent()) {
            frame.setState(EXCEPTIONAL_HALT);
        } else {
            frame.incrementPC(result.getPcIncrement());
        }
    }
}
```

### Nested Calls (CALL opcode)

When a CALL opcode executes, it creates a **new child MessageFrame** and runs the EVM recursively:

```
MessageFrame (Contract A)
 └── CALL → MessageFrame (Contract B)
              └── CALL → MessageFrame (Contract C)
                           └── RETURN
                          ← result propagated up
             ← result propagated up
 ← execution continues in A
```

If any frame REVERTs, changes to the `WorldUpdater` in that frame and all its children are discarded. The parent frame's state is unaffected (except gas is returned).

---

## Section 7: World State Architecture

The **world state** is a mapping from every Ethereum address to its account state. It is the most critical and complex data structure in Besu.

### Account State Fields

```java
public interface Account {
    Address getAddress();
    long getNonce();
    Wei getBalance();
    Hash getCodeHash();         // keccak256 of contract bytecode
    Bytes getCode();            // contract bytecode
    UInt256 getStorageValue(UInt256 key);  // contract storage slot
}
```

### WorldState Interface Hierarchy

```
WorldState                          ← read-only snapshot of state
 └── MutableWorldState              ← can be modified
      ├── WorldUpdater              ← transaction-scoped changes (commit or rollback)
      │    └── StackedUpdater       ← nested call scoping (stacked on top of parent)
      │
      ├── BonsaiWorldState          ← Bonsai implementation
      │    ├── BonsaiFlatDbStrategy ← flat key-value storage (account key → state)
      │    └── BonsaiTrieLogManager ← journal of per-block state diffs
      │
      └── ForestMutableWorldState   ← Forest implementation (Merkle Patricia Trie by hash)
```

### Bonsai vs Forest Access Pattern

**Forest (Traditional)**:
```
Read account at address 0xABCD...
 → compute key = keccak256(address)
 → look up in trie: root → branch → branch → ... → leaf  (many DB reads)
```

**Bonsai (Flat DB)**:
```
Read account at address 0xABCD...
 → look up flat DB: key = address  →  value = {nonce, balance, codeHash, storageRoot}
 → ONE DB read
```

### Trie Log — The State Change Journal

Every time a block is imported, Bonsai writes a **TrieLog** to the database:

```
TrieLog for Block #19,000,000
 ├── account changes: {0xAlice: balance 5→4 ETH, nonce 3→4}
 ├── account changes: {0xBob: balance 2→3 ETH}
 ├── storage changes: {0xContract: slot[0x01] 100→200}
 └── code changes: (none this block)
```

This log is what allows Bonsai to:
- **Roll back** state when a chain reorg occurs (replay the log in reverse)
- **Access historical state** (replay N logs backwards from head)
- The `--bonsai-historical-block-limit` (default 512) controls how far back you can go

---

## Section 8: Transaction Pool Architecture

The transaction pool holds **pending transactions** — valid transactions that have been received but not yet included in a block.

### Key Classes

```
ethereum/eth/
 └── transactions/
      ├── TransactionPool.java          ← main interface + impl
      ├── PendingTransactions.java      ← sorted set of pending txs
      ├── BaseFeePendingTransactions.java ← EIP-1559 specific ordering
      ├── TransactionPoolConfiguration.java ← limits (max size, price floor)
      └── TransactionBroadcaster.java   ← announce new txs to peers
```

### Transaction Pool Flow

```
New transaction received (from RPC or P2P)
        │
        ▼
TransactionPool.addTransaction(tx)
        │
        ├── Check: already known? → reject (duplicate)
        ├── Check: valid signature? → reject if bad
        ├── Check: chain ID matches? → reject if wrong chain
        ├── Check: nonce >= account nonce? → reject if stale
        ├── Check: sender has enough ETH? → reject if insufficient
        ├── Check: gas limit <= block gas limit? → reject if too large
        ├── Check: gas price >= min gas price? → reject if below floor
        │
        ▼
Add to PendingTransactions (sorted by effective gas price DESC, nonce ASC per sender)
        │
        ▼
TransactionBroadcaster.onTransactionAdded(tx)
        │ → announce to connected peers via ETH protocol (NewPooledTransactionHashes)
```

### Transaction Selection for Block Building

When it's time to build a block:

```java
BlockTransactionSelector.selectTransactions(worldState, gasLimit)
    // iterate txs from pool (highest gas price first)
    // for each tx:
    //   - would it exceed block gas limit? → skip
    //   - does sender have enough nonce/balance? → skip
    //   - execute tx in a WorldUpdater
    //   - if success: include in block, update cumulative gas
    //   - if fail: still include (tx is invalid but included, gas consumed)
    // stop when gas limit reached or pool exhausted
```

---

## Section 9: Sync Architecture

Besu's sync system downloads the blockchain from peers and brings the node to the current chain head.

### Sync Pipeline (Snap Sync)

The pipeline in `services/pipeline/` is a **staged async processor**:

```
┌──────────────────────────────────────────────────────────────┐
│                     SNAP SYNC PIPELINE                        │
│                                                               │
│  [CheckpointHeader] → [DownloadHeaders] → [ValidateHeaders]  │
│         │                                         │           │
│         └────────────────────────────────────────►│           │
│                                                   │           │
│  [DownloadBodies] ← [RequestBodies] ←─────────────┘           │
│         │                                                     │
│  [ValidateBlocks] → [ImportBlocks] → [UpdateChainHead]        │
│                                                               │
│  (Parallel):                                                  │
│  [SnapWorldStateDownloader] → downloads trie leaves + proofs │
└──────────────────────────────────────────────────────────────┘
```

Each stage runs in its own thread. Stages are connected by bounded queues (back-pressure: if a downstream stage is slow, upstream stages slow down automatically).

### Key Sync Classes

```
ethereum/eth/sync/
 ├── SyncController.java               ← top-level orchestrator, decides sync mode
 ├── DefaultSynchronizer.java          ← wires together all sync components
 ├── snapsync/
 │    ├── SnapSyncDownloader.java      ← orchestrates snap sync
 │    ├── SnapWorldStateDownloader.java← downloads world state via snap protocol
 │    └── RangeManager.java            ← splits state trie into ranges for parallel download
 ├── fastsync/                         ← deprecated fast sync
 ├── fullsync/
 │    └── FullSyncDownloader.java      ← full sync for archive nodes
 └── PipelineChainDownloader.java      ← the staged pipeline runner
```

---

## Section 10: P2P Network Architecture

### Layer Stack

```
┌─────────────────────────────────────────────┐
│          APPLICATION LAYER                   │
│  EthProtocolManager  │  SnapProtocolManager  │
│  (handles ETH msgs)  │  (handles snap sync)  │
├─────────────────────────────────────────────┤
│          SUBPROTOCOL LAYER                   │
│  EthPeer (per-peer state, request/response)  │
├─────────────────────────────────────────────┤
│          RLPX TRANSPORT LAYER               │
│  RlpxAgent → manages TCP connections        │
│  FramedConnection → ECIES encrypted frames  │
├─────────────────────────────────────────────┤
│          DISCOVERY LAYER                    │
│  DiscoveryAgent → UDP Kademlia DHT          │
│  PeerDiscoveryController                   │
└─────────────────────────────────────────────┘
```

### Key P2P Classes

```
ethereum/p2p/
 ├── network/
 │    ├── DefaultP2PNetwork.java         ← top-level P2P manager
 │    └── P2PNetwork.java               ← interface
 ├── discovery/
 │    ├── DiscoveryAgent.java           ← UDP discovery (find new peers)
 │    ├── PeerDiscoveryController.java  ← Kademlia routing table
 │    └── Endpoint.java                 ← IP:UDP:TCP triplet
 ├── rlpx/
 │    ├── RlpxAgent.java                ← manages all TCP peer connections
 │    ├── connections/
 │    │    └── RlpxConnection.java      ← one TCP connection to one peer
 │    └── handshake/
 │         └── EciesHandshaker.java     ← ECIES key exchange handshake
 └── peers/
      ├── PeerManager.java              ← track connected peers, reputation
      └── StaticNodes.java              ← manual peer list (--static-nodes-file)
```

### ETH Protocol Messages (key ones)

| Message | Direction | Purpose |
|---------|-----------|---------|
| `Status` | both | handshake: share chainId, head block, genesis hash |
| `NewBlockHashes` | both | announce new blocks (EIP-2464) |
| `NewBlock` | both | send a full new block |
| `Transactions` | both | gossip pending transactions |
| `GetBlockHeaders` | → peer | request range of headers |
| `BlockHeaders` | ← peer | response with headers |
| `GetBlockBodies` | → peer | request block bodies by hash |
| `BlockBodies` | ← peer | response with bodies |
| `GetReceipts` | → peer | request transaction receipts |
| `Receipts` | ← peer | response with receipts |
| `GetPooledTransactions` | → peer | request txs from pool by hash |
| `PooledTransactions` | ← peer | response with txs |

---

## Section 11: API Server Architecture

All API servers are built on **Vert.x** — a non-blocking, event-driven framework for the JVM.

### Server Overview

```
ethereum/api/
 ├── jsonrpc/
 │    ├── JsonRpcHttpService.java         ← HTTP server (port 8545 default)
 │    ├── websocket/
 │    │    └── WebSocketService.java      ← WS server (port 8546 default)
 │    ├── engine/
 │    │    └── EngineJsonRpcService.java  ← Engine API (port 8551, JWT auth)
 │    ├── internal/methods/
 │    │    ├── EthGetBalance.java         ← eth_getBalance handler
 │    │    ├── EthSendRawTransaction.java ← eth_sendRawTransaction handler
 │    │    ├── DebugTraceTransaction.java ← debug_traceTransaction handler
 │    │    └── ... (one class per JSON-RPC method)
 │    └── authentication/
 │         └── AuthenticationService.java ← JWT validation for Engine API
 ├── graphql/
 │    └── GraphQLHttpService.java         ← GraphQL server
 └── query/
      └── BlockchainQueries.java          ← high-level query API used by all RPC methods
```

### JSON-RPC Method Registration

Each JSON-RPC method is a class implementing `JsonRpcMethod`:

```java
public interface JsonRpcMethod {
    String getName();                          // "eth_getBalance"
    JsonRpcResponse response(JsonRpcRequest);  // handle the call
}
```

At startup, all method instances are collected into a `Map<String, JsonRpcMethod>` and registered with the HTTP server. Incoming requests are dispatched by method name.

### Supported RPC Namespaces

| Namespace | Purpose |
|-----------|---------|
| `eth_` | Standard Ethereum API (balances, txs, blocks, calls) |
| `net_` | Network info (version, peer count, listening) |
| `web3_` | Client info, sha3 utility |
| `debug_` | Debug/trace (traceTransaction, storageRangeAt) |
| `admin_` | Node administration (peers, logging, node info) |
| `txpool_` | Transaction pool inspection |
| `miner_` | Mining control (start/stop, coinbase) |
| `perm_` | Permissioning management |
| `engine_` | Consensus client Engine API (newPayload, forkchoiceUpdated) |
| `plugins_` | Plugin management (reloadPluginConfig) |

### Engine API (Post-Merge Critical Path)

The Engine API is how the **consensus client drives Besu**:

```
Consensus Client (Teku)
    │
    │ engine_forkchoiceUpdatedV3({headBlockHash, finalizedHash, payloadAttributes})
    ▼
EngineForkchoiceUpdated handler
    │ → MergeCoordinator.updateForkChoice(headHash, finalizedHash, safeHash)
    │ → if payloadAttributes: start building a new block payload
    │ → return {payloadStatus: VALID, payloadId: "0x..."}
    │
    │ engine_getPayloadV3(payloadId)
    ▼
EngineGetPayload handler
    │ → return the built block (ExecutionPayload + BlobsBundle)
    │
    │ [consensus client selects block, broadcasts to other nodes]
    │
    │ engine_newPayloadV3(executionPayload)
    ▼
EngineNewPayload handler
    │ → BlockImporter.importBlock(block)
    │ → return {status: "VALID"} or {status: "INVALID", validationError: "..."}
```

---

## Section 12: Consensus Module Architecture

### QBFT (Recommended for Private Networks)

QBFT uses the **Istanbul BFT** protocol. The consensus round for each block:

```
ROUND START (validator selected as proposer for this round)
     │
     ▼
PROPOSE: proposer creates block → broadcasts PROPOSE message to all validators
     │
     ▼ (all validators receive PROPOSE, validate block)
PRE-PREPARE → PREPARE: each validator broadcasts PREPARE message
     │
     ▼ (wait for 2f+1 PREPARE messages, where f = max faulty nodes)
COMMIT: each validator broadcasts COMMIT message with signature
     │
     ▼ (wait for 2f+1 COMMIT messages)
BLOCK COMMITTED: block is finalized, added to chain
     │
     ▼
NEXT ROUND begins immediately
```

Key QBFT classes:
```
consensus/qbft/
 ├── QbftBlockCreator.java          ← builds the block proposal
 ├── QbftRoundChangeManager.java    ← handles timeout/round change
 ├── QbftProtocolSchedule.java      ← QBFT-specific ProtocolSchedule
 └── jsonrpc/
      └── QbftJsonRpcMethods.java   ← qbft_ JSON-RPC methods (e.g., getValidatorsByBlockNumber)

consensus/qbft-core/
 ├── QbftController.java            ← state machine controller
 ├── QbftRound.java                 ← one round of QBFT consensus
 └── messagedata/
      ├── PrepareMessageData.java
      ├── CommitMessageData.java
      └── ProposalMessageData.java
```

### Post-Merge (PoS) Consensus

After The Merge, **Besu no longer does consensus itself** for public networks. Instead:

```
consensus/merge/
 ├── MergeCoordinator.java          ← coordinates with consensus client
 ├── MergeBlockCreator.java         ← builds blocks on request from Engine API
 ├── TransitionProtocolSchedule.java ← switches from PoW rules to PoS rules at TTD
 └── PayloadAttributes.java         ← fee recipient, timestamp, withdrawals, etc.
```

The `MergeCoordinator` responds to Engine API calls and delegates to `MergeBlockCreator` to assemble payloads.

---

## Section 13: Plugin System Architecture

### Plugin API Module (`plugin-api/`)

This module defines the **public contract** between Besu and plugins. It has zero dependencies on internal Besu modules:

```
plugin-api/src/main/java/org/hyperledger/besu/plugin/
 ├── BesuPlugin.java                ← interface every plugin must implement
 ├── BesuContext.java               ← the service locator passed to plugins
 └── services/
      ├── BesuEvents.java           ← subscribe to blockchain events
      ├── BlockchainService.java    ← read/write blockchain data
      ├── TransactionPoolService.java ← access transaction pool
      ├── StorageService.java       ← create custom key-value storage
      ├── MetricsSystem.java        ← register custom metrics
      ├── RpcEndpointService.java   ← register custom JSON-RPC methods
      ├── TransactionSelectionService.java ← customize tx selection
      └── PluginTransactionValidatorService.java ← custom tx validation
```

### Plugin Lifecycle

```
Besu startup
    │
    ▼
BesuPluginContextImpl.registerPlugins()
    │ scan `plugins/` directory for JARs
    │ for each JAR: use Java ServiceLoader to find BesuPlugin implementations
    │ call plugin.register(context)     ← plugin gets access to BesuContext
    │
    ▼
BesuPluginContextImpl.beforeExternalServices()
    │ call plugin.beforeExternalServices()
    │ (plugins can configure services before RPC/P2P start)
    │
    ▼
[RPC, P2P, metrics servers all start]
    │
    ▼
BesuPluginContextImpl.startPlugins()
    │ call plugin.start()
    │ (plugins begin their own work)
    │
    ▼
[Normal operation]
    │
    ▼
BesuPluginContextImpl.stopPlugins()
    │ call plugin.stop()
    │ (plugins clean up resources)
```

### Example: Writing a Minimal Plugin

```java
@AutoService(BesuPlugin.class)   // registers with ServiceLoader
public class MyPlugin implements BesuPlugin {

    private BesuContext context;

    @Override
    public void register(BesuContext context) {
        this.context = context;
        // get the events service and subscribe to new blocks
        context.getService(BesuEvents.class).ifPresent(events -> {
            events.addBlockAddedListener(block -> {
                System.out.println("New block: " + block.getHeader().getNumber());
            });
        });
    }

    @Override
    public void start() { /* nothing to start */ }

    @Override
    public void stop() { /* nothing to clean up */ }
}
```

---

## Section 14: Metrics Architecture

```
metrics/core/
 ├── MetricsSystem.java               ← interface: createCounter(), createTimer(), etc.
 ├── PrometheusMetricsSystem.java     ← Prometheus implementation
 └── noop/
      └── NoOpMetricsSystem.java      ← no-op for tests

metrics/rocksdb/
 └── RocksDBMetricsFactory.java       ← collects RocksDB internal stats
```

### Key Metrics Categories

| Category | Examples |
|----------|---------|
| **Blockchain** | `besu_blockchain_height`, `besu_blockchain_difficulty` |
| **Sync** | `besu_synchronizer_in_sync`, `besu_synchronizer_chain_head_*` |
| **Transaction Pool** | `besu_transaction_pool_transactions`, `besu_transaction_pool_*_size` |
| **P2P** | `besu_peers_connected`, `besu_peers_discovery_*` |
| **EVM** | `besu_executors_*_task_execution_time` |
| **RPC** | `besu_rpc_*_request_count`, `besu_rpc_*_response_time` |
| **Storage** | `besu_rocksdb_*` (compaction, reads, writes, latency) |

Prometheus scrapes `/metrics` on port 9545. Grafana dashboards visualize everything.

---

## Section 15: Full System Architecture Diagram

```
╔══════════════════════════════════════════════════════════════════════════════╗
║                           BESU PROCESS                                       ║
║                                                                              ║
║  ┌─────────────────────────────────────────────────────────────────────┐    ║
║  │  CLI LAYER  (app/)                                                  │    ║
║  │                                                                     │    ║
║  │  Besu.main() → DaggerBesuComponent → BesuCommand (PicoCLI)         │    ║
║  │  Parses: --network, --sync-mode, --rpc-http-enabled, ...            │    ║
║  └──────────────────────────────┬──────────────────────────────────────┘    ║
║                                 │ run()                                      ║
║  ┌──────────────────────────────▼──────────────────────────────────────┐    ║
║  │  CONTROLLER LAYER  (ethereum/core)                                  │    ║
║  │                                                                     │    ║
║  │  BesuControllerBuilder → BesuController                             │    ║
║  │    ├── ProtocolSchedule (hard fork rules timeline)                  │    ║
║  │    ├── ProtocolContext (blockchain + worldState + consensusCtx)     │    ║
║  │    └── TransactionPool                                              │    ║
║  └──────┬──────────┬──────────────┬──────────────────┬────────────────┘    ║
║         │          │              │                  │                      ║
║  ┌──────▼──┐ ┌─────▼──────┐ ┌────▼──────────┐ ┌────▼──────────────────┐  ║
║  │   EVM   │ │  WORLD     │ │  CONSENSUS    │ │   SYNC MANAGER        │  ║
║  │ MODULE  │ │  STATE     │ │  LAYER        │ │                       │  ║
║  │ (evm/)  │ │ (eth/trie) │ │               │ │  SnapSyncDownloader   │  ║
║  │         │ │            │ │ ┌───────────┐ │ │  FullSyncDownloader   │  ║
║  │ Opcodes │ │ Bonsai     │ │ │QBFT/IBFT2 │ │ │  PipelineDownloader   │  ║
║  │ Precomps│ │ Forest     │ │ │MergeCoord │ │ │  WorldStateDownloader │  ║
║  │ GasCalc │ │ TrieLog    │ │ └───────────┘ │ └───────────────────────┘  ║
║  └─────────┘ └──────┬─────┘ └───────────────┘                            ║
║                     │                                                      ║
║  ┌──────────────────▼──────────────────────────────────────────────────┐  ║
║  │  STORAGE LAYER  (services/kvstore, plugins/rocksdb)                 │  ║
║  │                                                                     │  ║
║  │  KeyValueStorageProvider                                            │  ║
║  │    └── RocksDBKeyValueStorage (column families per data type)       │  ║
║  │         ├── blockchain   (headers, bodies, receipts)                │  ║
║  │         ├── worldstate   (Bonsai flat DB / Forest trie nodes)       │  ║
║  │         └── trielog      (Bonsai state diffs per block)             │  ║
║  └─────────────────────────────────────────────────────────────────────┘  ║
║                                                                            ║
║  ┌─────────────────────────────────────────────────────────────────────┐  ║
║  │  P2P NETWORK LAYER  (ethereum/p2p)                                  │  ║
║  │                                                                     │  ║
║  │  DiscoveryAgent (UDP :30303) → find peers via Kademlia DHT          │  ║
║  │  RlpxAgent (TCP :30303)      → ECIES encrypted peer connections     │  ║
║  │  EthProtocolManager          → ETH subprotocol message handling     │  ║
║  │  EthPeer                     → per-peer state + request tracking    │  ║
║  └─────────────────────────────────────────────────────────────────────┘  ║
║                                                                            ║
║  ┌─────────────────────────────────────────────────────────────────────┐  ║
║  │  API LAYER  (ethereum/api)                                          │  ║
║  │                                                                     │  ║
║  │  JsonRpcHttpService  (:8545)  → eth_, net_, debug_, admin_, ...     │  ║
║  │  WebSocketService    (:8546)  → same + eth_subscribe subscriptions  │  ║
║  │  GraphQLHttpService  (:8547)  → GraphQL queries                     │  ║
║  │  EngineJsonRpcService(:8551)  → engine_* (JWT authenticated)        │  ║
║  │  MetricsHttpService  (:9545)  → Prometheus /metrics scrape          │  ║
║  └─────────────────────────────────────────────────────────────────────┘  ║
║                                                                            ║
║  ┌─────────────────────────────────────────────────────────────────────┐  ║
║  │  PLUGIN LAYER  (plugin-api/, plugins/)                              │  ║
║  │                                                                     │  ║
║  │  BesuPluginContextImpl → loads JARs from plugins/ via ServiceLoader │  ║
║  │  Plugin lifecycle: register → beforeExternal → start → stop        │  ║
║  │  Plugins can: add RPC methods, subscribe events, custom tx select   │  ║
║  └─────────────────────────────────────────────────────────────────────┘  ║
╚══════════════════════════════════════════════════════════════════════════════╝
         │ Engine API (JWT)                    │ DevP2P TCP/UDP
         ▼                                     ▼
  Consensus Client                     Other Besu / ETH Nodes
  (Teku, Lighthouse, etc.)             (Geth, Nethermind, etc.)
```

---

## Section 16: Quick Reference — Where to Find Things in the Code

| Feature / Concept | Module | Key Class |
|-------------------|--------|-----------|
| Application entry point | `app` | `Besu.java` |
| CLI argument parsing | `app` | `BesuCommand.java` |
| Dependency injection root | `app` | `BesuComponent.java` |
| Hard fork scheduling | `ethereum/core` | `MainnetProtocolSchedule.java` |
| EVM execution loop | `evm` | `EVM.java` |
| EVM execution context | `evm` | `MessageFrame.java` |
| EVM opcodes | `evm` | `evm/operation/*.java` |
| Gas cost calculation | `evm` | `GasCalculator` subclasses |
| Precompiled contracts | `evm` | `evm/precompile/*.java` |
| Ethereum core types | `datatypes` | `Address`, `Hash`, `Wei`, `Transaction` |
| Block processing | `ethereum/core` | `BlockProcessor.java` |
| Block import | `ethereum/core` | `BlockImporter.java` |
| Transaction processing | `ethereum/core` | `MainnetTransactionProcessor.java` |
| World state (Bonsai) | `ethereum/core` | `BonsaiWorldStateProvider.java` |
| World state (Forest) | `ethereum/core` | `ForestWorldStateArchive.java` |
| Trie implementation | `ethereum/trie` | `MerklePatriciaTrie.java` |
| Transaction pool | `ethereum/eth` | `TransactionPool.java` |
| Snap sync | `ethereum/eth` | `SnapSyncDownloader.java` |
| ETH wire protocol | `ethereum/eth` | `EthProtocolManager.java` |
| P2P network | `ethereum/p2p` | `DefaultP2PNetwork.java` |
| Peer discovery | `ethereum/p2p` | `DiscoveryAgent.java` |
| RLPx transport | `ethereum/p2p` | `RlpxAgent.java` |
| JSON-RPC HTTP server | `ethereum/api` | `JsonRpcHttpService.java` |
| Engine API | `ethereum/api` | `EngineJsonRpcService.java` |
| All RPC methods | `ethereum/api` | `jsonrpc/internal/methods/*.java` |
| QBFT consensus | `consensus/qbft` | `QbftController.java` |
| QBFT state machine | `consensus/qbft-core` | `QbftRound.java` |
| Post-Merge coordinator | `consensus/merge` | `MergeCoordinator.java` |
| Plugin interfaces | `plugin-api` | `BesuPlugin.java`, `BesuContext.java` |
| Plugin lifecycle | `app` | `BesuPluginContextImpl.java` |
| RocksDB storage | `plugins/rocksdb` | `RocksDBKeyValueStorage.java` |
| Metrics | `metrics/core` | `PrometheusMetricsSystem.java` |
| RLP encoding | `ethereum/rlp` | `RLP.java` |
| Cryptographic primitives | `crypto/algorithms` | `SignatureAlgorithm.java` |

---

## 🧠 Recap

- ✅ Besu is a **Gradle multi-project** with ~30 subprojects, each a focused JAR
- ✅ **Dagger 2** wires the application at compile time — no reflection, fast startup
- ✅ **BesuCommand** (PicoCLI) is the CLI entry point with hundreds of options
- ✅ **BesuController** is the central wiring point for all subsystems
- ✅ **ProtocolSchedule** is the hard-fork timeline — the right rules for every block
- ✅ **EVM module** is standalone and extractable — runs the bytecode for every smart contract call
- ✅ **World state** has two implementations: Bonsai (efficient, recommended) and Forest (archive nodes)
- ✅ **Snap sync** pipeline downloads state in parallel ranges — much faster than old fast sync
- ✅ **Vert.x** powers all API servers — non-blocking, high-throughput event loop
- ✅ **Engine API** is the JWT-authenticated bridge to the consensus client (post-Merge)
- ✅ **Plugin API** lets you extend Besu without touching core code
- ✅ **RocksDB** is the persistent store for everything — blockchain, state, trie logs

---

## ➡️ Next Up

You now understand the full architecture. Let's trace the **complete lifecycle** of the application from `./bin/besu` command all the way to processing a live transaction.

👉 Move on to **[A4_application_flow.md](./A4_application_flow.md)**