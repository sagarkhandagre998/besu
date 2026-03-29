# 07 — The EVM & Smart Contracts

> **Level:** Intermediate | **Read time:** ~30 minutes
> **Prerequisites:** [06 — Ethereum Accounts, Wallets & Transactions](06_ethereum_accounts_and_transactions.md)

---

## Table of Contents

1. [What Is the EVM?](#1-what-is-the-evm)
2. [The EVM as a State Machine](#2-the-evm-as-a-state-machine)
3. [EVM Components](#3-evm-components)
4. [EVM Opcodes](#4-evm-opcodes)
5. [How Smart Contracts Work](#5-how-smart-contracts-work)
6. [Contract Deployment Flow](#6-contract-deployment-flow)
7. [The ABI: Application Binary Interface](#7-the-abi-application-binary-interface)
8. [Function Calls: External, Internal, and Special](#8-function-calls-external-internal-and-special)
9. [Common Smart Contract Patterns](#9-common-smart-contract-patterns)
10. [Smart Contract Limitations](#10-smart-contract-limitations)
11. [Precompiles](#11-precompiles)
12. [Gas Metering in the EVM](#12-gas-metering-in-the-evm)
13. [The Extractable EVM: Besu's EVM as a Library](#13-the-extractable-evm-besus-evm-as-a-library)
14. [Recap Checklist](#14-recap-checklist)
15. [Check Your Understanding](#15-check-your-understanding)

---

## 1. What Is the EVM?

The **Ethereum Virtual Machine (EVM)** is the runtime engine at the heart of Ethereum. It is the component that actually *executes* smart contract code — every time a transaction triggers a contract, the EVM runs the contract's bytecode and produces a result.

Think of the EVM as a very unusual kind of computer:

- It exists simultaneously on **every full Ethereum node** in the world — thousands of machines running in parallel.
- Every single one of them executes the **exact same code** with the **exact same inputs** and arrives at the **exact same result**.
- It is completely **isolated** from the host machine — a contract running in the EVM cannot read files, make network requests, or interact with the operating system in any way.
- Its execution is **metered** — every instruction costs a precise amount of "gas," preventing infinite loops and ensuring nodes are compensated for their work.

This combination — deterministic, sandboxed, globally replicated execution — is what makes trustless smart contracts possible.

### A Sandboxed, Stack-Based Virtual Machine

The EVM is a **stack-based** architecture (as opposed to register-based architectures like x86 or ARM). Rather than storing intermediate values in named registers, the EVM uses a last-in-first-out (LIFO) stack. Operations pop their inputs from the top of the stack and push their results back onto it.

Each item on the EVM stack is a **256-bit (32-byte) word**. This is not arbitrary — 256 bits matches the output size of Keccak-256 and the size of secp256k1 private keys, making many cryptographic operations natural and efficient.

### Why "Virtual Machine"?

The EVM is called a virtual machine because it is a software simulation of a computer — it has its own instruction set (opcodes), its own memory model, and its own execution rules, all implemented in software by each Ethereum client. Besu implements the EVM in Java. Geth implements it in Go. Nethermind in C#. They all implement the same EVM specification, so they all produce identical results.

---

## 2. The EVM as a State Machine

Ethereum is fundamentally a **state machine**. Understanding this framing makes many confusing details click into place.

```
                        EVM = The Transition Function

  World State at            Transaction               World State at
  Block N - 1       +       (Input)          =        Block N
  ─────────────           ─────────────           ─────────────
  { all account            { from, to,             { all account
    balances,                value, data,             balances,
    nonces,                  gas, sig }               nonces,
    contract code,                                    contract code,
    contract storage }                                contract storage }
        │                                                   │
        └──────────────── EVM executes ────────────────────▶│
```

Every transaction is an **input** to the EVM. The EVM takes the current world state and the transaction, executes the transaction's instructions against the state, and produces a **new world state**. The `stateRoot` in each block header is the cryptographic fingerprint of the resulting state.

Key implications:
- The same transaction applied to the same state always produces the same new state — **no randomness, no external dependencies**.
- There is one canonical sequence of states: genesis → block 1's state → block 2's state → ... → current state.
- A full node can reconstruct the current state from scratch by replaying every transaction since genesis.

### EVM Execution Context

When a transaction triggers the EVM, it runs in a specific **execution context** that includes:

| Context Field | Contents |
|---|---|
| `code` | The bytecode being executed (the contract's compiled code) |
| `calldata` | Input data for this call (the function selector + arguments) |
| `caller` | The address that triggered this execution (`msg.sender` in Solidity) |
| `value` | The amount of ETH (wei) sent with this call (`msg.value`) |
| `gasPrice` | The effective gas price for this transaction |
| `origin` | The original EOA that sent the top-level transaction (`tx.origin`) |
| `blockNumber` | The current block's number (`block.number`) |
| `timestamp` | The current block's timestamp (`block.timestamp`) |
| `chainId` | The chain ID of the network |

---

## 3. EVM Components

The EVM has four distinct storage areas, each with very different characteristics:

```
┌──────────────────────────────────────────────────────────────────────┐
│                        EVM EXECUTION ENVIRONMENT                     │
│                                                                      │
│  ┌─────────────────┐   ┌─────────────────┐   ┌──────────────────┐  │
│  │      STACK      │   │     MEMORY      │   │    STORAGE       │  │
│  │                 │   │                 │   │                  │  │
│  │  256-bit words  │   │  Byte array     │   │  Key-value store │  │
│  │  Max 1024 items │   │  Temporary      │   │  Persistent      │  │
│  │  LIFO           │   │  Grows as needed│   │  32B key → 32B   │  │
│  │  Free (no cost  │   │  Cleared after  │   │  value           │  │
│  │  per word)      │   │  call ends      │   │  Expensive!      │  │
│  └─────────────────┘   └─────────────────┘   └──────────────────┘  │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    CALLDATA (read-only)                      │    │
│  │  The input data passed to this call. ABI-encoded function    │    │
│  │  selector + arguments. Cannot be modified during execution.  │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌───────────────────┐   ┌──────────────────────────────────────┐   │
│  │  PROGRAM COUNTER  │   │          GAS COUNTER                 │   │
│  │  (PC)             │   │                                      │   │
│  │  Points to the    │   │  Starts at gasLimit; decrements with │   │
│  │  next opcode to   │   │  each opcode executed. Hits 0 →      │   │
│  │  execute          │   │  out-of-gas exception, revert.       │   │
│  └───────────────────┘   └──────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
```

### Stack

- **Purpose:** The primary workspace for all arithmetic, logic, and control flow.
- **Size:** Each item is exactly **256 bits (32 bytes)**. Maximum depth is **1024 items**.
- **Access:** Only the top of the stack is directly accessible. Operations pop from the top and push results back. Some opcodes (`DUP`, `SWAP`) access items deeper in the stack.
- **Cost:** No cost for simply having items on the stack. The gas cost comes from the opcodes that manipulate it.
- **Overflow:** Pushing a 1025th item causes a stack overflow exception and reverts the transaction.

### Memory

- **Purpose:** Temporary byte-addressable storage for data needed during a single execution. Think of it like RAM.
- **Size:** Starts empty and expands dynamically as needed. There is no fixed limit, but memory expansion has a **quadratic gas cost** — doubling the memory usage more than doubles the cost. This discourages unbounded memory use.
- **Lifetime:** Cleared entirely after each message call. A contract's memory in one transaction is not visible in another.
- **Access:** Via `MLOAD` (read 32 bytes) and `MSTORE` / `MSTORE8` (write 32 bytes / 1 byte) opcodes.

### Storage

- **Purpose:** Persistent key-value store. This is where contract state lives between transactions — token balances, ownership records, configuration, anything that must survive beyond a single call.
- **Structure:** Maps 256-bit keys (called "slots") to 256-bit values. Each contract has its own completely separate storage namespace.
- **Lifetime:** Permanent — persists across all transactions until explicitly overwritten or the contract is selfdestruct-ed.
- **Cost:** The most expensive EVM resource.
  - Writing to a previously zero slot (`SSTORE`): **20,000 gas** (cold write).
  - Writing to a previously non-zero slot: **5,000 gas** (warm write).
  - Reading (`SLOAD`): **2,100 gas** (cold) or **100 gas** (warm, after the slot has been read once in this transaction).
- **Why so expensive?** Because storage changes must be committed to every full node's database on disk — a global, permanent operation.

### Calldata

- **Purpose:** The read-only input data for this execution — the ABI-encoded function call data.
- **Access:** Via `CALLDATALOAD` (read 32 bytes at an offset), `CALLDATASIZE`, and `CALLDATACOPY`.
- **Cost:** Cheap to read. Notably, calldata is **the cheapest way to pass large amounts of data** to a contract — cheaper than storing it, cheaper than using memory if you only need to read it.

---

## 4. EVM Opcodes

The EVM has a defined instruction set — a set of **opcodes** (operation codes). Each opcode is a single byte (0x00 to 0xFF), giving a theoretical maximum of 256 opcodes (not all are defined — some are invalid/reserved).

Every opcode has:
- A **name** (e.g., `ADD`, `SLOAD`, `CALL`)
- An **opcode byte** (e.g., `0x01`, `0x54`, `0xF1`)
- A **gas cost** (fixed or dynamic)
- A defined number of **stack inputs** it pops
- A defined number of **stack outputs** it pushes

### Selected Opcodes by Category

#### Arithmetic and Logic

| Opcode | Byte | Gas | Pops | Pushes | Description |
|---|---|---|---|---|---|
| `ADD` | 0x01 | 3 | 2 | 1 | a + b (mod 2²⁵⁶) |
| `MUL` | 0x02 | 5 | 2 | 1 | a × b (mod 2²⁵⁶) |
| `SUB` | 0x03 | 3 | 2 | 1 | a - b (mod 2²⁵⁶) |
| `DIV` | 0x04 | 5 | 2 | 1 | Integer division a ÷ b |
| `MOD` | 0x06 | 5 | 2 | 1 | a mod b |
| `EXP` | 0x0A | 10+ | 2 | 1 | a^b; extra cost per byte of exponent |
| `LT` | 0x10 | 3 | 2 | 1 | 1 if a < b, else 0 |
| `EQ` | 0x14 | 3 | 2 | 1 | 1 if a == b, else 0 |
| `AND` | 0x16 | 3 | 2 | 1 | Bitwise AND |
| `OR` | 0x17 | 3 | 2 | 1 | Bitwise OR |
| `NOT` | 0x19 | 3 | 1 | 1 | Bitwise NOT |
| `SHL` | 0x1B | 3 | 2 | 1 | Shift left |
| `SHR` | 0x1C | 3 | 2 | 1 | Logical shift right |

#### Environment and Block Information

| Opcode | Byte | Gas | Description |
|---|---|---|---|
| `ADDRESS` | 0x30 | 2 | Address of currently executing contract |
| `BALANCE` | 0x31 | 100/2600 | ETH balance of an address (warm/cold) |
| `ORIGIN` | 0x32 | 2 | Original EOA sender (`tx.origin`) |
| `CALLER` | 0x33 | 2 | Immediate caller (`msg.sender`) |
| `CALLVALUE` | 0x34 | 2 | ETH sent with this call (`msg.value`) |
| `CALLDATALOAD` | 0x35 | 3 | Read 32 bytes from calldata at offset |
| `CODESIZE` | 0x38 | 2 | Size of currently executing contract code |
| `GASPRICE` | 0x3A | 2 | Effective gas price of the transaction |
| `BLOCKHASH` | 0x40 | 20 | Hash of one of the last 256 blocks |
| `COINBASE` | 0x41 | 2 | Block proposer's address |
| `TIMESTAMP` | 0x42 | 2 | Current block timestamp |
| `NUMBER` | 0x43 | 2 | Current block number |
| `CHAINID` | 0x46 | 2 | Current chain ID (EIP-1344) |
| `SELFBALANCE` | 0x47 | 5 | ETH balance of the current contract (cheaper than BALANCE) |
| `BASEFEE` | 0x48 | 2 | Current block's baseFeePerGas (EIP-3198) |

#### Memory Operations

| Opcode | Byte | Gas | Description |
|---|---|---|---|
| `MLOAD` | 0x51 | 3 + expansion | Load 32 bytes from memory at offset |
| `MSTORE` | 0x52 | 3 + expansion | Store 32 bytes to memory at offset |
| `MSTORE8` | 0x53 | 3 + expansion | Store 1 byte to memory at offset |
| `MSIZE` | 0x59 | 2 | Size of active memory in bytes |

#### Storage Operations

| Opcode | Byte | Gas | Description |
|---|---|---|---|
| `SLOAD` | 0x54 | 100 or 2100 | Load a value from contract storage (warm/cold) |
| `SSTORE` | 0x55 | 100–20000 | Store a value in contract storage (complex pricing) |

#### Control Flow

| Opcode | Byte | Gas | Description |
|---|---|---|---|
| `JUMP` | 0x56 | 8 | Unconditional jump to a `JUMPDEST` |
| `JUMPI` | 0x57 | 10 | Conditional jump (jump if top of stack ≠ 0) |
| `JUMPDEST` | 0x5B | 1 | Marks a valid jump destination |
| `PC` | 0x58 | 2 | Current value of the program counter |
| `STOP` | 0x00 | 0 | Halt execution successfully |
| `RETURN` | 0xF3 | 0 | Halt execution and return data |
| `REVERT` | 0xFD | 0 | Halt, revert all state changes, return error data |
| `INVALID` | 0xFE | all gas | Invalid opcode — consumes all remaining gas and reverts |
| `SELFDESTRUCT` | 0xFF | 5000+ | Destroy contract, send ETH to target address |

#### Stack Manipulation

| Opcode | Byte | Gas | Description |
|---|---|---|---|
| `PUSH1`–`PUSH32` | 0x60–0x7F | 3 | Push 1 to 32 bytes as a value onto the stack |
| `DUP1`–`DUP16` | 0x80–0x8F | 3 | Duplicate stack item N (1 = top) |
| `SWAP1`–`SWAP16` | 0x90–0x9F | 3 | Swap top of stack with item N+1 |
| `POP` | 0x50 | 2 | Discard top of stack |

#### Logging (Events)

| Opcode | Byte | Gas | Description |
|---|---|---|---|
| `LOG0` | 0xA0 | 375 + data | Emit log with no topics |
| `LOG1` | 0xA1 | 375 + 375 + data | Emit log with 1 topic |
| `LOG2` | 0xA2 | 375 + 750 + data | Emit log with 2 topics |
| `LOG3` | 0xA3 | 375 + 1125 + data | Emit log with 3 topics |
| `LOG4` | 0xA4 | 375 + 1500 + data | Emit log with 4 topics |

#### Call Operations

| Opcode | Byte | Description |
|---|---|---|
| `CALL` | 0xF1 | Call another contract or send ETH — transfers value, has its own context |
| `DELEGATECALL` | 0xF4 | Call another contract's code, but execute in *caller's* storage/context |
| `STATICCALL` | 0xFA | Call another contract in read-only mode — state changes forbidden |
| `CALLCODE` | 0xF2 | Deprecated predecessor to DELEGATECALL |
| `CREATE` | 0xF0 | Deploy a new contract (address derived from sender + nonce) |
| `CREATE2` | 0xF5 | Deploy a new contract with deterministic address (EIP-1014) |
| `RETURN` | 0xF3 | Return data to the caller and end execution |
| `REVERT` | 0xFD | Return error data, revert all state changes |

### Reading Bytecode: A Tiny Example

Here is the compiled bytecode for the absolute simplest possible contract — one that does nothing but return the number 42:

```
Solidity:
  contract Answer { function get() public pure returns (uint256) { return 42; } }

Relevant runtime bytecode segment (simplified):
  PUSH1 0x2a    → push 42 (0x2a hex) onto the stack
  PUSH1 0x00    → push memory offset 0
  MSTORE        → store 42 at memory[0..31]
  PUSH1 0x20    → push 32 (the size of the return value)
  PUSH1 0x00    → push memory offset 0
  RETURN        → return 32 bytes starting at memory[0]
```

This is exactly what the EVM processes, one opcode at a time, decrementing the gas counter with each step.

---

## 5. How Smart Contracts Work

### From Solidity to Bytecode to On-Chain

Smart contracts are typically written in **Solidity** (a high-level language with syntax similar to JavaScript/C++), though other languages like **Vyper** and **Yul** (an intermediate language) exist.

The development pipeline:

```
┌────────────────┐     Solidity Compiler     ┌────────────────────────────────────┐
│  MyToken.sol   │  ─────────────────────▶   │  Compilation Output                │
│                │      (solc, hardhat,       │                                    │
│  High-level    │       foundry, etc.)       │  Bytecode (hex):                   │
│  Solidity      │                            │  0x608060405234801561001057600...  │
│  source code   │                            │                                    │
│                │                            │  ABI (JSON):                       │
│  Human-        │                            │  [{"name":"transfer","inputs":...}]│
│  readable      │                            │                                    │
│                │                            │  Source maps, debug info, etc.     │
└────────────────┘                            └────────────────────────────────────┘
```

The compiler output has two critical parts:
1. **EVM bytecode** — the raw machine code that the EVM executes.
2. **ABI (Application Binary Interface)** — a JSON description of the contract's public interface (functions, events, errors). Used by callers to encode/decode calls correctly.

### Deployment Transaction

A contract is deployed by sending a special transaction where the `to` field is **empty (null)**:

```json
{
  "to":    null,
  "data":  "0x608060405234801561001057600080fd...",
  "value": "0x0",
  "gas":   "0x493E0"
}
```

The `data` field contains:
1. The contract's **creation bytecode** (the "init code") — a special piece of code that runs only at deployment time.
2. Any **constructor arguments**, ABI-encoded and appended at the end.

### Init Code vs. Runtime Code

This is a subtle but important distinction:

| | Init Code (creation bytecode) | Runtime Code (deployed bytecode) |
|---|---|---|
| **When it runs** | Only once, during deployment | Every time the contract is called |
| **Purpose** | Sets up initial state, runs the constructor, returns the runtime code | Handles all external calls |
| **Stored on-chain?** | No — executed and discarded | Yes — stored permanently |
| **What it returns** | The runtime bytecode itself | Return data of the specific call |

When you deploy a contract:
1. The EVM runs the init code.
2. The init code executes the constructor (sets initial storage values, etc.).
3. The init code returns the runtime bytecode.
4. The EVM stores the returned runtime bytecode at the new contract address.
5. All future calls to that address execute the runtime bytecode.

---

## 6. Contract Deployment Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│  STEP 1: WRITE AND COMPILE                                              │
│                                                                         │
│  Developer writes MyToken.sol in Solidity                              │
│  Compiler produces: init bytecode + runtime bytecode + ABI             │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  STEP 2: CREATE DEPLOYMENT TRANSACTION                                  │
│                                                                         │
│  { to: null, data: <init_bytecode + abi_encoded_constructor_args>,     │
│    value: 0, gasLimit: <estimated>, maxFeePerGas: ..., ... }           │
│  Sign with deployer's private key                                       │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  STEP 3: EVM PROCESSES THE TRANSACTION                                  │
│                                                                         │
│  a) Recognizes to == null → this is a deployment                       │
│  b) Computes the new contract address:                                  │
│       addr = keccak256(rlp([sender, nonce]))[12:]                      │
│  c) Creates a new account at that address (balance=0, nonce=1)         │
│  d) Executes the init bytecode in a new EVM context                    │
│     - Constructor runs, sets initial storage values                    │
│     - Init code returns the runtime bytecode                           │
│  e) Stores the returned runtime bytecode as the account's code         │
│  f) Sets codeHash = keccak256(runtime_bytecode)                        │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  STEP 4: CONTRACT NOW LIVES ON-CHAIN                                    │
│                                                                         │
│  Address: 0xA0b8...eB48  (deterministic, computed in step 3b)          │
│  Code: stored permanently at that address                               │
│  Storage: initial values set by constructor are now persistent          │
│  Receipt: contractAddress field contains the new address                │
│                                                                         │
│  Anyone can now send transactions to 0xA0b8...eB48 to interact         │
│  with the contract.                                                     │
└─────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  STEP 5: (OPTIONAL) VERIFY SOURCE CODE                                  │
│                                                                         │
│  Upload the Solidity source to Etherscan or Sourcify.                  │
│  The verifier recompiles it and checks the bytecode matches.           │
│  If it matches, users can read and audit the source code.              │
│  Unverified contracts show only raw bytecode — hard to audit.          │
└─────────────────────────────────────────────────────────────────────────┘
```

### CREATE2: Deterministic Addresses

`CREATE2` (EIP-1014) is an alternative to `CREATE`. Instead of deriving the address from the deployer's nonce, it derives it from:

```
address = keccak256(0xFF || deployerAddress || salt || keccak256(initCode))[12:]
```

This means you can **compute the contract's address before it is deployed** — as long as you know the deployer address, the salt, and the init code. This is the basis of:

- **Counterfactual deployments** (used in state channels and L2s).
- **Upgradeable proxy patterns** where a proxy is deployed at a predictable address.
- **Factory contracts** that deploy instances at predictable addresses based on user-supplied salts.

---

## 7. The ABI: Application Binary Interface

### What Is the ABI?

The ABI is a JSON document that describes a contract's public interface — its functions, events, errors, and their parameter types. It is the contract between a contract's callers and the contract itself about how data is encoded and decoded.

```json
[
  {
    "type": "function",
    "name": "transfer",
    "stateMutability": "nonpayable",
    "inputs": [
      { "name": "to",    "type": "address" },
      { "name": "value", "type": "uint256" }
    ],
    "outputs": [
      { "name": "", "type": "bool" }
    ]
  },
  {
    "type": "event",
    "name": "Transfer",
    "inputs": [
      { "name": "from",  "type": "address", "indexed": true  },
      { "name": "to",    "type": "address", "indexed": true  },
      { "name": "value", "type": "uint256", "indexed": false }
    ]
  }
]
```

### Function Selectors

When you call a contract function, the first 4 bytes of the `data` field identify which function to call. This is the **function selector**:

```
selector = keccak256("transfer(address,uint256)")[0:4]
         = keccak256("transfer(address,uint256)") first 4 bytes
         = 0xa9059cbb
```

Note: the selector is computed from the **function signature** — the function name and its parameter types in canonical form, with no spaces except where unavoidable, with no parameter names. The exact format matters.

### ABI Encoding

After the 4-byte selector, the arguments are ABI-encoded. The EVM itself has no knowledge of the ABI — it just sees raw bytes. The ABI encoding/decoding is done by the calling application (wallet, dApp, SDK) and decoded by the contract's Solidity code.

Example: calling `transfer(0xBob, 1000000000000000000)` (transfer 1 ETH worth of tokens to Bob):

```
calldata = function_selector + abi_encoded_args

= 0xa9059cbb                                              ← selector for transfer(address,uint256)
  000000000000000000000000Bob_address_padded_to_32_bytes  ← address, zero-padded left
  0000000000000000000000000000000000000000000000000de0b6b3a7640000  ← 1e18, zero-padded
```

All ABI-encoded values are 32 bytes (256 bits) wide, zero-padded. Dynamic types (strings, bytes arrays) use an additional indirection layer.

### `stateMutability` Types

The ABI describes how a function interacts with state:

| `stateMutability` | Can read storage? | Can modify storage? | Can receive ETH? |
|---|---|---|---|
| `pure` | No | No | No |
| `view` | Yes | No | No |
| `nonpayable` | Yes | Yes | No |
| `payable` | Yes | Yes | Yes |

`pure` and `view` functions can be called without sending a transaction — they are executed locally on the node (via `eth_call`) and return a result instantly with no gas cost and no state change.

---

## 8. Function Calls: External, Internal, and Special

When contract code executes, it can invoke other code in several different ways, each with distinct semantics.

### External Calls (CALL)

```
Contract A  ──── CALL ────▶  Contract B
```

`CALL` creates a **new execution context** for the called contract:
- `msg.sender` in B = A's address.
- `msg.value` = ETH sent with the call (can be 0).
- B executes in its **own storage** — reads and writes go to B's storage, not A's.
- B has its own separate memory.
- B's stack is separate from A's.
- If B reverts, A can catch the failure and decide what to do (or A itself reverts).
- Gas is forwarded (minus a base cost); B can use it up.

**Re-entrancy:** The most famous smart contract vulnerability. If contract A calls contract B, B could immediately call back into A before A's state is updated. If A's balance check happens before balance deduction, B can drain A repeatedly. The fix: **Checks-Effects-Interactions pattern** — update all state before making external calls.

### Delegate Calls (DELEGATECALL)

```
Contract A  ──── DELEGATECALL ────▶  Contract B's code
              (but runs in A's context!)
```

`DELEGATECALL` is unusual and powerful:
- Contract B's **code** is executed.
- But execution happens in **Contract A's context**: A's storage, A's address, A's ETH balance.
- `msg.sender` and `msg.value` are **preserved** — they remain the original caller, not A.
- B's code reads and writes **A's storage slots**, not B's.

This is the foundation of **upgradeable proxy contracts**: a proxy contract stores all state and uses `DELEGATECALL` to execute logic from an implementation contract. To "upgrade," you just change which implementation address the proxy delegates to. The state in the proxy stays intact.

```
User ──▶ Proxy Contract (stores all state)
              │
              │ DELEGATECALL (runs implementation's code
              ▼              against proxy's storage)
         Implementation v1  (or v2 after upgrade)
```

**Storage collision danger:** If the proxy and implementation contract use the same storage slot for different variables, they will overwrite each other. EIP-1967 standardises specific "admin" storage slots using high, random-looking addresses to avoid collision with normal storage.

### Static Calls (STATICCALL)

```
Contract A  ──── STATICCALL ────▶  Contract B (read-only mode)
```

`STATICCALL` is identical to `CALL` but with one absolute restriction: **no state modifications are allowed**. Any opcode that would modify state (`SSTORE`, `LOG*`, `CREATE`, `SELFDESTRUCT`, any `CALL` with non-zero value) causes an immediate revert.

This guarantees that the call is **read-only** — it cannot change any state, even indirectly. Used when a contract needs to query another contract's state without the risk of side effects. Solidity automatically uses `STATICCALL` for `view` and `pure` function calls.

### Summary of Call Types

| | `CALL` | `DELEGATECALL` | `STATICCALL` |
|---|---|---|---|
| Executes whose code | Callee's | Callee's | Callee's |
| Runs in whose context | Callee's | **Caller's** | Callee's |
| `msg.sender` in callee | Caller's address | **Preserved** (original) | Caller's address |
| Can modify state | Yes | Yes (in **caller's** storage!) | **No** |
| Can receive ETH | Yes | No | No |
| Use case | Normal cross-contract calls | Proxy patterns, libraries | Read-only queries |

---

## 9. Common Smart Contract Patterns

### ERC-20: Fungible Tokens

The **ERC-20** standard (EIP-20) defines the interface that all fungible tokens on Ethereum must implement. "Fungible" means each unit is interchangeable — 1 USDC is equal to any other 1 USDC, just like dollars.

Required functions:

```solidity
interface IERC20 {
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function transfer(address to, uint256 amount) external returns (bool);
    function allowance(address owner, address spender) external view returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(address from, address to, uint256 amount) external returns (bool);

    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
}
```

Key mechanism — **allowances**: Rather than giving your tokens to a contract, you `approve` it to spend up to a certain amount on your behalf. The contract calls `transferFrom` to pull the approved amount. This two-step pattern avoids needing to physically transfer tokens to an intermediary.

Notable ERC-20 tokens: USDC, USDT, DAI, WETH (Wrapped ETH), UNI, LINK — essentially every DeFi token.

### ERC-721: Non-Fungible Tokens (NFTs)

The **ERC-721** standard defines non-fungible tokens. Each token has a unique `tokenId` and distinct ownership. No two tokens are the same — each represents a unique asset (digital art, a game item, a deed, an identity credential).

Core interface:

```solidity
interface IERC721 {
    function ownerOf(uint256 tokenId) external view returns (address);
    function transferFrom(address from, address to, uint256 tokenId) external;
    function approve(address to, uint256 tokenId) external;
    function getApproved(uint256 tokenId) external view returns (address);
    function setApprovalForAll(address operator, bool approved) external;
    function isApprovedForAll(address owner, address operator) external view returns (bool);
    function safeTransferFrom(address from, address to, uint256 tokenId) external;

    event Transfer(address indexed from, address indexed to, uint256 indexed tokenId);
    event Approval(address indexed owner, address indexed approved, uint256 indexed tokenId);
    event ApprovalForAll(address indexed owner, address indexed operator, bool approved);
}
```

The `tokenURI` function (from the optional metadata extension) returns a URL pointing to JSON metadata describing the token (name, image, attributes).

### ERC-1155: Multi-Token Standard

**ERC-1155** is a more flexible standard that handles both fungible and non-fungible tokens in a single contract. Instead of one token per contract (ERC-20) or one unique token per ID (ERC-721), ERC-1155 allows:

- **Fungible tokens:** ID 1 = Gold coins (10,000 supply, all interchangeable).
- **Non-fungible tokens:** ID 5001 = unique Sword of Destiny #1.
- **Semi-fungible tokens:** ID 100 = Concert ticket (fungible before the event; each is identical; after use, they become uniquely "used").

Key advantage: **batch transfers** — send multiple different token types to multiple recipients in a single transaction, saving significant gas vs. multiple ERC-20/ERC-721 transfers.

### Upgradeable Proxy Patterns

Because deployed contract code is immutable, upgradeability requires an architectural workaround:

```
┌─────────────────────┐         ┌──────────────────────────┐
│  Proxy Contract     │         │  Implementation Contract  │
│                     │         │                           │
│  Stores all state   │─────────│  Contains all logic       │
│  Receives all calls │ DELEGATE│  (no state, just code)    │
│                     │  CALL   │                           │
│  implementation =   │────────▶│  function transfer() {    │
│    0xImplV1...      │         │    // logic here          │
└─────────────────────┘         │  }                        │
                                └──────────────────────────┘

To upgrade: update proxy's `implementation` pointer to 0xImplV2...
```

Common proxy patterns:
- **Transparent Proxy (EIP-1967):** Admin calls go to proxy; user calls delegate to implementation.
- **UUPS (EIP-1822):** Upgrade logic lives in the implementation, making the proxy simpler and cheaper.
- **Beacon Proxy:** Multiple proxies share one "beacon" that points to the current implementation — upgrade once, all proxies update.

---

## 10. Smart Contract Limitations

The EVM's sandboxed, deterministic nature is its greatest strength — and also imposes hard limitations that every smart contract developer must work around.

### No Native Randomness

The EVM is fully deterministic. Every node must execute code and get the same result. This means there is **no built-in source of randomness**.

Common (bad) approaches and why they fail:
- `block.timestamp` — validators can manipulate this within a ~15-second window.
- `blockhash` — only available for the last 256 blocks; miners/validators can influence which blocks they produce.

The correct approach: **Verifiable Random Functions (VRFs)** via oracles like **Chainlink VRF**. A VRF generates a random number off-chain with a cryptographic proof that it was generated fairly. The proof is verified on-chain, and only then is the random number used. This is how most NFT projects do fair minting and most games do fair randomization.

Post-Merge, the `PREVRANDAO` opcode (formerly `DIFFICULTY`) returns a value from the beacon chain's randomness beacon — more reliable than blockhash but still manipulable by validators under adversarial conditions. Suitable for low-stakes randomness, not for high-value lotteries.

### No External Data (Oracles)

A contract cannot make HTTP requests. It cannot read from an API. It cannot know the current ETH/USD price, the weather in Tokyo, or the result of a sports match. The EVM is isolated from the outside world by design.

The solution: **Oracles** — trusted (or decentralized) services that fetch real-world data and post it on-chain via a transaction. The most popular is **Chainlink**, which uses a decentralized network of oracle nodes that each fetch and sign data, with the on-chain result being an aggregated, tamper-resistant feed.

```
Real-world data           Chainlink Oracle Network          Smart Contract
(ETH price: $3,200)  ──▶  (multiple nodes fetch,      ──▶  reads price
                           aggregate, and post              from oracle
                           on-chain)                        contract
```

### No Scheduling / Automation

A contract cannot schedule itself to run at a specific time. There is no `setInterval` equivalent. **A contract only runs when a transaction triggers it.**

Solutions:
- **Chainlink Automation** (formerly Keepers): Decentralized network of bots that monitor conditions and trigger contract functions when criteria are met.
- **Gelato Network**: Another decentralized automation service.
- **Centralized bots**: Many DeFi protocols use their own off-chain bots for non-critical automation (liquidations, reward distribution) while accepting the associated trust assumption.

### Immutability After Deployment

Once deployed, a contract's code **cannot be changed**. This is a feature (you can audit and trust the code), but it's also a challenge when bugs are discovered or requirements change.

Solutions:
- **Proxy patterns** (DELEGATECALL): Separate logic from state; upgrade the logic contract.
- **Parameterization**: Make behaviour configurable via parameters stored in storage (admin can update parameters without upgrading code).
- **Immutable code as a feature**: Some protocols intentionally make their contracts non-upgradeable to maximize trust (e.g., Uniswap v2 core contracts).

### Block Gas Limit Constraints

A single transaction cannot consume more gas than the block's `gasLimit`. This caps how much computation any single call can do. Algorithms that are O(n) in storage operations become problematic at large n — you may need to process data in chunks across multiple transactions.

---

## 11. Precompiles

### What Are Precompiles?

**Precompiles** are special "built-in" contracts at fixed, low-numbered addresses (starting from `0x0000...0001`) that implement cryptographic and mathematical operations too expensive to implement efficiently in EVM bytecode.

Instead of running EVM opcodes, calling a precompile address triggers **native code** in the client implementation (Java in Besu, Go in Geth, etc.). This is dramatically faster and cheaper than implementing the same operation in bytecode.

They look like regular contracts — you call them with a `CALL` or `STATICCALL` opcode — but they have no bytecode; the client handles the call specially.

### Precompile Reference Table

| Address | Name | EIP | Gas Cost | Description |
|---|---|---|---|---|
| `0x01` | `ecRecover` | — | 3,000 | Recover the signer address from an ECDSA signature `(v, r, s)`. Used in many contracts to verify signed messages. |
| `0x02` | `SHA256` | — | 60 + 12/word | Compute SHA-256 hash. Used for Bitcoin-compatible operations and some cross-chain bridges. |
| `0x03` | `RIPEMD160` | — | 600 + 120/word | Compute RIPEMD-160 hash. Used for Bitcoin address compatibility. |
| `0x04` | `identity` | — | 15 + 3/word | Copy input to output (data copy). Used as a cheap way to copy memory. |
| `0x05` | `modexp` | EIP-198 | complex | Modular exponentiation: `base^exp mod modulus`. Used in RSA verification and other number-theoretic cryptography. |
| `0x06` | `bn128Add` | EIP-196 | 150 | Point addition on the BN128 (alt_bn128) elliptic curve. Used in ZK-SNARK verification. |
| `0x07` | `bn128Mul` | EIP-196 | 6,000 | Scalar multiplication on BN128. Used in ZK-SNARK verification. |
| `0x08` | `bn128Pairing` | EIP-197 | complex | Bilinear pairing check on BN128. The core of efficient on-chain ZK-SNARK proof verification. |
| `0x09` | `blake2f` | EIP-152 | variable | BLAKE2b compression function. Enables Ethereum contracts to verify Zcash transactions and other BLAKE2-based proofs. |
| `0x0a` | `kzgPointEvaluation` | EIP-4844 | 50,000 | Verify a KZG polynomial commitment proof. Used by L2 rollups to prove blob data correctness. |

### How Besu Implements Precompiles

In Besu's Java codebase, each precompile is a class implementing the `PrecompileContract` interface. The EVM's `CALL` handler checks: "Is the target address a precompile?" If yes, it dispatches to the corresponding Java class rather than loading and executing bytecode.

This means:
- **Exact gas accounting** — Besu computes gas costs using the same formulas as the EVM spec.
- **Correctness** — Besu's implementations are tested against the official Ethereum test vectors.
- **Extensibility** — Private Besu networks can add custom precompiles at additional addresses for application-specific native-speed operations.

### The Importance of Pairing Precompiles (0x06–0x08)

The BN128 pairing precompiles at `0x06`, `0x07`, and `0x08` are critically important for the Ethereum ecosystem because they enable **efficient on-chain ZK-SNARK verification**.

ZK-SNARKs (Zero-Knowledge Succinct Non-Interactive Arguments of Knowledge) are cryptographic proofs that allow one party to prove they know something without revealing what they know, and do so with a small, quickly-verifiable proof.

Without these precompiles, verifying a ZK-SNARK on-chain would cost millions of gas (computationally intractable). With them, verification costs a few hundred thousand gas — expensive but feasible.

This is what enables ZK-rollups (Linea, zkSync, Polygon zkEVM, Scroll) to post compact validity proofs to Ethereum and have the L1 verify them on-chain.

---

## 12. Gas Metering in the EVM

### Why Gas Exists

Without a cost for computation, the network would be vulnerable to **DoS attacks**: an attacker could submit a transaction with an infinite loop, forcing every node to run forever. Gas solves this by:

1. **Capping computation:** A transaction can consume at most `gasLimit` gas. When gas runs out, execution stops immediately.
2. **Pricing computation appropriately:** More expensive operations (disk I/O for storage) cost more gas than cheap operations (arithmetic). This prevents attacks that abuse cheap opcodes to consume disproportionate resources.
3. **Compensating nodes:** Gas fees, paid in ETH, compensate the validator (and via the base fee burn mechanism, all ETH holders) for the resources consumed.

### How Gas Metering Works Step by Step

```
Transaction arrives with: gasLimit = 100,000, maxFeePerGas = 10 gwei

Initial gas counter: 100,000

EVM begins execution:
  Opcode: PUSH1 0x02   → cost: 3 gas   → counter: 99,997
  Opcode: PUSH1 0x03   → cost: 3 gas   → counter: 99,994
  Opcode: ADD          → cost: 3 gas   → counter: 99,991
  Opcode: PUSH1 0x00   → cost: 3 gas   → counter: 99,988
  Opcode: SSTORE       → cost: 20,000  → counter: 79,988  ← expensive!
  ...continues...
  Opcode: STOP         → cost: 0 gas   → counter: 79,341

Gas used = 100,000 - 79,341 = 20,659
Gas refund = 79,341 gas units returned to the sender
Fee paid = 20,659 × effectiveGasPrice (in wei)
```

### Out-of-Gas Exception

If the gas counter reaches 0 during execution:

1. Execution **stops immediately** — the current opcode does not complete.
2. All state changes made during this transaction are **reverted** — the state returns to what it was before the transaction started.
3. The transaction is still **included in the block** — it occupies a slot and consumes the full `gasLimit` of gas (all gas is charged, none refunded).
4. The receipt records `status: 0` (failure) and `gasUsed = gasLimit`.
5. The sender has paid the maximum possible gas fee for a failed transaction.

This is why setting `gasLimit` too low is risky — you pay for the work done before the gas ran out, achieve nothing, and have to try again.

### Gas Refunds

Some operations earn partial gas refunds:
- **Clearing storage** (`SSTORE` that sets a value to 0 from non-zero): up to 4,800 gas refund.
- **SELFDESTRUCT** (deprecated/discouraged): formerly offered a large refund; heavily nerfed in EIP-3529 (London fork) because it was being exploited by "gas tokens."

Post-EIP-3529, refunds are capped at **20% of gas used**. This prevents gaming the refund mechanism while still incentivising storage cleanup.

### Gas Costs: A Hierarchy of Expense

Understanding the relative cost of operations is essential for writing gas-efficient contracts:

```
CHEAPEST ─────────────────────────────────────────────── MOST EXPENSIVE

Arithmetic   Reading       Reading       External    Writing to
(ADD, MUL)   calldata      memory        call        storage
3 gas        3 gas/word    3 gas/word    ~2,100 gas  20,000 gas (first write)
                                         minimum     5,000 gas (subsequent)


Key principles for gas-efficient contracts:
 ✓ Use calldata instead of memory for function arguments (if read-only)
 ✓ Pack multiple small values into a single storage slot (uint128 + uint128 in one slot)
 ✓ Cache storage reads in memory if reading the same slot multiple times
 ✓ Prefer events over storage for data that only needs to be read off-chain
 ✓ Use mappings over arrays for large datasets (arrays require shifting)
 ✓ Short-circuit expensive checks with cheap checks first
```

### Dynamic Gas Costs

Some opcodes have **dynamic** gas costs that depend on inputs:

- **`EXP`:** 10 + 50 per byte in the exponent. `2^256` is 10 + 50×32 = 1,610 gas.
- **`KECCAK256`:** 30 + 6 per 32-byte word of input.
- **Memory expansion:** Quadratic — memory[0..31] costs 3 gas; memory[0..1023] costs ~36 gas; memory[0..32767] costs ~512 gas; grows quadratically to discourage huge memory allocations.
- **`CALL`/`DELEGATECALL`/`STATICCALL`:** ~2,100 base if the callee is accessed cold (first time in this transaction); ~100 if warm.

### The EIP-1559 Gas Price Calculation

For type-2 transactions, the actual gas price paid is:

```
effectiveGasPrice = min(maxFeePerGas, baseFeePerGas + maxPriorityFeePerGas)

validator receives: effectiveGasPrice - baseFeePerGas = min(maxPriorityFeePerGas, maxFeePerGas - baseFeePerGas)
protocol burns:     baseFeePerGas × gasUsed
sender pays total:  effectiveGasPrice × gasUsed  (in wei)
unused gas refund:  (gasLimit - gasUsed) × effectiveGasPrice
```

---

## 13. The Extractable EVM: Besu's EVM as a Library

One of Besu's notable architectural decisions is that its EVM implementation is packaged as a **standalone Java library** — the `evm` module — that can be used independently of the full Besu node.

### What This Means

```
Standard usage:
  Full Besu Node
  ├── Networking (DevP2P, LibP2P)
  ├── JSON-RPC API
  ├── Consensus integration (Engine API)
  ├── State management (RocksDB, bonsai trie)
  └── EVM (executes transactions)

Standalone library usage:
  Your Java Application
  └── besu-evm library (just the EVM, no network, no node)
       ├── Execute bytecode in isolation
       ├── Simulate transactions against a custom state
       ├── Run EVM tests
       └── Build custom tooling
```

### Use Cases for the Standalone EVM

- **Testing frameworks:** Execute smart contract code in unit tests without spinning up a full node.
- **Simulation:** "What would happen if this transaction executed against this state?" — without broadcasting anything.
- **Layer 2 systems:** An L2 sequencer can use the Besu EVM library to execute L2 transactions and compute state roots locally before posting to L1.
- **Static analysis tools:** Analyse bytecode behaviour without network connectivity.
- **Custom precompiles for private networks:** Add domain-specific native operations (e.g., a custom hash function, a regulatory compliance check) to a private Besu network's EVM without modifying the full node codebase.

### Example: Using the Besu EVM Library

```java
// Add to your Maven/Gradle project:
// org.hyperledger.besu:evm:<version>

// Create a simple EVM instance
EVM evm = MainnetEVMs.london(EvmConfiguration.DEFAULT);

// Provide a code executor with custom state
SimpleWorld world = new SimpleWorld();
world.createAccount(Address.fromHexString("0xdeadbeef..."), 0, Wei.of(1_000_000));

// Execute bytecode
MessageFrame frame = MessageFrame.builder()
    .type(MessageFrame.Type.MESSAGE_CALL)
    .worldUpdater(world.updater())
    .initialGas(100_000L)
    .contract(contractAddress)
    .inputData(Bytes.fromHexString("0xa9059cbb..."))  // transfer() calldata
    .build();

evm.runToHalt(frame, OperationTracer.NO_TRACING);

System.out.println("Status: " + frame.getState());
System.out.println("Gas used: " + (100_000 - frame.getRemainingGas()));
```

### EVM Upgrades and Besu

Every Ethereum hard fork may change the EVM: new opcodes are added, gas costs are repriced, opcodes are deprecated. The Besu EVM library is versioned accordingly:

- `MainnetEVMs.frontier()` — original EVM
- `MainnetEVMs.berlin()` — includes EIP-2929 (cold/warm storage costs)
- `MainnetEVMs.london()` — includes EIP-3529 (refund cap), EIP-3541 (0xEF prefix reserved)
- `MainnetEVMs.shanghai()` — includes EIP-3855 (PUSH0 opcode)
- `MainnetEVMs.cancun()` — includes EIP-1153 (transient storage: TLOAD/TSTORE), EIP-5656 (MCOPY), EIP-4788 (beacon root access)

This means developers can target specific hard fork levels for compatibility testing and simulation, choosing the exact EVM rules they want to apply.

---

## 14. Recap Checklist

After reading this guide, you should be able to confidently say:

- [ ] I understand that the EVM is a sandboxed, deterministic, stack-based virtual machine that runs on every full Ethereum node
- [ ] I can describe the EVM as a state machine: current state + transaction input → new state
- [ ] I know the four storage areas of the EVM: Stack (256-bit words, max 1024), Memory (temporary byte array), Storage (persistent key-value, expensive), and Calldata (read-only input)
- [ ] I understand what EVM opcodes are and can give examples from the arithmetic, storage, control flow, and call categories
- [ ] I know the difference between init code (runs at deployment, returns runtime code) and runtime code (stored on-chain, runs on every call)
- [ ] I understand what the ABI is, what a function selector is (first 4 bytes of keccak256 of function signature), and how arguments are ABI-encoded
- [ ] I can explain the difference between CALL (callee's context), DELEGATECALL (caller's context), and STATICCALL (read-only)
- [ ] I know the ERC-20 (fungible), ERC-721 (non-fungible), and ERC-1155 (multi-token) standards and what each is used for
- [ ] I can list the four main smart contract limitations: no randomness, no external data, no scheduling, immutable once deployed — and the workaround for each
- [ ] I know what precompiles are (native-code contracts at fixed low addresses) and can name at least 5 with their purposes
- [ ] I understand gas metering: each opcode has a cost, gas counter decrements, hitting 0 causes a revert, unused gas is refunded
- [ ] I understand that storage operations are the most expensive (~20,000 gas for a new write vs. 3 gas for arithmetic)
- [ ] I know that Besu's EVM is available as a standalone Java library, usable outside of a full node

---

## 15. Check Your Understanding

Try to answer these without looking back:

1. **The EVM stack can hold a maximum of 1024 items, each 256 bits wide. A recursive function in a smart contract keeps pushing data onto the stack without popping. What happens when the 1025th item is pushed? Is gas refunded?**

2. **A proxy contract uses `DELEGATECALL` to an implementation contract. The implementation contract has a `uint256 public count` at storage slot 0. The proxy contract has a `address public implementation` at storage slot 0. Describe exactly what happens to the proxy's `implementation` variable when someone calls `increment()` on the proxy, which increments `count` via DELEGATECALL.**

3. **A Solidity function is marked `view`. When a dApp calls this function via `eth_call` (not a transaction), does it consume gas? Does it appear in a block? Can it modify storage? Explain each answer.**

4. **Why does writing to storage cost 20,000 gas for a zero-to-nonzero write, but only 5,000 gas for a nonzero-to-nonzero write? What real-world resource does this pricing reflect?**

5. **A ZK-rollup on Ethereum posts a validity proof on-chain for verification. Which precompile address(es) are most likely used during verification, and why couldn't this be done efficiently with pure EVM bytecode?**

6. **A developer wants to call `transfer(address,uint256)` on an ERC-20 contract. Without a library, they need to manually construct the calldata. Walk through the exact steps: how is the function selector computed, and how are the two arguments encoded? What is the total size of the calldata in bytes?**

---

## Next Up

**[08 — Solidity Basics →](08_solidity_basics.md)**

Now that you understand the EVM's internals — how it executes bytecode, what opcodes do, how gas is metered — the next guide moves up the stack to **Solidity**: Ethereum's most popular smart contract programming language. We'll cover data types, state variables, functions, events, modifiers, inheritance, and how to write your first contract from scratch.