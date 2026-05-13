# Bonsai Archive — Complete Learning Guide

> A self-contained explanation series that takes you from Ethereum basics all the way to
> understanding and reading the source code of Besu's Bonsai Archive feature.
>
> No deep prior knowledge of Besu is required. Basic familiarity with blockchain concepts
> (blocks, transactions, addresses) is assumed.

---

## Why This Guide Exists

Besu introduced a new storage format called **Bonsai Archive** (`X_BONSAI_ARCHIVE`).
To understand _why_ it exists and _how_ it works, you need to understand a chain of
concepts — from how Ethereum stores state, to how Besu's original storage strategies
work, to the specific problems Bonsai Archive was designed to solve.

This guide builds that understanding one layer at a time.

---

## Reading Order

Read the files in numeric order. Each file builds on the previous one.

```
01 → 02 → 03 → 04 → 05 → 06 → 07
```

---

## File Index

| # | File | What You Learn | Prerequisite Knowledge |
|---|------|---------------|----------------------|
| 01 | `01_ethereum_state_basics.md` | What the Ethereum world state is, what an account contains, why nodes store history, full node vs archive node | Basic blockchain concepts |
| 02 | `02_merkle_patricia_trie.md` | Merkle trees, Patricia tries, how Ethereum's MPT works, write amplification, the stateRoot | File 01 |
| 03 | `03_forest_vs_bonsai.md` | Besu's two original storage strategies (Forest and Bonsai), the flat database concept, trie log rollback, and why standard Bonsai cannot serve old historic state | Files 01–02 |
| 04 | `04_bonsai_deep_dive.md` | Bonsai internals — all five DB segments, partial vs full flat DB modes, the full block import pipeline, the world state update accumulator, state healing | File 03 |
| 05 | `05_trie_logs.md` | TrieLog structure, PathBasedValue (before/after), how rollback works step by step, the 512-block retention window, the TrieLogPruner, chain reorg handling | File 04 |
| 06 | `06_bonsai_archive.md` | The Bonsai Archive design — block-number-suffixed keys, four new DB segments, `getNearestBefore`, the BonsaiArchiver background worker, deleted-account tombstones, performance characteristics | File 05 |
| 07 | `07_code_walkthrough.md` | A traced call stack from `eth_getBalance` JSON-RPC handler all the way down to the RocksDB `seekForPrev`, plus the write path and async archiver path | File 06 |

---

## The Story in One Page

### The Problem

Ethereum nodes must answer queries like `eth_getBalance(address, blockNumber=1000000)`.
To do that, a node must know what the world state looked like at block 1,000,000 — not
just today.

### Forest Mode (old approach)

Besu's original **Forest** storage keeps every Merkle Patricia Trie node from every
block ever processed. Historic state is trivially available by following the old trie root.
But this consumes 1.5 TB+ of disk and makes current-state reads slow (8–12 random disk
hops through the trie).

### Standard Bonsai (2021)

**Bonsai** replaced Forest's trie-heavy approach with a **flat database**: one key-value
entry per account (`keccak(address) → RLP{nonce, balance, ...}`). Current-state reads
become a single database lookup. Disk usage drops to ~700 GB.

But Bonsai only keeps **one copy** of each account (the current value). Historic state is
reconstructed by replaying **Trie Logs** — compact per-block diffs that record before/after
values for every changed account. Trie Logs are only kept for the last ~512 blocks. Older
history is gone.

### Bonsai Archive (new)

**Bonsai Archive** extends the flat database with a simple key change:

```
Standard Bonsai key:   keccak(address)                ← 32 bytes
Bonsai Archive key:    keccak(address) ++ blockNumber  ← 40 bytes
```

Now one entry per account-per-block-it-changed can coexist. To find the state at any
historic block, Besu does a single `seekForPrev` (nearest-before) seek in RocksDB:

```
seekForPrev( keccak(address) ++ blockNumber )
→ finds the entry at the last block the account changed at or before blockNumber
```

Two database operations. Any block. Any time. Faster than Forest. Half the disk of Forest.

---

## Key Classes in the Codebase

| Class | Path | Role |
|-------|------|------|
| `BonsaiArchiveFlatDbStrategy` | `ethereum/core/.../bonsai/storage/flat/` | Read/write with block-numbered keys and `getNearestBefore` |
| `BonsaiArchiver` | `ethereum/core/.../bonsai/worldview/` | Background worker moving old state to archive segments |
| `BonsaiArchiveWorldStateProvider` | `ethereum/core/.../bonsai/` | Routes historic RPC queries to archive or TrieLog path |
| `BonsaiWorldStateKeyValueStorage` | `ethereum/core/.../bonsai/storage/` | Storage facade; exposes `getAccount`, `updater`, etc. |
| `TrieLogLayer` | `ethereum/core/.../pathbased/common/trielog/` | Per-block diff object (before + after for every changed item) |
| `TrieLogPruner` | `ethereum/core/.../pathbased/common/trielog/` | Deletes TrieLogs beyond the retention window |
| `KeyValueSegmentIdentifier` | `ethereum/core/.../storage/keyvalue/` | Defines all RocksDB column families including the 4 new archive/freezer ones |
| `BonsaiContext` | `ethereum/core/.../pathbased/common/` | Tiny wrapper carrying the block number for archive key construction |

---

## Configuration

Enable Bonsai Archive when starting a new Besu node:

```
besu --data-storage-format=X_BONSAI_ARCHIVE [other flags]
```

The `X_` prefix indicates this is experimental. You cannot migrate an existing Forest or
standard Bonsai node to Bonsai Archive without resyncing from scratch.

---

## Quick Reference: DB Segments

```
Segment                       Key format                          Present in
─────────────────────────────────────────────────────────────────────────────────────
ACCOUNT_INFO_STATE            keccak(addr)                        BONSAI + ARCHIVE
ACCOUNT_STORAGE_STORAGE       keccak(addr) ++ keccak(slot)        BONSAI + ARCHIVE
CODE_STORAGE                  keccak(code) or keccak(addr)        BONSAI + ARCHIVE
TRIE_BRANCH_STORAGE           trie path (+ metadata keys)         BONSAI + ARCHIVE
TRIE_LOG_STORAGE              block_hash                          BONSAI + ARCHIVE
─────────────────────────────────────────────────────────────────────────────────────
ACCOUNT_INFO_STATE_ARCHIVE    keccak(addr) ++ block_num           ARCHIVE only
ACCOUNT_STORAGE_ARCHIVE       keccak(addr)++keccak(slot)++blockN  ARCHIVE only
ACCOUNT_INFO_STATE_FREEZER    keccak(addr) ++ block_num           ARCHIVE only
ACCOUNT_STORAGE_FREEZER       keccak(addr)++keccak(slot)++blockN  ARCHIVE only
─────────────────────────────────────────────────────────────────────────────────────
```

---

## External References

- [Bonsai Archive feature wiki page](https://lf-hyperledger.atlassian.net/wiki/spaces/BESU/pages/22156895/Bonsai+archive+feature)
- Besu repository: `besu/ethereum/core/src/main/java/org/hyperledger/besu/ethereum/trie/pathbased/bonsai/`
