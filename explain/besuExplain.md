🔷 Hyperledger Besu — Complete Overview

## What Is Besu?

**Hyperledger Besu** is an open-source, enterprise-grade **Ethereum client** written in **Java**, licensed under **Apache 2.0**. It is a **graduated** Hyperledger project and is one of the most feature-complete Ethereum clients available. It supports both **public Ethereum networks** and **private permissioned networks**, making it uniquely positioned for enterprise use cases.

- **GitHub**: https://github.com/hyperledger/besu
- **Docs**: https://besu.hyperledger.org
- **Community**: Discord (`#besu` and `#besu-contributors` channels)

---

## 🌐 Public Networks

Besu acts as an **execution client** on public Ethereum networks:
- **Ethereum Mainnet**
- **Hoodi**, **Sepolia**, **Ephemery** (testnets)
- **Linea** (Ethereum L2)

### Execution vs. Consensus Client

Since the 2022 **Merge**, Ethereum moved from Proof of Work (PoW) to **Proof of Stake (PoS)**. This split the client role into two:

| Layer | Client Type | Example |
|---|---|---|
| Execution Layer | Execution Client | **Besu** |
| Consensus Layer | Consensus Client (Beacon Node) | **Teku**, Lighthouse, etc. |

Besu handles: executing transactions, managing the world state, serving JSON-RPC API requests, and P2P communication. It uses the **Engine API** to communicate with the consensus client.

A **validator node** also runs a **validator client**, which handles attestations and block proposals on the consensus layer.

---

## 🔐 Private (Permissioned) Networks

Besu is equally capable of running private enterprise networks that are not connected to Ethereum Mainnet. These use:
- Custom **Chain IDs**
- **Proof of Authority (PoA)** consensus

---

## ⚙️ Core Features

### 1. The Ethereum Virtual Machine (EVM)
A Turing-complete VM that executes smart contracts via transactions. Besu's EVM can also be used as a **standalone library** (extractable EVM).

---

### 2. Consensus Algorithms

#### For Public Networks:
- **Proof of Stake (PoS)** — Used on Ethereum Mainnet post-Merge, in conjunction with a consensus client.

#### For Private Networks (Proof of Authority — PoA):
| Protocol | Finality | Min Validators | Fault Tolerance | Notes |
|---|---|---|---|---|
| **QBFT** | ✅ Immediate | 4 | ≥ ⅔ must be up | Recommended for production |
| **IBFT 2.0** | ✅ Immediate | 4 | ≥ ⅔ must be up | No forks, stable |
| **Clique** | ❌ No (forks possible) | 1 | Up to ½ can fail | **Deprecated** (block production from 25.12.0) |
| **PoW (Ethash)** | ❌ | N/A | N/A | **Deprecated**, used for Ethereum Classic |

**QBFT** is the recommended enterprise-grade protocol. Validators take turns producing blocks; a supermajority (≥ ⅔) must sign each block.

---

### 3. Data Storage Formats

Besu uses **RocksDB** as its underlying key-value store, with two world state storage formats:

| Feature | **Bonsai Tries** (default) | **Forest of Tries** |
|---|---|---|
| Access method | Direct via account key | Traverses branches by hash |
| Full node storage | ~650 GB | ~750 GB |
| Archive node | ❌ Not supported | ✅ ~12 TB |
| Pruning | Implicit (built-in) | Manual (deprecated `--pruning-enabled`) |
| Read performance | Faster (recent data) | Slower |

Blockchain data is further split into:
- **Block headers** — form the cryptographic chain
- **Block bodies** — ordered transaction lists
- **Transaction receipts** — execution metadata and logs
- **World State** — address-to-account mapping (balances, code, storage)

---

### 4. Node Synchronization Modes

| Mode | Description | Status | Speed | Disk |
|---|---|---|---|---|
| **Snap** (default) | Downloads trie leaves, reconstructs state locally | ✅ Recommended | ~3 hrs world state + ~13 hrs blockchain | Smallest |
| **Checkpoint** | Snap from a specific block in genesis | ⚠️ Will be deprecated | ~3 hrs + ~13 hrs | Smallest |
| **Fast** | Downloads headers + receipts, verifies from genesis | ❌ Deprecated (24.12.0+) | ~1.5 days | Average |
| **Full** | Reprocesses all transactions from genesis | ✅ For archive nodes | Weeks | Largest |

> Recommended: **Snap sync + Bonsai Tries** — fastest sync and lowest storage.

#### Node Types:
- **Full Node** — stores current + some historical state. Suitable for most use cases.
- **Archive Node** — stores every historical state since genesis. Must use Forest + Full sync.

---

### 5. Transaction Types

| Type | EIP | Key Parameters |
|---|---|---|
| `FRONTIER` | Legacy | `gasPrice`, `chainId`, `nonce`, etc. |
| `ACCESS_LIST` | EIP-2930 | Legacy + `accessList` |
| `EIP1559` | EIP-1559 | `maxFeePerGas`, `maxPriorityFeePerGas` (no `gasPrice`) |
| `BLOB` | EIP-4844 | Blob commitment (data off-chain), enables L2 scaling |

- **EIP-1559** uses a dynamic **base fee** (burned) + **priority fee** (paid to validator).
- **BLOB** transactions store large data commitments for rollups/L2s — temporarily held by consensus clients like Teku.

---

### 6. APIs

Besu exposes multiple APIs:
- **JSON-RPC over HTTP** — standard `eth_`, `net_`, `web3_`, `debug_`, `miner_` methods
- **JSON-RPC over WebSocket** — same methods with WS support
- **GraphQL** — query-based API
- **Engine API** — for consensus client communication
- **RPC Pub/Sub** — event subscriptions over WebSocket

> Besu does **not** manage keys internally. Use **Web3Signer** for key management and transaction signing.

---

### 7. P2P Networking

Besu implements Ethereum's **devp2p** protocols:
- **Discovery (UDP)** — finds peers on the network
- **RLPx (TCP)** — peer-to-peer communication via sub-protocols:
  - **ETH sub-protocol (Ethereum Wire Protocol)** — syncs blockchain state, propagates transactions
  - **IBF sub-protocol** — facilitates IBFT2 consensus decisions

---

### 8. Permissioning (Private Networks)

Besu supports both **node** and **account** permissioning:

- **Node permissioning** — restricts which nodes can join the network
- **Account permissioning** — controls which accounts can transact, enforces identity/onboarding, can suspend accounts

**Local permissioning** operates at the individual node level via a config file — it protects your node but doesn't enforce rules network-wide.

> Privacy (private transactions) and network-level permissioning are **deprecated** features.

---

### 9. Monitoring

- **Node performance**: Prometheus metrics + `debug_metrics` JSON-RPC + **Grafana** dashboards
- **Network performance**: Grafana + **Quorum Explorer** (via Developer Quickstart)

---

### 10. Plugin System

Besu has a **modular plugin framework** (Java-based). Plugins can:
- Extend monitoring, add event streaming, integrate with third-party systems
- Access data about blocks, balances, transactions, smart contracts, logs, syncing state
- Be installed by placing `.jar` files in the `plugins/` directory

Plugin lifecycle: **Register → Start → Stop**. Can be dynamically reloaded via `plugins_reloadPluginConfig` JSON-RPC method.

---

## 🛠️ Getting Started

| Goal | Path |
|---|---|
| Connect to Ethereum Mainnet/testnet | Install Besu + connect a consensus client (e.g., Teku) |
| Private enterprise network | Use the **Developer Quickstart** or create a QBFT/IBFT 2.0 network |
| Smart contract dev | Works with **Hardhat**, **Remix**, **web3j** |
| Kubernetes deployment | Official Kubernetes tutorials available |
| Azure deployment | IBFT 2.0 network on Microsoft Azure tutorial available |

---

## 📌 Key Takeaways

1. **Besu is Java-based**, enterprise-friendly, and Apache 2.0 licensed.
2. It's dual-purpose: public Ethereum **execution client** AND private PoA network node.
3. **QBFT** is the gold standard for private enterprise consensus.
4. **Snap sync + Bonsai Tries** is the best combo for public network nodes.
5. The **Plugin API** makes it highly extensible without forking the codebase.
6. Key management is **external** (use Web3Signer).
7. Privacy and PoW/Clique features are **deprecated** — focus is on PoS (public) and QBFT (private).