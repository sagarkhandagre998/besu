# 02 — Cryptography Basics: Hashing, Keys & Digital Signatures

> **Level:** Beginner | **Read time:** ~25 minutes
> **Prerequisites:** [01 — What Is Blockchain?](01_what_is_blockchain.md)

---

## Table of Contents

1. [Why Blockchain Needs Cryptography](#1-why-blockchain-needs-cryptography)
2. [Hash Functions](#2-hash-functions)
3. [Merkle Trees](#3-merkle-trees)
4. [Public Key Cryptography](#4-public-key-cryptography)
5. [Digital Signatures](#5-digital-signatures)
6. [Elliptic Curve Cryptography (secp256k1)](#6-elliptic-curve-cryptography-secp256k1)
7. [Putting It All Together: Sending an Ethereum Transaction](#7-putting-it-all-together-sending-an-ethereum-transaction)
8. [Common Terms Glossary](#8-common-terms-glossary)
9. [Recap Checklist](#9-recap-checklist)
10. [Check Your Understanding](#10-check-your-understanding)

---

## 1. Why Blockchain Needs Cryptography

Imagine a ledger that is shared with thousands of strangers around the world. Nobody is in charge. Anyone can try to write to it. How do you:

- **Prove** that a transaction was created by the person who claims to have sent it?
- **Guarantee** that historical records haven't been quietly changed?
- **Trust** the data without trusting any single person?

The answer to all three questions is **cryptography** — a set of mathematical tools that let computers make guarantees that are practically impossible to fake, even for nation-states with enormous computing power.

Blockchain relies on three cryptographic primitives:

| Primitive | What it does |
|---|---|
| **Hash functions** | Produce a fixed-size "fingerprint" of any data |
| **Public key cryptography** | Let you prove ownership without revealing a secret |
| **Digital signatures** | Prove a specific person authorized a specific action |

Understanding these three tools gives you a solid mental model for how blockchains work at a deep level.

---

## 2. Hash Functions

### The Fingerprint Analogy

Think of a hash function as a fingerprint machine. You can feed it *anything* — a single letter, a 10 GB video file, or the entire text of a transaction — and it always spits out a short, fixed-length string of characters.

- The same input **always** produces the same fingerprint.
- Even the tiniest change to the input produces a completely different fingerprint.
- You **cannot** reconstruct the original input just by looking at the fingerprint.

### SHA-256 Example

SHA-256 (Secure Hash Algorithm, 256-bit output) is one of the most widely used hash functions. Ethereum uses a variant called **Keccak-256** (sometimes loosely called SHA-3).

```
Input:  "Hello, blockchain!"
SHA-256 output:
  b45c4f3a9e2d1c8f6a0b7e5d2c4a1f9b3e8d6c2a5f0b3e7d1c9a4f6b2e8d5c1a

Input:  "Hello, blockchain." (just a period instead of !)
SHA-256 output:
  1a9f3c2d7b4e8a6f0c5d2e9b1a8f4c3e7d6b2a9f5c1e8d4b7a3f6c2d9e5b1a8f
```

*(The exact hex values above are illustrative; the key point is that the outputs look completely unrelated despite near-identical inputs.)*

### The 5 Key Properties

#### 1. Deterministic
The same input always produces the exact same output. If you hash "Alice sends 5 ETH to Bob" a million times, you always get the same hash. This makes hashes reliable as identifiers.

#### 2. Fixed-size Output
No matter how large or small the input, the output is always 256 bits (32 bytes) for SHA-256 and Keccak-256. A single character and the entire Wikipedia compressed archive produce outputs of the same length. This is essential for storing transaction and block identifiers efficiently.

#### 3. One-way (Pre-image Resistant)
You cannot reverse a hash. Given a hash output, there is no mathematical shortcut to figure out what input produced it. The only approach is brute force — try every possible input until you find one that matches. For a 256-bit hash, that means trying roughly 2²⁵⁶ combinations. For perspective, the number of atoms in the observable universe is about 2²⁶⁶. It is computationally impossible.

#### 4. Avalanche Effect
Change one bit of the input and roughly half of the output bits flip. There is no gradual transition — a tiny change causes an avalanche of differences in the output. This means you cannot nudge a hash toward a desired target; every attempt is essentially a fresh random output.

#### 5. Collision Resistant
A collision means two different inputs produce the same hash output. While collisions are theoretically possible (infinitely many inputs map to a finite set of outputs), finding one intentionally requires astronomical computing effort. For Keccak-256, no practical collision has ever been found.

### How Blockchain Uses Hashes to Chain Blocks

Each block contains the **hash of the previous block** in its header. This creates an unbreakable chain:

```
Block 1                Block 2                Block 3
┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│ prevHash: 0  │       │ prevHash:    │       │ prevHash:    │
│              │  ───▶ │  hash(B1)    │  ───▶ │  hash(B2)    │
│ data: [txs]  │       │ data: [txs]  │       │ data: [txs]  │
│              │       │              │       │              │
│ hash: h1     │       │ hash: h2     │       │ hash: h3     │
└──────────────┘       └──────────────┘       └──────────────┘
```

If an attacker tries to modify a transaction in Block 1, the hash of Block 1 changes. But Block 2 stores the *old* hash of Block 1 — so Block 2 now has an invalid parent hash. To cover up the change, the attacker must re-hash Block 2, which changes Block 2's hash, breaking Block 3... and so on forever. The entire chain from the tampered block onward must be recomputed and re-accepted by the network — an impossibility in practice.

---

## 3. Merkle Trees

### What Is a Merkle Tree?

A Merkle tree is a **tree of hashes**. Its purpose: efficiently prove that a particular transaction is included in a block, without having to process every transaction in the block.

### Structure

All transactions in a block are the **leaf nodes**. Their hashes are combined pairwise, and those combined hashes are themselves hashed — all the way up to a single **root hash** (the Merkle root).

```
                    ┌─────────────────┐
                    │   Merkle Root   │
                    │  Hash(H12+H34)  │
                    └────────┬────────┘
                             │
            ┌────────────────┴────────────────┐
            │                                 │
     ┌──────┴──────┐                   ┌──────┴──────┐
     │     H12     │                   │     H34     │
     │ Hash(H1+H2) │                   │ Hash(H3+H4) │
     └──────┬──────┘                   └──────┬──────┘
            │                                 │
     ┌──────┴──────┐                   ┌──────┴──────┐
     │             │                   │             │
  ┌──┴──┐       ┌──┴──┐            ┌──┴──┐       ┌──┴──┐
  │ H1  │       │ H2  │            │ H3  │       │ H4  │
  │Hash │       │Hash │            │Hash │       │Hash │
  │(Tx1)│       │(Tx2)│            │(Tx3)│       │(Tx4)│
  └─────┘       └─────┘            └─────┘       └─────┘
    Tx1           Tx2                Tx3           Tx4
 (leaf node)  (leaf node)        (leaf node)  (leaf node)
```

The Merkle root is stored in the **block header** as `transactionsRoot`.

### Why Is This Useful?

**Efficient verification (Merkle Proofs):** Suppose you have a block with 10,000 transactions. You want to verify that Tx3 is in the block. Without a Merkle tree, you'd need to download and check all 10,000 transactions. With a Merkle tree, you only need:
- The hash of Tx3
- A small "proof path" of sibling hashes up the tree (H4, H12, and the root)

You can re-compute the root yourself and check it matches the block header's `transactionsRoot`. This is how **light clients** (like wallets on mobile phones) can verify transactions without downloading the whole blockchain.

**Tamper detection:** Changing any transaction changes its leaf hash, which cascades up and changes the root. The block header's `transactionsRoot` would no longer match — immediately detectable.

**Ethereum also uses Merkle Patricia Tries** (a more complex variant) for the world state, transaction receipts, and storage, allowing efficient proofs for all kinds of data.

---

## 4. Public Key Cryptography

### The Core Concept

Public key cryptography (also called asymmetric cryptography) gives each participant two mathematically linked keys:

| Key | Also called | Can be shared? | Purpose |
|---|---|---|---|
| **Private key** | Secret key | **Never** — keep it secret | Prove ownership; sign transactions |
| **Public key** | Verification key | Yes — share freely | Let others verify your signatures |

The magic: you can derive the public key from the private key, but you **cannot** derive the private key from the public key. It's a one-way relationship.

### The Lockbox Analogy

Imagine a public key as an open padlock that you freely distribute to anyone. Your private key is the key that opens it.

- If someone wants to send you a secret message, they lock it with your open padlock (encrypt with your public key). Only your private key can open it.
- If you want to prove a message came from you, you stamp it with your private key (sign it). Anyone with your public key can verify that stamp — but only you could have made it.

In Ethereum, we primarily use the **signing** use case, not the encryption use case.

### How an Ethereum Address Is Derived

An Ethereum address is not randomly generated — it is **mathematically derived** from your private key through a deterministic process:

```
Step 1: Start with a random 256-bit private key
        ┌────────────────────────────────────────────────────────┐
        │  0x4c0883a69102937d6231471b5dbb6e538eba2ef... (32 bytes)│
        └────────────────────────────────────────────────────────┘
                              │
                              │  ECDSA (secp256k1 curve)
                              ▼
Step 2: Generate the public key (uncompressed: 04 prefix + x + y coords)
        ┌────────────────────────────────────────────────────────┐
        │  04 + 64-byte x-coordinate + 64-byte y-coordinate      │
        │  (65 bytes total, 130 hex characters)                  │
        └────────────────────────────────────────────────────────┘
                              │
                              │  Keccak-256 hash of the public key
                              │  (strip the 04 prefix first — hash 64 bytes)
                              ▼
Step 3: Take the Keccak-256 hash of the public key (64 bytes)
        ┌────────────────────────────────────────────────────────┐
        │  full 32-byte Keccak-256 hash output                   │
        └────────────────────────────────────────────────────────┘
                              │
                              │  Take only the LAST 20 bytes (40 hex chars)
                              ▼
Step 4: Ethereum Address
        ┌────────────────────────────────────────────────────────┐
        │  0x71C7656EC7ab88b098defB751B7401B5f6d8976F             │
        └────────────────────────────────────────────────────────┘
```

Key observations:
- The address is **deterministic** — the same private key always yields the same address.
- The address is **20 bytes (40 hex characters)**, prefixed with `0x`.
- You **cannot** reverse this process to find the private key from an address (two one-way functions are applied).
- **Losing your private key means losing access to your funds forever** — there is no password recovery.

---

## 5. Digital Signatures

### What Is a Digital Signature?

A digital signature is a mathematical proof that:
1. A specific private key was used to authorize a specific piece of data.
2. The data has not been changed since it was signed.

It's the equivalent of a handwritten signature on a legal document — except it's mathematically impossible to forge (unlike handwritten signatures, which can be faked).

### How Signing Works

When you submit an Ethereum transaction, your wallet performs these steps behind the scenes:

```
Step 1: Assemble the transaction data
        { from, to, value, data, nonce, gasLimit, gasPrice }

Step 2: RLP-encode the transaction
        (Serialize it into bytes — see the glossary for RLP)

Step 3: Keccak-256 hash the encoded transaction
        txHash = keccak256(rlp_encoded_tx)
        → a 32-byte hash representing this specific transaction

Step 4: Sign the hash with your private key using ECDSA
        (v, r, s) = ECDSA_sign(txHash, privateKey)

Step 5: Attach (v, r, s) to the transaction and broadcast it
```

The three output values `(v, r, s)` are the signature:

| Value | Size | Meaning |
|---|---|---|
| `r` | 32 bytes | First part of the ECDSA signature (x-coordinate related) |
| `s` | 32 bytes | Second part of the ECDSA signature |
| `v` | 1 byte | Recovery identifier — helps reconstruct the public key from the signature |

### How Verification Works

Every Ethereum node that receives your transaction does the following **without ever seeing your private key**:

```
Step 1: Receive the transaction + (v, r, s)

Step 2: Re-hash the transaction data
        txHash = keccak256(rlp_encoded_tx)

Step 3: Use (v, r, s) + txHash to mathematically recover the public key
        recoveredPublicKey = ECDSA_recover(txHash, v, r, s)

Step 4: Derive the Ethereum address from the recovered public key
        recoveredAddress = last20bytes(keccak256(recoveredPublicKey))

Step 5: Check: does recoveredAddress == transaction.from?
        ✅ Yes → the signature is valid; this account authorized this transaction
        ❌ No  → reject the transaction
```

### Why Digital Signatures Are So Powerful

- **Non-repudiation:** You cannot deny sending a transaction — only your private key could have produced that signature for that specific data. Unlike "someone used my password," you cannot claim someone forged your signature.
- **Integrity:** If even one byte of the transaction changes after signing, the hash changes, and signature verification fails. You know the data was not tampered with.
- **No passwords needed:** There is no server storing your password that could be hacked. The math itself is the proof. No account recovery, no password reset — you either have the private key or you don't.
- **Trustless verification:** Any node in the world can verify your signature independently, with no need to contact a central authority.

---

## 6. Elliptic Curve Cryptography (secp256k1)

Ethereum uses a specific mathematical system called **Elliptic Curve Cryptography (ECC)** with the curve named **secp256k1**.

### What Is an Elliptic Curve?

An elliptic curve is a type of mathematical curve defined by the equation:

```
y² = x³ + ax + b  (over a finite field)

For secp256k1:  y² = x³ + 7
```

Points on this curve have a special mathematical property: you can define an operation called **point multiplication** where multiplying a known starting point (`G`, the generator point) by a number (`k`) gives you a new point on the curve.

```
PublicKey = k × G

Where:
  k  = private key (a very large random integer)
  G  = generator point (a fixed, known point on the curve — baked into the standard)
  ×  = elliptic curve point multiplication (not regular multiplication!)
```

### Why Is This Secure?

Multiplication is easy to compute (`k × G` takes milliseconds). The reverse — given the result and `G`, find `k` — is the **Elliptic Curve Discrete Logarithm Problem (ECDLP)**. No efficient algorithm exists to solve it. With a 256-bit key space and the secp256k1 parameters, brute-forcing it would take longer than the age of the universe.

### Why secp256k1?

- Chosen by Bitcoin's creator Satoshi Nakamoto; Ethereum adopted it for compatibility.
- It has no known weaknesses (some other curves have raised concerns about potential backdoors — particularly NIST curves).
- Allows efficient computation and compact signatures.
- The security level is approximately 128 bits (meaning ~2¹²⁸ operations to break).

---

## 7. Putting It All Together: Sending an Ethereum Transaction

Here is the complete flow of what happens cryptographically when you press "Send" in MetaMask or any Ethereum wallet:

```
┌─────────────────────────────────────────────────────────────────┐
│                    YOU (wallet software)                        │
│                                                                 │
│  1. CREATE TRANSACTION                                          │
│     { to: 0xBob, value: 1 ETH, nonce: 7, gasLimit: 21000, ... }│
│                                                                 │
│  2. RLP ENCODE                                                  │
│     Serialize the transaction into a binary byte sequence       │
│                                                                 │
│  3. HASH                                                        │
│     txHash = keccak256(rlp_encoded_transaction)                 │
│                                                                 │
│  4. SIGN                                                        │
│     (v, r, s) = ECDSA_sign(txHash, your_private_key)           │
│                                                                 │
│  5. BROADCAST                                                   │
│     Send { transaction_data, v, r, s } to an Ethereum node     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │  over the internet (p2p network)
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              ETHEREUM NODE (e.g., Besu)                         │
│                                                                 │
│  6. RECEIVE & VERIFY SIGNATURE                                  │
│     - Re-hash the transaction data                              │
│     - ECDSA_recover(txHash, v, r, s) → recoveredAddress        │
│     - Check: recoveredAddress == transaction.from ✅            │
│                                                                 │
│  7. VALIDATE TRANSACTION                                        │
│     - Nonce is correct (= account's current nonce)?            │
│     - Sender has enough ETH to pay gas + value?                 │
│     - Gas limit is sufficient?                                  │
│                                                                 │
│  8. ADD TO MEMPOOL                                              │
│     Transaction waits here until a validator picks it up        │
│                                                                 │
│  9. INCLUDED IN BLOCK                                           │
│     Validator assembles a block, includes this transaction,     │
│     computes transactionsRoot (Merkle root of all txs)         │
│                                                                 │
│  10. BLOCK PROPAGATED & ACCEPTED                                │
│      Other nodes verify the block header hashes chain correctly │
│      stateRoot matches re-executed state transitions            │
│                                                                 │
│  11. TRANSACTION FINALIZED                                      │
│      Your transaction is now part of the permanent chain        │
└─────────────────────────────────────────────────────────────────┘
```

Every node on the network independently performs steps 6–10. There is no central server, no trusted intermediary, no account login. Pure mathematics guarantees the result.

---

## 8. Common Terms Glossary

| Term | What It Means |
|---|---|
| **keccak256** | The specific hash function used by Ethereum. Produces a 32-byte (256-bit) output. Similar to (but different from) the official SHA-3 standard. Always used for hashing transaction data, addresses, and state. |
| **secp256k1** | The name of the specific elliptic curve used by Ethereum (and Bitcoin). The "256" refers to the 256-bit field size; "k1" refers to the Koblitz curve type. Defines the math behind key generation and signing. |
| **ECDSA** | Elliptic Curve Digital Signature Algorithm. The specific signing algorithm Ethereum uses. Takes a private key and a message hash, outputs `(v, r, s)`. |
| **(v, r, s)** | The three components of an Ethereum signature. `r` and `s` are the mathematical signature values; `v` is the recovery identifier (27 or 28 for legacy; 0 or 1 for modern) that allows nodes to recover the public key without storing it. |
| **RLP** | Recursive Length Prefix. Ethereum's custom binary serialization format. Converts structured data (like a transaction object) into a byte sequence for hashing and network transmission. Compact and unambiguous. |
| **Merkle Trie** | The data structure Ethereum uses to store its world state, transactions, and receipts. A combination of a Merkle tree (hash chaining) and a Patricia trie (efficient key-value lookup). Allows cryptographic proofs of any stored value. |
| **nonce** | Overloaded term in Ethereum: (1) In transactions, it's a per-account counter that increments with each sent transaction — prevents replay attacks. (2) In Proof of Work, it's a number miners iterate through to find a valid block hash. |

---

## 9. Recap Checklist

After reading this guide, you should be able to confidently say:

- [ ] I understand that hash functions produce a fixed-size "fingerprint" of any data
- [ ] I can name the 5 properties of a good hash function: deterministic, fixed-size, one-way, avalanche effect, collision resistant
- [ ] I understand how hashing blocks together (each block contains the hash of the previous block) makes tampering detectable
- [ ] I understand what a Merkle tree is and why it allows efficient transaction verification
- [ ] I understand the difference between a private key (secret, never share) and a public key (safe to share)
- [ ] I can describe how an Ethereum address is derived from a private key (ECDSA → public key → keccak256 → last 20 bytes)
- [ ] I understand that a digital signature proves who authorized a transaction, without revealing the private key
- [ ] I know that `(v, r, s)` are the three components of an Ethereum signature
- [ ] I understand at a high level what secp256k1 is and why it's secure
- [ ] I can describe the end-to-end flow from creating a transaction to it being accepted by the network

---

## 10. Check Your Understanding

Test yourself — try to answer these without re-reading:

1. **Why can't an attacker quietly edit a transaction in block 100 without detection?** Walk through what happens to the hashes.

2. **You receive a file and its SHA-256 hash. How do you verify the file hasn't been corrupted?** What property of hash functions makes this work?

3. **If an Ethereum address is derived from the public key, and the public key is derived from the private key, why is the address considered safe to share publicly?**

4. **What does the `v` component in an Ethereum signature `(v, r, s)` allow nodes to do?** Why is this necessary?

5. **A transaction is sent from address 0xAlice with nonce 5. An attacker copies the exact same signed transaction and tries to rebroadcast it 10 minutes later. Why does this not work?**

6. **What is the difference between a Merkle tree and simply hashing all transactions together into one hash?** What does the tree structure enable that a single hash does not?

---

## Next Up

**[03 — How Blockchain Works Internally →](03_how_blockchain_works.md)**

In the next guide, we'll use everything we just learned about cryptography and apply it to understanding the full internal structure of a blockchain: block headers, the transaction lifecycle, the mempool, forks, genesis blocks, and how nodes validate everything.