# A1 — What is the Business Requirement? Why Does Besu Exist?

> **Series:** Besu Deep-Dive — Document 1 of N
> **Next:** [A2 — How Does Besu Solve These Problems?](./A2_how_besu_solves_it.md)

---

## Table of Contents

1. [The Problem Space](#1-the-problem-space)
2. [The Dual Business Requirement](#2-the-dual-business-requirement)
   - [Customer Type 1 — Public Ethereum Participants](#customer-type-1--public-ethereum-participants)
   - [Customer Type 2 — Enterprise / Consortium Networks](#customer-type-2--enterprise--consortium-networks)
3. [Why a Java-Based Ethereum Client?](#3-why-a-java-based-ethereum-client)
4. [The Specific Product Requirements](#4-the-specific-product-requirements)
   - [Functional Requirements](#functional-requirements)
   - [Non-Functional Requirements](#non-functional-requirements)
5. [Besu vs Other Ethereum Clients](#5-besu-vs-other-ethereum-clients)
6. [Summary — The Business Value Proposition](#6-summary--the-business-value-proposition)

---

## 1. The Problem Space

### The World Needs Trustless Infrastructure

Modern digital economies run on trust — trust in banks to hold your money, trust in clearing houses to settle trades, trust in governments to maintain registries, trust in software vendors to run the systems you depend on. This trust is expensive to establish, expensive to maintain, and catastrophically costly when it breaks.

Blockchain technology emerged as a proposed solution: replace institutional trust with mathematical and cryptographic guarantees. Instead of trusting that a bank correctly debited and credited accounts, you trust that a deterministic program executed correctly on thousands of independent computers simultaneously and that the result is immutably recorded. No single party controls the ledger. No single party can reverse a transaction. No single party can secretly inflate supply or forge a record.

Ethereum extended the concept beyond simple value transfer. It introduced a **Turing-complete virtual machine** running on a decentralized network — the Ethereum Virtual Machine (EVM). This meant you could encode arbitrary business logic (smart contracts) into an unstoppable, globally-replicated computer. Supply chain provenance, decentralized finance, on-chain identity, tokenised real-world assets — all became theoretically possible without a central operator.

But "theoretically possible" is not the same as "practically deployable." The gap between the vision and the operational reality is enormous, and that gap is exactly the space Hyperledger Besu was built to fill.

---

### Traditional Enterprise Software Problems

Before examining what Besu solves, it is worth cataloguing what it is solving against. Enterprise software — particularly in regulated industries like banking, insurance, healthcare, and government — suffers from a specific and well-understood set of pathologies:

**Vendor lock-in.** When a bank deploys a core banking system from a major vendor, it becomes structurally dependent on that vendor for upgrades, patches, API changes, and pricing. Switching costs are enormous — data migration, staff retraining, integration work — so incumbents extract rent indefinitely. The bank cannot independently verify that the software is doing what it claims to do; it must trust the vendor's audit reports.

**Single point of failure.** A centralised database or application server is a single target for attack, outage, or misconfiguration. When a payment network's central switch goes down, millions of transactions fail simultaneously. Redundancy exists within a single operator's infrastructure, but not across operators — so systemic risk is never fully eliminated.

**High intermediary fees.** Every reconciliation step between two parties that do not share a database requires an intermediary — a clearing house, a custodian, a notary, a central securities depository. Each intermediary charges for the trust it provides. Cross-border payments, for instance, traverse multiple correspondent banking relationships, each adding latency and cost.

**No auditability or immutability.** Records in a centralised database can be altered after the fact. Audit trails exist but are controlled by the same party that controls the data — making them insufficient for regulatory purposes without expensive third-party auditing arrangements. Proving that a record has not been tampered with requires trusting the record-keeper.

**Proprietary standards and fragmentation.** Financial institutions use dozens of incompatible messaging formats, settlement systems, and data schemas. Integration between institutions is bespoke, expensive, and fragile.

---

### Why Enterprises Cannot Simply "Use Ethereum Mainnet" Directly

Given that Ethereum Mainnet is a live, functioning, decentralised global computer, a reasonable question is: why don't enterprises just use it? The answer is a combination of hard technical constraints, regulatory requirements, and economic realities.

**Privacy.** Ethereum Mainnet is a public ledger. Every transaction, every contract call, every storage value is visible to every node on the network and to anyone running a block explorer. A bank cannot broadcast that it is transferring $500 million between accounts. A hospital cannot record patient treatment events on a public chain. A government procurement system cannot expose tender bids to all competitors. For any commercially or legally sensitive data, a fully public ledger is simply not viable.

**Compliance and identity.** Regulated industries operate under strict know-your-customer (KYC) and anti-money-laundering (AML) requirements. Ethereum Mainnet pseudonymity — where addresses are not linked to legal identities at the protocol level — conflicts with requirements to verify counterparty identity before transacting. Enterprises need to be able to say: "Only pre-approved legal entities may join this network."

**Throughput and latency.** Ethereum Mainnet targets a 12-second slot time and processes blocks of ~30–60 million gas. This limits practical throughput to tens of transactions per second under typical workloads. A stock exchange processes hundreds of thousands of trades per second. A supply chain event system might need millisecond acknowledgements. Mainnet throughput is not sufficient for many enterprise workloads.

**Gas costs and economic model.** Every transaction on Mainnet requires ETH to pay for gas. The cost fluctuates with network congestion. For an enterprise consortium, this is operationally untenable — you cannot budget unpredictable gas costs into a business process, and you cannot require consortium members to acquire and hold ETH just to run internal workflows.

**Finality.** Ethereum Mainnet uses a probabilistic finality model under proof-of-work (now economic finality under proof-of-stake with Casper FFG). Blocks can technically be reorganised in the short term. Enterprise use cases — especially settlement systems — require deterministic, immediate finality: once a block is produced, it is final. No reorgs. No uncertainty.

**EIP compatibility and upgrade cadence.** Mainnet upgrades on a schedule driven by the Ethereum core developer community, with changes that are designed for the global public network. An enterprise consortium may need to stay on a specific version of the EVM for regulatory or audit reasons, or may need custom opcodes that would never be accepted into Mainnet.

---

### The Fundamental Gap

The situation creates a structural gap that neither existing technology category fills:

| Property | Public Blockchain (Mainnet) | Private/Permissioned Database | What Enterprises Need |
|---|---|---|---|
| Trustlessness | ✅ | ❌ | ✅ |
| Privacy | ❌ | ✅ | ✅ |
| Permissioned access | ❌ | ✅ | ✅ |
| Immediate finality | Partial | ✅ | ✅ |
| EVM smart contracts | ✅ | ❌ | ✅ |
| Open standards | ✅ | ❌ | ✅ |
| Auditability | ✅ | Depends | ✅ |
| No gas fees | ❌ | ✅ | ✅ (internal) |
| Known validators | ❌ | N/A | ✅ |

Public blockchains are too open. Private databases are not trustless. The enterprise world needs something that sits precisely in the middle: a **permissioned, EVM-compatible blockchain runtime** that delivers trustlessness within a defined membership boundary, combined with a fully capable **public Ethereum execution client** for organisations that also need to interact with Mainnet.

**Hyperledger Besu is the answer to that gap.**

---

## 2. The Dual Business Requirement

What makes Besu architecturally unique among Ethereum clients is that it explicitly serves **two distinct customer types** with quite different — and in some respects opposing — requirements. Most clients are built exclusively for one or the other. Besu is designed from the ground up to serve both simultaneously from a single codebase.

---

### Customer Type 1 — Public Ethereum Participants

This customer operates on Ethereum Mainnet (or public testnets: Sepolia, Holesky). They are:

- **Node operators** providing JSON-RPC infrastructure for wallets, dApps, and analytics services (e.g., infrastructure providers, RPC providers)
- **Validators** (post-Merge) who run Besu as the Execution Layer (EL) client alongside a Consensus Layer (CL) client (Teku, Lighthouse, Prysm, Nimbus, Lodestar) to propose and attest blocks and earn staking rewards
- **Solo stakers** running home nodes to contribute to Ethereum decentralisation
- **dApp backends** that need a trusted local node rather than dependence on a third-party RPC provider
- **MEV infrastructure** (builders, searchers, relays) requiring direct mempool access and block construction control
- **Archive node operators** serving historical state to developers and analytics platforms

**What these customers require from Besu:**

- **Mainnet compatibility:** Implement every Ethereum Improvement Proposal (EIP) that has been activated on Mainnet — from Frontier (2015) through London (EIP-1559), through the Merge (Paris), through Shanghai (withdrawals), Cancun (EIP-4844 blobs, EIP-4788, EIP-6780), and onward to Prague (EIP-7702, Verkle preparation).
- **Consensus client integration:** Post-Merge, the execution client no longer runs consensus. It must expose the **Engine API** — a separate, JWT-authenticated JSON-RPC endpoint — through which the CL client drives block production and fork-choice updates. Besu must implement the Engine API spec exactly and stay in lockstep with CL client teams.
- **JSON-RPC completeness:** Serve the full `eth_`, `net_`, `web3_`, `debug_`, `admin_`, `txpool_`, `trace_`, and `engine_` namespaces. Applications have hard dependencies on specific RPC methods; any gap breaks compatibility.
- **Snap Sync:** Download the Ethereum state as fast as possible when a new node joins the network. Snap Sync (the current state-of-the-art) downloads state leaves directly rather than reconstructing the trie top-down, dramatically reducing sync time. Besu must both perform Snap Sync itself and serve Snap Sync requests to other peers.
- **Checkpoint Sync:** Bootstrap a node by trusting a known checkpoint block (a Weak Subjectivity checkpoint from a trusted source), then Snap Sync from that point, skipping years of historical chain validation.
- **Low storage footprint:** Ethereum's state is hundreds of gigabytes and growing. Bonsai Tries (Besu's state storage format) dramatically reduces disk usage compared to traditional Forest of Tries, making node operation cheaper.
- **High performance:** Blocks must be imported and executed within the 12-second slot time with margin to spare. This means block execution, state updates, RPC serving, and P2P gossip must all be fast enough not to miss attestation deadlines.
- **MEV infrastructure support:** `eth_sendBundle`, support for MEV-Boost relay connections, block building configuration.

**What happens without a good execution client:**

Without a properly functioning execution client, a validator node cannot:
- Verify the validity of new blocks received from the network (the EL is responsible for EVM execution and state transition verification)
- Construct new execution payloads when selected as block proposer
- Serve transaction receipts, state queries, or historical data to applications
- Participate in the mempool (incoming transactions are submitted via EL's `eth_sendRawTransaction`)
- Provide MEV-related block building services

A validator missing these capabilities will miss attestations, miss block proposals, get slashed for equivocation if something goes wrong, and ultimately deliver negative returns to stakers.

---

### Customer Type 2 — Enterprise / Consortium Networks

This customer is running a **private, permissioned Ethereum network** — a network that uses the EVM and Ethereum data structures but does not connect to Mainnet. Examples include:

- **Banking consortia** settling interbank payments or securities transfers on a shared ledger (e.g., tokenised bond settlement between multiple banks and a central depository)
- **Supply chain networks** tracking provenance of goods across manufacturers, logistics providers, customs, and retailers — where each participant is a known legal entity
- **Healthcare consortia** sharing patient data consent records or clinical trial data across hospitals and pharmaceutical companies under strict HIPAA/GDPR controls
- **Government agencies** running inter-agency data sharing systems where auditability and non-repudiation are legally required
- **Capital markets infrastructure** building tokenised asset platforms (tokenised money market funds, repo on blockchain, etc.)
- **Central Bank Digital Currency (CBDC) projects** building regulated digital currency infrastructure

**What these customers require from Besu:**

- **Permissioned membership:** Only nodes whose `enode://` public key is on an approved list may connect to the network. Only accounts whose address is on an approved list (or has been approved by a smart contract) may submit transactions. This is non-negotiable for KYC/AML compliance — an unknown entity must not be able to participate.
- **Immediate, deterministic finality:** QBFT (Quorum Byzantine Fault Tolerant) consensus ensures that once a block is added to the chain, it is final. There are no forks, no reorgs, no probabilistic waiting periods. Settlement systems require this.
- **No mining, no gas fees (in the traditional sense):** The network is operated by known validators. There is no proof-of-work puzzle. Gas can be set to zero or managed as an internal unit of account rather than a market-driven fee.
- **Configurable genesis:** The chain ID, initial account balances, validator set, consensus parameters (block time, round change timeout, minimum validators), gas limit, and initial contract deployments are all specified in a genesis JSON file controlled by the consortium operators.
- **EVM compatibility:** Enterprises want to reuse Solidity tooling, Hardhat/Truffle deployment pipelines, OpenZeppelin libraries, MetaMask (for internal users), and the entire Ethereum developer ecosystem. A proprietary VM would require retraining developers and rebuilding tooling from scratch.
- **Plugin architecture:** Regulated industries need custom logic: specific transaction validation rules (e.g., "only allow transactions that reference a valid on-chain KYC credential"), custom block reward distribution, custom metrics exporters for regulatory reporting. Besu's plugin system allows these extensions without forking the core codebase.
- **Monitoring and operations:** Enterprise IT departments need operational visibility. Prometheus metrics, Grafana dashboards, log correlation with enterprise logging systems (Splunk, ELK stack) are not nice-to-haves; they are baseline requirements.
- **Support and governance:** Enterprises need a project with clear governance, commercial support pathways, and a license that is safe to use in proprietary products. Hyperledger (a Linux Foundation project) provides exactly this governance structure, and Apache 2.0 eliminates license risk.

**What happens without Besu (or a similar solution):**

Without a suitable permissioned EVM client, enterprises face two bad options:

*Option A — Use Ethereum Mainnet:* Expose all transaction data publicly, deal with unpredictable gas costs, be unable to control network membership, accept probabilistic finality, and be subject to protocol upgrades driven by a global community with different priorities.

*Option B — Use a proprietary database:* Get privacy and control, but lose trustlessness (the operator of the database can alter records), lose the EVM developer ecosystem (must build custom tooling), and lose interoperability with the broader blockchain ecosystem (cannot connect to DeFi, cannot bridge to Mainnet, cannot use standard wallets and tools).

Besu uniquely provides a third option: **a permissioned, enterprise-governed EVM network with trustlessness guarantees within the consortium boundary**.

---

## 3. Why a Java-Based Ethereum Client?

At the time of Besu's creation (2018–2019, initially called "Pantheon" by PegaSys/ConsenSys before donation to Hyperledger), the existing execution clients were written in Go (go-ethereum / Geth), C++ (cpp-ethereum, later Aleth), Rust (early stage), and Python (pyethereum, reference only). The decision to build in Java was deliberate and strategic.

### The Java Ecosystem is Mature and Enterprise-Friendly

Java has been the dominant language in enterprise software for over two decades. The JVM ecosystem has:

- Decades of production-hardened libraries for cryptography (Bouncy Castle), networking (Netty, Vert.x), serialisation (Jackson, Protocol Buffers), logging (SLF4J, Log4j2), metrics (Micrometer, Prometheus), and dependency injection (Dagger, Guice, Spring).
- A vast pool of developers already employed in enterprise IT departments — particularly in banking and financial services. Choosing Java means organisations can contribute to and maintain their deployments without hiring specialised systems programmers.
- Excellent static analysis tooling (SpotBugs, Checkstyle, SonarQube), IDEs (IntelliJ IDEA, Eclipse), and build systems (Gradle, Maven) with deep enterprise CI/CD integration.
- Strong typing that catches a large class of errors at compile time — critical for financial infrastructure where runtime bugs can have real monetary consequences.

### The JVM is Battle-Tested for High-Throughput Services

Banks, trading platforms, and payment systems have run on the JVM for decades at extraordinary scale. The JVM's JIT compiler, mature garbage collectors (G1GC, ZGC, Shenandoah), and profiling tooling (JFR, async-profiler, VisualVM) are well-understood by enterprise operations teams. Running Besu is operationally similar to running any other Java-based service — familiar deployment patterns, familiar monitoring approaches, familiar tuning knobs.

The JVM also provides genuine platform portability: run the same JAR on x86_64 Linux for production, on Apple Silicon for developer laptops, and on Windows for enterprise environments that mandate it — without recompilation.

### Apache 2.0 License — Enterprise-Safe

The Apache 2.0 license is specifically chosen to be safe for enterprise use:

- **No copyleft:** Unlike the GPL (used by go-ethereum), Apache 2.0 does not require derivative works to be open-sourced. An enterprise can build proprietary products on top of Besu, write proprietary plugins, and distribute them without being obligated to release source code.
- **Patent grant:** Apache 2.0 includes an explicit patent licence from contributors, reducing legal risk for enterprise adopters.
- **Hyperledger governance:** The Linux Foundation's Hyperledger project provides neutral governance — no single company controls Besu. This is critical for consortium deployments where members would be reluctant to depend on infrastructure controlled by a competitor.

### Plugin System Enabled by Java's Architecture

Java's `ServiceLoader` mechanism provides a standard, zero-dependency way to discover and load implementations of interfaces at runtime from JARs on the classpath. Combined with Dagger for compile-time dependency injection within the core, this gives Besu a clean plugin architecture: the core defines interfaces (`BesuPlugin`, service interfaces), the runtime discovers implementing JARs, and plugins receive typed access to internal services via a context object. This pattern is natural in Java and would be significantly more complex to achieve cleanly in Go or Rust.

---

## 4. The Specific Product Requirements

### Functional Requirements

The following functional requirements define what Besu must be able to **do** — observable capabilities that users and operators depend on.

**FR-1: Execute Ethereum Transactions and Update World State**

Besu must implement the Ethereum Virtual Machine completely and correctly. For every hard fork from Frontier through the current head of Mainnet, it must execute smart contract bytecode using the correct rules — correct gas costs, correct opcode semantics, correct precompile implementations, correct state transition logic. A single incorrect EVM execution produces an incorrect state root, which causes the node to be rejected by peers.

This includes: executing contract creation transactions, executing message call transactions, computing the correct receipt (logs, gas used, status), applying the correct state changes (account balances, nonce increments, contract storage), and computing the correct post-execution state root.

**FR-2: Store Blockchain Data Persistently**

Besu must persist:
- Block headers and bodies (transactions + ommers/uncles)
- Transaction receipts
- The Ethereum world state (all account balances, nonces, code, and storage)
- Trie nodes (for state, storage, receipt, and transaction tries)

This data must survive process restarts and hardware failures (to the extent the underlying storage engine allows). The storage layer must be fast enough to keep up with block production rates and serve RPC queries with low latency.

**FR-3: Sync with the Network**

A newly started Besu node with no chain data must be able to download and validate the canonical chain from the network. Besu must support multiple sync strategies:
- **Full sync:** Download and execute every block from genesis (slowest, maximum security)
- **Snap sync:** Download a recent snapshot of world state leaves directly from peers, reconstruct the trie, then switch to following the head (fast and the recommended default)
- **Checkpoint sync:** Trust a provided checkpoint block hash and number, Snap sync from that point, skip full history validation

**FR-4: Serve JSON-RPC APIs**

Besu must expose a standards-compliant JSON-RPC server accessible over HTTP and WebSocket. The required API namespaces include:
- `eth_` — core Ethereum APIs (getBlock, sendRawTransaction, call, estimateGas, getLogs, getBalance, etc.)
- `net_` — network info (version, peerCount, listening)
- `web3_` — client version, sha3
- `admin_` — peer management, node info
- `txpool_` — mempool inspection (content, status, inspect)
- `debug_` — low-level debugging (traceTransaction, dumpBlock, getBadBlocks, storageRangeAt)
- `trace_` — Parity-compatible tracing API
- `engine_` — Engine API for consensus client communication
- `miner_` — mining control (legacy, used for block production config on private networks)
- `ibft_` / `qbft_` — consensus management APIs for private networks
- `perm_` — permissioning APIs for private networks
- `plugins_` — plugin management

**FR-5: Communicate with a Consensus Client via Engine API**

Post-Merge, Besu does not run consensus internally on Mainnet. The Consensus Layer client drives block production and fork-choice. Besu exposes the Engine API (defined by the `execution-apis` specification) on a separate authenticated HTTP endpoint. The CL calls:
- `engine_newPayloadVX` — deliver a new execution payload (block) for validation
- `engine_forkchoiceUpdatedVX` — update the fork choice state and optionally start payload building
- `engine_getPayloadVX` — retrieve a built execution payload for the CL to propose
- `engine_getPayloadBodiesByHashVX` / `engine_getPayloadBodiesByRangeVX` — retrieve payload bodies for the CL's history sync

Besu must implement all versions of these APIs as they evolve with each hard fork.

**FR-6: Participate in P2P Network**

Besu must implement Ethereum's P2P networking stack:
- **Node discovery (Discv4/Discv5):** UDP-based Kademlia distributed hash table for finding peers
- **RLPx:** TCP-based encrypted and authenticated transport for peer connections
- **ETH wire protocol:** For gossiping new blocks and transactions, serving block headers/bodies, serving receipts, serving node data
- **SNAP protocol:** For serving snap sync requests (account ranges, storage ranges, bytecodes, trie nodes)

**FR-7: Support Multiple Consensus Protocols**

- On public Ethereum (post-Merge): Proof-of-Stake, driven externally by the CL
- On private networks: QBFT (primary, recommended), IBFT 2.0 (legacy), Clique (PoA, simple, for development/testnets)

**FR-8: Support Permissioning**

For enterprise/private networks:
- **Node permissioning:** Control which nodes (by enode public key) are allowed to connect as peers
- **Account permissioning:** Control which accounts are allowed to submit transactions (and optionally which contracts they can interact with)
- Both local (flat file / in-memory list) and smart contract-based permissioning (on-chain governance of the permission lists)

**FR-9: Be Extensible via Plugins**

A plugin JAR placed in the `plugins/` directory must be discovered and loaded at startup. Plugins must be able to:
- Register custom JSON-RPC methods
- Intercept and filter transactions before they enter the transaction pool
- Intercept block proposal events and influence transaction selection
- Subscribe to block and transaction events
- Provide custom storage backends
- Access and query the blockchain and world state

**FR-10: Expose Operational Monitoring**

- Prometheus metrics endpoint (scrape-able by a Prometheus server)
- Structured logging with configurable log levels and formats
- Health check endpoints (for load balancer integration, Kubernetes liveness/readiness probes)
- JVM metrics exposed alongside application metrics

---

### Non-Functional Requirements

These requirements define the *quality attributes* of Besu's implementation — the "-ilities."

**NFR-1: Performance**

Block import and execution must complete comfortably within the 12-second Ethereum Mainnet slot time, even for maximum-size blocks (currently ~60 million gas). The node must not fall behind the head of the chain under normal operating conditions on commodity hardware (8-core CPU, 32 GB RAM, 2 TB NVMe SSD).

Target metrics (indicative, hardware-dependent):
- Full block import (including EVM execution, state update, trie hashing): < 2 seconds for a 60M gas block on target hardware
- JSON-RPC response latency: < 100ms for simple queries (eth_getBalance, eth_getBlockByNumber)
- Snap sync completion time: < 12 hours for a fresh node joining Mainnet
- Transaction pool throughput: handle thousands of pending transactions without significant latency degradation

**NFR-2: Storage Efficiency**

An Ethereum full node (no archive) should fit within 1 TB of disk using Bonsai Tries state storage. Archive nodes (all historical state) are naturally larger but should be clearly documented. The system must not have storage leaks — long-running nodes should not grow disk usage indefinitely beyond the natural growth of chain state.

**NFR-3: Security — No Key Management Inside the Client**

Besu should not be the place where private keys are managed for validators or signers. Key management is delegated to external tools (Web3Signer). This separates the attack surface: a compromised Besu process cannot leak signing keys. Besu integrates with Web3Signer via an HTTP API.

The P2P communication must use properly authenticated and encrypted connections (RLPx). The Engine API endpoint must require JWT authentication. The admin JSON-RPC APIs must not be exposed on public-facing interfaces.

**NFR-4: Interoperability**

- Implement the Enterprise Ethereum Alliance (EEA) specification for private transaction formats and consensus interfaces (for private network customers)
- Implement all standard Ethereum JSON-RPC methods exactly as specified so that any standard Ethereum tool (MetaMask, Hardhat, Ethers.js, Web3.js, Foundry, etc.) works against Besu without modification
- Support the standard Ethereum genesis file format for easy migration between clients

**NFR-5: Modularity and Testability**

Each major subsystem (EVM, storage, networking, RPC, consensus, transaction pool) must be independently testable without requiring the entire node to be running. The EVM module (`ethereum/evm/`) must be extractable as a standalone library usable in testing frameworks. The storage layer must support in-memory backends for fast unit tests. The network layer must be mockable for consensus and sync testing.

**NFR-6: Reliability and Recovery**

Besu must handle:
- Process restart at any point during sync (restart should resume, not restart from scratch)
- Chain reorganisations (revert to the common ancestor, then replay the new canonical chain)
- Peer misbehaviour (malformed messages, sending invalid blocks) without crashing
- Disk full conditions with graceful error reporting rather than data corruption
- Network partitions with correct behaviour upon reconnection (follow the longest chain / highest total difficulty / correct fork choice)

---

## 5. Besu vs Other Ethereum Clients

The Ethereum Mainnet execution layer has a diversity of clients. This is intentional — client diversity ensures that a bug in any single client does not take down the entire network. Besu occupies a distinct position in this ecosystem.

| Property | **Besu** | **Geth** | **Nethermind** | **Erigon** |
|---|---|---|---|---|
| **Language** | Java | Go | C# / .NET | Go (fork of Geth, rewritten) |
| **License** | Apache 2.0 | LGPL-3.0 | LGPL-3.0 | LGPL-3.0 |
| **Primary Maintainer** | Hyperledger (Linux Foundation) / ConsenSys, various contributors | Ethereum Foundation | Nethermind team | Erigon team (formerly Turbo-Geth) |
| **Key Differentiator** | Enterprise features, plugin system, dual public/private support | Reference client, largest ecosystem support, most widely deployed | .NET ecosystem, Windows-friendly, excellent tracing APIs | Lowest disk usage on archive nodes, staged sync optimisation |
| **Enterprise / Private Network Support** | ✅ First-class — QBFT, IBFT2, Clique, permissioning, plugin API | ⚠️ Limited (Clique only, no permissioning) | ⚠️ Limited | ❌ Minimal |
| **Plugin / Extension System** | ✅ Full Java ServiceLoader plugin API | ❌ None | ❌ None | ❌ None |
| **Permissioning** | ✅ Node + account level, smart contract or local | ❌ | ❌ | ❌ |
| **QBFT / IBFT2 Consensus** | ✅ | ❌ | ✅ (Nethermind supports IBFT/QBFT for private nets) | ❌ |
| **Snap Sync** | ✅ (client + server) | ✅ (origin of Snap protocol) | ✅ | ✅ |
| **Bonsai State Storage** | ✅ (default, minimal footprint) | Uses LevelDB/Pebble with MPT | Uses its own state DB | Uses its own flat state DB |
| **Archive Node** | ✅ (Forest of Tries mode) | ✅ | ✅ | ✅ (most efficient) |
| **GraphQL API** | ✅ | ✅ | ✅ | ❌ |
| **EEA Compliance** | ✅ | ❌ | ❌ | ❌ |
| **Governance** | Hyperledger / Linux Foundation | Ethereum Foundation | Private company | Private company |
| **License Risk for Proprietary Use** | ✅ None (Apache 2.0) | ⚠️ LGPL (linking restrictions) | ⚠️ LGPL | ⚠️ LGPL |

**Key takeaway:** Geth is the reference client and the most widely deployed for public Mainnet use. Erigon leads on archive node storage efficiency. Nethermind offers good .NET ecosystem integration. **Besu is the only client that comprehensively addresses enterprise requirements** — permissioning, enterprise consensus (QBFT), plugin extensibility, Apache 2.0 licensing, and Hyperledger governance — while simultaneously maintaining full Mainnet compatibility.

---

## 6. Summary — The Business Value Proposition

Hyperledger Besu exists because no other tool fills the intersection of three requirements simultaneously:

1. **Full Ethereum Mainnet compatibility** — every EIP, every consensus upgrade, Engine API, Snap Sync, all JSON-RPC
2. **Enterprise-grade private network capabilities** — QBFT finality, permissioning, plugin extensibility, Apache 2.0 license, neutral governance
3. **Java ecosystem integration** — enterprise-friendly runtime, mature tooling, massive existing developer base

This positions Besu as the **"one client, two worlds"** solution:

- A bank can use Besu to run its private interbank settlement network (Customer Type 2), and simultaneously use Besu to maintain a Mainnet-connected node for interacting with public DeFi protocols (Customer Type 1) — from the same codebase, with the same operational model.

- An infrastructure provider running Mainnet validators can use Besu as their execution client, while the same software is deployed by one of their enterprise clients in a private QBFT network — the operators have the same monitoring, configuration, and operational skills for both deployments.

### Enterprise Ethereum Alliance (EEA) Compliance

Besu is the reference implementation of the **Enterprise Ethereum Alliance (EEA) specification** — the industry standard that defines how Ethereum-compatible enterprise networks should behave. This gives Besu formal standing as *the* enterprise Ethereum client, and gives EEA member companies confidence that building on Besu means building on a standards-compliant foundation.

### Hyperledger Graduation

Besu is the only **graduated** Hyperledger project specifically focused on Ethereum compatibility. Hyperledger graduation means:
- The project has demonstrated production readiness
- Governance, security processes, and contribution guidelines meet Linux Foundation standards
- Long-term support commitments are in place
- The project is recommended for enterprise adoption by the Hyperledger Governing Board

### The Bottom Line

Without Besu:
- Enterprises building permissioned blockchain systems must choose between inadequate public chains and proprietary non-trustless databases
- Java/JVM shops participating in Ethereum have no native-language execution client that matches their operational norms
- The Ethereum ecosystem loses a critical source of client diversity on Mainnet

With Besu:
- Enterprises get a production-grade, standards-compliant, extensible EVM runtime they can deploy and maintain with existing Java skills
- Mainnet participants get a fully featured, high-performance execution client with excellent operational tooling
- The Ethereum network gets another robust, independently implemented client, improving resilience

---

## Next Document

Continue to **[A2 — How Does Besu Solve These Problems?](./A2_how_besu_solves_it.md)** for a deep dive into the architectural decisions and specific design patterns Besu uses to fulfil each of these requirements — from EVM execution internals to the plugin API's lifecycle management.

---

*Document: A1 | Series: Besu Deep-Dive | Audience: Engineers, Architects, Enterprise Evaluators*