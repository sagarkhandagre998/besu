# 05 — Ethereum: The Programmable Blockchain

> **Level:** Intermediate | **Read time:** ~30 minutes
> **Prerequisites:** [04 — Consensus Mechanisms](04_consensus_mechanisms.md)

---

## Table of Contents

1. [A Brief History of Ethereum](#1-a-brief-history-of-ethereum)
2. [Bitcoin vs Ethereum](#2-bitcoin-vs-ethereum)
3. [The Key Innovation: Smart Contracts](#3-the-key-innovation-smart-contracts)
4. [Decentralized Applications (dApps)](#4-decentralized-applications-dapps)
5. [Ethereum's Architecture: Two Layers](#5-ethereums-architecture-two-layers)
6. [The EVM: Ethereum Virtual Machine](#6-the-evm-ethereum-virtual-machine)
7. [Ether (ETH): The Native Currency](#7-ether-eth-the-native-currency)
8. [Ethereum Mainnet and Testnets](#8-ethereum-mainnet-and-testnets)
9. [Key Ethereum Improvement Proposals (EIPs)](#9-key-ethereum-improvement-proposals-eips)
10. [Ethereum's Roadmap](#10-ethereums-roadmap)
11. [Layer 2 Solutions](#11-layer-2-solutions)
12. [Recap Checklist](#12-recap-checklist)
13. [Check Your Understanding](#13-check-your-understanding)

---

## 1. A Brief History of Ethereum

### The Vision

In late **2013**, a 19-year-old programmer named **Vitalik Buterin** published a whitepaper proposing a new kind of blockchain. He had been involved in the Bitcoin community and noticed a fundamental limitation: Bitcoin's scripting language was deliberately constrained. It was designed to do one thing well — transfer value — but it was not a general-purpose programming environment.

Vitalik's insight was simple and radical:

> *What if the blockchain were a general-purpose computer that anyone could program?*

He called this idea **Ethereum** — a decentralized platform for running arbitrary programs, where no single party controls the execution and the results are guaranteed by mathematics.

### Key Milestones

| Year | Event |
|---|---|
| **2013** | Vitalik Buterin publishes the Ethereum whitepaper |
| **2014** | Ethereum Foundation formed; public crowdsale raises ~$18M in Bitcoin |
| **2015** | **Frontier** — Ethereum mainnet launches on July 30, 2015 |
| **2016** | **The DAO hack** — 3.6M ETH drained; controversial hard fork splits chain into ETH and ETC |
| **2017** | **Metropolis** upgrades (Byzantium, Constantinople); ICO boom drives adoption |
| **2020** | **Beacon Chain** launches — the foundation for Proof of Stake goes live |
| **2021** | **London** hard fork — EIP-1559 overhauls the fee market |
| **2022** | **The Merge** — Ethereum transitions from PoW to PoS on September 15, 2022 |
| **2023** | **Shanghai/Capella** — staked ETH withdrawals enabled |
| **2024** | **Dencun** — EIP-4844 blob transactions reduce L2 costs by ~90% |

### The "World Computer" Vision

Ethereum was originally described as a **"world computer"** — a single, global, shared virtual machine that anyone can write programs for, that runs exactly as programmed, that cannot be shut down or censored, and whose state is transparent and auditable by anyone.

Whether or not that exact vision has been fully realized is debated, but Ethereum has undeniably become the foundation for a vast ecosystem of financial applications, digital ownership, and decentralized governance.

---

## 2. Bitcoin vs Ethereum

Both Bitcoin and Ethereum are public blockchains, but they were designed with fundamentally different goals.

| Property | Bitcoin | Ethereum |
|---|---|---|
| **Launched** | January 2009 | July 2015 |
| **Primary purpose** | Store of value / digital gold / peer-to-peer payments | Programmable platform for decentralized applications |
| **Scripting** | Bitcoin Script — intentionally limited, not Turing-complete | Solidity / Vyper / others — Turing-complete (with gas limit) |
| **Smart contracts** | Very limited (basic multisig, timelocks) | Full-featured, general-purpose |
| **Block time** | ~10 minutes | 12 seconds (post-Merge, fixed slot time) |
| **Consensus (current)** | Proof of Work (SHA-256) | Proof of Stake (beacon chain) |
| **Native token** | BTC | ETH |
| **Token supply** | Capped at 21 million BTC | No hard cap; issuance reduced post-Merge; net deflationary at high usage |
| **Account model** | UTXO (Unspent Transaction Output) | Account-based (address → balance, nonce) |
| **State** | Only UTXO set | Rich world state: accounts, contract storage, code |
| **Virtual machine** | None (no general execution) | EVM (Ethereum Virtual Machine) |
| **Programmability** | Minimal | Core design goal |

### Why the Account Model Matters

Bitcoin's UTXO model tracks individual "coins" (unspent outputs). Ethereum uses an **account model** where each address has a running balance and nonce — much closer to how a traditional bank account works. This makes it far easier to write complex programs (smart contracts) that need to track state over time.

---

## 3. The Key Innovation: Smart Contracts

### What Is a Smart Contract?

A **smart contract** is a program stored on the blockchain. It has its own Ethereum address, its own ETH balance, and its own persistent storage. When you send a transaction to a contract's address, the contract's code executes on every node in the network.

The name "smart contract" can be misleading — they are not legally binding contracts (though they can represent one), and they are not necessarily "smart." They are simply **self-executing code**.

Key properties:

- **Immutable once deployed:** The code cannot be changed after it is deployed (without using special proxy upgrade patterns).
- **Deterministic:** Given the same inputs and state, the contract always produces the same outputs — on every node, everywhere in the world.
- **Trustless:** You do not need to trust the author of the contract. You can read and verify the code yourself (if the source is published). The outcome is guaranteed by the code, not by any person.
- **Permissionless:** Anyone with ETH to pay gas fees can deploy a smart contract. No approval needed.
- **Transparent:** All contract code and state is publicly readable on-chain (or at minimum verifiable via the state root).

### A Simple Analogy

Think of a traditional vending machine:

- You insert money.
- You select an item.
- The machine automatically dispenses the item and returns change.
- No cashier involved. No need to trust anyone. The machine executes the rules mechanically.

A smart contract is a vending machine — but for any kind of agreement: "if Alice deposits 100 ETH by Friday and Bob deposits 100 ETH by Friday, release the combined funds to the winner of this on-chain vote."

### What Makes Them Revolutionary

Before smart contracts, executing an agreement between untrusting parties required a **trusted intermediary** — a bank, a lawyer, an escrow agent. These intermediaries charge fees, can be corrupted, can make errors, and can be unavailable.

Smart contracts remove the intermediary. The code *is* the intermediary, and it cannot be bribed, cannot make arbitrary decisions, and cannot selectively enforce the rules.

---

## 4. Decentralized Applications (dApps)

### What Is a dApp?

A **decentralized application (dApp)** is an application whose backend logic runs on a blockchain (via smart contracts) rather than on centralized servers controlled by a company.

Typical structure:

```
┌──────────────────────────────────────────────────────────────┐
│                    Traditional App                           │
│                                                              │
│  User Browser ──▶ Company's Web Server ──▶ Company's DB      │
│                   (can be taken down,      (single point     │
│                    censored, hacked)        of failure)       │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│                       dApp                                   │
│                                                              │
│  User Browser ──▶ Frontend (static HTML/JS,                  │
│       │            often hosted on IPFS or a CDN)            │
│       │                                                      │
│       └──▶ Smart Contracts on Ethereum                       │
│             (running on thousands of nodes worldwide,        │
│              no single point of failure or control)          │
└──────────────────────────────────────────────────────────────┘
```

The frontend of a dApp is usually a normal web application. The key difference is that all state-changing operations go through transactions sent to smart contracts on the blockchain, not to a company's private database.

### Notable dApp Examples

| dApp | Category | What It Does |
|---|---|---|
| **Uniswap** | DEX (Decentralized Exchange) | Swap ERC-20 tokens without a centralized exchange; uses an automated market maker (AMM) algorithm in smart contracts |
| **Aave** | DeFi Lending | Deposit crypto to earn interest or borrow against collateral; interest rates set by supply/demand algorithms |
| **Compound** | DeFi Lending | Similar to Aave; pioneered the concept of yield farming |
| **OpenSea** | NFT Marketplace | Buy, sell, and auction NFTs (ERC-721 tokens); marketplace rules enforced by smart contracts |
| **MakerDAO** | Stablecoin | Issues DAI, a decentralized stablecoin pegged to $1 USD, backed by crypto collateral locked in smart contracts |
| **ENS** | Naming | Ethereum Name Service — maps human-readable names (e.g., `vitalik.eth`) to Ethereum addresses |
| **Gnosis Safe** | Multisig Wallet | Smart contract wallet requiring multiple approvals for transactions — used by DAOs and teams |

### Limitations of dApps

dApps are powerful but come with real trade-offs:

- **No free lunch on gas:** Every state change costs gas. Users must pay transaction fees.
- **UX friction:** Requiring a wallet (MetaMask etc.) and understanding gas is a higher barrier than traditional apps.
- **Immutability cuts both ways:** Bugs in deployed contracts cannot be patched without special upgrade mechanisms.
- **Speed:** Ethereum's 12-second block time means dApps are slower than traditional apps for writes.

---

## 5. Ethereum's Architecture: Two Layers

After **The Merge** in September 2022, Ethereum's architecture was formally split into two distinct layers, each handled by separate software:

```
┌──────────────────────────────────────────────────────────────┐
│                    CONSENSUS LAYER                           │
│                                                              │
│  • Implements Proof of Stake                                 │
│  • Manages validators, attestations, finality                │
│  • Runs the beacon chain                                     │
│  • Chooses which blocks to finalize                          │
│                                                              │
│  Software: Teku, Lighthouse, Prysm, Nimbus, Lodestar        │
│  (Besu pairs with Teku for production deployments)           │
└────────────────────────┬─────────────────────────────────────┘
                         │  Engine API (JSON-RPC over HTTP)
                         │  Consensus client tells execution client
                         │  which blocks to execute and finalize
                         ▼
┌──────────────────────────────────────────────────────────────┐
│                    EXECUTION LAYER                           │
│                                                              │
│  • Processes transactions                                    │
│  • Runs the EVM (executes smart contract code)               │
│  • Maintains the world state (all account balances/storage)  │
│  • Handles the mempool (pending transactions)                │
│  • Exposes JSON-RPC API to wallets and dApps                 │
│                                                              │
│  Software: Besu, Geth, Nethermind, Erigon, Reth             │
└──────────────────────────────────────────────────────────────┘
```

### Before The Merge (Pre-September 2022)

Before The Merge, execution clients like Geth handled everything: PoW mining, transaction processing, EVM execution, and peer networking for block propagation. The beacon chain ran in parallel as a separate chain preparing for the transition.

### After The Merge

The execution layer **stopped doing consensus** (no more mining). The consensus layer became the sole source of truth about which blocks are canonical. The two layers communicate through the **Engine API** — a JSON-RPC interface where the consensus client instructs the execution client to:

- `engine_forkchoiceUpdated` — "this is the current head of the chain"
- `engine_newPayload` — "here is a new block to execute and validate"
- `engine_getPayload` — "give me a block to propose"

**Besu is an execution layer client.** It handles the EVM and transactions. It must always be paired with a consensus client (like Teku) to run on the public Ethereum network.

---

## 6. The EVM: Ethereum Virtual Machine

> *This section is a high-level introduction. File 07 covers the EVM in depth.*

The **Ethereum Virtual Machine (EVM)** is the runtime environment in which all smart contract code executes on Ethereum.

### Key Properties

- **Sandboxed:** Smart contract code runs in a completely isolated environment. It cannot access the filesystem, network, or any external resource. It can only interact with what is explicitly passed to it and with other smart contracts on-chain.
- **Deterministic:** Given the same starting state and the same transaction, the EVM always produces the same output — on every single node in the world. This is what makes trustless computation possible.
- **Stack-based:** The EVM uses a stack (not registers like most real CPUs) to perform computations. Each item on the stack is a 256-bit word.
- **Gas-metered:** Every operation has a precise gas cost. If a transaction runs out of gas mid-execution, the execution is reverted (as if it never happened), but the gas fee is still charged.

### The EVM as a State Machine

```
Current World State (S)
        +
Transaction Input (T)
        │
        ▼
┌───────────────┐
│               │
│     EVM       │  ← executes T against S
│               │
└───────────────┘
        │
        ▼
New World State (S')
```

Ethereum is essentially a **state machine**. The EVM is the transition function. Every transaction is an input that moves the world from one state to another. The `stateRoot` in each block header is the cryptographic fingerprint of the resulting state.

### Why the EVM Matters for Besu

Besu contains a full, production-quality EVM implementation written in Java. It is:

- Used to execute all transactions when Besu processes blocks.
- Exposed as a **standalone library** (the `evm` module) that can be embedded in other Java applications — for example, to simulate transactions, run tests, or build Layer 2 systems.
- Kept in sync with EVM upgrades (new opcodes, gas cost changes) as Ethereum hard forks are released.

---

## 7. Ether (ETH): The Native Currency

### What Is ETH?

**Ether (ETH)** is the native cryptocurrency of the Ethereum network. It serves two essential purposes:

1. **Paying for computation (gas fees):** Every transaction must pay for the EVM computation it requires, denominated in ETH. Without gas fees, nodes would have no incentive to process transactions, and the network would be vulnerable to spam.
2. **Economic security (staking):** Validators stake ETH as collateral to participate in Proof of Stake consensus. Misbehavior destroys this stake, making attacks economically irrational.

### Denominations

ETH is divisible to 18 decimal places. Different units have names for convenience:

| Unit | Wei Value | Common Use |
|---|---|---|
| **wei** | 1 wei | Smallest unit; used in code/contracts |
| **kwei** | 1,000 wei | Rarely used |
| **mwei** | 1,000,000 wei | Rarely used |
| **gwei** (gigawei) | 1,000,000,000 wei (10⁹) | **Gas prices** — "5 gwei per gas" |
| **szabo** | 1,000,000,000,000 wei (10¹²) | Rarely used |
| **finney** | 10¹⁵ wei | Rarely used |
| **ether** | 10¹⁸ wei | Human-readable amounts — "1 ETH" |

```
1 ETH = 1,000,000,000 gwei
1 ETH = 1,000,000,000,000,000,000 wei  (10^18 wei)

Examples:
  Gas price of 15 gwei = 15,000,000,000 wei per unit of gas
  A simple ETH transfer costs 21,000 gas
  Fee = 21,000 × 15 gwei = 315,000 gwei = 0.000315 ETH
```

### ETH Supply

- **No hard cap** — unlike Bitcoin's 21 million BTC ceiling, ETH has no fixed maximum supply.
- **Pre-Merge:** Miners earned ~2 ETH per block (plus fees), creating ~4.5% annual inflation.
- **Post-Merge:** New ETH issuance dropped by ~90% (validators earn much smaller rewards).
- **EIP-1559:** The `baseFeePerGas` is **burned** (destroyed) with every transaction. At high network usage, more ETH is burned than is issued, making ETH net deflationary.
- This dynamic is sometimes called **"ultrasound money"** by ETH proponents.

---

## 8. Ethereum Mainnet and Testnets

### Mainnet

**Ethereum Mainnet** (chain ID: 1) is the live, production Ethereum network. ETH on mainnet has real monetary value. Contracts deployed here are live and control real funds. You never test on mainnet.

### Testnets

Testnets are public networks that mirror mainnet's behavior but use valueless test ETH. They allow developers to test contracts and applications in a realistic environment without risking real money.

All of the following testnets are supported by Besu:

| Testnet | Chain ID | Consensus | Status | Notes |
|---|---|---|---|---|
| **Sepolia** | 11155111 | PoS | ✅ Active | Primary long-lived developer testnet; recommended for most testing |
| **Hoodi** | 560048 | PoS | ✅ Active | Launched 2025; designed for staking infrastructure testing |
| **Ephemery** | (rotating) | PoS | ✅ Active | Resets periodically (every ~28 days); great for clean-slate testing |
| ~~Goerli~~ | ~~5~~ | ~~PoS~~ | ❌ Deprecated | Shut down January 2025 |
| ~~Rinkeby~~ | ~~4~~ | ~~Clique~~ | ❌ Deprecated | Shut down September 2022 |
| ~~Ropsten~~ | ~~3~~ | ~~PoW~~ | ❌ Deprecated | Shut down December 2022 |

### Getting Testnet ETH

Testnets have **faucets** — websites that send you small amounts of test ETH for free:

- Sepolia faucets: `sepoliafaucet.com`, Alchemy/Infura faucets, `faucet.chainstack.com`
- Hoodi faucets: Check the Ethereum Foundation's developer resources

### Local Development Networks

For fast iteration during development, you can run a local Ethereum network:

- **Besu in dev mode** (`--network=dev`): A single-node network with instant mining. No real consensus, just immediate block production on demand.
- **Hardhat Network / Anvil**: In-process EVM networks used in testing frameworks; very fast, forking of mainnet/testnets supported.

---

## 9. Key Ethereum Improvement Proposals (EIPs)

Ethereum evolves through a structured governance process. Anyone can write an **EIP (Ethereum Improvement Proposal)** — a design document describing a proposed change to the protocol. After community review, testing, and client implementation, accepted EIPs are bundled into scheduled hard forks.

Here are the most impactful EIPs to know:

### EIP-1559 — Fee Market Reform (London Hard Fork, August 2021)

**The problem it solved:** Before EIP-1559, gas prices were set by a first-price auction — users bid gas prices and miners picked the highest bids. This led to volatile, unpredictable fees and systematic overbidding.

**How it works:**
- Every block has a **`baseFeePerGas`** calculated by the protocol based on how full the previous block was.
- The base fee is **burned** (sent to address 0x0, removed from supply).
- Users set a **`maxFeePerGas`** (ceiling they'll pay) and a **`maxPriorityFeePerGas`** (tip to the validator).
- Actual fee paid: `min(maxFeePerGas, baseFee + priorityFee)`

**Key benefits:**
- More predictable fees — wallets can estimate fees accurately.
- ETH burning creates deflationary pressure.
- Validators only earn the tip (priority fee), not the base fee, reducing their incentive to manipulate block fullness.

### EIP-4844 — Shard Blob Transactions (Dencun Hard Fork, March 2024)

**The problem it solved:** Layer 2 rollups (Optimism, Arbitrum, Base, etc.) post their compressed transaction data to Ethereum mainnet as **calldata** — which is expensive because it's stored permanently. This made L2 transaction fees higher than necessary.

**How it works:**
- Introduces a new transaction type carrying **"blobs"** — large chunks of data (~128 KB each) attached to a transaction.
- Blobs are **not accessible to the EVM** (not stored in state) — they are only needed for data availability.
- Blobs are pruned after ~18 days (full nodes don't need to keep them forever).
- Blob space has its own separate fee market, typically much cheaper than calldata.

**Key benefits:**
- L2 transaction costs dropped by ~90% immediately after Dencun.
- Lays the groundwork for **danksharding** — the long-term scaling solution.

### EIP-2930 — Optional Access Lists (Berlin Hard Fork, April 2021)

**The problem it solved:** EIP-2929 increased gas costs for storage and account access to reflect their true disk I/O cost — but this broke some contracts that had been optimized for the old costs.

**How it works:**
- Introduces a new transaction type (type 1) with an optional **access list**: a pre-declared list of addresses and storage slots the transaction will access.
- Pre-declaring access grants a discount on the first access (the "cold access" penalty is avoided for declared slots).

**Key benefits:**
- Partially mitigates gas cost increases from EIP-2929.
- Can reduce gas costs for contracts that access many different storage slots.
- Enables gas cost optimization without changing contract code.

### Other Notable EIPs

| EIP | Name | What It Changed |
|---|---|---|
| **EIP-155** | Replay Attack Protection | Added chain ID to transaction signing |
| **EIP-721** | NFT Standard | Defined the interface for non-fungible tokens |
| **EIP-20** | Token Standard (ERC-20) | Defined the interface for fungible tokens |
| **EIP-1967** | Proxy Storage Slots | Standardized storage layout for upgradeable contracts |
| **EIP-3675** | The Merge Specification | Formally specified the PoW → PoS transition |
| **EIP-4895** | Beacon Chain Withdrawals | Allowed validators to withdraw staked ETH (Shanghai) |

---

## 10. Ethereum's Roadmap

Ethereum's development roadmap was articulated by Vitalik Buterin as a series of named phases. Each phase addresses a different technical challenge. The phases can (and do) progress in parallel — they are not strictly sequential.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     ETHEREUM'S ROADMAP                                  │
├──────────────────┬──────────────────────────────────────────────────────┤
│  THE MERGE ✅    │  Switch from PoW to PoS                              │
│  (Done 2022)     │  Result: 99.95% energy reduction                     │
├──────────────────┼──────────────────────────────────────────────────────┤
│  THE SURGE 🔄    │  Scale Ethereum via rollups + data availability       │
│  (In Progress)   │  EIP-4844 (blobs) is phase 1                         │
│                  │  Goal: 100,000+ TPS via L2s                          │
│                  │  Key tech: Danksharding, PeerDAS                     │
├──────────────────┼──────────────────────────────────────────────────────┤
│  THE SCOURGE 🔄  │  Address MEV and censorship concerns                 │
│  (In Progress)   │  Separate block building from block proposing (PBS)  │
│                  │  Inclusion lists to prevent censorship                │
├──────────────────┼──────────────────────────────────────────────────────┤
│  THE VERGE 📋    │  Make verification cheaper / trustless               │
│  (Planned)       │  Verkle Trees replace Merkle Patricia Tries          │
│                  │  Stateless clients — no need to store full state     │
│                  │  SNARKify the consensus (validity proofs for PoS)    │
├──────────────────┼──────────────────────────────────────────────────────┤
│  THE PURGE 📋    │  Simplify the protocol; reduce historical burden     │
│  (Planned)       │  EIP-4444: stop requiring nodes to serve old history │
│                  │  Reduce protocol complexity and technical debt        │
├──────────────────┼──────────────────────────────────────────────────────┤
│  THE SPLURGE 📋  │  Everything else — miscellaneous improvements        │
│  (Planned)       │  Account abstraction (EIP-4337 / EIP-7702)           │
│                  │  EVM improvements, new precompiles                   │
│                  │  Deeper L1/L2 integration                            │
└──────────────────┴──────────────────────────────────────────────────────┘
```

### What This Means for Besu

Every hard fork in Ethereum's roadmap requires updates to execution clients like Besu. The Besu team tracks upcoming EIPs, implements them, tests them on testnets, and ships updates before each scheduled mainnet upgrade. This is why keeping your Besu installation up to date is important — running an outdated client means falling behind on hard fork rules and eventually being unable to sync the chain.

---

## 11. Layer 2 Solutions

### Why Layer 2?

Ethereum's base layer (L1) is intentionally conservative in its throughput — roughly 15–30 transactions per second on the execution layer. This is a deliberate design choice: every node must re-execute every transaction, so higher throughput means higher hardware requirements for nodes, which leads to centralization.

**Layer 2 (L2)** solutions move transaction execution *off* the main chain while still **inheriting Ethereum's security**. The core idea: do the heavy computational lifting elsewhere, but post enough data (and/or proofs) to Ethereum so that the L1 can serve as a final arbiter if something goes wrong.

### Rollups: The Primary L2 Approach

A **rollup** processes thousands of transactions off-chain, compresses them, and posts a summary to Ethereum. There are two main types:

#### Optimistic Rollups

```
┌─────────────────────────────────────────────────────────────┐
│  OPTIMISTIC ROLLUP (e.g., Optimism, Arbitrum, Base)        │
│                                                             │
│  1. Users send transactions to the rollup sequencer        │
│  2. Sequencer executes them off-chain (fast, cheap)        │
│  3. Sequencer posts compressed tx data to Ethereum L1      │
│  4. Block is "optimistically" accepted as valid            │
│  5. 7-day challenge window — anyone can submit a           │
│     "fraud proof" if they can prove the sequencer lied     │
│  6. If no valid fraud proof: batch finalized               │
│  7. If fraud proof valid: batch rejected, sequencer slashed│
└─────────────────────────────────────────────────────────────┘
```

- **Pros:** EVM-compatible (easy to port existing contracts), simpler technology.
- **Cons:** 7-day withdrawal delay to L1 (due to the challenge window); relies on at least one honest watcher.

#### ZK-Rollups (Zero-Knowledge Rollups)

```
┌─────────────────────────────────────────────────────────────┐
│  ZK-ROLLUP (e.g., zkSync Era, Starknet, Polygon zkEVM,     │
│             Linea, Scroll)                                  │
│                                                             │
│  1. Users send transactions to the rollup                  │
│  2. Rollup executes them off-chain                         │
│  3. Rollup generates a cryptographic validity proof        │
│     (ZK-SNARK or ZK-STARK) that proves all txs were        │
│     executed correctly without revealing the details       │
│  4. Proof + compressed data posted to Ethereum L1         │
│  5. L1 verifies the proof (cheap) — if valid, batch        │
│     is immediately finalized                               │
└─────────────────────────────────────────────────────────────┘
```

- **Pros:** Immediate finality (no challenge window); mathematically proven correctness.
- **Cons:** Generating ZK proofs is computationally expensive; EVM compatibility is harder to achieve (zkEVMs are a major engineering effort).

### Besu's Role in the L2 Ecosystem

Besu is primarily an **Ethereum L1 execution client** — it runs nodes on Ethereum mainnet and testnets. However, Besu's reach extends into the L2 space:

- **Linea** — a ZK-rollup developed by Consensys (Besu's parent company) — uses a **modified version of Besu** as its execution client. The Linea team contributes improvements back to Besu upstream.
- Besu's EVM library is used by some L2 projects as a standalone component for local simulation and testing.
- Private enterprise networks built with Besu can act as application-specific L2-like chains.

### L2 Comparison

| Property | Optimistic Rollup | ZK-Rollup |
|---|---|---|
| **Withdrawal delay** | ~7 days | Minutes (after proof verification) |
| **Proof type** | Fraud proofs (post-hoc) | Validity proofs (pre-validated) |
| **EVM compatibility** | Near-perfect | Harder (zkEVM required) |
| **Cost to post data to L1** | Moderate (calldata / blobs) | Moderate + proof verification cost |
| **Trust assumption** | ≥1 honest validator watching | Pure cryptographic proof |
| **Examples** | Optimism, Arbitrum, Base | zkSync Era, Linea, Starknet, Scroll |

---

## 12. Recap Checklist

After reading this guide, you should be able to confidently say:

- [ ] I know the key milestones in Ethereum's history from the 2013 whitepaper through The Merge and Dencun
- [ ] I can compare Bitcoin and Ethereum across purpose, account model, block time, consensus, and programmability
- [ ] I understand what a smart contract is: immutable, deterministic, self-executing code on the blockchain
- [ ] I can describe what a dApp is and give 3+ real-world examples with their categories
- [ ] I understand that post-Merge Ethereum has two separate layers: the execution layer (Besu) and the consensus layer (Teku)
- [ ] I understand the Engine API connects the two layers
- [ ] I know the EVM is a sandboxed, deterministic, stack-based virtual machine that executes smart contract code
- [ ] I can convert between wei, gwei, and ether (1 ETH = 10^18 wei = 10^9 gwei)
- [ ] I can name and describe Ethereum's active testnets: Sepolia, Hoodi, Ephemery
- [ ] I can explain EIP-1559 (fee market: baseFee burned, tips to validators), EIP-4844 (blobs for L2 data), and EIP-2930 (access lists)
- [ ] I can name all 6 phases of Ethereum's roadmap and their main goals
- [ ] I can explain the difference between Optimistic Rollups and ZK-Rollups
- [ ] I know Besu's role: L1 execution client, and also used in Linea (a ZK-rollup)

---

## 13. Check Your Understanding

Try to answer these without looking back:

1. **What fundamental limitation of Bitcoin motivated Vitalik Buterin to propose Ethereum?** What is the key property that Ethereum's scripting adds that Bitcoin's does not have?

2. **A friend claims: "I store my ETH in MetaMask." Technically, is this accurate? Where does ETH actually 'live,' and what does MetaMask actually store?** (Hint: think about the world state and private keys.)

3. **EIP-1559 introduced a base fee that is burned rather than paid to validators. What problem does this solve, and why might validators have previously had an incentive to behave badly under the old first-price auction system?**

4. **An Optimistic Rollup claims to "inherit Ethereum's security." What does this mean concretely?** What would have to happen for a rollup sequencer to successfully steal user funds, and what prevents it?

5. **After The Merge, Besu alone is not sufficient to run an Ethereum mainnet node — you also need a consensus client like Teku. What specific function does each client handle, and how do they communicate?**

6. **EIP-4844 added "blobs" to Ethereum. A blob is not accessible to the EVM and is pruned after ~18 days. If blobs are not permanent and not accessible to contracts, why are they useful for Layer 2 rollups?**

---

## Next Up

**[06 — Ethereum Accounts, Wallets & Transactions →](06_ethereum_accounts_and_transactions.md)**

Now that you understand Ethereum as a platform, the next guide goes deep into its fundamental data model: how accounts work, what a wallet really is, how transactions are structured and signed, what happens when a transaction executes, and how the entire world state is stored as a cryptographic data structure.