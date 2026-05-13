# Merkle Patricia Tries

> A beginner-friendly guide to the data structure that holds all of Ethereum's state.

---

## 1. What is a Hash?

A hash is a fixed-length fingerprint of any piece of data — change even one byte of the input and you get a completely different fingerprint.

```
"hello"  -->  [ SHA-256 ]  -->  2cf24dba5fb0a30e...
"hellO"  -->  [ SHA-256 ]  -->  9b71d224bd62f378...  (totally different!)
```

---

## 2. What is a Merkle Tree?

A **Merkle Tree** is a tree where every node contains the hash of its children. You start by hashing the raw data at the leaves, then repeatedly hash pairs of nodes up the tree until you reach a single value at the top: the **root hash**.

```
                      [ Root Hash ]
                      H(H_AB + H_CD)
                     /              \
             [ H_AB ]              [ H_CD ]
             H(H_A+H_B)            H(H_C+H_D)
            /         \            /         \
        [ H_A ]    [ H_B ]    [ H_C ]    [ H_D ]
        hash(A)    hash(B)    hash(C)    hash(D)
           |          |          |          |
        Data A     Data B     Data C     Data D
```

Each parent is just the hash of its two children concatenated together. The root hash is computed bottom-up: leaves first, then branches, then the root.

---

## 3. Why is a Merkle Tree Useful?

**Tamper-proof:** If anyone alters Data C, then H_C changes, which changes H_CD, which changes the Root Hash. The entire fingerprint of the tree is invalidated. You cannot silently edit any leaf.

**Efficient verification (Merkle proofs):** To prove that Data C is in the tree, you only need to provide H_D (its sibling) and H_AB (the other branch). A verifier can recompute the path upward and check that it arrives at the known Root Hash — without downloading the entire tree.

```
  Proving Data C exists:

  Known root: H(H_AB + H_CD)

  Provided proof path:
    1. hash(C)  -->  H_C
    2. H_D      -->  H_CD = H(H_C + H_D)   ✓ sibling supplied
    3. H_AB     -->  Root = H(H_AB + H_CD)  ✓ sibling supplied

  If the recomputed root matches the known root, proof is valid.
```

This means a light client with only the root hash can verify any individual piece of data in O(log n) steps instead of downloading everything.

---

## 4. What is a Patricia Trie (Prefix Tree)?

A **Patricia Trie** (also called a Radix Tree or Prefix Tree) is a tree where keys are stored character-by-character along paths from root to leaf. Nodes that share a common prefix share the same path.

Imagine storing three keys: `"cat"`, `"car"`, and `"dog"`:

```
          [ root ]
         /        \
      [c]          [d]
       |             |
      [a]           [o]
      / \             |
    [t]  [r]         [g]
     |    |           |
  "cat"  "car"      "dog"
```

The prefix `"ca"` is shared by both `"cat"` and `"car"`, so the tree does not store it twice. A Patricia Trie goes a step further and **compresses** long single-child chains into a single edge labelled with the whole substring:

```
      [ root ]
      /       \
  [ca]         [dog] --> value("dog")
  /    \
[t]    [r]
 |      |
value  value
("cat")("car")
```

This compression keeps the tree shallow and reduces wasted nodes when keys share long prefixes.

---

## 5. What is a Merkle Patricia Trie (MPT)?

A **Merkle Patricia Trie** is what you get when you bolt a Merkle Tree on top of a Patricia Trie:

- The **Patricia Trie** part gives you efficient key-value lookup using prefix-compressed paths.
- The **Merkle Tree** part gives you cryptographic integrity: every node stores the hash of its children, so the root hash is a fingerprint of all keys and values in the trie.

Ethereum's MPT uses **hex-encoded** keys (each nibble — 4 bits — is one step in the trie, giving a branching factor of 16) and three node types:

| Node type    | Description                                                            |
|-------------|------------------------------------------------------------------------|
| Leaf node    | Stores the remaining key suffix and the final value                    |
| Extension node | Stores a shared key prefix and a single child                        |
| Branch node  | Has up to 16 children (one per hex nibble 0–f) plus an optional value |

A simplified picture of an MPT with two accounts:

```
                    [ Branch node ]
                    hash = H(children)
                    /              \
         [ Extension ]          [ Extension ]
         prefix: "3a"           prefix: "7f"
              |                       |
         [ Leaf ]                [ Leaf ]
         key: "...c1"            key: "...82"
         value: Account A        value: Account B
```

Every node is identified by the hash of its encoded content. Parent nodes store the hashes of their children, so the root hash covers the entire trie recursively — exactly like a Merkle Tree.

---

## 6. How Ethereum Uses MPT for the State Trie

Ethereum keeps one global **state trie** that maps every account address to that account's data.

- **Key:** `keccak256(address)` — the 32-byte Keccak hash of the 20-byte address (using the hash avoids adversarial trie-depth attacks).
- **Value:** RLP-encoded account data: `(nonce, balance, storageRoot, codeHash)`.

```
  Key path through the trie:

  Address: 0xAbCd...1234
        |
        v
  keccak256(0xAbCd...1234)
  = 0x3f7a...e901  (32 bytes = 64 hex nibbles)
        |
        v
  Traverse 64 nibbles from root to leaf:
  root -> [3] -> [f] -> [7] -> [a] -> ... -> leaf node
                                              |
                                              v
                                     { nonce:    5,
                                       balance:  1.2 ETH,
                                       storageRoot: 0x...,
                                       codeHash:    0x... }
```

Every externally-owned account (EOA) and every smart contract account is a leaf somewhere in this trie.

---

## 7. How the stateRoot is Derived

Each node in the MPT is serialized (using RLP encoding) and then hashed with Keccak-256. Parent nodes embed the hashes of their children rather than the children themselves.

```
  Leaf node bytes  -->  keccak256()  -->  leaf_hash
  Branch node bytes
    (contains leaf_hash inside it)  -->  keccak256()  -->  branch_hash
  Root node bytes
    (contains branch_hash inside it)  -->  keccak256()  -->  stateRoot
```

The **stateRoot** is therefore a single 32-byte value that is a cryptographic commitment to **every account, every balance, every nonce, and every piece of contract storage** on the entire network at that block.

This stateRoot is embedded directly in the block header:

```
  Block Header
  +--------------------------+
  | parentHash               |
  | stateRoot   <---------+  |   <-- fingerprint of ALL state
  | transactionsRoot       |  |
  | receiptsRoot           |  |
  | ...                    |  |
  +--------------------------+  |
                                |
            keccak256(root node of state MPT)
```

Two nodes that disagree about any account's balance will compute a different stateRoot and therefore disagree about the block hash — consensus enforces agreement on the whole world state through this single hash.

---

## 8. The Problem with MPT: Write Amplification

Every time a single account's value changes (e.g. a balance update after a transfer), every node on the path from that leaf up to the root must be recomputed and rewritten, because a parent's hash depends on its children.

```
  Updating Account B's balance:

  Before:                        After:

  [ Root: H_old ]                [ Root: H_new ]   <-- rewritten
       |                               |
  [ Branch: H_old ]             [ Branch: H_new ]  <-- rewritten
       |                               |
  [ Extension: H_old ]         [ Extension: H_new ] <-- rewritten
       |                               |
  [ Leaf: old balance ]         [ Leaf: new balance ] <-- rewritten
```

If the trie has depth D, a single account update triggers D node rewrites. With 64 nibble levels and millions of accounts, a busy block that touches thousands of accounts rewrites a very large number of nodes — this is called **write amplification**.

This is one of the biggest performance bottlenecks in Ethereum clients, including Besu. It is why proposals like Verkle Tries are actively being researched as a replacement for the current MPT.

---

## 9. What is a Storage Trie?

Each smart contract account has its own private key-value store for its state variables (e.g. token balances in an ERC-20 contract, votes in a governance contract, etc.). This is called the **storage trie**.

```
  Global State Trie
  +-----------------+
  | EOA Account     |  <-- no storage trie (storageRoot = empty hash)
  +-----------------+
  | Contract Account|
  |  nonce: 1       |
  |  balance: 0     |
  |  codeHash: 0xAB |
  |  storageRoot ---+--------> [ Storage Trie ]
  +-----------------+          /       |       \
                            [slot 0] [slot 1] [slot 2]
                            value:5  value:0  value:99
```

- **Key:** `keccak256(slot_number)` — the 32-byte hash of the 256-bit storage slot index.
- **Value:** The RLP-encoded 256-bit value stored in that slot.

The `storageRoot` of a contract is the root hash of its storage trie, and it is stored inside the account's leaf node in the global state trie. This means the global stateRoot transitively commits to every contract's every storage slot.

---

## 10. Reading a Single Account's Balance

Nothing in Ethereum state is stored in a flat array you can index directly. Reading a balance means navigating a tree. Here is the full journey:

```
  Step 1: Start at the stateRoot (stored in the block header)
          stateRoot = 0x3f9a...

  Step 2: Look up the root node in the database by its hash
          DB[ 0x3f9a... ]  -->  Branch node (16 children)

  Step 3: Take the first nibble of keccak256(address)
          e.g. nibble = '7', follow child[7]
          DB[ child[7] hash ]  -->  Extension node, prefix "f4"

  Step 4: Skip the prefix, follow its child pointer
          DB[ child hash ]  -->  Branch node

  Step 5: Take the next nibble, follow that child...
          ... repeat for all 64 nibbles ...

  Step 6: Arrive at the Leaf node
          DB[ leaf hash ]  -->  Leaf { nonce, balance, storageRoot, codeHash }

  Step 7: Decode the RLP and read the balance field.
```

In the worst case this is around 8–9 actual node lookups (because branch nodes collapse many nibbles and extension nodes skip shared prefixes), but each lookup is a random read from the key-value store (LevelDB / RocksDB in Besu). On a busy node with a cold cache this adds up quickly.

This traversal cost is also why **snap sync** (Besu's fast sync strategy) downloads a flat snapshot of all accounts directly, bypassing the trie traversal entirely during initial sync, and only reconstructs the trie structure afterward.

---

## Summary

| Concept          | One-line recap                                                             |
|-----------------|-----------------------------------------------------------------------------|
| Hash             | Fixed-length fingerprint; any change produces a completely different output |
| Merkle Tree      | Tree of hashes; root commits to all leaves; enables compact proofs          |
| Patricia Trie    | Prefix-compressed key-value tree; efficient lookup and storage              |
| MPT              | Merkle + Patricia combined; Ethereum's core data structure                  |
| State Trie       | Global MPT mapping keccak(address) --> account data                         |
| stateRoot        | Root hash of the state trie; the fingerprint of all Ethereum state          |
| Storage Trie     | Per-contract MPT mapping keccak(slot) --> slot value                        |
| Write amplification | Single leaf update forces rewrite of every node on the path to root      |
| Balance lookup   | ~8–9 DB reads traversing root-to-leaf through the state trie                |