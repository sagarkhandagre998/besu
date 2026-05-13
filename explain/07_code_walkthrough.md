# Code Walkthrough: Tracing `eth_getBalance` Through Bonsai Archive

> A line-by-line journey through the Besu source code, following a single historic
> `eth_getBalance` RPC call from the JSON-RPC handler down to the RocksDB seek.

---

## Table of Contents

1. [The Query We Are Tracing](#1-the-query-we-are-tracing)
2. [Layer Map: Where Each Class Lives](#2-layer-map-where-each-class-lives)
3. [Step 1 — JSON-RPC Entry Point](#3-step-1--json-rpc-entry-point)
4. [Step 2 — WorldStateQueryParams: Packaging the Request](#4-step-2--worldstatequeryparams-packaging-the-request)
5. [Step 3 — BonsaiArchiveWorldStateProvider.getWorldState()](#5-step-3--bonsaiarchiveworldstateprovidergetworldstate)
6. [Step 4 — The Decision Fork: TrieLog vs Archive](#6-step-4--the-decision-fork-trielog-vs-archive)
7. [Step 5 — rollMutableArchiveStateToBlockHash()](#7-step-5--rollmutablearchivestatetoblockhash)
8. [Step 6 — Reading the Account: BonsaiWorldStateKeyValueStorage.getAccount()](#8-step-6--reading-the-account-bonsaiworldstatekeyvaluestoragegetaccount)
9. [Step 7 — BonsaiArchiveFlatDbStrategy.getFlatAccount()](#9-step-7--bonsaiarchiveflatdbstrategyflataccount)
10. [Step 8 — getStateArchiveContextForRead()](#10-step-8--getstatearchivecontextforread)
11. [Step 9 — calculateArchiveKeyWithMaxSuffix()](#11-step-9--calculatearchivekeywithmaxsuffix)
12. [Step 10 — getNearestBefore() in RocksDB](#12-step-10--getnearestbefore-in-rocksdb)
13. [Step 11 — Prefix Validation and Deleted-Value Filtering](#13-step-11--prefix-validation-and-deleted-value-filtering)
14. [Step 12 — Returning the Balance](#14-step-12--returning-the-balance)
15. [The Block Import Path: How Archive Entries Get Written](#15-the-block-import-path-how-archive-entries-get-written)
16. [The Archiver Path: Moving Old State Asynchronously](#16-the-archiver-path-moving-old-state-asynchronously)
17. [Key Data Structures Quick Reference](#17-key-data-structures-quick-reference)
18. [Full Call Stack Summary](#18-full-call-stack-summary)

---

## 1. The Query We Are Tracing

Imagine a client sends this JSON-RPC request to a Besu node running in
`X_BONSAI_ARCHIVE` mode:

```json
{
  "jsonrpc": "2.0",
  "method": "eth_getBalance",
  "params": [
    "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045",
    "0xF4240"
  ],
  "id": 1
}
```

- **Address**: `0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045` (vitalik.eth, as an example)
- **Block**: `0xF4240` = decimal **1,000,000** (block one million on mainnet)
- **Current chain head**: block **20,000,000**
- **Distance from head**: 19,000,000 blocks — **far beyond** the 512-block TrieLog window

This is the most interesting case: a query that can only be answered by the archive flat
DB, never by TrieLog rollback.

---

## 2. Layer Map: Where Each Class Lives

Before we dive in, here is a map of every class we will visit and its location in the
project:

```
besu/ethereum/
│
├── api/src/main/java/.../jsonrpc/internal/methods/
│   └── EthGetBalance.java                         ← JSON-RPC handler (Step 1)
│
├── core/src/main/java/.../trie/pathbased/
│   │
│   ├── common/provider/
│   │   ├── WorldStateQueryParams.java             ← Query packaging (Step 2)
│   │   └── PathBasedWorldStateProvider.java       ← Base provider
│   │
│   ├── bonsai/
│   │   ├── BonsaiArchiveWorldStateProvider.java   ← Archive routing (Steps 3–5)
│   │   ├── BonsaiWorldStateProvider.java          ← Standard Bonsai provider
│   │   │
│   │   ├── storage/
│   │   │   ├── BonsaiWorldStateKeyValueStorage.java ← Storage facade (Step 6)
│   │   │   │
│   │   │   └── flat/
│   │   │       ├── BonsaiFlatDbStrategy.java       ← Abstract flat DB ops
│   │   │       ├── BonsaiArchiveFlatDbStrategy.java← Archive read/write (Steps 7–13)
│   │   │       └── BonsaiFullFlatDbStrategy.java   ← Standard Bonsai flat ops
│   │   │
│   │   └── worldview/
│   │       ├── BonsaiArchiver.java                ← Background archiver (Section 16)
│   │       └── BonsaiWorldState.java              ← Live world state object
│   │
│   └── common/
│       ├── BonsaiContext.java                     ← Block number context wrapper
│       └── storage/flat/FlatDbStrategy.java       ← Base strategy interface
│
└── core/src/main/java/.../storage/keyvalue/
    └── KeyValueSegmentIdentifier.java             ← DB segment definitions
```

---

## 3. Step 1 — JSON-RPC Entry Point

**File**: `ethereum/api/src/main/java/org/hyperledger/besu/ethereum/api/jsonrpc/internal/methods/EthGetBalance.java`

The RPC method handler is registered as `"eth_getBalance"`. Its `response()` method
is called for every incoming request:

```java
// Simplified — actual code is similar
@Override
public JsonRpcResponse response(final JsonRpcRequestContext requestContext) {
    final Address address = requestContext.getRequiredParameter(0, Address.class);
    final BlockParameter blockParameter =
        requestContext.getRequiredParameter(1, BlockParameter.class);

    // Resolve the block parameter to a concrete BlockHeader
    // "latest", "safe", "finalized", or a hex block number like "0xF4240"
    final Optional<BlockHeader> maybeBlockHeader =
        blockchainQueries.getBlockHeader(blockParameter);

    if (maybeBlockHeader.isEmpty()) {
        return new JsonRpcErrorResponse(requestContext.getRequest().getId(), RpcErrorType.BLOCK_NOT_FOUND);
    }

    // Ask blockchainQueries for the balance at this block
    final Optional<Wei> balance = blockchainQueries.accountBalance(address, maybeBlockHeader.get());

    return balance
        .map(b -> new JsonRpcSuccessResponse(
            requestContext.getRequest().getId(),
            b.toShortHexString()))
        .orElse(new JsonRpcErrorResponse(...));
}
```

`blockchainQueries.accountBalance()` calls into the world state layer:

```java
// BlockchainQueries.java (simplified)
public Optional<Wei> accountBalance(final Address address, final BlockHeader blockHeader) {
    // Build query parameters for the world state provider
    WorldStateQueryParams queryParams = WorldStateQueryParams.withBlockHeader(blockHeader);

    return worldStateArchive
        .getWorldState(queryParams)              // <-- enters BonsaiArchiveWorldStateProvider
        .map(ws -> ws.get(address))             // get the Account object
        .map(Account::getBalance);              // extract the Wei balance
}
```

At this point we have:
- `address` = `0xd8dA...96045`
- `blockHeader` = the header for block 1,000,000 (retrieved from the blockchain DB)
- We are about to call `getWorldState()` on the archive provider

---

## 4. Step 2 — WorldStateQueryParams: Packaging the Request

**File**: `ethereum/core/src/main/java/org/hyperledger/besu/ethereum/trie/pathbased/common/provider/WorldStateQueryParams.java`

`WorldStateQueryParams` is a simple value object that carries everything the world state
provider needs to know about what is being requested:

```java
public class WorldStateQueryParams {
    private final BlockHeader blockHeader;
    private final boolean shouldWorldStateUpdateHead;  // true only for block import
    private final boolean shouldPersistState;          // true only when committing
    // ... other flags

    // Factory for a read-only historic query
    public static WorldStateQueryParams withBlockHeader(final BlockHeader blockHeader) {
        return new WorldStateQueryParams(
            blockHeader,
            false,  // shouldWorldStateUpdateHead = false (not importing a block)
            false   // shouldPersistState = false (read only)
        );
    }

    public BlockHeader getBlockHeader() { return blockHeader; }
    public boolean shouldWorldStateUpdateHead() { return shouldWorldStateUpdateHead; }
    // ...
}
```

For our query:
- `blockHeader` = block 1,000,000's header
- `shouldWorldStateUpdateHead` = **false** (we are reading, not importing)
- `shouldPersistState` = **false**

The `shouldWorldStateUpdateHead` flag is the first branch point in the provider.

---

## 5. Step 3 — BonsaiArchiveWorldStateProvider.getWorldState()

**File**: `ethereum/core/src/main/java/org/hyperledger/besu/ethereum/trie/pathbased/bonsai/BonsaiArchiveWorldStateProvider.java`

This is the heart of the archive routing logic. Let us read the actual source carefully:

```java
@Override
public Optional<MutableWorldState> getWorldState(final WorldStateQueryParams queryParams) {

    // Branch 1: Is this a block import (head update)?
    if (queryParams.shouldWorldStateUpdateHead()) {
        // Standard path: execute the block, update the head state, verify stateRoot
        return getFullWorldState(queryParams);
    }

    // Branch 2: Historic read path
    // Compare the requested block's age against the TrieLog window
    final BlockHeader chainHeadBlockHeader = blockchain.getChainHeadHeader();

    if (chainHeadBlockHeader.getNumber() - queryParams.getBlockHeader().getNumber()
            >= trieLogManager.getMaxLayersToLoad()) {

        // The requested block is BEYOND the TrieLog window
        // (20,000,000 - 1,000,000 = 19,000,000 >= 512)
        // → Use the archive flat DB path

        LOG.debug(
            "Returning archive state without verifying state root {}",
            trieLogManager.getMaxLayersToLoad());

        return cachedWorldStorageManager
            .getWorldState(chainHeadBlockHeader.getHash())   // Step A: get head world state
            .map(MutableWorldState::disableTrie)             // Step B: skip trie verification
            .flatMap(
                worldState ->
                    rollMutableArchiveStateToBlockHash(      // Step C: set block context
                        (PathBasedWorldState) worldState,
                        queryParams.getBlockHeader().getBlockHash()))
            .map(MutableWorldState::freezeStorage);          // Step D: make read-only
    }

    // The requested block is WITHIN the TrieLog window
    // (e.g., block 19,999,600 — only 400 blocks behind head)
    // → Use the standard Bonsai TrieLog rollback path
    return super.getWorldState(queryParams);
}
```

### Evaluating Our Query

- `chainHeadBlockHeader.getNumber()` = **20,000,000**
- `queryParams.getBlockHeader().getNumber()` = **1,000,000**
- Difference = **19,000,000**
- `trieLogManager.getMaxLayersToLoad()` = **512** (default)
- `19,000,000 >= 512` → **true** → archive path

We take the archive branch. Let us follow each step A–D.

### Step A: `cachedWorldStorageManager.getWorldState(chainHeadBlockHeader.getHash())`

This retrieves the **current chain-head world state** from the cache. The cache holds a
`BonsaiWorldState` object pointed at the current head (block 20,000,000). Its underlying
storage is the live `BonsaiWorldStateKeyValueStorage` (the persistent flat DB).

Why the **head** world state? Because:
1. We will not roll back via TrieLogs (they don't go back 19M blocks).
2. We will instead just **change the block number context** stored in the world state,
   so that all flat DB reads use the target block number as their key suffix.
3. The actual account data is in the archive segments, not derived from the head state.

### Step B: `.map(MutableWorldState::disableTrie)`

```java
// MutableWorldState interface — implemented by PathBasedWorldState
default MutableWorldState disableTrie() {
    // Sets a flag that bypasses all trie operations:
    // - No stateRoot computation
    // - No MPT node reads/writes
    // - Flat DB only
    this.worldStateConfig = worldStateConfig.withTrieDisabled();
    return this;
}
```

This is critical. Without `disableTrie()`, the world state would try to verify that the
flat DB data matches the MPT for block 1,000,000. But the MPT for block 1,000,000 was
overwritten long ago by 19 million subsequent blocks. Trie verification would either fail
or return garbage.

### Step C: `rollMutableArchiveStateToBlockHash(...)` — details in Step 5 below.

### Step D: `.map(MutableWorldState::freezeStorage)`

```java
default MutableWorldState freezeStorage() {
    // Wraps the underlying storage in a read-only view.
    // Any attempt to write will throw UnsupportedOperationException.
    this.storage = new FrozenWorldStateKeyValueStorage(this.storage);
    return this;
}
```

The caller (blockchainQueries) receives a world state that:
- Is scoped to block 1,000,000 (via the block context set in Step C)
- Does not perform any trie operations (archive flat DB only)
- Cannot accidentally write anything to disk

---

## 6. Step 4 — The Decision Fork: TrieLog vs Archive

To make the branching logic crystal clear, here is a visual decision tree:

```
  getWorldState(queryParams) called
           │
           ▼
  queryParams.shouldWorldStateUpdateHead()?
     YES → getFullWorldState()         [block import — not our path]
     NO  ↓
           │
           ▼
  chainHead - requestedBlock >= maxLayersToLoad (512)?
     YES → ARCHIVE PATH  ← we are here for block 1,000,000
     NO  ↓
           │
           ▼
  Standard Bonsai TrieLog rollback path
  (super.getWorldState — rolls back up to 512 blocks using TrieLogs)
```

The threshold condition uses `>=` (greater-than-or-equal), meaning block
`head - 512` is the **last** block served by TrieLog rollback. Block `head - 513` and
older go through the archive path. This avoids a gap exactly at the boundary.

---

## 7. Step 5 — rollMutableArchiveStateToBlockHash()

**File**: `BonsaiArchiveWorldStateProvider.java`

```java
protected Optional<MutableWorldState> rollMutableArchiveStateToBlockHash(
        final PathBasedWorldState mutableState,
        final Hash blockHash) {

    LOG.trace(
        "Rolling mutable archive world state to block hash "
            + blockHash.getBytes().toHexString());

    try {
        // THE KEY OPERATION:
        // persist() does NOT write world state data to the flat DB.
        // In archive mode with trie disabled, it ONLY writes the three
        // metadata keys in TRIE_BRANCH_STORAGE:
        //
        //   WORLD_ROOT_HASH_KEY   ← stateRoot of block 1,000,000
        //   WORLD_BLOCK_HASH_KEY  ← hash of block 1,000,000
        //   WORLD_BLOCK_NUMBER_KEY← 1000000L (as 8-byte big-endian)
        //
        // This sets the "context" that BonsaiArchiveFlatDbStrategy will
        // use when constructing archive keys for subsequent reads.
        mutableState.persist(blockchain.getBlockHeader(blockHash).get());

        LOG.trace(
            "Archive rolling finished, {} now at {}",
            mutableState.getWorldStateStorage().getClass().getSimpleName(),
            blockHash);

        return Optional.of(mutableState);

    } catch (final MerkleTrieException re) {
        // Re-throw to trigger state healing (shouldn't happen in archive mode)
        throw re;
    } catch (final Exception e) {
        LOG.atInfo()
            .setMessage("State rolling failed on {} for block hash {}: {}")
            .addArgument(mutableState.getWorldStateStorage().getClass().getSimpleName())
            .addArgument(blockHash)
            .addArgument(e)
            .log();
        return Optional.empty();
    }
}
```

### What persist() does with WORLD_BLOCK_NUMBER_KEY

Inside `PathBasedWorldState.persist()` (simplified):

```java
public void persist(final BlockHeader blockHeader) {
    // ... other state operations ...

    // Write the block number metadata
    final Updater updater = worldStateKeyValueStorage.updater();
    updater.getWorldStateTransaction().put(
        TRIE_BRANCH_STORAGE,
        WORLD_BLOCK_NUMBER_KEY,                            // special metadata key
        Bytes.ofUnsignedLong(blockHeader.getNumber())     // block 1,000,000 → 0x00000000000F4240
              .toArrayUnsafe()
    );
    updater.getWorldStateTransaction().put(
        TRIE_BRANCH_STORAGE,
        WORLD_BLOCK_HASH_KEY,
        blockHeader.getBlockHash().toArrayUnsafe()
    );
    updater.commitComposedOnly();   // only write the metadata, not full state
}
```

After `persist()`, `TRIE_BRANCH_STORAGE[WORLD_BLOCK_NUMBER_KEY]` = `0x00000000000F4240`
(the big-endian encoding of 1,000,000).

This one write is what makes the subsequent account read use block 1,000,000 as its
archive key suffix. Everything hinges on this value.

---

## 8. Step 6 — Reading the Account: BonsaiWorldStateKeyValueStorage.getAccount()

**File**: `ethereum/core/src/main/java/org/hyperledger/besu/ethereum/trie/pathbased/bonsai/storage/BonsaiWorldStateKeyValueStorage.java`

After the world state is set up with the block 1,000,000 context, the caller does:

```java
ws.get(address)   // get Account at address 0xd8dA...
```

This calls through to the storage layer:

```java
// BonsaiWorldState.java
@Override
public Account get(final Address address) {
    final Hash addressHash = address.addressHash();  // keccak256(address)

    // Ask storage for the RLP-encoded account bytes
    return worldStateKeyValueStorage
        .getAccount(addressHash)
        .map(bytes -> BonsaiAccount.fromRLP(this, address, addressHash, bytes, false))
        .orElse(null);
}
```

```java
// BonsaiWorldStateKeyValueStorage.java
public Optional<Bytes> getAccount(final Hash accountHash) {
    return getFlatDbStrategy()            // returns BonsaiArchiveFlatDbStrategy instance
        .getFlatAccount(
            this::getWorldStateRootHash,  // supplier for current stateRoot (not used in archive)
            this::getAccountStateTrieNode,// node loader for trie (not used, trie disabled)
            accountHash,                  // keccak256(0xd8dA...96045)
            composedWorldStateStorage     // the RocksDB-backed SegmentedKeyValueStorage
        );
}
```

`getFlatDbStrategy()` returns an instance of `BonsaiArchiveFlatDbStrategy` because the
flat DB mode version byte in the DB is `0x02` (archive mode). The factory
`BonsaiFlatDbStrategyProvider` reads this byte at startup and instantiates the correct
strategy.

`accountHash` = `keccak256(0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045)`

In hex, this is some 32-byte value like `0x3e26253c5a8b02bb30c92798fa8083f0c966a954...`
(the exact value depends on the actual keccak of that address).

---

## 9. Step 7 — BonsaiArchiveFlatDbStrategy.getFlatAccount()

**File**: `ethereum/core/src/main/java/org/hyperledger/besu/ethereum/trie/pathbased/bonsai/storage/flat/BonsaiArchiveFlatDbStrategy.java`

This is the most important method in the entire archive feature:

```java
@Override
public Optional<Bytes> getFlatAccount(
        final Supplier<Optional<Bytes>> worldStateRootHashSupplier,
        final NodeLoader nodeLoader,
        final Hash accountHash,
        final SegmentedKeyValueStorage storage) {

    // Increment the "total getAccount calls" counter (metrics)
    getAccountCounter.inc();

    // ── STEP 8: Determine the block context ──────────────────────────────
    // Read WORLD_BLOCK_NUMBER_KEY from TRIE_BRANCH_STORAGE.
    // After rollMutableArchiveStateToBlockHash(), this returns 1,000,000.
    Optional<BonsaiContext> readContext = getStateArchiveContextForRead(storage);

    // ── STEP 9: Build the "nearest before" query key ──────────────────────
    // If we have a context (block 1,000,000), the key is:
    //     keccak(address) ++ 0x00000000000F4240
    // If no context (edge case), use MAX_BLOCK_SUFFIX to get the most recent.
    Bytes keyNearest = calculateArchiveKeyWithMaxSuffix(
        readContext,
        accountHash.getBytes().toArrayUnsafe()
    );

    // ── STEP 10: Search ACCOUNT_INFO_STATE_ARCHIVE ────────────────────────
    Optional<SegmentedKeyValueStorage.NearestKeyValue> nearestAccount =
        storage
            .getNearestBefore(ACCOUNT_INFO_STATE_ARCHIVE, keyNearest)
            .filter(
                found ->
                    // ── STEP 11a: Verify the address prefix ───────────────
                    // The found key must start with our accountHash (32 bytes).
                    // If the nearest key belongs to a different account, discard it.
                    accountHash.getBytes().commonPrefixLength(found.key())
                        >= accountHash.getBytes().size()
            );

    Optional<SegmentedKeyValueStorage.NearestKeyValue> accountFound;

    if (nearestAccount.isEmpty()) {
        // ── STEP 10b: Not found in ARCHIVE — try FREEZER ─────────────────
        accountFound =
            storage
                .getNearestBefore(ACCOUNT_INFO_STATE_FREEZER, keyNearest)
                .filter(
                    found ->
                        accountHash.getBytes().commonPrefixLength(found.key())
                            >= accountHash.getBytes().size()
                );

        if (accountFound.isPresent()) {
            // Found in the freezer — increment the archive-read counter
            getAccountFromArchiveCounter.inc();
        } else {
            // Not found anywhere — account did not exist at this block
            getAccountNotFoundInFlatDatabaseCounter.inc();
        }
    } else {
        // Found in the primary archive segment — increment found counter
        accountFound = nearestAccount;
        getAccountFoundInFlatDatabaseCounter.inc();
    }

    // ── STEP 11b: Filter deleted values ──────────────────────────────────
    if (accountFound.isPresent()) {
        return accountFound
            .filter(
                found ->
                    // DELETED_ACCOUNT_VALUE = new byte[0]
                    // An empty value means the account was deleted at this block.
                    !Arrays.areEqual(
                        DELETED_ACCOUNT_VALUE,
                        found.value().orElse(DELETED_ACCOUNT_VALUE)
                    )
            )
            // Convert the NearestKeyValue to Optional<Bytes>
            .flatMap(SegmentedKeyValueStorage.NearestKeyValue::wrapBytes);
    }

    // ── STEP 12: Return empty → account does not exist ───────────────────
    return Optional.empty();
}
```

---

## 10. Step 8 — getStateArchiveContextForRead()

**File**: `BonsaiArchiveFlatDbStrategy.java`

```java
private Optional<BonsaiContext> getStateArchiveContextForRead(
        final SegmentedKeyValueStorage storage) {

    // Read the block number that was set by rollMutableArchiveStateToBlockHash()
    Optional<byte[]> archiveContext =
        storage.get(TRIE_BRANCH_STORAGE, WORLD_BLOCK_NUMBER_KEY);

    if (archiveContext.isPresent()) {
        try {
            long blockNumber = Bytes.wrap(archiveContext.get()).toLong();
            // blockNumber = 1,000,000
            return Optional.of(new BonsaiContext(blockNumber));
        } catch (NumberFormatException e) {
            throw new IllegalStateException(
                "World state archive context invalid format: "
                    + new String(archiveContext.get(), StandardCharsets.UTF_8));
        }
    }

    // No block number in DB → no context (e.g., genesis state)
    return Optional.empty();
}
```

`BonsaiContext` is a tiny wrapper:

```java
// BonsaiContext.java
public class BonsaiContext {
    private final long blockNumber;

    public BonsaiContext(final long blockNumber) {
        this.blockNumber = blockNumber;
    }

    public long getBlockNumber() { return blockNumber; }
}
```

The context is now: `BonsaiContext { blockNumber = 1,000,000 }`.

---

## 11. Step 9 — calculateArchiveKeyWithMaxSuffix()

**File**: `BonsaiArchiveFlatDbStrategy.java`

```java
public static Bytes calculateArchiveKeyWithMaxSuffix(
        final Optional<BonsaiContext> context,
        final byte[] naturalKey) {

    return calculateArchiveKeyWithSuffix(
        context,
        naturalKey,
        MAX_BLOCK_SUFFIX   // = Bytes.ofUnsignedLong(Long.MAX_VALUE) = 0x7FFFFFFFFFFFFFFF
    );
}

public static byte[] calculateArchiveKeyWithSuffix(
        final Optional<BonsaiContext> context,
        final byte[] naturalKey,
        final byte[] suffix) {

    if (context.isPresent()) {
        // We have a block context: use the block number as the suffix.
        // Encode blockNumber as 8-byte big-endian unsigned long.
        byte[] blockSuffix =
            Bytes.ofUnsignedLong(context.get().getBlockNumber()).toArrayUnsafe();
        return Bytes.concatenate(
            Bytes.of(naturalKey),
            Bytes.of(blockSuffix)
        ).toArrayUnsafe();
    }
    // No context: use the provided default suffix (MAX or MIN)
    return Bytes.concatenate(
        Bytes.of(naturalKey),
        Bytes.of(suffix)
    ).toArrayUnsafe();
}
```

Wait — if we have a context (block 1,000,000), why does `calculateArchiveKeyWithMaxSuffix`
use the block number instead of `MAX_BLOCK_SUFFIX`?

The method name can be slightly confusing. The logic is:

- **With context**: the key suffix IS the block number. We want the nearest entry AT OR
  BEFORE block 1,000,000. So we build a key with exactly block 1,000,000 as the suffix,
  and `getNearestBefore` finds the largest key ≤ this.

- **Without context** (e.g., asking for the absolute current value without a specific
  block): use `MAX_BLOCK_SUFFIX` (0x7FFF...) so that `getNearestBefore` finds the
  highest block-numbered entry available.

For our query, the key constructed is:

```
  naturalKey (32 bytes):  keccak256(0xd8dA...96045)
                          = 0x3e26253c5a8b02bb30c92798fa8083f0c966a954b019cad73aa12cdf3b344a7e
                            (example — actual value depends on real keccak)

  blockSuffix (8 bytes):  block 1,000,000 = 0x00000000000F4240

  queryKey (40 bytes):    0x3e26253c5a8b02bb30c92798fa8083f0c966a954b019cad73aa12cdf3b344a7e
                          ++ 0x00000000000F4240
```

This 40-byte key is the argument passed to `getNearestBefore`.

---

## 12. Step 10 — getNearestBefore() in RocksDB

**File**: `plugin-api/.../storage/SegmentedKeyValueStorage.java` (interface) and
`services/rocksdb/.../RocksDBColumnarKeyValueStorage.java` (implementation)

The interface declares:

```java
public interface SegmentedKeyValueStorage {

    /**
     * Returns the entry whose key is the largest key that is
     * lexicographically ≤ the given key, within the given segment.
     *
     * Returns Optional.empty() if no such entry exists.
     */
    Optional<NearestKeyValue> getNearestBefore(
        SegmentIdentifier segment,
        Bytes key
    );

    record NearestKeyValue(Bytes key, Optional<byte[]> value) {
        public Optional<Bytes> wrapBytes() {
            return value.map(Bytes::wrap);
        }
    }
}
```

The RocksDB implementation uses a **seek-for-prev** iterator operation:

```java
// RocksDBColumnarKeyValueStorage.java (conceptual — actual code uses try-with-resources)
@Override
public Optional<NearestKeyValue> getNearestBefore(
        final SegmentIdentifier segment,
        final Bytes key) {

    // Get the column family handle for ACCOUNT_INFO_STATE_ARCHIVE
    final ColumnFamilyHandle cfHandle = getColumnFamilyHandle(segment);

    try (final RocksIterator iterator = db.newIterator(cfHandle)) {

        // seekForPrev positions the iterator at the largest key ≤ targetKey.
        // This is an O(log N) operation on RocksDB's LSM-tree index.
        iterator.seekForPrev(key.toArrayUnsafe());

        if (!iterator.isValid()) {
            // No key ≤ targetKey exists in this column family
            return Optional.empty();
        }

        final Bytes foundKey   = Bytes.wrap(iterator.key());
        final byte[] foundValue = iterator.value();

        return Optional.of(new NearestKeyValue(foundKey, Optional.of(foundValue)));
    }
}
```

### What RocksDB Sees

RocksDB's `ACCOUNT_INFO_STATE_ARCHIVE` column family contains (among millions of other
entries) something like this for our address (sorted order):

```
  ...
  3e26...7e  0000000000000001  →  { nonce:0, balance:0 ETH }     ← block 1
  3e26...7e  0000000000000064  →  { nonce:1, balance:1.5 ETH }   ← block 100
  3e26...7e  00000000000F3E50  →  { nonce:2, balance:0.9 ETH }   ← block 999,504
  3e26...7e  00000000000F4240  →  { nonce:3, balance:1.2 ETH }   ← block 1,000,000  ← EXACT MATCH
  3e26...7e  000000000027C1D0  →  { nonce:4, balance:0.5 ETH }   ← block 2,605,520
  ...
```

In our example, the account changed on exactly block 1,000,000, so we get an exact match.
In the general case (if the account didn't change on that exact block), we'd get the
entry for the last block the account changed before 1,000,000.

```
  seekForPrev(3e26...7e  00000000000F4240)
  → iterator positions at 3e26...7e  00000000000F4240   ← exact match ✓
  → returns NearestKeyValue {
        key:   0x3e26...7e  00000000000F4240
        value: RLP { nonce:3, balance:1.2 ETH, storageRoot:..., codeHash:... }
    }
```

This is a single B-tree seek — logarithmic in the total number of archive entries, but
effectively constant with a warm RocksDB block cache.

---

## 13. Step 11 — Prefix Validation and Deleted-Value Filtering

Back in `getFlatAccount()`, after `getNearestBefore` returns:

### Prefix Validation

```java
.filter(
    found ->
        accountHash.getBytes().commonPrefixLength(found.key())
            >= accountHash.getBytes().size()
)
```

This checks: does the found key START with our address hash?

```
  accountHash (32 bytes):  3e26253c5a8b02bb30c92798fa8083f0c966a954b019cad73aa12cdf3b344a7e
  found.key()  (40 bytes): 3e26253c5a8b02bb30c92798fa8083f0c966a954b019cad73aa12cdf3b344a7e
                           00000000000F4240

  commonPrefixLength = 32   (all 32 bytes of accountHash match the first 32 bytes of found.key)
  32 >= 32 → true → valid! ✓
```

Why is this check necessary? Consider what happens at a segment boundary. If our address
hash is the lexicographically smallest key in the archive, `getNearestBefore` might return
the last entry of the previous account (which has a smaller address hash). The prefix
check rejects it:

```
  accountHash:  0x4D95FBAF...
  found.key():  0x3e26253c...00000000000F4240   ← different account (3e < 4D)

  commonPrefixLength = 1 (only 0x4D vs 0x3e match for first nibble... actually 0)
  Actually 0 < 32 → false → rejected ✓
```

### Deleted-Value Filtering

```java
return accountFound
    .filter(
        found ->
            !Arrays.areEqual(
                DELETED_ACCOUNT_VALUE,        // = new byte[0]  (empty array)
                found.value().orElse(DELETED_ACCOUNT_VALUE)
            )
    )
    .flatMap(SegmentedKeyValueStorage.NearestKeyValue::wrapBytes);
```

If the value at the found key is an empty byte array, it is a deletion tombstone —
the account was deleted at that block. The filter rejects it and the method returns
`Optional.empty()`, meaning "account does not exist at this block."

For our query, the value is a real RLP-encoded account (not empty), so the filter passes.

---

## 14. Step 12 — Returning the Balance

The RLP bytes returned by `getFlatAccount()` flow back up:

```java
// BonsaiWorldStateKeyValueStorage.getAccount() returns Optional<Bytes>
// BonsaiWorldState.get(address) maps it:

return worldStateKeyValueStorage
    .getAccount(addressHash)
    .map(bytes ->
        BonsaiAccount.fromRLP(this, address, addressHash, bytes, false)
    )
    .orElse(null);
```

`BonsaiAccount.fromRLP()` decodes the RLP:

```java
// Simplified
public static BonsaiAccount fromRLP(
        final BonsaiWorldState context,
        final Address address,
        final Hash addressHash,
        final Bytes encoded,
        final boolean mutable) {

    // RLP decode: { nonce, balance, storageRoot, codeHash }
    final RLPInput input = RLP.input(encoded);
    input.enterList();
    final long nonce        = input.readLongScalar();
    final Wei  balance      = Wei.of(input.readUInt256Scalar());
    final Hash storageRoot  = Hash.wrap(input.readBytes32());
    final Hash codeHash     = Hash.wrap(input.readBytes32());
    input.leaveList();

    return new BonsaiAccount(
        context, address, addressHash, nonce, balance, storageRoot, codeHash, mutable
    );
}
```

Back in `blockchainQueries.accountBalance()`:

```java
return worldStateArchive
    .getWorldState(queryParams)      // Optional<MutableWorldState>
    .map(ws -> ws.get(address))     // Optional<Account>  ← BonsaiAccount { balance: 1.2 ETH }
    .map(Account::getBalance);      // Optional<Wei>       ← Wei(1_200_000_000_000_000_000)
```

And in `EthGetBalance.response()`:

```java
return new JsonRpcSuccessResponse(
    requestContext.getRequest().getId(),
    balance.toShortHexString()    // "0x10a741a462780000"  (1.2 ETH in Wei hex)
);
```

The JSON-RPC response sent to the client:

```json
{
  "jsonrpc": "2.0",
  "result": "0x10a741a462780000",
  "id": 1
}
```

That is the complete read path. ✓

---

## 15. The Block Import Path: How Archive Entries Get Written

Now let us trace the **write path** — how did that archive entry get into the DB in the
first place? This happens during normal block import.

### 15a. Block Import Calls putFlatAccount()

When block 1,000,000 is being committed to disk, the `BonsaiWorldStateKeyValueStorage.Updater`
calls:

```java
// BonsaiWorldStateKeyValueStorage.Updater.putAccountInfoState()
public Updater putAccountInfoState(final Hash accountHash, final Bytes accountValue) {
    if (accountValue.isEmpty()) return this;

    flatDbStrategy.putFlatAccount(
        worldStorage,               // ← needed by archive strategy to read block context
        composedWorldStateTransaction,
        accountHash,
        accountValue
    );
    return this;
}
```

### 15b. BonsaiArchiveFlatDbStrategy.putFlatAccount()

```java
@Override
public void putFlatAccount(
        final SegmentedKeyValueStorage storage,
        final SegmentedKeyValueStorageTransaction transaction,
        final Hash accountHash,
        final Bytes accountValue) {

    // ── Determine block context for writing ──────────────────────────────
    // At write time, WORLD_BLOCK_NUMBER_KEY holds the number of the
    // PREVIOUS block (the one currently persisted as "head").
    // The block being imported is previous + 1.
    Optional<BonsaiContext> writeContext = getStateArchiveContextForWrite(storage);

    // ── Write to the archive segment with the block number suffix ─────────
    byte[] archiveKey = calculateArchiveKeyWithSuffix(
        writeContext,
        accountHash.getBytes().toArrayUnsafe(),
        MIN_BLOCK_SUFFIX    // default if no context — only used at genesis
    );

    transaction.put(
        ACCOUNT_INFO_STATE_ARCHIVE,   // write to the archive segment
        archiveKey,                   // keccak(address) ++ blockNumber
        accountValue.toArrayUnsafe()  // RLP-encoded account state
    );

    // ── Also write to the primary segment (for fast current-state reads) ──
    // The parent class BonsaiFlatDbStrategy.putFlatAccount() writes:
    //   ACCOUNT_INFO_STATE[ keccak(address) ] = accountValue
    super.putFlatAccount(storage, transaction, accountHash, accountValue);
}
```

### 15c. getStateArchiveContextForWrite()

```java
private Optional<BonsaiContext> getStateArchiveContextForWrite(
        final SegmentedKeyValueStorage storage) {

    Optional<byte[]> archiveContext =
        storage.get(TRIE_BRANCH_STORAGE, WORLD_BLOCK_NUMBER_KEY);

    if (archiveContext.isPresent()) {
        long previousBlockNumber = Bytes.wrap(archiveContext.get()).toLong();
        // The world state currently records block N-1.
        // The block we are importing is block N.
        // Write archive entries with suffix = N.
        return Optional.of(new BonsaiContext(previousBlockNumber + 1));
    } else {
        // No block number → genesis block, use suffix 0
        return Optional.of(new BonsaiContext(0L));
    }
}
```

Note the `+ 1`: at the moment of writing, `WORLD_BLOCK_NUMBER_KEY` still reflects the
**parent** block (the previous head). The new block being written is one higher.

### 15d. The Double Write

Every block import in archive mode writes the account state **twice**:

```
  Block 1,000,000 imported:

  Write 1 (archive):
  ACCOUNT_INFO_STATE_ARCHIVE[ keccak(Alice) ++ 0x00000000000F4240 ] = RLP{...}

  Write 2 (primary):
  ACCOUNT_INFO_STATE[ keccak(Alice) ] = RLP{...}   ← overwrites previous value
```

The primary segment always reflects current state (O(1) reads). The archive segment
accumulates all historical versions (O(log N) reads via `getNearestBefore`).

---

## 16. The Archiver Path: Moving Old State Asynchronously

Beyond writing new state, `BonsaiArchiver` also moves **superseded** state from the
primary segments to the archive segments. Let us trace what happens when block 1,000,001
is imported and the archiver runs for block 999,991 (10 blocks behind head).

### 16a. onBlockAdded() is triggered

```java
// BonsaiArchiver.java
@Override
public void onBlockAdded(final BlockAddedEvent addedBlockContext) {
    initialize();   // refresh latestArchivedBlock from DB

    final Optional<Long> blockNumber =
        Optional.of(addedBlockContext.getHeader().getNumber());

    blockNumber.ifPresent(
        blockNum -> {
            // Schedule async — don't block the import thread
            executeAsync.accept(
                () -> {
                    // Try-lock: if another archiver run is in progress, skip this one
                    if (archiveMutex.tryLock()) {
                        try {
                            moveBlockStateToArchive();
                        } finally {
                            archiveMutex.unlock();
                        }
                    }
                });
        });
}
```

### 16b. moveBlockStateToArchive()

```java
public int moveBlockStateToArchive() {
    // Stay 10 blocks behind head — safety buffer for short reorgs
    final long retainAboveThisBlock =
        blockchain.getChainHeadBlockNumber()
            - DISTANCE_FROM_HEAD_BEFORE_ARCHIVING_OLD_STATE;  // = head - 10

    // Build the list of blocks to archive in this run
    final SortedMap<Long, Hash> blocksToArchive;
    synchronized (this) {
        blocksToArchive = new TreeMap<>();
        long nextToArchive = latestArchivedBlock.get() + 1;

        // Process up to CATCHUP_LIMIT=1000 blocks per run
        while (blocksToArchive.size() <= CATCHUP_LIMIT
                && nextToArchive < retainAboveThisBlock) {
            blocksToArchive.put(
                nextToArchive,
                blockchain.getBlockByNumber(nextToArchive).get().getHash()
            );
            nextToArchive++;
        }
    }

    // For each block, use its TrieLog to know which accounts changed,
    // then move their PREVIOUS state to the archive segment
    blocksToArchive.entrySet().forEach(block -> {
        Hash blockHash = block.getValue();

        Optional<TrieLog> trieLog = trieLogManager.getTrieLogLayer(blockHash);

        if (trieLog.isPresent()) {
            // Archive account state
            trieLog.get().getAccountChanges().forEach((address, change) -> {
                rootWorldStateStorage.archivePreviousAccountState(
                    blockchain.getBlockHeader(
                        blockchain.getBlockHeader(blockHash).get().getParentHash()
                    ),
                    address.addressHash()
                );
            });

            // Archive storage state
            trieLog.get().getStorageChanges().forEach((address, storageSlotKey) -> {
                storageSlotKey.forEach((slotKey, slotValue) -> {
                    rootWorldStateStorage.archivePreviousStorageState(
                        blockchain.getBlockHeader(
                            blockchain.getBlockHeader(blockHash).get().getParentHash()
                        ),
                        Bytes.concatenate(
                            address.addressHash().getBytes(),
                            slotKey.getSlotHash().getBytes()
                        )
                    );
                });
            });
        }

        // Update progress pointer
        rootWorldStateStorage.setLatestArchivedBlock(block.getKey());
        latestArchivedBlock.set(block.getKey());
    });

    return archivedAccountStateCount.get() + archivedAccountStorageCount.get();
}
```

### 16c. archivePreviousAccountState()

The `archivePreviousAccountState()` method in `PathBasedWorldStateKeyValueStorage`:

```java
// Moves the entry currently in ACCOUNT_INFO_STATE for `accountHash`
// to ACCOUNT_INFO_STATE_ARCHIVE with the parent block's number as the suffix,
// but ONLY if a more recent version exists (i.e., the account has been updated).
public int archivePreviousAccountState(
        final Optional<BlockHeader> parentBlockHeader,
        final Hash accountHash) {

    if (parentBlockHeader.isEmpty()) return 0;

    // Read the CURRENT value in the primary segment
    Optional<byte[]> currentValue =
        composedWorldStateStorage.get(ACCOUNT_INFO_STATE, accountHash.getBytes().toArrayUnsafe());

    if (currentValue.isEmpty()) return 0;

    // Write it to the archive segment under the PARENT block's number
    // (this is the value that was "current" AS OF the parent block)
    byte[] archiveKey = BonsaiArchiveFlatDbStrategy.calculateArchiveKeyWithSuffix(
        Optional.of(new BonsaiContext(parentBlockHeader.get().getNumber())),
        accountHash.getBytes().toArrayUnsafe(),
        BonsaiArchiveFlatDbStrategy.MIN_BLOCK_SUFFIX
    );

    final SegmentedKeyValueStorageTransaction txn = composedWorldStateStorage.startTransaction();
    txn.put(ACCOUNT_INFO_STATE_ARCHIVE, archiveKey, currentValue.get());
    txn.commit();

    return 1;
}
```

### Why is this separate from the block import write?

During block import of block N, `putFlatAccount()` writes the **new** value (after block N)
to the archive with block N's number.

`archivePreviousAccountState()`, called by the archiver for block N, writes the **old**
value (before block N, i.e., the value as of the parent block N-1) to the archive.

Together they ensure the archive contains both:
- The value **before** the change (written by the archiver asynchronously)
- The value **after** the change (written by `putFlatAccount()` during import)

This is why the archive can serve the state "at block N-1" even though block N overwrote
the primary segment.

---

## 17. Key Data Structures Quick Reference

### BonsaiContext

```java
// BonsaiContext.java
public class BonsaiContext {
    private final long blockNumber;

    public BonsaiContext(final long blockNumber) {
        this.blockNumber = blockNumber;
    }

    public long getBlockNumber() { return blockNumber; }
}
```

A trivial wrapper. Carries the block number through the key-construction calls so that
the archive strategy knows which block's worth of data to construct the key for.

### SegmentedKeyValueStorage.NearestKeyValue

```java
// In the plugin-api
public interface SegmentedKeyValueStorage {
    record NearestKeyValue(Bytes key, Optional<byte[]> value) {
        public Optional<Bytes> wrapBytes() {
            return value.map(Bytes::wrap);
        }
    }

    Optional<NearestKeyValue> getNearestBefore(SegmentIdentifier segment, Bytes key);
    Optional<NearestKeyValue> getNearestAfter(SegmentIdentifier segment, Bytes key);
}
```

A `NearestKeyValue` holds both the found **key** (needed for prefix validation) and the
**value** (the actual account/storage data).

### KeyValueSegmentIdentifier (relevant entries)

```java
// All nine segments used by X_BONSAI_ARCHIVE:
ACCOUNT_INFO_STATE(      {6},  EnumSet.of(BONSAI, X_BONSAI_ARCHIVE), ...)  // current accounts
CODE_STORAGE(            {7},  EnumSet.of(BONSAI, X_BONSAI_ARCHIVE), ...)  // bytecode
ACCOUNT_STORAGE_STORAGE( {8},  EnumSet.of(BONSAI, X_BONSAI_ARCHIVE), ...)  // current slots
TRIE_BRANCH_STORAGE(     {9},  EnumSet.of(BONSAI, X_BONSAI_ARCHIVE), ...)  // MPT + metadata
TRIE_LOG_STORAGE(        {10}, EnumSet.of(BONSAI, X_BONSAI_ARCHIVE), ...)  // block diffs

// Four new archive/freezer segments (use full-name keys for debuggability):
ACCOUNT_INFO_STATE_ARCHIVE(  "ACCOUNT_INFO_STATE_ARCHIVE".getBytes(),  EnumSet.of(X_BONSAI_ARCHIVE), ...)
ACCOUNT_STORAGE_ARCHIVE(     "ACCOUNT_STORAGE_ARCHIVE".getBytes(),     EnumSet.of(X_BONSAI_ARCHIVE), ...)
ACCOUNT_INFO_STATE_FREEZER(  "ACCOUNT_INFO_STATE_FREEZER".getBytes(),  EnumSet.of(X_BONSAI_ARCHIVE), ...)
ACCOUNT_STORAGE_FREEZER(     "ACCOUNT_STORAGE_FREEZER".getBytes(),     EnumSet.of(X_BONSAI_ARCHIVE), ...)
```

### Key Suffix Constants

```java
// In BonsaiArchiveFlatDbStrategy:
static final byte[] MAX_BLOCK_SUFFIX = Bytes.ofUnsignedLong(Long.MAX_VALUE).toArrayUnsafe();
// = 0x7FFFFFFFFFFFFFFF  — used as upper bound in getNearestBefore without context

static final byte[] MIN_BLOCK_SUFFIX = Bytes.ofUnsignedLong(0L).toArrayUnsafe();
// = 0x0000000000000000  — used as lower bound in range scans

public static final byte[] DELETED_ACCOUNT_VALUE  = new byte[0];  // tombstone for deleted accounts
public static final byte[] DELETED_STORAGE_VALUE  = new byte[0];  // tombstone for deleted slots
```

---

## 18. Full Call Stack Summary

Here is the complete call stack for our `eth_getBalance(address, block=1000000)` query,
condensed into one view:

```
  EthGetBalance.response()
  └─ blockchainQueries.accountBalance(address, blockHeader_1000000)
     └─ worldStateArchive.getWorldState(queryParams{block=1000000, readOnly})
        └─ BonsaiArchiveWorldStateProvider.getWorldState(queryParams)
           │  [Decision: 20M - 1M = 19M >= 512 → archive path]
           ├─ cachedWorldStorageManager.getWorldState(headHash)
           │     returns BonsaiWorldState{head=block_20000000}
           ├─ worldState.disableTrie()
           │     worldStateConfig.trieDisabled = true
           ├─ rollMutableArchiveStateToBlockHash(worldState, hash_1000000)
           │  └─ mutableState.persist(blockHeader_1000000)
           │        writes WORLD_BLOCK_NUMBER_KEY = 0x00000000000F4240 to TRIE_BRANCH_STORAGE
           └─ worldState.freezeStorage()
                 wraps storage in FrozenWorldStateKeyValueStorage

     → caller now has MutableWorldState scoped to block 1,000,000

     └─ worldState.get(address)
        └─ BonsaiWorldState.get(address)
           └─ worldStateKeyValueStorage.getAccount(keccak(address))
              └─ BonsaiArchiveFlatDbStrategy.getFlatAccount(keccak, storage)
                 ├─ getStateArchiveContextForRead(storage)
                 │     reads WORLD_BLOCK_NUMBER_KEY → 1,000,000
                 │     returns BonsaiContext{blockNumber=1000000}
                 ├─ calculateArchiveKeyWithMaxSuffix(context, keccak(address))
                 │     returns keccak(address) ++ 0x00000000000F4240  [40 bytes]
                 ├─ storage.getNearestBefore(ACCOUNT_INFO_STATE_ARCHIVE, queryKey)
                 │  └─ RocksDB.seekForPrev(queryKey)
                 │        seeks to largest key ≤ keccak(address)++1000000
                 │        returns NearestKeyValue{ key: ..., value: RLP{nonce,bal,...} }
                 ├─ filter: commonPrefixLength(keccak(address), found.key) >= 32  ✓
                 └─ filter: found.value != empty (not a tombstone)  ✓
                    returns Optional<Bytes>{ RLP{nonce:3, balance:1.2eth, ...} }

           └─ BonsaiAccount.fromRLP(...)
                 decodes RLP → BonsaiAccount{balance: Wei(1_200_000_000_000_000_000)}

     └─ account.getBalance()
           returns Wei(1_200_000_000_000_000_000)

  EthGetBalance → JsonRpcSuccessResponse{ result: "0x10a741a462780000" }
```

Total RocksDB operations for this query:
1. One `get` on `TRIE_BRANCH_STORAGE` (to read `WORLD_BLOCK_NUMBER_KEY`)
2. One `seekForPrev` on `ACCOUNT_INFO_STATE_ARCHIVE` (the archive lookup)

That is it. **Two database operations** to serve a historic balance query from 19 million
blocks ago. Compare this to Forest mode's 8–12 random node reads just to traverse the
trie, and the power of Bonsai Archive's design becomes clear.

---

## What You Have Now Learned

Working through all seven files in this series, you understand:

| File | What you learned |
|------|-----------------|
| `01_ethereum_state_basics.md` | What world state is, what accounts contain, why nodes need history |
| `02_merkle_patricia_trie.md` | The MPT data structure, write amplification, stateRoot |
| `03_forest_vs_bonsai.md` | Why Forest is disk-heavy, how Bonsai's flat DB improves reads |
| `04_bonsai_deep_dive.md` | DB segments, flat DB modes, block import pipeline, state healing |
| `05_trie_logs.md` | TrieLog as a block-diff journal, rollback, pruning, the 512-block window |
| `06_bonsai_archive.md` | Block-number-suffixed keys, `getNearestBefore`, archiver, archive provider |
| `07_code_walkthrough.md` | The actual Besu source code, traced end-to-end for a historic RPC call |

The Bonsai Archive feature elegantly solves the "full archive node" problem by observing
that the key insight is not storing more trie nodes — it is storing flat state with a
richer key that includes time (block number). RocksDB's sorted-key `seekForPrev` does
the rest.