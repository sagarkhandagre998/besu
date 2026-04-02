# A4 — Complete Application Flow of Besu (Gradle Application)

> 🔴 **Level:** Contributor | ⏱️ **Read Time:** ~60 minutes

---

## Introduction

This document traces the **complete lifecycle** of Hyperledger Besu — from the moment you type `./bin/besu` in your terminal to the moment a real Ethereum transaction is executed, committed to disk, and acknowledged back to a dApp.

Think of this as reading the story of what happens inside the JVM, step by step.

---

## Section 1: The Build System — Gradle Multi-Project

Before we run Besu, we need to **build** it. Understanding the build system helps you navigate the codebase.

### Project Layout

```
besu/                          ← repo root
├── settings.gradle            ← declares ALL subprojects
├── build.gradle               ← root build config (shared rules)
├── gradle.properties          ← version numbers, JVM args
├── gradlew                    ← Gradle wrapper script (use this, not system gradle)
└── gradlew.bat                ← Windows version
```

### settings.gradle — All Subprojects

```groovy
rootProject.name = 'besu'

include 'app'
include 'config'
include 'consensus:clique'
include 'consensus:common'
include 'consensus:ibft'
include 'consensus:ibftlegacy'
include 'consensus:merge'
include 'consensus:qbft'
include 'consensus:qbft-core'
include 'crypto:algorithms'
include 'crypto:services'
include 'datatypes'
include 'ethereum:api'
include 'ethereum:blockcreation'
include 'ethereum:core'
include 'ethereum:eth'
include 'ethereum:evmtool'
include 'ethereum:p2p'
include 'ethereum:permissioning'
include 'ethereum:referencetests'
include 'ethereum:rlp'
include 'ethereum:trie'
include 'ethereum:verkletrie'
include 'evm'
include 'metrics:core'
include 'metrics:rocksdb'
include 'nat'
include 'platform'
include 'plugin-api'
include 'plugins:rocksdb'
include 'services:kvstore'
include 'services:pipeline'
include 'services:tasks'
include 'testfuzz'
include 'testutil'
include 'util'
```

Each entry is an independent Gradle subproject — its own `build.gradle`, its own `src/` tree, its own unit tests.

### Key Gradle Tasks

| Task | What It Does |
|------|-------------|
| `./gradlew build` | Compile ALL modules + run unit tests + generate JARs |
| `./gradlew installDist` | Build + assemble the runnable distribution in `build/install/besu/` |
| `./gradlew test` | Run unit tests across all modules |
| `./gradlew integrationTest` | Run integration tests (spin up in-process components) |
| `./gradlew acceptanceTest` | Run acceptance tests (launch real Besu Docker containers) |
| `./gradlew ethereum:referencetests:referenceTests` | Run Ethereum Foundation EVM test vectors |
| `./gradlew spotlessApply` | Auto-format all Java code (Google Java Format) |
| `./gradlew :evm:jar` | Build only the EVM subproject JAR |
| `./gradlew clean` | Delete all build output |

### Module Dependency Graph (Key Edges)

```
app
 ├──► ethereum/api      ──► ethereum/core ──► evm ──► datatypes
 ├──► ethereum/eth      ──► ethereum/p2p
 ├──► consensus/qbft    ──► consensus/qbft-core
 ├──► consensus/merge
 ├──► metrics/core
 ├──► plugin-api
 └──► plugins/rocksdb   ──► services/kvstore
                                  └──► plugin-api
```

**Rule**: dependencies flow downward only. `evm` never imports `ethereum/api`. Gradle enforces this.

### After `./gradlew installDist`

```
build/install/besu/
├── bin/
│   ├── besu           ← Linux/Mac shell script
│   └── besu.bat       ← Windows batch script
└── lib/
    ├── besu.jar       ← main app JAR (contains Besu.java main class)
    ├── ethereum-core-*.jar
    ├── evm-*.jar
    ├── rocksdb-*.jar
    └── ... (all dependency JARs on classpath)
```

---

## Section 2: Phase 1 — JVM Startup

You type:

```bash
./build/install/besu/bin/besu \
  --network=mainnet \
  --sync-mode=SNAP \
  --rpc-http-enabled \
  --engine-rpc-enabled \
  --engine-jwt-secret=/path/to/jwtsecret
```

Here is exactly what happens:

### Step 1: Shell Script Executes

The `bin/besu` script:
```bash
#!/bin/sh
# (simplified)
exec java \
  -Dvertx.disableFileCPResolving=true \
  -Dbesu.home=/path/to/besu \
  -cp "/path/to/besu/lib/*" \
  org.hyperledger.besu.Besu \
  "$@"
```

It sets the classpath to **all JARs in `lib/`** and calls the main class `org.hyperledger.besu.Besu` with your CLI arguments forwarded as `$@`.

### Step 2: JVM Loads `Besu.java`

```
app/src/main/java/org/hyperledger/besu/Besu.java
```

```java
public final class Besu {

  public static void main(final String... args) {
    setupLogging();                              // Step 3
    final BesuComponent besuComponent =
        DaggerBesuComponent.create();            // Step 4
    final BesuCommand besuCommand =
        besuComponent.getBesuCommand();          // Step 5
    int exitCode =
        besuCommand.parse(
            new RunLast(),
            besuCommand.parameterExceptionHandler(),
            besuCommand.executionExceptionHandler(),
            System.in,
            besuComponent,
            args);                               // Step 6
    System.exit(exitCode);
  }
}
```

### Step 3: `setupLogging()`

- Sets Netty's internal logger to **Log4j2**
- Sets Vert.x logger delegate to **Log4j2**
- Sets `log4j.configurationFactory` to `BesuLoggingConfigurationFactory`
- This ensures all async networking frameworks use the same log4j2 output

### Step 4: `DaggerBesuComponent.create()`

This is compile-time generated code. Dagger built `DaggerBesuComponent.java` when you ran `./gradlew build`. At runtime it:
- Instantiates `BesuPluginContextImpl` (the plugin manager)
- Instantiates `BesuCommand` with all its `@Inject` dependencies satisfied
- All of this without any reflection — just plain `new` calls

### Step 5: `besuComponent.getBesuCommand()`

Returns the fully constructed `BesuCommand` instance. At this point the object exists in memory but nothing has been parsed or started yet.

### Step 6: `besuCommand.parse(..., args)`

PicoCLI begins parsing. This kicks off Phase 2.

---

## Section 3: Phase 2 — CLI Parsing (PicoCLI)

PicoCLI is the command-line framework. It reads `args[]` and populates fields annotated with `@Option` inside `BesuCommand`.

### What Happens During Parse

```
args[] = ["--network=mainnet", "--sync-mode=SNAP", "--rpc-http-enabled", ...]
    │
    ▼
PicoCLI reads each argument
    │
    ├── "--network=mainnet"      → besuCommand.network = NetworkName.MAINNET
    ├── "--sync-mode=SNAP"       → besuCommand.syncMode = SyncMode.SNAP
    ├── "--rpc-http-enabled"     → besuCommand.rpcHttpEnabled = true
    ├── "--engine-rpc-enabled"   → besuCommand.engineRpcEnabled = true
    └── "--engine-jwt-secret=.." → besuCommand.engineJwtKeyFile = Path(...)
```

### Config File Loading

If `--config-file=besu.toml` is provided, PicoCLI also reads the TOML file:

```toml
# besu.toml example
network="mainnet"
sync-mode="SNAP"
data-path="/var/lib/besu"
rpc-http-enabled=true
metrics-enabled=true
```

Precedence: **CLI flags > config file > built-in defaults**.

### Subcommand Detection

PicoCLI checks if the first non-option argument is a subcommand:

```
besu blocks import --from=chain.rlp   → BlocksSubCommand.ImportSubCommand runs, then exit
besu operator generate-blockchain-config ...  → OperatorSubCommand runs, then exit
besu rlp encode ...   → RLPSubCommand.EncodeSubCommand runs, then exit
```

If no subcommand: PicoCLI calls `BesuCommand.run()` via the `RunLast` execution strategy.

---

## Section 4: Phase 3 — Configuration and Validation

`BesuCommand.run()` now executes. First it validates everything and builds configs.

### Step 1: Validate Option Combinations

```java
// Examples of validation checks:
if (isMiningEnabled && coinbase == null) {
    throw new ParameterException("--miner-coinbase required when --miner-enabled");
}
if (syncMode == CHECKPOINT && !genesisHasCheckpoint()) {
    logger.warn("Checkpoint sync specified but no checkpoint in genesis, falling back to snap sync");
}
if (engineRpcEnabled && engineJwtKeyFile == null) {
    throw new ParameterException("--engine-jwt-secret required when --engine-rpc-enabled");
}
```

### Step 2: Load Genesis File

```java
GenesisConfigFile genesisConfigFile;

if (network == NetworkName.MAINNET) {
    // load built-in genesis from classpath:
    // config/src/main/resources/mainnet.json
    genesisConfigFile = GenesisConfigFile.mainnet();
} else if (genesisFile != null) {
    genesisConfigFile = GenesisConfigFile.fromFile(genesisFile);
}
```

The genesis JSON contains:
```json
{
  "config": {
    "chainId": 1,
    "homesteadBlock": 1150000,
    "berlinBlock": 12244000,
    "londonBlock": 12965000,
    "terminalTotalDifficulty": "58750003716598352816469"
  },
  "alloc": {
    "0xde0B295669a9FD93d5F28D9Ec85E40f4cb697BAe": {
      "balance": "0x59..."
    }
  },
  "difficulty": "0x400000000",
  "gasLimit": "0x1388"
}
```

### Step 3: Detect Consensus Type

```java
GenesisConfigOptions genesisOptions = genesisConfigFile.getConfigOptions();

if (genesisOptions.isQbft())   → consensusType = QBFT
if (genesisOptions.isIbft2())  → consensusType = IBFT2
if (genesisOptions.isClique()) → consensusType = CLIQUE
// default for Mainnet:         → consensusType = MERGE (PoS post-TTD)
```

### Step 4: Print Configuration Overview

Added in 2022, Besu prints a startup summary:

```
╔════════════════════════════════════════════════╗
║                 BESU CONFIGURATION             ║
╠════════════════════════════════════════════════╣
║ Java           : Oracle OpenJDK 21.0.2         ║
║ Network        : Mainnet                       ║
║ Sync mode      : SNAP                          ║
║ Data directory : /var/lib/besu                 ║
║ Engine RPC     : ENABLED (port 8551)           ║
╚════════════════════════════════════════════════╝
```

### Step 5: Initialize Metrics System

```java
MetricsSystem metricsSystem;
if (metricsEnabled) {
    metricsSystem = new PrometheusMetricsSystem(metricsConfig, ...);
} else {
    metricsSystem = new NoOpMetricsSystem();
}
```

The `MetricsSystem` is passed to virtually every subsystem so they can register counters, gauges, and timers.

---

## Section 5: Phase 4 — Storage Initialization

Before building the controller, Besu must open the database.

### Step 1: Determine Data Directory

```java
Path dataDir = dataPath.orElse(
    Path.of(System.getProperty("user.home"), ".besu", network.name().toLowerCase())
);
// e.g., /home/user/.besu/mainnet/
```

### Step 2: Check Existing Database

```java
DatabaseMetadata dbMetadata = DatabaseMetadata.lookUpFrom(dataDir);

if (dbMetadata.exists()) {
    int existingVersion = dbMetadata.getVersion();
    if (existingVersion < CURRENT_DB_VERSION) {
        // run migration
        StorageMigration.migrate(dataDir, existingVersion, CURRENT_DB_VERSION);
    }
}
```

### Step 3: Open RocksDB

RocksDB is opened with multiple **column families** — think of these as separate namespaces in the same database file:

```java
RocksDBKeyValueStorageFactory storageFactory = new RocksDBKeyValueStorageFactory(...);

// Column families opened:
// BLOCKCHAIN      - block headers, block bodies, receipts, tx index
// WORLD_STATE     - Bonsai flat DB accounts + storage
// TRIE_BRANCH_STORAGE  - Bonsai trie nodes
// TRIE_LOG_STORAGE     - Bonsai trie logs (state diffs per block)
// VARIABLES            - chain metadata (head block hash, etc.)
// PRUNING_STATE        - (if using Forest pruning)
```

### Step 4: Build Blockchain

```java
DefaultBlockchain blockchain = DefaultBlockchain.createOrLoad(
    genesisState,        // genesis block
    storageProvider,     // RocksDB-backed
    metricsSystem
);
```

If a genesis block doesn't exist in the DB, `GenesisState.writeStateTo()` writes all pre-funded accounts from the genesis `alloc` section into the world state, then stores genesis block as block #0.

### Step 5: Build World State Archive

```java
WorldStateArchive worldStateArchive;
if (dataStorageFormat == BONSAI) {
    worldStateArchive = new BonsaiWorldStateProvider(
        storageProvider,
        blockchain,
        bonsaiHistoricalBlockLimit,   // default 512
        metricsSystem
    );
} else {
    worldStateArchive = new ForestWorldStateArchive(storageProvider);
}
```

---

## Section 6: Phase 5 — BesuController Construction

Now all the major subsystems are wired together.

### Step 1: Select the Right Builder

```java
BesuControllerBuilder builder =
    new BesuControllerBuilderFactory()
        .fromEthNetworkConfig(networkConfig, consensusType);
// Returns one of:
//   MainnetBesuControllerBuilder  → for Mainnet/testnets (PoS)
//   QbftBesuControllerBuilder     → for QBFT private networks
//   IbftBesuControllerBuilder     → for IBFT 2.0 private networks
//   CliqueBesuControllerBuilder   → for Clique networks
```

### Step 2: Build ProtocolSchedule

```java
ProtocolSchedule protocolSchedule =
    MainnetProtocolSchedule.create(genesisConfigOptions, privacyParameters, ...);
```

This creates the full hard-fork timeline. For Mainnet it looks like:

```
Block 0          → FrontierProtocolSpec
Block 1,150,000  → HomesteadProtocolSpec
...
Block 12,965,000 → LondonProtocolSpec      (EIP-1559 fee market)
Block 15,537,394 → MergeProtocolSpec       (PoW → PoS transition)
Block 17,034,870 → ShanghaiProtocolSpec    (withdrawals)
Block 19,426,587 → CancunProtocolSpec      (EIP-4844 blobs)
```

Each `ProtocolSpec` bundles: `EVM`, `GasCalculator`, block/tx validators, precompile registry.

### Step 3: Build ProtocolContext

```java
ProtocolContext protocolContext = new ProtocolContext(
    blockchain,           // the canonical chain
    worldStateArchive,    // Bonsai or Forest world state access
    consensusContext      // QBFT: ValidatorProvider; PoS: MergeContext
);
```

### Step 4: Build TransactionPool

```java
TransactionPool transactionPool = TransactionPoolFactory.createTransactionPool(
    protocolSchedule,
    protocolContext,
    ethContext,
    metricsSystem,
    syncState,
    transactionPoolConfig  // max size, price floor, etc.
);
```

### Step 5: Build MiningCoordinator (or MergeCoordinator)

For **PoS (Mainnet)**:
```java
MergeCoordinator miningCoordinator = new MergeCoordinator(
    protocolContext,
    protocolSchedule,
    transactionPool,
    miningParameters
);
```

For **QBFT (private)**:
```java
QbftBlockCreatorFactory blockCreatorFactory = new QbftBlockCreatorFactory(...);
QbftMiningCoordinator miningCoordinator = new QbftMiningCoordinator(
    bftEventQueue,
    qbftController,
    blockCreatorFactory,
    ...
);
```

### Step 6: Load Plugins

```java
besuPluginContext.registerPlugins(pluginsDir);
// Java ServiceLoader scans plugins/ dir for JARs
// For each discovered BesuPlugin implementation:
//   plugin.register(besuContext)
//   → plugin can now query BesuContext for services
```

---

## Section 7: Phase 6 — Runner Construction (RunnerBuilder)

The `Runner` ties together all the network-facing services. It is built by `RunnerBuilder`.

### EthProtocolManager

```java
EthProtocolManager ethProtocolManager = new EthProtocolManager(
    blockchain,
    networkId,
    worldStateArchive,
    transactionPool,
    ethProtocolConfiguration,
    peerManager,
    syncConfig,
    scheduler
);
```

This handles all ETH wire protocol messages from peers: `GetBlockHeaders`, `BlockBodies`, `Transactions`, `NewBlock`, etc.

### Synchronizer

```java
Synchronizer synchronizer = new DefaultSynchronizer(
    syncConfig,
    protocolSchedule,
    protocolContext,
    worldStateStorage,
    ethProtocolManager.getEthContext(),
    pivotBlockSelector,
    syncState,
    metricsSystem
);
```

### P2P Network

```java
P2PNetwork p2pNetwork = DefaultP2PNetwork.builder()
    .config(networkConfig)
    .nodeKey(nodeKey)                // our node's private key
    .blockchain(blockchain)
    .metricsSystem(metricsSystem)
    .supportedCapabilities(capabilities)  // ETH/68, SNAP/1, etc.
    .build();
```

### NAT Detection

```java
NatService natService = new NatService(natConfig);
// Detects: are we behind UPnP router? Kubernetes? Docker?
// Discovers external IP + port if needed
// Sets advertised address for peer discovery
```

### API Servers

```java
// JSON-RPC HTTP
JsonRpcHttpService jsonRpcHttpService = new JsonRpcHttpService(
    vertx,
    dataDir,
    jsonRpcConfig,       // port 8545, cors, allowlist
    metricsSystem,
    natService,
    methodsMap,          // Map<String, JsonRpcMethod>
    ...
);

// WebSocket
WebSocketService webSocketService = new WebSocketService(
    vertx,
    webSocketConfig,     // port 8546
    requestHandler
);

// Engine API (JWT-authenticated)
EngineJsonRpcService engineRpcService = new EngineJsonRpcService(
    vertx,
    dataDir,
    engineConfig,        // port 8551
    metricsSystem,
    natService,
    engineMethodsMap,
    jwtAuthProvider     // validates JWT from consensus client
);
```

### All of this is assembled into a `Runner`:

```java
Runner runner = new Runner(
    vertx,
    p2pNetwork,
    ethProtocolManager,
    synchronizer,
    jsonRpcHttpService,
    webSocketService,
    graphQLService,
    engineRpcService,
    metricsService,
    natService,
    miningCoordinator,
    besuPluginContext,
    dataDir
);
```

---

## Section 8: Phase 7 — Starting All Services

`runner.startExternalServices()` and then `runner.startEthereumMainLoop()` are called.

### Startup Order (matters!)

```
 1. metricsService.start()
    └── Prometheus HTTP server starts on :9545
        └── /metrics endpoint now live

 2. p2pNetwork.start()
    └── DiscoveryAgent starts UDP socket on :30303
        └── sends PING to bootnodes
    └── RlpxAgent starts TCP listener on :30303
        └── ready to accept incoming peer connections

 3. besuPluginContext.beforeExternalServices()
    └── plugins get a chance to configure before APIs start

 4. jsonRpcHttpService.start()
    └── Vert.x HTTP server starts on :8545
        └── accepts eth_*, net_*, debug_*, admin_* calls

 5. webSocketService.start()
    └── Vert.x WS server starts on :8546
        └── accepts same + eth_subscribe

 6. engineRpcService.start()
    └── Vert.x HTTP server starts on :8551 (JWT only)
        └── accepts engine_newPayload, engine_forkchoiceUpdated, etc.

 7. besuPluginContext.startPlugins()
    └── plugins.forEach(plugin -> plugin.start())

 8. synchronizer.start()
    └── begins sync process (connect to peers, download headers, etc.)

 9. miningCoordinator.start()
    └── for QBFT: starts listening for PROPOSE messages
    └── for PoS: MergeCoordinator waits for Engine API commands

10. besuPluginContext.afterExternalServicesMainLoop()
    └── plugins notified everything is running

11. Register JVM shutdown hook:
    Runtime.getRuntime().addShutdownHook(new Thread(() -> runner.close()));
```

---

## Section 9: Phase 8 — The Running Node (Concurrent Loops)

Besu is now fully running. Many things happen concurrently:

### Loop 1: Peer Discovery (UDP Thread)

```
Every few seconds:
  DiscoveryAgent processes incoming UDP packets:
    PING  → reply with PONG
    PONG  → add peer to routing table
    FindNode → reply with Neighbors (closest known peers)
    Neighbors → add new peer candidates

  PeerDiscoveryController:
    → try to establish RLPx TCP connections to discovered peers
    → maintain routing table (Kademlia k-buckets)
```

### Loop 2: Peer Connection Management (RLPx)

```
RlpxAgent maintains N TCP connections (target: --max-peers, default 25):
  New inbound connection:
    → ECIES handshake (key exchange)
    → RLPx hello message exchange (capabilities negotiation)
    → ETH Status message exchange (genesis hash, head block, chain ID)
    → peer added to EthPeers

  Peer disconnects:
    → remove from peer list
    → try to find replacement via discovery
```

### Loop 3: Block Sync Loop

```
SyncController monitors:
  Are we at chain head? → IDLE
  Behind chain head?    → trigger appropriate sync mode

Snap Sync (main sync from scratch):
  Phase 1: Download and validate block headers (from checkpoint to head)
  Phase 2: Download block bodies
  Phase 3: In parallel - SnapWorldStateDownloader:
    → request account ranges from peers (SNAP protocol)
    → request storage ranges
    → request bytecodes
    → verify range proofs (Merkle proofs)
    → build local trie from downloaded leaves
  Phase 4: When world state complete - switch to live block import
```

### Loop 4: Transaction Gossip

```
EthProtocolManager receives NewPooledTransactionHashes from peer:
  → request full txs for unknown hashes (GetPooledTransactions)
  → receive txs (PooledTransactions)
  → validate each tx
  → add to TransactionPool
  → re-announce to other peers (minus the one who sent it)
```

### Loop 5: Block Announcement (Live, post-sync)

```
EthProtocolManager receives NewBlock from peer:
  → validate block header (parentHash matches our head)
  → request block if we don't have it (GetBlockBodies)
  → BlockImporter.importBlock(block)
  → if valid: append to blockchain, notify subscribers
  → announce to other connected peers (minus sender)
```

### Loop 6: Engine API Polling (PoS Mainnet)

```
Consensus client (Teku) calls us via Engine API every ~12 seconds:

  engine_forkchoiceUpdatedV3:
    → update our view of canonical chain head
    → if payloadAttributes: start building a block

  engine_getPayloadV3:
    → return the block we built (ExecutionPayload)

  engine_newPayloadV3:
    → consensus client tells us to execute and validate a block
    → BlockImporter.importBlock(block)
    → return {status: "VALID"} or {status: "INVALID"}
```

### Loop 7: Vert.x Event Loops (API Requests)

```
Multiple Vert.x event loop threads run simultaneously:
  → Each event loop handles many concurrent HTTP connections
  → Non-blocking: while waiting for DB read, thread handles another request
  → JSON-RPC request arrives → parse → dispatch to JsonRpcMethod → DB queries
  → → → build response → send back
```

---

## Section 10: Phase 9 — Deep Trace: One Transaction End to End

Let's follow a single transaction from a user's dApp all the way through Besu.

### The Scenario

A dApp calls `eth_sendRawTransaction` to transfer 0.1 ETH from Alice to Bob.

### Step 1: Raw Transaction Arrives

```
dApp sends HTTP POST to :8545:

{
  "jsonrpc": "2.0",
  "method": "eth_sendRawTransaction",
  "params": [
    "0x02f872018459682f008459682f0e82520894..."
  ],
  "id": 1
}
```

The hex string is an **RLP-encoded, signed EIP-1559 transaction**.

### Step 2: Vert.x HTTP Server Receives the Request

```
JsonRpcHttpService (running on Vert.x event loop)
  → HttpServerRequest received
  → read body (async, non-blocking)
  → parse JSON → create JsonRpcRequest object
  → look up method: methodsMap.get("eth_sendRawTransaction")
  → returns: EthSendRawTransaction instance
  → call: ethSendRawTransaction.response(request)
```

### Step 3: EthSendRawTransaction.response()

```java
// ethereum/api/.../EthSendRawTransaction.java
public JsonRpcResponse response(JsonRpcRequest request) {
    String rawTxHex = request.getRequiredParameter(0, String.class);
    Bytes rawTxBytes = Bytes.fromHexString(rawTxHex);

    // RLP decode the transaction
    Transaction transaction = Transaction.readFrom(RLP.input(rawTxBytes));

    // Submit to transaction pool
    ValidationResult<TransactionInvalidReason> result =
        transactionPool.addTransactionViaApi(transaction);

    if (result.isValid()) {
        return new JsonRpcSuccessResponse(
            request.getId(),
            transaction.getHash().toString()  // "0xabc123..."
        );
    } else {
        return new JsonRpcErrorResponse(
            request.getId(),
            convertToRpcError(result.getInvalidReason())
        );
    }
}
```

### Step 4: RLP Decoding the Transaction

```
Bytes: 0x02f872018459682f008459682f0e82520894...
         │
         ▼
RLP.decode()
         │
         ▼
Transaction {
  type:                 EIP1559 (0x02)
  chainId:              1 (Mainnet)
  nonce:                42
  maxPriorityFeePerGas: 1500000000 (1.5 gwei)
  maxFeePerGas:         1500000014 (1.5 gwei + margin)
  gasLimit:             21000
  to:                   0xBob...
  value:                100000000000000000 (0.1 ETH in wei)
  data:                 0x (empty - simple transfer)
  accessList:           []
  v, r, s:              (ECDSA signature components)
}
```

### Step 5: Transaction Pool Validation

```
TransactionPool.addTransactionViaApi(transaction)
  │
  ├── DUPLICATE CHECK:
  │   Is transaction.getHash() already in pendingTransactions?
  │   → YES: return ALREADY_KNOWN
  │
  ├── SIGNATURE VALIDATION:
  │   sender = ECDSA.recoverSender(transaction)
  │   → recovers Alice's address from (v, r, s) + transaction hash
  │   → verifies it's a valid secp256k1 signature
  │   → FAIL: return INVALID_SIGNATURE
  │
  ├── CHAIN ID CHECK:
  │   transaction.getChainId() == network.getChainId()?
  │   → 1 == 1 ✓
  │
  ├── NONCE CHECK:
  │   worldState.getAccount(Alice).getNonce() == 42?
  │   current nonce = 42? → VALID (tx nonce matches, not stale or future)
  │
  ├── BALANCE CHECK:
  │   Alice's balance >= (value + gasLimit * maxFeePerGas)?
  │   = 0.1 ETH + 21000 * 1500000014 wei
  │   = 0.1 ETH + ~0.0000315 ETH
  │   Alice has 1.5 ETH → ✓ VALID
  │
  ├── GAS LIMIT CHECK:
  │   transaction.getGasLimit() <= blockGasLimit?
  │   21000 <= 30,000,000 → ✓ VALID
  │
  └── GAS PRICE FLOOR CHECK:
      transaction.getMaxFeePerGas() >= minGasPrice?
      1500000000 >= 1000000000 → ✓ VALID
```

All checks pass. Transaction is valid.

### Step 6: Added to Pending Transactions

```java
pendingTransactions.addTransaction(transaction, Optional.of(Alice));
// Internally: PriorityQueue sorted by effective gas price (DESC) then nonce (ASC per sender)
// Alice's tx now sits at some position in the queue
```

### Step 7: Broadcast to Peers

```java
transactionBroadcaster.onTransactionAdded(transaction);
// For each connected peer (up to 25):
//   → send NewPooledTransactionHashes(type=0x02, [tx.getHash()])
//   → peers can then request the full tx if they want it
```

### Step 8: Return Hash to Caller

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": "0xabc123def456..."
}
```

The dApp now has the transaction hash. The transaction is in the pool but **not yet confirmed**.

---

### Time Passes... Block Proposal Time (~12 seconds for PoS)

### Step 9: Consensus Client Requests a Block Payload

```
Teku (consensus client) → engine_forkchoiceUpdatedV3({
  headBlockHash: "0xLatestHead...",
  safeBlockHash: "0xSafeHead...",
  finalizedBlockHash: "0xFinalHead...",
  payloadAttributes: {
    timestamp: 1710000012,
    prevRandao: "0x...",
    suggestedFeeRecipient: "0xValidatorAddress...",
    withdrawals: [...]
  }
})
```

### Step 10: MergeCoordinator Builds a Block

```
MergeCoordinator.preparePayload(payloadAttributes)
  │
  ▼
MergeBlockCreator.createBlock(parentHeader, timestamp, ...)
  │
  ├── Pull transactions from pool:
  │   BlockTransactionSelector.selectTransactions(worldState, gasLimit)
  │   │
  │   ├── iterate pendingTransactions (highest gas price first)
  │   ├── Alice's tx is selected (gas price 1.5 gwei, within block gas limit)
  │   └── collect selected txs into List<Transaction>
  │
  ├── Execute all selected transactions:
  │   for each tx in selectedTxs:
  │     WorldUpdater blockUpdater = worldState.updater()
  │     TransactionProcessor.processTransaction(tx, blockUpdater)
  │       → see Step 14 for details
  │     receipts.add(txReceipt)
  │
  ├── Process withdrawals (Shanghai+):
  │   for each withdrawal in payloadAttributes.withdrawals:
  │     add ETH directly to validator's balance (no EVM execution)
  │
  ├── Compute stateRoot:
  │   blockUpdater.commit()
  │   stateRoot = worldState.rootHash()
  │
  ├── Compute transactionsRoot (Merkle root of tx list)
  ├── Compute receiptsRoot (Merkle root of receipts)
  ├── Compute withdrawalsRoot
  │
  └── Assemble ExecutionPayload {
        parentHash, feeRecipient, stateRoot, receiptsRoot,
        logsBloom, prevRandao, blockNumber, gasLimit,
        gasUsed, timestamp, extraData, baseFeePerGas,
        blockHash, transactions, withdrawals
      }
```

### Step 11: Consensus Client Retrieves the Payload

```
Teku → engine_getPayloadV3(payloadId)
Besu ← ExecutionPayload + BlobsBundle (if any blobs)
```

Teku broadcasts the payload to the beacon network as a `SignedBeaconBlock`.

### Step 12: Consensus Client Confirms the Block

After the block gets enough attestations from other validators:

```
Teku → engine_newPayloadV3(executionPayload)
```

### Step 13: Block Import — BlockImporter

```
BlockImporter.importBlock(block, HeaderValidationMode.FULL, ...)
  │
  ├── HEADER VALIDATION:
  │   BlockHeaderValidator.validateHeader(header, parentHeader, ...)
  │   ├── parentHash == parent.getHash() ✓
  │   ├── blockNumber == parent.getNumber() + 1 ✓
  │   ├── timestamp > parent.getTimestamp() ✓
  │   ├── gasLimit within allowed range of parent ✓
  │   ├── gasUsed <= gasLimit ✓
  │   └── (PoS specific) prevRandao, feeRecipient, etc.
  │
  ├── BODY VALIDATION:
  │   BlockBodyValidator.validateBody(block, ...)
  │   ├── transactionsRoot == merkleRoot(block.transactions) ✓
  │   └── withdrawalsRoot == merkleRoot(block.withdrawals) ✓
  │
  └── BLOCK PROCESSING:
      BlockProcessor.processBlock(block, worldState, ...)   ← Step 14
```

### Step 14: Block Processing — Transaction Execution

```
BlockProcessor.processBlock(block, mutableWorldState, ...)
  │
  ├── coinbaseAccount = worldState.getOrCreate(block.getCoinbase())
  │
  ├── for each transaction in block.getBody().getTransactions():
  │   │
  │   ├── WorldUpdater txUpdater = mutableWorldState.updater()
  │   │
  │   ├── TransactionProcessor.processTransaction(blockchain, txUpdater, blockHeader, tx, ...)
  │   │   │
  │   │   ├── INTRINSIC GAS CHECK:
  │   │   │   intrinsicGas = 21000 (base) + 0 (empty data)
  │   │   │   tx.getGasLimit() >= intrinsicGas? → 21000 >= 21000 ✓
  │   │   │
  │   │   ├── UPFRONT COST DEDUCTION:
  │   │   │   upfrontCost = gasLimit * maxFeePerGas + value
  │   │   │   Alice.balance -= upfrontCost
  │   │   │
  │   │   ├── baseFee = block.getBaseFee()   (e.g., 1499999990 wei)
  │   │   │   effectivePriorityFee = min(maxPriorityFeePerGas, maxFeePerGas - baseFee)
  │   │   │                       = min(1500000000, 24) = 24 wei
  │   │   │   effectiveGasPrice = baseFee + effectivePriorityFee
  │   │   │
  │   │   ├── EVM EXECUTION:
  │   │   │   to = 0xBob (not a contract, just an address)
  │   │   │   tx.getData() = empty
  │   │   │   → This is a simple ETH transfer, NO EVM execution needed
  │   │   │   → Just: Bob.balance += value (0.1 ETH)
  │   │   │
  │   │   │   (If 'to' were a smart contract address with code:)
  │   │   │   (→ MessageCallProcessor.process(messageFrame, evm))
  │   │   │   (→ EVM.runToHalt(messageFrame))
  │   │   │   (→ execute opcodes until STOP/RETURN/REVERT)
  │   │   │
  │   │   ├── GAS REFUND:
  │   │   │   gasUsed = 21000 (intrinsic, no EVM opcodes for simple transfer)
  │   │   │   gasRefund = (gasLimit - gasUsed) * effectiveGasPrice
  │   │   │   Alice.balance += gasRefund
  │   │   │
  │   │   ├── COINBASE REWARD:
  │   │   │   priorityFeeTotal = gasUsed * effectivePriorityFee
  │   │   │   coinbase.balance += priorityFeeTotal
  │   │   │   (baseFee portion is BURNED — not given to anyone)
  │   │   │
  │   │   └── txUpdater.commit()    ← persist state changes for this tx
  │   │
  │   └── Create TransactionReceipt {
  │         status: 1 (success),
  │         cumulativeGasUsed: 21000,
  │         logs: [],
  │         bloomFilter: (empty for simple transfer)
  │       }
  │
  ├── Process withdrawals:
  │   for each withdrawal: validatorAccount.balance += withdrawalAmount
  │
  ├── Validate stateRoot:
  │   computedRoot = mutableWorldState.rootHash()
  │   computedRoot == block.getHeader().getStateRoot() ✓
  │   (if mismatch → INVALID block, reject)
  │
  └── return BlockProcessingResult { receipts: [...] }
```

### Step 15: Persist to Storage

```
DefaultBlockchain.appendBlock(block, receipts)
  │
  ├── Write to RocksDB BLOCKCHAIN column family:
  │   key: "header_" + blockHash → value: RLP(blockHeader)
  │   key: "body_"   + blockHash → value: RLP(blockBody)
  │   key: "receipt_"+ blockHash → value: RLP(receipts)
  │   key: "txlookup_"+ txHash   → value: {blockHash, txIndex}
  │
  ├── Update canonical chain head:
  │   key: "chainhead" → value: blockHash
  │
  └── BonsaiWorldStateProvider.persist(worldState, block)
      ├── Write TrieLog (state diff for this block):
      │   key: "trielog_" + blockHash → {accountChanges, storageChanges, codeChanges}
      │
      └── Update flat DB:
          key: Alice's address → {nonce: 43, balance: newBalance, ...}
          key: Bob's address   → {nonce: 0,  balance: +0.1ETH, ...}
```

### Step 16: Notify All Subscribers

```
BlockAddedEvent fired to all listeners:
  │
  ├── TransactionPool.onBlockAdded(block):
  │   → remove Alice's tx from pending transactions
  │   → it's now confirmed, no longer pending
  │
  ├── WebSocket subscription manager:
  │   → for each eth_subscribe("newHeads") subscriber:
  │     → push notification: { blockNumber: N, blockHash: "0x...", ... }
  │
  ├── eth_subscribe("logs") subscribers:
  │   → check if any block logs match subscriber's filter
  │   → push matching logs
  │
  ├── Metrics updated:
  │   besu_blockchain_height.set(blockNumber)
  │   besu_transaction_pool_transactions.dec()
  │   block_import_timer.record(importDuration)
  │
  └── EthProtocolManager:
      → announce new block to connected peers
      → send NewBlockHashes to all peers
```

### Step 17: Return to Consensus Client

```
engine_newPayloadV3 response:
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "status": "VALID",
    "latestValidHash": "0xBlockHash...",
    "validationError": null
  }
}
```

The consensus client now knows Besu successfully executed and stored the block.

### Step 18: Final Forkchoice Update

```
Teku → engine_forkchoiceUpdatedV3({
  headBlockHash: "0xThisNewBlock...",
  safeBlockHash: "0x...",
  finalizedBlockHash: "0x..."
})

Besu:
  → updates canonical chain head to this block
  → marks older finalized blocks as "finalized" (cannot reorg past them)
  → Bonsai: prunes trie logs older than finalized block (saves disk space)
```

**Alice's transaction is now FINALIZED on Ethereum Mainnet.**

---

## Section 11: Phase 10 — Graceful Shutdown

When you press `Ctrl+C` or the system sends `SIGTERM`:

### JVM Shutdown Hook Fires

```
Runner.close() called (registered via Runtime.getRuntime().addShutdownHook())
  │
  ├── 1. Stop accepting new RPC requests
  │      jsonRpcHttpService.stop()
  │      webSocketService.stop()
  │      engineRpcService.stop()
  │
  ├── 2. Stop block production
  │      miningCoordinator.stop()
  │
  ├── 3. Plugin stop lifecycle
  │      besuPluginContext.stopPlugins()
  │      → plugin.stop() for each loaded plugin
  │
  ├── 4. Stop synchronizer
  │      synchronizer.stop()
  │      → cancels all pending sync requests
  │
  ├── 5. Stop P2P network
  │      p2pNetwork.close()
  │      → disconnect all peers gracefully (send Disconnect message)
  │      → close UDP discovery socket
  │      → close all TCP RLPx connections
  │
  ├── 6. Stop metrics server
  │      metricsService.stop()
  │
  ├── 7. Close database
  │      storageProvider.close()
  │      → flush RocksDB Write-Ahead Log (WAL) to disk
  │      → close all column family handles
  │      → release file locks
  │
  └── 8. Vert.x shutdown
         vertx.close()
         → drain event loops, close all HTTP servers
```

If restarted, Besu will:
1. Detect existing database in the data directory
2. Read the last `chainhead` pointer from RocksDB
3. Resume sync from exactly where it left off

---

## Section 12: Key Design Patterns in the Code

Understanding these patterns helps you read Besu source faster:

### Builder Pattern

Almost every major object is constructed via a builder:

```java
// Instead of a massive constructor:
JsonRpcConfiguration config = JsonRpcConfiguration.createDefault();
config.setPort(8545);
config.setEnabled(true);

// Or:
Runner runner = new RunnerBuilder()
    .vertx(vertx)
    .besuController(besuController)
    .p2pEnabled(p2pEnabled)
    .jsonRpcConfiguration(jsonRpcConfig)
    // ... many more options
    .build();
```

Where to find: `RunnerBuilder.java`, `BesuControllerBuilder.java`, `JsonRpcConfiguration.java`

### Strategy Pattern (ProtocolSpec)

Different strategies (hard fork rule sets) are encapsulated and swapped at runtime:

```java
ProtocolSpec spec = protocolSchedule.getByBlockHeader(blockHeader);
// For block 12,964,999 → LondonProtocolSpec with London gas rules
// For block 12,965,000 → LondonProtocolSpec (same London rules still apply)
// For block 17,034,870 → MergeProtocolSpec with PoS rules

spec.getBlockImporter().importBlock(protocolContext, block, ...);
// → the right importer for this fork runs automatically
```

Where to find: `ProtocolSpec.java`, `MainnetProtocolSchedule.java`

### Observer / Event Pattern

```java
// Subscribe to events:
long listenerId = blockchainService.addBlockAddedListener(blockAddedEvent -> {
    long blockNumber = blockAddedEvent.getBlock().getHeader().getNumber();
    doSomethingWith(blockNumber);
});

// Unsubscribe:
blockchainService.removeBlockAddedListener(listenerId);
```

Where to find: `BesuEvents.java` (plugin-api), `BlockAddedObserver.java`

### Pipeline Pattern (Sync)

```java
// services/pipeline — like Java streams but async with back-pressure:
Pipeline<BlockHeader> pipeline = PipelineBuilder
    .createPipelineFrom("headers", headerSource, 10)
    .thenProcessAsync("downloadBodies", this::downloadBody, 4)
    .thenProcess("validate", this::validateBlock)
    .thenProcessAsync("import", this::importBlock, 1)
    .build();
```

Where to find: `services/pipeline/`, `PipelineChainDownloader.java`

### Layered WorldUpdater (Decorator Pattern)

```java
// Main block updater:
WorldUpdater blockUpdater = worldState.updater();

// Per-transaction updater stacked on top:
WorldUpdater txUpdater = blockUpdater.updater();
// → changes go to txUpdater first
// → txUpdater.commit() → pushed to blockUpdater
// → txUpdater.revert() → discarded, blockUpdater unchanged

// Nested call updater stacked on txUpdater:
WorldUpdater callUpdater = txUpdater.updater();
// REVERT in sub-call → callUpdater.revert() → txUpdater unaffected
```

Where to find: `AbstractWorldUpdater.java`, `StackedUpdater.java`

---

## Section 13: Testing Strategy

### Test Levels

```
besu/
 ├── src/test/java/           ← Unit tests (JUnit 5 + Mockito)
 │    └── *Test.java          ← fast, isolated, mock dependencies
 │
 ├── src/integration-test/java/ ← Integration tests
 │    └── *IT.java            ← spin up in-process components, real DB
 │
 ├── acceptance-tests/        ← Full acceptance tests
 │    └── tests/              ← spawn real Besu processes via TestContainers
 │         └── *AT.java       ← test full node behavior over real network
 │
 └── ethereum/referencetests/ ← EVM correctness
      └── *.json              ← Ethereum Foundation test vectors
```

### How to Run Tests

```bash
# Unit tests for one module:
./gradlew :ethereum:core:test

# All unit tests:
./gradlew test

# Integration tests:
./gradlew integrationTest

# Acceptance tests (requires Docker):
./gradlew acceptanceTest

# EVM reference tests:
./gradlew ethereum:referencetests:referenceTests

# Check code formatting:
./gradlew spotlessCheck

# Fix code formatting:
./gradlew spotlessApply
```

---

## Section 14: Where to Look for Each Feature

| What You Want to Understand | Module | Key File(s) |
|----------------------------|---------|----|
| Application entry point | `app` | `Besu.java` |
| CLI options | `app` | `BesuCommand.java` |
| Dagger DI setup | `app` | `BesuComponent.java`, `DaggerBesuComponent.java` |
| Genesis file loading | `config` | `GenesisConfigFile.java`, `GenesisState.java` |
| Hard fork schedule | `ethereum/core` | `MainnetProtocolSchedule.java` |
| Protocol spec per fork | `ethereum/core` | `MainnetProtocolSpecs.java` |
| Block import pipeline | `ethereum/core` | `BlockImporter.java`, `BlockProcessor.java` |
| Transaction execution | `ethereum/core` | `MainnetTransactionProcessor.java` |
| EVM main loop | `evm` | `EVM.java` |
| EVM execution context | `evm` | `MessageFrame.java` |
| A specific opcode | `evm` | `evm/operation/{Name}Operation.java` |
| Gas costs | `evm` | `CancunGasCalculator.java` (latest) |
| Precompiles | `evm` | `evm/precompile/` |
| World state (Bonsai) | `ethereum/core` | `BonsaiWorldStateProvider.java` |
| State trie | `ethereum/trie` | `StoredMerklePatriciaTrie.java` |
| Trie logs | `ethereum/core` | `TrieLogManager.java` |
| Transaction pool | `ethereum/eth` | `TransactionPool.java` |
| Transaction selection | `ethereum/core` | `BlockTransactionSelector.java` |
| Snap sync | `ethereum/eth` | `SnapSyncDownloader.java` |
| ETH wire protocol | `ethereum/eth` | `EthProtocolManager.java` |
| P2P peer discovery | `ethereum/p2p` | `DiscoveryAgent.java` |
| RLPx connections | `ethereum/p2p` | `RlpxAgent.java` |
| JSON-RPC server | `ethereum/api` | `JsonRpcHttpService.java` |
| A specific RPC method | `ethereum/api` | `jsonrpc/internal/methods/Eth*.java` |
| Engine API | `ethereum/api` | `EngineJsonRpcService.java`, `engine/` folder |
| QBFT consensus | `consensus/qbft` | `QbftController.java`, `QbftRound.java` |
| Post-Merge PoS | `consensus/merge` | `MergeCoordinator.java` |
| Plugin interfaces | `plugin-api` | `BesuPlugin.java`, `BesuContext.java` |
| Plugin lifecycle | `app` | `BesuPluginContextImpl.java` |
| RocksDB storage | `plugins/rocksdb` | `RocksDBKeyValueStorage.java` |
| RLP encoding/decoding | `ethereum/rlp` | `RLP.java`, `RLPInput.java`, `RLPOutput.java` |
| Cryptographic signing | `crypto/algorithms` | `SECP256K1.java`, `SignatureAlgorithm.java` |
| Metrics | `metrics/core` | `PrometheusMetricsSystem.java` |

---

## 🧠 Final Recap — The Complete Flow

```
1. ./bin/besu --network=mainnet ...
         │
         ▼
2. JVM starts → Besu.main() → setupLogging() → DaggerBesuComponent.create()
         │
         ▼
3. BesuCommand.parse() → PicoCLI populates @Option fields
         │
         ▼
4. BesuCommand.run() → validates options → loads genesis → detects consensus
         │
         ▼
5. Storage init → RocksDB opened → DefaultBlockchain → BonsaiWorldStateProvider
         │
         ▼
6. BesuControllerBuilder → ProtocolSchedule + ProtocolContext + TransactionPool
   + MergeCoordinator + plugins loaded
         │
         ▼
7. RunnerBuilder → P2PNetwork + EthProtocolManager + API servers assembled
         │
         ▼
8. Runner.start() → metrics → P2P → RPC servers → sync → mining coordinator
         │
         ▼
9. Node running:
   - Vert.x loops handle RPC requests
   - P2P loops discover + connect to peers
   - Sync downloads headers → bodies → world state
   - Engine API: consensus client drives block execution every 12s
         │
         ▼
10. eth_sendRawTransaction:
    RLP decode → validate → pool → gossip → return hash
         │
         ▼ (12 seconds later)
11. Block proposal → transaction selected → EVM executed → state committed
    → receipts written → subscribers notified → Engine API returns VALID
         │
         ▼
12. SIGTERM → shutdown hook → RPC stops → P2P disconnects → DB flushed → exit
```

---

## ➡️ What's Next?

You now understand Besu from business requirement all the way to the JVM shutdown hook. 

**Suggested next steps for contributing:**

1. **Clone the repo**: `git clone https://github.com/hyperledger/besu.git`
2. **Build it**: `./gradlew build`
3. **Run a dev node**: `./gradlew installDist && build/install/besu/bin/besu --network=dev --rpc-http-enabled`
4. **Find a good first issue**: https://github.com/hyperledger/besu/labels/good%20first%20issue
5. **Join Discord**: `#besu-contributors` channel
6. **Read CONTRIBUTING.md** in the repo root

👉 See **[20_how_to_contribute_to_besu.md](./20_how_to_contribute_to_besu.md)** for the full contributor guide.

---

*📌 Sources: Besu GitHub (github.com/hyperledger/besu), Besu Docs (besu.hyperledger.org), Besu Wiki (lf-hyperledger.atlassian.net/wiki/spaces/BESU)*