# 03 — How Blockchain Works Internally

> **Level:** Beginner → Intermediate | **Read time:** ~25 minutes
> **Prerequisites:** [02 — Cryptography Basics](02_cryptography_basics.md)

---

## Table of Contents

1. [The Anatomy of a Block](#1-the-anatomy-of-a-block)
2. [ASCII Diagram: Block Structure](#2-ascii-diagram-block-structure)
3. [How a Transaction Flows Through the Network](#3-how-a-transaction-flows-through-the-network)
4. [What Is the Mempool?](#4-what-is-the-mempool)
5. [How Nodes Validate a Block](#5-how-nodes-validate-a-block)
6. [The Concept of Finality](#6-the-concept-of-finality)
7. [What Is a Fork?](#7-what-is-a-fork)
8. [What Is a Genesis Block?](#8-what-is-a-genesis-block)
9. [The Genesis File](#9-the-genesis-file)
10. [Chain ID and Network ID](#10-chain-id-and-network-id)
11. [ASCII Diagram: Blocks Chaining with Hashes](#11-ascii-diagram-blocks-chaining-with-hashes)
12. [Recap Checklist](#12-recap-checklist)
13. [Check Your Understanding](#13-check-your-understanding)

---

## 1. The Anatomy of a Block

A block is the fundamental unit of a blockchain. Think of it like a page in a ledger book: each page records a batch of transactions, references the previous page, and is sealed with a unique fingerprint (hash) so that nobody can quietly alter it later.

Every block is divided into two parts: the **header** and the **body**.

---

### 1.1 The Block Header

The header is a compact summary of the block. It contains metadata — not the raw transactions themselves, but cryptographic commitments (hashes) that prove exactly what the body contains.

| Field | Type | Description |
|---|---|---|
| `parentHash` | 32-byte hash | The Keccak-256 hash of the **previous block's header**. This is the link that chains blocks together. Changing any block breaks every subsequent `parentHash`. |
| `stateRoot` | 32-byte hash | The root hash of the **Merkle Patricia Trie** representing the entire world state (every account's balance, nonce, code, and storage) **after** all transactions in this block have been executed. |
| `transactionsRoot` | 32-byte hash | The root hash of the Merkle trie containing all transactions in this block. Any node can prove a specific transaction is included using only this hash + a short proof path. |
| `receiptsRoot` | 32-byte hash | The root hash of the Merkle trie of transaction receipts (results). Receipts include gas used, logs/events emitted, and success/failure status. |
| `number` | integer | The block height — how many blocks have come before this one. The genesis block is block 0. |
| `timestamp` | integer | Unix timestamp (seconds since Jan 1, 1970 UTC) set by the block producer. Nodes reject blocks with timestamps too far in the future. |
| `difficulty` / `nonce` | varies | Used in **Proof of Work** to prove the miner did computational work. In PoS (post-Merge Ethereum), `difficulty` is 0 and `nonce` is `0x0000000000000000`. |
| `gasLimit` | integer | The maximum total gas that all transactions in this block may consume. Acts as a cap on block size / computation. Validators can vote to slowly adjust this up or down. |
| `gasUsed` | integer | The actual total gas consumed by all transactions in this block. Always ≤ `gasLimit`. |
| `miner` / `coinbase` | 20-byte address | The Ethereum address of the block producer (miner in PoW, validator/proposer in PoS). Block rewards and priority fees are sent here. |
| `logsBloom` | 256-byte filter | A Bloom filter encoding all the event topics and addresses from logs in this block. Allows light clients to quickly check if a block *might* contain a specific event without downloading all receipts. |
| `extraData` | ≤32 bytes | Arbitrary data the block producer can include. Often used by mining pools for identification, or by PoA networks to carry validator signatures (e.g., IBFT 2.0 / QBFT). |
| `baseFeePerGas` | integer | Introduced in EIP-1559. The minimum gas price for transactions in this block, calculated by protocol based on how full the previous block was. This fee is **burned** (not paid to the validator). |
| `mixHash` | 32-byte hash | In PoW (Ethash), combined with `nonce` to prove the work. In PoS, repurposed as `prevRandao` — a randomness beacon contributed by the validator. |

### 1.2 The Block Body

The body is simpler — it contains the actual list of signed transactions included in this block, in order. Each transaction is fully encoded (RLP-serialized) including its `(v, r, s)` signature.

The body also contains the list of **uncle headers** (PoW only — blocks that were mined at the same time as this block but didn't make it into the main chain; miners were rewarded for including them to reduce wasted work).

In PoS Ethereum, uncle blocks no longer exist.

---

## 2. ASCII Diagram: Block Structure

```
┌══════════════════════════════════════════════════════════════════╗
║                        BLOCK N                                   ║
╠══════════════════════════════════════════════════════════════════╣
║  HEADER                                                          ║
║  ┌────────────────────────────────────────────────────────────┐  ║
║  │ parentHash       : 0x4a3b...f1c2  ← hash of Block N-1     │  ║
║  │ number           : 18,500,000                              │  ║
║  │ timestamp        : 1710000000                              │  ║
║  │ miner/coinbase   : 0xAbCd...1234                           │  ║
║  │ gasLimit         : 30,000,000                              │  ║
║  │ gasUsed          : 14,382,910                              │  ║
║  │ baseFeePerGas    : 12 gwei                                 │  ║
║  │ stateRoot        : 0x9f1a...cc03  ← fingerprint of ALL    │  ║
║  │                                     account states        │  ║
║  │ transactionsRoot : 0x7e2b...a8f0  ← Merkle root of txs   │  ║
║  │ receiptsRoot     : 0x3c4d...b7e1  ← Merkle root of rcpts │  ║
║  │ logsBloom        : 0x0000...0000  (256 bytes)             │  ║
║  │ extraData        : 0x                                      │  ║
║  └────────────────────────────────────────────────────────────┘  ║
║                                                                  ║
║  BODY                                                            ║
║  ┌────────────────────────────────────────────────────────────┐  ║
║  │ transactions: [                                            │  ║
║  │   Tx0: { from: 0xAlice, to: 0xBob, value: 1 ETH, ... }   │  ║
║  │   Tx1: { from: 0xCarol, to: 0xDEX, data: swap(), ... }   │  ║
║  │   Tx2: { from: 0xDave,  to: 0x000, data: deploy(), ... } │  ║
║  │   ... up to gasLimit worth of transactions                │  ║
║  │ ]                                                         │  ║
║  └────────────────────────────────────────────────────────────┘  ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 3. How a Transaction Flows Through the Network

From the moment you click "Send" to the moment your transaction is permanently recorded, it passes through several distinct stages. Here is the complete journey:

```
┌──────────────────────────────────────────────────────────────────┐
│  STAGE 1: CREATION (your wallet)                                 │
│                                                                  │
│  User fills in: recipient, amount, gas settings                  │
│  Wallet assembles tx object, fetches correct nonce from node     │
│  Wallet signs the transaction with the user's private key        │
│  → produces signed transaction with (v, r, s) attached           │
└────────────────────────────┬─────────────────────────────────────┘
                             │  eth_sendRawTransaction (JSON-RPC)
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│  STAGE 2: SUBMISSION (your chosen Ethereum node, e.g. Besu)      │
│                                                                  │
│  Node receives the raw signed transaction                        │
│  Node performs basic validation:                                 │
│    ✓ Valid RLP encoding?                                         │
│    ✓ Valid signature? (recover sender address)                   │
│    ✓ Sender nonce matches expected nonce?                        │
│    ✓ Sender has enough ETH for gas + value?                      │
│    ✓ Gas limit ≥ 21,000 (minimum for a transfer)?               │
│    ✓ Gas price ≥ current baseFeePerGas?                         │
└────────────────────────────┬─────────────────────────────────────┘
                             │  passes validation
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│  STAGE 3: THE MEMPOOL (transaction pool)                         │
│                                                                  │
│  Transaction sits in the node's local mempool                    │
│  Node gossips the transaction to its peers                       │
│  Peers gossip to their peers → propagates across the network     │
│  Transaction is now in thousands of mempools simultaneously      │
└────────────────────────────┬─────────────────────────────────────┘
                             │  a validator/miner picks it up
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│  STAGE 4: BLOCK BUILDING (validator / miner)                     │
│                                                                  │
│  Validator selects profitable transactions from mempool          │
│  Typically sorted by: highest priority fee (tip) first          │
│  Assembles a block: applies txs one by one, tracking gasUsed     │
│  Stops when gasLimit reached or no more txs available            │
│  Computes: transactionsRoot, stateRoot, receiptsRoot             │
│  Signs the block (PoA) or finds valid nonce (PoW)               │
└────────────────────────────┬─────────────────────────────────────┘
                             │  broadcasts new block
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│  STAGE 5: BLOCK PROPAGATION & VALIDATION (all other nodes)       │
│                                                                  │
│  Nodes receive the new block                                     │
│  Each node independently validates the block (see Section 5)     │
│  Valid block → appended to the node's local chain                │
│  Transaction removed from mempool (it's been included)           │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│  STAGE 6: CONFIRMATION & FINALITY                                │
│                                                                  │
│  Transaction is now "confirmed" (in a block)                     │
│  More blocks built on top → more confirmations → more certain    │
│  After sufficient confirmations (or PoS finalization):           │
│  → Transaction is considered final / irreversible                │
└──────────────────────────────────────────────────────────────────┘
```

---

## 4. What Is the Mempool?

The **mempool** (memory pool, also called the transaction pool or txpool) is a temporary holding area in each node's memory where valid, pending transactions wait to be included in a block.

Key characteristics:

- **Not part of the blockchain.** The mempool is local to each node and is not stored on-chain. Different nodes can have slightly different mempools at any given moment.
- **Volatile.** Transactions disappear from the mempool once they are included in a block. Transactions that stay too long without being picked up may be dropped (evicted) by nodes to free memory.
- **Not ordered globally.** There is no single global ordering. Each validator chooses which transactions to include and in what order — this is the basis of **MEV (Maximal Extractable Value)**.
- **Replacement.** You can replace a pending transaction by sending a new one with the same nonce but a higher gas price. This is how "speed up" and "cancel" work in wallets like MetaMask.
- **Priority.** In EIP-1559, validators prioritize transactions with higher **priority fees** (tips). The `maxPriorityFeePerGas` field is what you set when you want faster inclusion.

```
Mempool at a given moment (simplified view):

┌─────────────────────────────────────────────────────────────────┐
│                         MEMPOOL                                 │
│                                                                 │
│  [HIGH PRIORITY — included first]                               │
│  Tx: 0xAaaa... | from: 0x1234 | tip: 10 gwei | value: 2 ETH   │
│  Tx: 0xBbbb... | from: 0x5678 | tip:  8 gwei | data: swap()   │
│                                                                 │
│  [MEDIUM PRIORITY]                                              │
│  Tx: 0xCccc... | from: 0x9abc | tip:  3 gwei | value: 0.1 ETH  │
│  Tx: 0xDddd... | from: 0xdef0 | tip:  2 gwei | data: mint()   │
│                                                                 │
│  [LOW PRIORITY — may wait many blocks]                          │
│  Tx: 0xEeee... | from: 0x1111 | tip:  1 gwei | value: 0.5 ETH  │
│  ...                                                            │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. How Nodes Validate a Block

When a node receives a new block, it does **not** blindly trust it. Every node independently performs a full set of validation checks. This is the heart of the "trustless" nature of blockchain — you don't trust the sender of the block, you trust the math.

Here is what a node like Besu checks:

### 5.1 Block Header Checks

1. **Parent hash exists:** Does `parentHash` match the hash of the current chain tip? If not, this block belongs to a different chain or a fork.
2. **Block number is sequential:** `number` must equal `parent.number + 1`.
3. **Timestamp is valid:** `timestamp` must be greater than the parent's timestamp. Also must not be too far in the future (nodes reject blocks timestamped more than 15 seconds into the future).
4. **Gas limit is within bounds:** The `gasLimit` must not change by more than 1/1024 of the parent's gas limit in either direction.
5. **Gas used does not exceed gas limit:** `gasUsed ≤ gasLimit`.
6. **Base fee is correct:** The `baseFeePerGas` must be the exact value calculated by the EIP-1559 formula from the parent block's gas usage. Nodes can compute this independently and reject any block that gets it wrong.
7. **Consensus-specific checks:** For PoW, verify the `nonce` and `mixHash` solve the puzzle. For PoS, verify validator attestations. For PoA (QBFT/IBFT), verify the validator signatures in `extraData`.

### 5.2 Transaction Checks

For each transaction in the block:

1. **Valid signature:** Re-derive the sender address from `(v, r, s)`. Does it match `tx.from`?
2. **Valid nonce:** Does the transaction's nonce match the sender's current nonce in the state?
3. **Sufficient balance:** Does the sender have enough ETH to cover `gasLimit × maxFeePerGas + value`?
4. **Gas price ≥ base fee:** `tx.maxFeePerGas ≥ block.baseFeePerGas`.

### 5.3 State Execution & Root Verification

This is the most intensive part:

1. **Re-execute every transaction** in the block, in order, starting from the parent block's world state.
2. Track state changes: update account balances, nonces, contract storage.
3. After executing all transactions, compute the resulting `stateRoot` (the Merkle root of the new world state).
4. **Compare computed `stateRoot` with the header's `stateRoot`.** They must match exactly.
5. Do the same for `transactionsRoot` and `receiptsRoot`.

If the state root doesn't match, the block is invalid — even if every individual transaction looked fine. This means the block producer made an error or deliberately tried to cheat.

---

## 6. The Concept of Finality

**Finality** means a transaction is permanently and irreversibly part of the blockchain. But "finality" works differently in different consensus systems.

### 6.1 Probabilistic Finality (Proof of Work)

In PoW, finality is never 100% guaranteed — it's probabilistic. After a transaction is included in a block, subsequent blocks are built on top of it:

```
Block 100   Block 101   Block 102   Block 103   Block 104
(your tx)  (confirms)  (confirms)  (confirms)  (confirms)
   ▼           ▼           ▼           ▼           ▼
───●───────────●───────────●───────────●───────────●──── chain
```

Each additional block makes reverting the transaction exponentially harder — an attacker would need to outpace the honest network and rebuild all subsequent blocks. Common thresholds:
- **1 confirmation** — very risky (only suitable for tiny amounts)
- **6 confirmations** — considered safe for most use cases on Bitcoin (legacy)
- **12–30 confirmations** — used by exchanges for large amounts on PoW chains

### 6.2 Finality in Proof of Stake (Ethereum)

Post-Merge Ethereum has **checkpointing finality** via the beacon chain:

- Time is divided into **slots** (12 seconds each) and **epochs** (32 slots = 6.4 minutes).
- At the end of each epoch, validators **vote** on the epoch boundary block.
- When 2/3 of all staked ETH has voted for a checkpoint, it is **justified**.
- When two consecutive checkpoints are justified, the earlier one is **finalized**.
- A finalized block cannot be reverted without destroying at least 1/3 of all staked ETH (which would be **slashed** — burned as a penalty).

In practice: **~13 minutes (2 epochs)** to achieve finality on Ethereum mainnet.

### 6.3 Immediate Finality (Proof of Authority)

In PoA networks like QBFT and IBFT 2.0 used in private Besu networks:
- A block is **final as soon as it is produced** — if ≥2/3 of validators signed it, the protocol guarantees no fork can override it.
- There is no waiting for confirmations.
- This is ideal for enterprise use cases where speed matters and validators are known/trusted.

---

## 7. What Is a Fork?

A fork occurs when two different valid blocks are produced at approximately the same time, both claiming to be the legitimate next block in the chain. This creates two competing versions of the chain.

### 7.1 How a Fork Happens

```
               Block 99 (agreed upon by all)
                    │
          ┌─────────┴──────────┐
          │                    │
       Block 100A           Block 100B
      (Miner Alice)        (Miner Bob)
          │                    │
       Block 101A           Block 101B
```

Different nodes receive different blocks first:
- Nodes that heard about Block 100A first extend that chain.
- Nodes that heard about Block 100B first extend that chain.

### 7.2 Fork Resolution (PoW — Longest Chain Rule)

In PoW, the rule is simple: **the chain with the most cumulative difficulty wins**. Since blocks take computational work to produce, the longer chain represents more honest effort. As soon as one chain gets a block ahead:

```
Block 99 → Block 100A → Block 101A → Block 102A   ← WINNING CHAIN
         → Block 100B → (orphaned, abandoned)
```

Nodes that were following the B-chain switch to the A-chain. Block 100B becomes an **uncle block** (or stale block). Transactions in 100B that are not in 100A go back to the mempool.

### 7.3 Fork Resolution (PoS)

Post-Merge Ethereum uses the **LMD-GHOST** fork choice rule (Latest Message Driven Greedy Heaviest Observed SubTree). Validators vote with their stake, and the fork with the most stake weight wins. Forks are rarer and resolve quickly.

### 7.4 Hard Forks vs Soft Forks

A **hard fork** is a protocol upgrade that changes the rules in a non-backward-compatible way. Nodes that don't upgrade reject the new blocks. Examples: The Merge, London (EIP-1559), Constantinople.

A **soft fork** is backward-compatible — old nodes still accept new blocks. Rarer in practice.

---

## 8. What Is a Genesis Block?

The genesis block is **Block 0** — the very first block in the chain. It is special in several ways:

- It has **no parent** — the `parentHash` is all zeros: `0x0000...0000`.
- It is **not mined** or produced through consensus — it is simply configured and agreed upon at network launch.
- It **pre-funds accounts** (allocates initial ETH balances) as specified in the genesis configuration.
- Its hash becomes the `parentHash` of Block 1, anchoring the entire chain.
- Every node on the same network must start from the **exact same genesis block**. Different genesis = different chain. You cannot connect to a network if your genesis doesn't match.

```
Genesis Block (Block 0)
┌──────────────────────────────────────────────────┐
│ parentHash : 0x000000...000000 (no parent)        │
│ number     : 0                                    │
│ timestamp  : (configured in genesis.json)         │
│ stateRoot  : (derived from initial allocations)  │
│ nonce      : (configured)                         │
│ difficulty : (configured)                         │
│ gasLimit   : (configured)                         │
│                                                   │
│ [pre-funded accounts from genesis config]         │
│   0xAlice : 1000 ETH                              │
│   0xBob   :  500 ETH                              │
│   ...                                             │
└──────────────────────────────────────────────────┘
```

---

## 9. The Genesis File

In Besu (and Ethereum generally), the genesis configuration is defined in a **`genesis.json`** file. When you start a new private network, every node must be given the same genesis file.

### Example `genesis.json` for a QBFT private network

```json
{
  "config": {
    "chainId": 1337,
    "berlinBlock": 0,
    "londonBlock": 0,
    "qbft": {
      "blockperiodseconds": 2,
      "epochlength": 30000,
      "requesttimeoutseconds": 4
    }
  },
  "nonce": "0x0",
  "timestamp": "0x0",
  "gasLimit": "0x29B92700",
  "difficulty": "0x1",
  "mixHash": "0x63746963616c2062797a616e74696e65206661756c7420746f6c6572616e6365",
  "coinbase": "0x0000000000000000000000000000000000000000",
  "alloc": {
    "0xfe3b557e8fb62b89f4916b721be55ceb828dbd73": {
      "privateKey": "8f2a55949038a9610f50fb23b5883af3b4ecb3c3bb792cbcefbd1542c692be63",
      "comment": "private key and wallet address used in the 4 nodes private network",
      "balance": "0xad78ebc5ac6200000"
    },
    "0x627306090abaB3A6e1400e9345bC60c78a8BEf57": {
      "balance": "90000000000000000000000"
    }
  },
  "extraData": "0x..."
}
```

### Key Fields Explained

| Field | Description |
|---|---|
| `config.chainId` | Unique identifier for this chain. Used in transaction signing to prevent replay attacks. Must be unique if you want to connect to other networks. |
| `config.qbft` (or `ibft2`, `clique`, `ethash`) | Specifies which consensus engine to use and its parameters (block period, epoch length, timeouts). |
| `config.londonBlock: 0` | Activates the London hard fork (EIP-1559) from block 0, meaning this network starts with EIP-1559 rules immediately. |
| `gasLimit` | The initial gas limit for the genesis block. Determines how much computation fits in a block. |
| `difficulty` | Initial mining difficulty. In PoA networks, set to `0x1` (effectively irrelevant since there's no mining). |
| `timestamp` | Unix timestamp of the genesis block. Typically `0x0` for new private networks. |
| `alloc` | Pre-funded accounts. A map of Ethereum addresses to initial ETH balances. This is how you give test accounts their starting funds. |
| `extraData` | In QBFT/IBFT 2.0, this field contains the encoded list of initial validator addresses. |
| `mixHash` | In QBFT, a special magic value that signals to Besu this is a QBFT genesis block. |

---

## 10. Chain ID and Network ID

These two IDs are often confused but serve related, distinct purposes.

### Chain ID

**Chain ID** is embedded in the transaction signature (introduced by EIP-155). When you sign a transaction, the chain ID is included in the data that gets hashed and signed.

This prevents **replay attacks**: if you sign a transaction on the Sepolia testnet (chainId 11155111) and someone copies the signed transaction and broadcasts it on Ethereum mainnet (chainId 1), the signature will be invalid because the chain ID embedded in the signing input doesn't match.

Without chain ID protection, any transaction on one chain could be replayed on any other chain.

### Network ID

**Network ID** is used at the peer-to-peer networking layer to identify which network a node belongs to. Nodes only connect to peers with the same network ID.

In most cases, chain ID and network ID are set to the same value. They can differ, but this is uncommon.

### Common Chain IDs

| Network | Chain ID |
|---|---|
| Ethereum Mainnet | 1 |
| Sepolia Testnet | 11155111 |
| Hoodi Testnet | 560048 |
| Ephemery Testnet | (changes each period) |
| Local development (Besu default) | 1337 |

In your genesis file, always set a **unique chain ID** for private networks. If you accidentally use chain ID 1 (mainnet) on a private network, users' wallets may behave unexpectedly — or worse, transactions could theoretically be replayable if private-key/account overlap occurred.

---

## 11. ASCII Diagram: Blocks Chaining with Hashes

This diagram shows how the `parentHash` field creates an unbreakable chain, and how the state root commits to the full world state at each block:

```
                        BLOCKCHAIN

  GENESIS             BLOCK 1              BLOCK 2             BLOCK 3
  (Block 0)
┌───────────┐       ┌───────────┐       ┌───────────┐       ┌───────────┐
│parentHash │       │parentHash │       │parentHash │       │parentHash │
│ 0x000000  │       │ = hash(B0)│       │ = hash(B1)│       │ = hash(B2)│
│           │──────▶│           │──────▶│           │──────▶│           │
│stateRoot  │       │stateRoot  │       │stateRoot  │       │stateRoot  │
│  SR0      │       │  SR1      │       │  SR2      │       │  SR3      │
│           │       │           │       │           │       │           │
│txsRoot:   │       │txsRoot:   │       │txsRoot:   │       │txsRoot:   │
│ (empty)   │       │  TR1      │       │  TR2      │       │  TR3      │
│           │       │           │       │           │       │           │
│ Block     │       │ Block     │       │ Block     │       │ Block     │
│ Hash: B0  │       │ Hash: B1  │       │ Hash: B2  │       │ Hash: B3  │
└───────────┘       └───────────┘       └───────────┘       └───────────┘
      │                   │                   │                   │
      ▼                   ▼                   ▼                   ▼
  State Trie          State Trie          State Trie          State Trie
  (SR0)               (SR1)               (SR2)               (SR3)
  ┌────────┐          ┌────────┐          ┌────────┐          ┌────────┐
  │ Alice  │          │ Alice  │          │ Alice  │          │ Alice  │
  │  1000  │          │   999  │          │   998  │          │  1098  │
  │ ETH    │          │ ETH    │          │ ETH    │          │ ETH    │
  │        │          │        │          │        │          │        │
  │ Bob    │          │ Bob    │          │ Bob    │          │ Bob    │
  │    0   │          │     1  │          │     2  │          │     2  │
  │ ETH    │          │ ETH    │          │ ETH    │          │ ETH    │
  └────────┘          └────────┘          └────────┘          └────────┘

  [If an attacker modifies a transaction in Block 1:]

  ✗ Block 1's transactions change → transactionsRoot changes
  ✗ Block 1's stateRoot changes   → Block 1's block hash changes
  ✗ Block 2's parentHash no longer matches Block 1's (new) hash
  ✗ Block 2 is now invalid → Block 3 is invalid → entire chain from B1 onward breaks
  ✗ Attacker must redo ALL subsequent blocks AND convince the network to accept them
  ✗ Practically impossible on a live network with thousands of honest nodes
```

---

## 12. Recap Checklist

After reading this guide, you should be comfortable with all of these:

- [ ] I can name and explain all the major fields in a block header
- [ ] I understand the difference between the block header (metadata/hashes) and the block body (transactions)
- [ ] I can describe the 6 stages a transaction goes through from wallet to finality
- [ ] I understand what the mempool is, why it exists, and how transactions are prioritized within it
- [ ] I know the 3 main things a node validates when it receives a new block: header rules, transaction signatures, and state root re-execution
- [ ] I can explain probabilistic finality (PoW), checkpoint finality (PoS), and immediate finality (PoA)
- [ ] I understand what a fork is, how it occurs, and how the network resolves it
- [ ] I know what the genesis block is and why every node on the same network must have the same genesis configuration
- [ ] I can identify the key fields in a `genesis.json` file and explain what each one controls
- [ ] I understand the difference between chain ID and network ID, and why chain ID prevents replay attacks

---

## 13. Check Your Understanding

Try to answer these without looking back:

1. **A block's `stateRoot` is computed *after* executing all transactions, not before. Why does this matter?** What would happen if someone included a `stateRoot` that didn't match the actual result of executing the transactions?

2. **Why must every node on the same network have the exact same `genesis.json`?** What would happen if one node started with a different pre-funded account allocation?

3. **A transaction is included in Block 500 on a PoW chain. The block has 6 confirmations. What does "6 confirmations" mean, and why doesn't it guarantee 100% finality?**

4. **Two miners produce Block 101A and Block 101B simultaneously. Both are valid. A node receives Block 101A first. Later it hears about a longer chain built on top of Block 101B. What does the node do, and what happens to transactions that were only in Block 101A?**

5. **Why is Chain ID included in the transaction signature? Give a concrete example of what could go wrong without it.**

6. **You deploy a private Besu network for testing and accidentally set `chainId: 1` (Ethereum mainnet) in your `genesis.json`. Your team's wallets work fine. Should you be worried? Why or why not?**

---

## Next Up

**[04 — Consensus Mechanisms: PoW, PoS & PoA →](04_consensus_mechanisms.md)**

Now that you understand how blocks are built and validated, the next guide dives into the engine behind it all: how do thousands of nodes with no central authority actually *agree* on which block to add next? We'll explore Proof of Work, Proof of Stake, and the Proof of Authority algorithms supported by Besu.