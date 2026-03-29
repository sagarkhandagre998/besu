# 06 — Ethereum Accounts, Wallets & Transactions

> **Level:** Intermediate | **Read time:** ~25 minutes
> **Prerequisites:** [05 — Ethereum: The Programmable Blockchain](05_ethereum_introduction.md)

---

## Table of Contents

1. [Two Types of Accounts](#1-two-types-of-accounts)
2. [Account State Fields](#2-account-state-fields)
3. [What Is a Nonce?](#3-what-is-a-nonce)
4. [What Is a Wallet?](#4-what-is-a-wallet)
5. [Transaction Anatomy](#5-transaction-anatomy)
6. [Transaction Types](#6-transaction-types)
7. [Transaction Lifecycle](#7-transaction-lifecycle)
8. [Transaction Receipts](#8-transaction-receipts)
9. [Events and Logs](#9-events-and-logs)
10. [RLP Encoding](#10-rlp-encoding)
11. [The World State](#11-the-world-state)
12. [Recap Checklist](#12-recap-checklist)
13. [Check Your Understanding](#13-check-your-understanding)

---

## 1. Two Types of Accounts

Everything on Ethereum — every balance, every deployed contract, every piece of state — lives inside an **account**. There are exactly two kinds.

---

### 1.1 Externally Owned Accounts (EOA)

An EOA is what most people think of when they say "my Ethereum wallet." It is an account controlled by a **private key** held by a human (or an automated system acting on a human's behalf).

Key characteristics:

- **Controlled by a private key.** Whoever holds the private key can authorise transactions from this account. There is no other form of access control — no password, no recovery email, no "forgot my key" option.
- **Has an ETH balance.** The account holds a balance of ether, measured in wei.
- **Has a nonce.** A counter that starts at 0 and increments by 1 with every transaction sent from this account.
- **Has no code.** An EOA cannot contain executable bytecode. It is a pure value-holding account.
- **Initiates all activity.** Every chain of events on Ethereum begins with an EOA signing and broadcasting a transaction. Smart contracts can call other contracts, but the very first call always originates from an EOA.

```
EOA: 0x71C7656EC7ab88b098defB751B7401B5f6d8976F
┌────────────────────────────────────┐
│ nonce    : 42                      │  ← sent 42 transactions so far
│ balance  : 3,500,000,000,000,000,  │
│            000 wei  (= 3.5 ETH)    │
│ codeHash : (hash of empty string)  │  ← no code
│ storageRoot: (empty trie root)     │  ← no storage
└────────────────────────────────────┘
```

---

### 1.2 Contract Accounts

A contract account is created when someone deploys a smart contract to the blockchain. Unlike an EOA, it is **not** controlled by a private key — it is controlled entirely by its own code.

Key characteristics:

- **Controlled by code.** The contract's bytecode defines every rule about when and how its balance and storage can change. There is no private key; nobody "owns" a contract account in the traditional sense.
- **Has an ETH balance.** Contracts can receive, hold, and send ether.
- **Has a nonce.** Starts at 1 (for historical reasons) and increments each time the contract deploys another contract.
- **Has code.** The contract's compiled EVM bytecode is stored on-chain, permanently.
- **Has storage.** A persistent key-value store unique to this contract. This is where the contract's state lives (e.g., token balances, ownership records, configuration).
- **Cannot initiate transactions on its own.** A contract can send ether and call other contracts, but only when triggered by an incoming transaction from an EOA (directly or through a chain of calls).

```
Contract Account: 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48  (USDC token)
┌────────────────────────────────────────────────────────────────┐
│ nonce       : 1                                                │
│ balance     : 0 wei  (USDC contract holds minimal ETH)        │
│ codeHash    : 0x3c5f7b...  ← keccak256 of the contract's EVM  │
│                              bytecode                          │
│ storageRoot : 0x9a2d4f...  ← Merkle root of the contract's    │
│                              storage trie (holds all token     │
│                              balances, allowances, etc.)       │
└────────────────────────────────────────────────────────────────┘
```

---

### 1.3 Side-by-Side Comparison

| Property | EOA | Contract Account |
|---|---|---|
| Created by | Generating a private key | Deploying a transaction (to = empty) |
| Controlled by | Private key | Code (bytecode) |
| Has ETH balance | Yes | Yes |
| Has nonce | Yes (tx count) | Yes (contract deploy count) |
| Has code | No | Yes |
| Has storage | No | Yes |
| Can initiate txs | Yes | No (only reacts) |
| Can hold ETH | Yes | Yes |
| Existence requires on-chain tx | No (just a key pair) | Yes |

---

## 2. Account State Fields

Every account on Ethereum — whether EOA or contract — is defined by exactly four fields stored in the world state trie. Understanding these fields is essential to understanding how Ethereum tracks value and computation.

### `nonce`

A counter. For EOAs, it counts the number of transactions successfully sent from this account. For contract accounts, it counts the number of contracts this account has deployed.

- Starts at **0** for EOAs, **1** for contracts (by convention).
- Ethereum requires transactions to be processed in exact nonce order from a given sender. If your last sent transaction had nonce 5, your next must have nonce 6.
- The nonce is included in the signed transaction data, so it cannot be forged.

### `balance`

The amount of ether held by this account, always expressed in **wei** (the smallest unit, 10⁻¹⁸ ETH). This is a simple integer — there are no "sub-accounts" or denominations in the state; only wei.

- Increased by: receiving ETH via a value-transfer transaction, receiving ETH from a contract call, block rewards (for validators).
- Decreased by: sending ETH in a transaction, paying gas fees.

### `codeHash`

The Keccak-256 hash of the account's EVM bytecode.

- For **EOAs**: This is always the hash of the empty byte string: `keccak256("") = 0xc5d2460186f7233c927e7db2dcc703c0e500b653ca82273b7bfad8045d85a470`. This special value signals "no code."
- For **contract accounts**: This is the hash of the deployed bytecode. The bytecode itself is stored separately in the database and looked up by this hash. Once deployed, the codeHash (and thus the bytecode) is **immutable** — it cannot be changed.

### `storageRoot`

The root hash of the contract's **storage trie** — a Merkle Patricia Trie that maps 32-byte storage slot keys to 32-byte values.

- For **EOAs**: Always the empty trie root hash (no storage).
- For **contract accounts**: This root changes every time the contract writes to storage (`SSTORE` opcode). The root in the block's state is a cryptographic commitment to all of the contract's stored values at that block height.

---

## 3. What Is a Nonce?

The nonce is simple but critically important. It deserves its own section because it solves two distinct problems simultaneously.

### Problem 1: Replay Attacks

Without a nonce, an attacker could record your signed transaction (e.g., "send 1 ETH to Alice") and rebroadcast it repeatedly. The signature would still be valid each time, and you would keep losing ETH.

With a nonce, once a transaction with nonce `N` is included in a block, any future transaction reusing nonce `N` from the same sender is **immediately rejected** by every node. The nonce was consumed; it cannot be reused.

### Problem 2: Transaction Ordering

Ethereum guarantees that transactions from the same sender are processed **in nonce order**. You cannot skip a nonce.

- If you have sent transactions with nonces 0 through 9, your next transaction must use nonce 10.
- If a transaction with nonce 10 is in the mempool and you broadcast another with nonce 12, the nonce-12 transaction sits in the mempool and waits until nonce 10 and 11 are confirmed — regardless of gas price.
- This gives you a way to **cancel or replace** a stuck transaction: broadcast a new transaction with the same nonce but a higher gas price. The network accepts whichever gets mined first; the other becomes invalid.

### Fetching the Current Nonce

Before signing a transaction, your wallet queries a node for the current nonce:

```
JSON-RPC call:
eth_getTransactionCount("0xYourAddress", "pending")

Returns: 42  ← your next transaction should use nonce 42
```

The `"pending"` tag means "count all confirmed transactions plus any pending ones in the mempool" — important if you're sending multiple transactions in rapid succession.

---

## 4. What Is a Wallet?

This is one of the most commonly misunderstood concepts in Ethereum.

### A Wallet Does NOT Store ETH

Your ETH is **not** stored in MetaMask. It is not stored on your phone. It is not stored anywhere on your computer. Your ETH exists as a balance recorded in Ethereum's world state, which lives on thousands of nodes around the world.

What a wallet actually stores is your **private key** (or a seed phrase that can derive it). The wallet is a key management tool and a signing interface.

```
Common misconception:
  "I have 5 ETH in MetaMask"

Reality:
  "The Ethereum world state records that address 0xAbCd...
   has a balance of 5,000,000,000,000,000,000 wei.
   MetaMask holds the private key that can authorize
   transactions from that address."
```

If you lose your private key (or seed phrase), you lose access to your funds — but your ETH doesn't disappear. It remains in the world state forever, at your address, simply inaccessible to anyone (including you).

### What a Wallet Actually Does

1. **Key generation and storage:** Generates a secure random private key; stores it encrypted locally.
2. **Address derivation:** Derives your public key and Ethereum address from the private key.
3. **Transaction construction:** Builds the transaction object (fills in nonce, estimates gas, etc.).
4. **Transaction signing:** Hashes the transaction and signs it with your private key using ECDSA (secp256k1). This happens locally — your private key never leaves your device.
5. **Transaction broadcast:** Sends the signed transaction to an Ethereum node via JSON-RPC (`eth_sendRawTransaction`).
6. **Balance and history display:** Queries nodes for your address's balance, nonce, and transaction history.

### Types of Wallets

| Type | Examples | How private key is stored | Security |
|---|---|---|---|
| **Browser extension** | MetaMask, Rabby | Encrypted in browser storage | Moderate — exposed to browser risks |
| **Mobile wallet** | Rainbow, Trust Wallet | Encrypted on device | Moderate |
| **Hardware wallet** | Ledger, Trezor | On dedicated secure chip (never leaves device) | High — private key never touches internet-connected device |
| **Paper wallet** | Printed key/QR code | Written on paper | High offline, catastrophic if lost |
| **Smart contract wallet** | Gnosis Safe, Argent | No private key — controlled by smart contract logic | High — multisig, recovery options |

### HD Wallets and Seed Phrases

Modern wallets use **Hierarchical Deterministic (HD)** key derivation (BIP-32/BIP-39). A single **seed phrase** (12 or 24 random words) deterministically generates an entire tree of private keys and addresses. This means:

- One backup (the seed phrase) recovers all your accounts.
- Different accounts for different purposes can all be derived from the same seed.
- **The seed phrase is the master key.** Anyone who obtains it can derive every private key you have.

---

## 5. Transaction Anatomy

A transaction is a signed instruction from an EOA telling the network to perform some action. Here is every field in a fully-formed Ethereum transaction (EIP-1559 / type-2 format):

```
{
  "type":                 "0x2",            // Transaction type (2 = EIP-1559)
  "chainId":              "0x1",            // Chain ID — prevents replay across networks
  "nonce":                "0x2a",           // 42 in decimal — sender's tx count
  "from":                 "0xAlice...",     // Sender address (recovered from signature)
  "to":                   "0xBob...",       // Recipient: an EOA or contract address
                                            // Empty/null = contract deployment
  "value":                "0xDE0B6B3A7640000", // Amount of ETH to send (in wei)
                                               // = 1 ETH = 10^18 wei
  "data":                 "0x",            // Input data:
                                           //  - Empty for simple ETH transfers
                                           //  - ABI-encoded function call for contracts
                                           //  - Contract bytecode for deployments
  "gasLimit":             "0x5208",        // 21000 — max gas this tx may consume
                                           // 21,000 is the minimum for a plain transfer
  "maxFeePerGas":         "0x2540BE400",   // 10,000,000,000 = 10 gwei
                                           // Max total gas price sender will pay
  "maxPriorityFeePerGas": "0x3B9ACA00",    // 1,000,000,000 = 1 gwei
                                           // Max tip to the validator
  "accessList":           [],              // EIP-2930 access list (usually empty)
  "v":                    "0x1",           // Signature component (recovery id)
  "r":                    "0xabcd...",     // Signature component
  "s":                    "0xef12..."      // Signature component
}
```

### Field-by-Field Breakdown

#### `to`
- For **ETH transfer**: the recipient's address.
- For **contract call**: the contract's address.
- For **contract deployment**: omitted entirely (or set to `null`/`0x`). The network recognises the absence of `to` as a deployment transaction and creates a new contract account.

#### `value`
The amount of ETH (in wei) transferred to the `to` address along with this transaction. Can be 0 — many contract calls transfer no ETH. For a pure ETH send, this is the main payload.

#### `data`
Sometimes called `input`. This field carries arbitrary bytes:
- **Plain ETH transfer:** empty (`0x`).
- **Contract function call:** ABI-encoded function selector (first 4 bytes = keccak256 of the function signature) followed by encoded arguments.
- **Contract deployment:** the contract's compiled bytecode (plus any constructor arguments ABI-encoded at the end).

#### `gasLimit`
The maximum number of gas units this transaction is allowed to consume. If execution hits this limit before finishing, it reverts (as if it never happened), but all gas up to the limit is still charged. Setting it too low causes an "out of gas" revert; setting it too high wastes nothing (unused gas is refunded).

- Minimum for a plain ETH transfer: **21,000 gas**.
- Contract calls typically need 50,000 – 500,000+ gas depending on complexity.

#### `maxFeePerGas` and `maxPriorityFeePerGas` (EIP-1559)
- `maxPriorityFeePerGas` — the "tip" offered to the validator to incentivise inclusion. Higher tip = higher priority in the mempool.
- `maxFeePerGas` — the absolute ceiling on total gas price. Must be ≥ `baseFeePerGas + maxPriorityFeePerGas`.
- **Actual fee paid per gas unit:** `min(maxFeePerGas, baseFeePerGas + maxPriorityFeePerGas)`
- **Base fee is burned; priority fee goes to the validator.**

#### `v`, `r`, `s`
The ECDSA signature produced by signing the transaction hash with the sender's private key. Together they allow any node to cryptographically recover and verify the sender's address, as described in guide 02.

---

## 6. Transaction Types

Ethereum has evolved its transaction format through several iterations, each introduced by an EIP. Each type is backward-compatible — older types remain valid.

| Type | EIP | Name | Key feature |
|---|---|---|---|
| **Type 0** | — | Legacy (Frontier) | Original format; single `gasPrice` field; no chain ID in earlier versions |
| **Type 1** | EIP-2930 | Access List | Adds optional `accessList`; chain ID always included; same `gasPrice` model |
| **Type 2** | EIP-1559 | Fee Market | Replaces `gasPrice` with `maxFeePerGas` + `maxPriorityFeePerGas`; the current standard |
| **Type 3** | EIP-4844 | Blob | Adds `blobVersionedHashes` + `maxFeePerBlobGas`; carries blob data for L2 rollups |

### Type 0 — Legacy

```
{ nonce, gasPrice, gasLimit, to, value, data, v, r, s }
```

The original transaction format from Ethereum's launch. Uses a single `gasPrice` field (first-price auction). Still fully supported but not recommended for new development.

### Type 1 — Access List (EIP-2930)

```
{ chainId, nonce, gasPrice, gasLimit, to, value, data, accessList, v, r, s }
```

Adds the `accessList` — a list of addresses and storage slots the transaction will access. Pre-declaring access receives a gas discount (avoids the "cold access" surcharge introduced by EIP-2929). Rarely used directly but important for gas optimisation in some scenarios.

### Type 2 — EIP-1559 (the current default)

```
{ chainId, nonce, maxPriorityFeePerGas, maxFeePerGas, gasLimit, to, value, data, accessList, v, r, s }
```

The modern standard. Almost all wallets and dApps send type-2 transactions. Provides predictable fees, reduces overbidding, and introduces the base fee burn mechanism.

### Type 3 — Blob (EIP-4844)

```
{ chainId, nonce, maxPriorityFeePerGas, maxFeePerGas, gasLimit, to, value, data,
  accessList, maxFeePerBlobGas, blobVersionedHashes, v, r, s }
```

A specialised type used exclusively by Layer 2 rollup sequencers to post batch data to Ethereum. Carries references to "blobs" (up to 128 KB each, up to 6 per transaction) via `blobVersionedHashes`. Blobs exist in a separate fee market (`maxFeePerBlobGas`) and are pruned from full nodes after ~18 days. Not used in normal EOA-to-EOA transfers.

---

## 7. Transaction Lifecycle

Here is the complete journey of a transaction, from the moment it leaves your wallet to the moment it achieves finality:

```
┌─────────────────────────────────────────────────────────────────────┐
│  STEP 1: CONSTRUCTION (wallet)                                      │
│                                                                     │
│  Wallet fetches: current nonce, baseFeePerGas, suggested tip        │
│  User fills in: to, value, data (or dApp fills it automatically)   │
│  Wallet sets: gasLimit (estimated via eth_estimateGas)              │
│  Result: unsigned transaction object                                │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│  STEP 2: SIGNING (wallet, locally)                                  │
│                                                                     │
│  1. RLP-encode the transaction fields                               │
│  2. keccak256(rlp_encoded_tx) → 32-byte hash                       │
│  3. ECDSA sign hash with private key → (v, r, s)                   │
│  4. Attach (v, r, s) to the transaction                             │
│  Private key NEVER leaves the wallet                                │
└────────────────────────────────┬────────────────────────────────────┘
                                 │  eth_sendRawTransaction(signedTxHex)
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│  STEP 3: SUBMISSION & INITIAL VALIDATION (receiving node, e.g Besu) │
│                                                                     │
│  ✓ Valid RLP format?                                                │
│  ✓ Valid chain ID?                                                  │
│  ✓ Signature valid? → recover sender address                        │
│  ✓ Nonce = sender's current nonce?                                  │
│  ✓ Sender balance ≥ (gasLimit × maxFeePerGas) + value?             │
│  ✓ maxFeePerGas ≥ current baseFeePerGas?                           │
│  ✓ gasLimit ≥ 21,000?                                               │
│                                                                     │
│  Returns: transaction hash (txHash = keccak256(signed_tx_bytes))    │
└────────────────────────────────┬────────────────────────────────────┘
                                 │  passes all checks
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│  STEP 4: MEMPOOL                                                    │
│                                                                     │
│  Transaction added to node's local transaction pool                 │
│  Gossiped to peer nodes → spreads across the network               │
│  Sits here until a validator picks it up                            │
│  Priority: higher tip (maxPriorityFeePerGas) = picked sooner       │
└────────────────────────────────┬────────────────────────────────────┘
                                 │  validator selected for this slot
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│  STEP 5: BLOCK BUILDING (validator)                                 │
│                                                                     │
│  Validator selects transactions from its mempool                    │
│  Executes them in order against current state                       │
│  Tracks gas used, state changes, logs emitted                       │
│  Builds block with: transactionsRoot, stateRoot, receiptsRoot      │
│  Signs and broadcasts block to consensus network                    │
└────────────────────────────────┬────────────────────────────────────┘
                                 │  block propagated
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│  STEP 6: BLOCK VALIDATION (all other nodes)                         │
│                                                                     │
│  Each node independently:                                           │
│    - Verifies all transaction signatures                            │
│    - Re-executes all transactions                                   │
│    - Computes stateRoot, transactionsRoot, receiptsRoot            │
│    - Compares computed roots with block header roots                │
│  If all match → block accepted, transaction is confirmed            │
│  Transaction removed from mempool (included in chain)               │
└────────────────────────────────┬────────────────────────────────────┘
                                 │  subsequent blocks build on top
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│  STEP 7: FINALITY                                                   │
│                                                                     │
│  PoS (mainnet): ~13 minutes → checkpoint finality                  │
│  PoA (QBFT private net): immediate on block inclusion               │
│  Transaction is now permanently and irreversibly part of the chain  │
└─────────────────────────────────────────────────────────────────────┘
```

### What Happens If a Transaction Reverts?

If the EVM runs into an error during execution (e.g., an assertion fails, an out-of-gas error, the code calls `revert()`):

- All **state changes** from this transaction are rolled back (as if it never ran).
- However, the transaction is still **included in the block** and the **gas fee is still charged**.
- The receipt records `status: 0` (failure).
- The sender loses the gas but the state is unchanged (except for the nonce incrementing and gas being deducted).

---

## 8. Transaction Receipts

Once a transaction is executed (successfully or not), the network generates a **transaction receipt** — an immutable record of what happened. Receipts are stored in the block's `receiptsRoot` trie and can be queried via `eth_getTransactionReceipt`.

### Receipt Fields

```json
{
  "transactionHash":   "0xabc123...",     // The tx hash you received on submission
  "transactionIndex":  "0x2",            // Position of this tx in the block (3rd tx)
  "blockHash":         "0xdef456...",     // Hash of the block containing this tx
  "blockNumber":       "0x11D4A40",       // Block number (hex) = 18,500,672
  "from":              "0xAlice...",      // Sender (recovered from signature)
  "to":                "0xBob...",        // Recipient (null for deployments)
  "contractAddress":   null,             // If this was a deployment: the new contract's
                                         // address. Otherwise null.
  "cumulativeGasUsed": "0x8A04F",        // Total gas used in block up to and including
                                         // this transaction
  "gasUsed":           "0x5208",         // Gas consumed by THIS transaction (21,000)
  "effectiveGasPrice": "0x3B9ACA00",     // Actual price paid per gas unit (in wei)
  "status":            "0x1",            // 0x1 = success, 0x0 = failure (revert)
  "logs":              [...],            // Array of event logs emitted during execution
  "logsBloom":         "0x000000...",    // 256-byte Bloom filter of logs in this tx
  "type":              "0x2"             // Transaction type
}
```

### The `status` Field

| Value | Meaning |
|---|---|
| `"0x1"` (1) | Transaction executed successfully — all state changes applied |
| `"0x0"` (0) | Transaction reverted — all state changes rolled back, gas fee still charged |

> Note: Before the Byzantium hard fork (2017), receipts did not have a `status` field. Instead they had a `root` field (the state root after the transaction). Very old transactions have this format.

### `contractAddress`

If the transaction was a contract deployment (`to` was empty), this field contains the address of the newly created contract. This address is deterministically computed as:

```
contractAddress = keccak256(rlp([sender_address, sender_nonce]))[12:]
```

The last 20 bytes of the hash of the RLP-encoded sender address and nonce. Predictable before deployment (useful for pre-computing addresses), but unique per sender-nonce combination.

---

## 9. Events and Logs

Smart contracts can **emit events** during execution. Events are the primary mechanism by which contracts communicate results to the outside world — to frontends, indexers, and other off-chain systems.

### What Are Events?

An event is declared in Solidity like this:

```solidity
event Transfer(address indexed from, address indexed to, uint256 value);
```

When a contract emits an event, the EVM executes a `LOG` opcode family (`LOG0` through `LOG4`), which appends a **log entry** to the transaction's receipt.

### Log Structure

Each log entry contains:

```
{
  address: "0xTokenContract...",    // Which contract emitted this log
  topics:  [                        // Up to 4 indexed topics (32 bytes each)
    "0xddf252...",                  // Topic 0: keccak256("Transfer(address,address,uint256)")
                                    //   — always the event signature hash
    "0x000000...Alice",             // Topic 1: indexed `from` address
    "0x000000...Bob"                // Topic 2: indexed `to` address
  ],
  data: "0x000000...0DE0B6B3A7640000"  // Non-indexed data: the `value` (1 ETH in wei)
}
```

### Indexed vs Non-Indexed Parameters

| Property | Indexed | Non-Indexed |
|---|---|---|
| Storage | In `topics` array | In `data` field |
| Max count | 3 (+ topic 0 = event signature) | Unlimited |
| Searchable? | Yes — Bloom filter + direct query | No — must decode data field |
| Gas cost | Slightly more expensive | Cheaper |

### Why Events Matter

- **dApp frontends** subscribe to events via `eth_getLogs` or WebSocket subscriptions to update their UI in real time (e.g., show a transfer notification).
- **Indexing services** (The Graph, Dune Analytics) crawl logs to build queryable databases of on-chain activity.
- **Cheaper than storage:** Emitting an event is much cheaper (in gas) than writing to contract storage. For data that only needs to be accessible off-chain, events are preferred.
- **Audit trails:** Events create a permanent, queryable history of contract activity — ideal for financial applications.

### Querying Logs

```json
// JSON-RPC call to get all Transfer events from the USDC contract
{
  "method": "eth_getLogs",
  "params": [{
    "fromBlock": "0x11D4A40",
    "toBlock":   "latest",
    "address":   "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
    "topics":    ["0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef"]
  }]
}
```

The `logsBloom` field in the block header is a 256-byte **Bloom filter** encoding all the addresses and topics of all logs in the block. Light clients can query the Bloom filter to quickly determine if a block *might* contain a log they care about — without downloading all receipts. If the Bloom filter says no, the block definitely has no matching logs. If it says yes, download the receipts to confirm.

---

## 10. RLP Encoding

### What Is RLP?

**Recursive Length Prefix (RLP)** is Ethereum's custom binary serialization format. It converts structured data (nested lists and byte strings) into a flat byte sequence for hashing, signing, and network transmission.

Ethereum invented RLP rather than using existing formats (like JSON or Protocol Buffers) for a key reason: **simplicity and determinism**. RLP has a single canonical encoding for any given data structure. There is no ambiguity, no optional fields, no whitespace — the same data always encodes to the exact same bytes, everywhere, every time. This is essential when the output will be cryptographically hashed.

### How RLP Works (Simplified)

RLP encodes two kinds of items:

1. **Byte strings** (raw bytes or integers): prefixed with their length information.
2. **Lists** (ordered collections of items): prefixed with the total encoded length of their contents.

```
Examples of RLP encoding:

"dog" (string)
→ [0x83, 0x64, 0x6f, 0x67]
   0x83 = 0x80 + 3  (string of length 3)
   0x64 0x6f 0x67   ("d", "o", "g" in ASCII)

["cat", "dog"] (list)
→ [0xc8, 0x83, 0x63, 0x61, 0x74, 0x83, 0x64, 0x6f, 0x67]
   0xc8 = 0xc0 + 8  (list of total length 8)
   0x83 0x63 0x61 0x74  ("cat")
   0x83 0x64 0x6f 0x67  ("dog")
```

### RLP in Transaction Signing

When you sign an EIP-1559 transaction, the signing input is:

```
keccak256(0x02 || RLP([chainId, nonce, maxPriorityFeePerGas, maxFeePerGas,
                        gasLimit, to, value, data, accessList]))
```

Where `||` means byte concatenation and `0x02` is the transaction type prefix. The signature covers all fields, so changing any field invalidates the signature.

### RLP in Transaction Hashing

The transaction hash (what you see as the `txHash`) is:

```
txHash = keccak256(0x02 || RLP([chainId, nonce, maxPriorityFeePerGas, maxFeePerGas,
                                 gasLimit, to, value, data, accessList, v, r, s]))
```

This time, the signature `(v, r, s)` is included in the encoding. The transaction hash uniquely identifies a fully signed transaction.

---

## 11. The World State

### What Is the World State?

The **world state** is the complete snapshot of all Ethereum account states at a given block. It answers the question: "What is the exact balance, nonce, code, and storage of every account that has ever existed on Ethereum, right now?"

Conceptually it is a giant mapping:

```
world state = { address → account_state }

where account_state = { nonce, balance, codeHash, storageRoot }
```

At any block height, the world state encodes everything needed to validate the next transaction. It is not stored as a flat list — it is stored as a **Merkle Patricia Trie**.

### Merkle Patricia Trie

Ethereum uses a data structure called the **Merkle Patricia Trie (MPT)** — a combination of:

- **Patricia Trie (Radix Trie):** An efficient key-value lookup structure. Keys (Ethereum addresses, storage slots) are broken into nibbles (4-bit chunks) and encoded as a path through the trie. Lookup is O(key length), and the structure shares common prefixes to save space.
- **Merkle Tree properties:** Every node in the trie is identified by the hash of its contents. The root node's hash — the **root hash** — cryptographically commits to the entire contents of the trie. Changing any value changes its node's hash, which propagates up to change the root.

```
World State Trie (simplified)

                stateRoot
               0x9a2d4f...
                    │
          ┌─────────┴──────────┐
          │                    │
       Branch                Branch
      (prefix: 0x7)          (prefix: 0xA)
          │                    │
    ┌─────┴──────┐         ┌───┴──────────┐
    │            │         │              │
  Leaf          Leaf     Leaf           Leaf
0x71C7...    0x7b4f...  0xA0b8...    0xAbCd...
(Alice EOA)  (Bob EOA)  (USDC       (Contract Y)
                         Contract)
```

### Three State Tries per Block

Each block header contains three root hashes, each from a separate trie:

| Root | Trie Contents | What It Proves |
|---|---|---|
| `stateRoot` | World state — all accounts | "After all txs in this block, every account has exactly these values" |
| `transactionsRoot` | All transactions in this block | "This block contains exactly these transactions, in this order" |
| `receiptsRoot` | All transaction receipts | "The results of executing all transactions are exactly these" |

### Why the MPT Matters

**Compact proofs:** The MPT allows anyone to prove that a specific account has a specific balance (or that a specific transaction is included in a block) with a short **Merkle proof** — a list of sibling hashes up the trie path. This proof can be verified using only the root hash, without downloading the entire state. This is how:

- **Light clients** (mobile wallets) verify balances without a full node.
- **Cross-chain bridges** prove account state on one chain to a contract on another.
- **L2 rollups** prove their state transitions to the L1.

**State growth:** The world state only grows — accounts are never deleted from the trie (though they can be cleared to an "empty" state). As of 2025, Ethereum's state trie contains tens of millions of accounts and continues to grow. This is a known scalability challenge that Ethereum's Verge roadmap phase (Verkle Trees) aims to address.

---

## 12. Recap Checklist

After reading this guide, you should be able to confidently say:

- [ ] I can clearly explain the difference between an EOA and a contract account (private key vs. code, no storage vs. persistent storage)
- [ ] I know the four fields in every account's state: `nonce`, `balance`, `codeHash`, `storageRoot`, and what each means
- [ ] I understand the nonce: it is a sequential counter that prevents transaction replay and enforces ordering
- [ ] I understand that a wallet stores private keys, not ETH — ETH lives in the world state on thousands of nodes
- [ ] I can describe what every field in a type-2 (EIP-1559) transaction does
- [ ] I know the four transaction types (0/legacy, 1/access list, 2/EIP-1559, 3/blob) and what each adds
- [ ] I can describe the 7 stages of a transaction lifecycle from wallet signing to finality
- [ ] I understand that a reverted transaction still consumes gas and increments the nonce
- [ ] I know what a transaction receipt contains: status, gasUsed, logs, contractAddress, blockNumber
- [ ] I understand what events/logs are, why indexed parameters matter, and how dApps use them
- [ ] I understand that RLP is Ethereum's canonical binary serialization format used for signing and hashing
- [ ] I understand the world state as a mapping of address → account state, stored as a Merkle Patricia Trie
- [ ] I know that `stateRoot`, `transactionsRoot`, and `receiptsRoot` in a block header are MPT root hashes

---

## 13. Check Your Understanding

Try to answer these without looking back:

1. **Alice sends a transaction from her EOA with nonce 5. The transaction reverts during EVM execution. What is Alice's nonce after the transaction is included in a block? What happened to her ETH balance?**

2. **A developer tells you: "Our contract is upgradeable — we just modify the code after deployment." Is this possible? What Ethereum property prevents direct code modification, and what pattern do developers actually use to achieve upgradeability?**

3. **Your MetaMask shows a balance of 2.5 ETH. Your laptop is destroyed. You have your 12-word seed phrase written on paper. Walk through exactly what happens when you recover your wallet on a new device. Where was your ETH during the time your laptop was destroyed?**

4. **A transaction is sent with `gasLimit: 100,000` but only uses 45,000 gas before completing successfully. What happens to the remaining 55,000 gas units? What is the exact formula for computing the total ETH fee paid?**

5. **An ERC-20 token contract emits a `Transfer` event every time tokens move. A user wants to find all token transfers to their address across the entire history of the chain. What JSON-RPC method would they use, and which event field (`topics` vs `data`) would they filter on, and why?**

6. **Two transactions in the mempool both come from address 0xAlice. Tx A has nonce 10, tip 5 gwei. Tx B has nonce 11, tip 50 gwei. A validator wants to include the high-tip Tx B first to maximise revenue. Can they? What happens if they try to include Tx B without first including Tx A?**

---

## Next Up

**[07 — The EVM & Smart Contracts →](07_evm_and_smart_contracts.md)**

Now that you understand accounts and transactions deeply — the *inputs* to Ethereum's computation engine — the next guide opens the hood on the engine itself: the Ethereum Virtual Machine. We'll explore how it executes bytecode, what opcodes are, how gas metering works instruction by instruction, and how smart contracts go from Solidity source code to running on-chain.