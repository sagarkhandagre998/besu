# 04 — Consensus Mechanisms: PoW, PoS, and PoA

> **Level:** Intermediate | **Read time:** ~30 minutes
> **Prerequisites:** [03 — How Blockchain Works Internally](03_how_blockchain_works.md)

---

## Table of Contents

1. [What Is Consensus?](#1-what-is-consensus)
2. [The Byzantine Generals Problem](#2-the-byzantine-generals-problem)
3. [Proof of Work (PoW)](#3-proof-of-work-pow)
4. [Proof of Stake (PoS)](#4-proof-of-stake-pos)
5. [Proof of Authority (PoA)](#5-proof-of-authority-poa)
   - [QBFT](#51-qbft-quorum-byzantine-fault-tolerant)
   - [IBFT 2.0](#52-ibft-20-istanbul-byzantine-fault-tolerant)
   - [Clique](#53-clique)
6. [Consensus Comparison Table](#6-consensus-comparison-table)
7. [Why Besu Supports Multiple Consensus Mechanisms](#7-why-besu-supports-multiple-consensus-mechanisms)
8. [Recap Checklist](#8-recap-checklist)
9. [Check Your Understanding](#9-check-your-understanding)

---

## 1. What Is Consensus?

Imagine a spreadsheet that thousands of strangers all maintain simultaneously. Nobody is in charge. Anyone can try to write new data. Many participants may be acting in bad faith — trying to corrupt the data, double-spend money, or simply lie.

How do all the honest participants agree on a single, correct version of the spreadsheet?

That is the consensus problem. **Consensus** is the process by which a distributed network of nodes — without any central authority — arrives at a single agreed-upon truth.

For a blockchain, consensus answers one specific question at a time:

> **"What is the next valid block to append to the chain?"**

Every consensus mechanism answers this question differently. They make different trade-offs across three dimensions:

| Dimension | What it means |
|---|---|
| **Security** | How hard is it for an attacker to take control? |
| **Performance** | How fast can the network reach agreement and produce blocks? |
| **Decentralization** | How many independent parties are involved? |

These three properties are in constant tension. Optimizing for any two tends to weaken the third — a trade-off sometimes called the **Blockchain Trilemma**.

---

## 2. The Byzantine Generals Problem

Before diving into specific mechanisms, we need to understand the theoretical problem they all solve: the **Byzantine Generals Problem**, first described by computer scientists Leslie Lamport, Robert Shostak, and Marshall Pease in 1982.

### The Setup

Picture several divisions of a Byzantine army camped outside an enemy city. Each division is commanded by a general. The generals can only communicate by messenger. They must agree on a common plan: **attack** or **retreat**. If they attack at different times, they will be defeated.

The complication: **some generals may be traitors**. Traitors will send different messages to different generals — telling some "attack" and others "retreat" — to cause confusion and disagreement.

```
         General A
            │
     ┌──────┼──────┐
     │      │      │
  General  General General
     B      C*      D
           (traitor)
```

General C* might tell A and B "attack" while telling D "retreat." This causes the honest generals to act on conflicting information and lose the battle.

### The Question

How can the honest generals reach a reliable agreement **even when some participants are actively lying?**

### The Answer

It turns out that if there are `f` traitors (Byzantine nodes), you need at least `3f + 1` total participants to guarantee consensus. In other words:

- With **1 traitor**, you need at least **4 generals total** to still reach correct consensus.
- With **2 traitors**, you need at least **7 generals**.
- The network can tolerate at most **⌊(n-1)/3⌋** Byzantine (malicious) nodes.

This is the foundation of **Byzantine Fault Tolerance (BFT)** — a consensus guarantee that holds even when up to 1/3 of participants are actively malicious.

### Why It Matters for Blockchain

- **PoW** solves Byzantine fault tolerance through computational cost (lying is expensive).
- **PoS** solves it through economic stake (lying causes your stake to be slashed/burned).
- **BFT-based PoA algorithms** (QBFT, IBFT 2.0) directly implement BFT protocols where known validators vote, requiring a supermajority (≥2/3) to agree.

---

## 3. Proof of Work (PoW)

### The Core Idea

In Proof of Work, nodes called **miners** compete in a computational race to produce the next valid block. The winner earns the right to add their block to the chain and collect a **block reward**.

The race is deliberately designed to be:
- **Hard to win** — requires real energy and computation.
- **Easy to verify** — anyone can instantly check the winner's answer.
- **Unpredictable** — you cannot shortcut the process; you must brute-force it.

### The Mining Puzzle

Every block header is hashed. The rule is:

```
keccak256(block_header) < target
```

The **target** is a threshold value. A valid block's hash must be numerically *smaller* than the target. Because hash outputs are essentially random, the only way to find a valid hash is to try billions of different inputs.

The **nonce** (number used once) in the block header is the variable miners change with each attempt:

```
Attempt 1: hash(header with nonce=0)       = 0xf3a2... → too large, try again
Attempt 2: hash(header with nonce=1)       = 0x8b1c... → too large, try again
Attempt 3: hash(header with nonce=2)       = 0x4f90... → too large, try again
...
Attempt 2,847,193,021: hash(header with nonce=2847193021) = 0x0000003a... → VALID! ✓
```

The miner broadcasts the winning block. Every other node checks the answer in a single hash operation — instant verification, but enormously expensive to find.

### Difficulty Adjustment

To keep block times consistent (roughly 10 minutes for Bitcoin, ~13 seconds for old Ethereum), the **target** adjusts automatically:

- If blocks are being found too quickly → raise the difficulty (lower the target threshold → harder to find valid hashes).
- If blocks are being found too slowly → lower the difficulty (raise the target → easier to find valid hashes).

Bitcoin adjusts every **2016 blocks** (~2 weeks). Ethereum (before The Merge) adjusted every block.

### Ethash: Ethereum's PoW Algorithm

Ethereum used the **Ethash** algorithm (originally called Dagger-Hashimoto), still used by Ethereum Classic. Key design goals:

- **ASIC-resistant (partially):** Ethash requires a large dataset (~1 GB) stored in memory. This makes it harder (though not impossible) for custom mining chips (ASICs) to dominate vs. consumer GPUs.
- **Memory-hard:** The bottleneck is memory bandwidth, not raw computation. This levels the playing field between GPUs and CPUs more than pure compute-bound algorithms.
- **DAG (Directed Acyclic Graph):** A large dataset regenerated every ~5 days (each "epoch"). Miners must keep this in GPU VRAM to mine efficiently.

### The 51% Attack

PoW security assumes that **honest miners control more than 50% of the network's total hash rate**. If an attacker controls >50%:

```
Honest chain:    B1 → B2 → B3 → B4 → B5  (public)
Attacker's chain: B1 → B2 → B3' → B4' → B5' → B6'  (secret, longer!)

Attacker:
  1. Sends 100 ETH to an exchange (on the honest chain)
  2. Exchange credits attacker's account
  3. Attacker withdraws (spends the funds elsewhere)
  4. Attacker reveals their secret longer chain
  5. Honest nodes switch to the longer chain (chain reorg)
  6. The transaction sending 100 ETH to the exchange is now gone
  → Double spend successful
```

This is why decentralization of hash power matters enormously in PoW. On small PoW networks, 51% attacks are feasible and have occurred in practice (Ethereum Classic, Bitcoin Gold, etc.).

### Trade-offs of PoW

| Advantage | Disadvantage |
|---|---|
| Proven security model (Bitcoin has run for 15+ years) | Enormous energy consumption |
| Highly decentralized (anyone with hardware can mine) | Slow finality (probabilistic) |
| No need to trust or know validators | Hardware waste (obsolete miners) |
| Sybil resistance via physical cost | ASIC centralization risk |

---

## 4. Proof of Stake (PoS)

### The Core Idea

Proof of Stake replaces the computational race with an economic one. Instead of spending electricity, participants **lock up (stake) cryptocurrency as collateral**. This stake is their "skin in the game."

The key insight: if a validator acts maliciously, their stake can be **slashed** (partially or fully destroyed as a penalty). Honest behavior is economically rational; dishonest behavior is economically devastating.

### How Ethereum's PoS Works (Post-Merge)

Ethereum switched from PoW to PoS in **"The Merge" on September 15, 2022** — one of the most significant technical events in blockchain history. Here is how it works:

#### Becoming a Validator

- Anyone who wants to participate as a validator deposits exactly **32 ETH** into the Beacon Chain deposit contract.
- They run two pieces of software:
  - An **execution client** (e.g., Besu) — handles transactions and the EVM.
  - A **consensus client** (e.g., Teku, Lighthouse, Prysm) — handles the PoS protocol.
- If a validator wants to stake more than 32 ETH, they run multiple validator instances.

#### Slots and Epochs

Time in Ethereum PoS is divided into fixed units:

```
1 Slot  = 12 seconds  (one potential block)
1 Epoch = 32 Slots    = 384 seconds ≈ 6.4 minutes

Timeline:
│←── Epoch 1 ────────────────────────────────────────────────────────────────────────────────►│
│ Slot 0 │ Slot 1 │ Slot 2 │ ... │ Slot 31 │ Slot 32 │ ... │ Slot 63 │ ← Epoch 2 starts
│ 12s    │ 12s    │ 12s    │     │ 12s     │ 12s     │     │ 12s     │
```

Every slot, one validator is **pseudo-randomly selected** as the **block proposer** for that slot. The remaining validators are divided into **committees** that vote (attest) on the proposed block.

#### Block Proposal and Attestation

```
Slot N:
  1. Assigned proposer builds a block and broadcasts it
  2. Committee members (attestors) receive the block
  3. Attestors check: Is this block valid? Does it build on the correct parent?
  4. Attestors broadcast their votes (attestations)
  5. If a majority of the committee votes for the block, it is included in the chain
  6. Proposer and attestors earn rewards in ETH
```

#### Finality via Checkpoints

At the end of each epoch, the last block is a **checkpoint**. The finalization process:

1. Validators vote on whether each checkpoint is valid.
2. When ≥2/3 of all staked ETH has voted for a checkpoint → it becomes **justified**.
3. When two consecutive checkpoints are justified → the earlier one is **finalized**.
4. A finalized block **cannot be reverted** without destroying at least 1/3 of all staked ETH.

```
Epoch 1           Epoch 2           Epoch 3
Checkpoint A      Checkpoint B      Checkpoint C
     │                 │                 │
     ●─────────────────●─────────────────●
     
After Epoch 2:  A is justified, B is justified
After Epoch 3:  A is FINALIZED (two justified checkpoints exist after it)
                B is justified, C is justified → B will be finalized next epoch
```

In practice, **finality is achieved in ~13 minutes (2 epochs)** under normal conditions.

#### Slashing

If a validator behaves maliciously (e.g., signs two different blocks for the same slot, or votes to finalize two conflicting checkpoints), they are **slashed**:

- A portion of their 32 ETH stake is immediately burned.
- They are forcibly ejected from the validator set.
- Additional penalties accumulate during the "slashing period."
- In the worst case, a validator can lose their entire 32 ETH.

This makes attacks enormously expensive — unlike PoW, where a failed attack just wastes electricity, in PoS it destroys capital.

### Trade-offs of PoS

| Advantage | Disadvantage |
|---|---|
| ~99.95% less energy than PoW | Validators need 32 ETH to participate directly (high barrier) |
| Economic penalties make attacks very costly | "Nothing at stake" concern (mitigated by slashing) |
| Stronger finality guarantees than PoW | Slightly more complex to implement correctly |
| Rewards proportional to stake, not hardware | Wealth concentration risk (more stake = more rewards) |

---

## 5. Proof of Authority (PoA)

### The Core Idea

Proof of Authority takes a fundamentally different approach: **validators are known, approved entities**. Instead of spending resources (compute or capital) to earn the right to produce blocks, validators are explicitly permissioned by the network operators.

PoA is **not suitable for public, permissionless blockchains** — because it requires trusting the validators, and there is no open entry. However, it is **ideal for private, consortium, and enterprise networks** where:

- All participants are known organizations (banks, supply chain partners, healthcare providers).
- Fast finality is required (seconds, not minutes).
- Energy efficiency matters.
- High throughput is needed (high transactions per second).

Besu supports three PoA algorithms: **QBFT**, **IBFT 2.0**, and **Clique**.

---

### 5.1 QBFT (Quorum Byzantine Fault Tolerant)

QBFT is the **recommended consensus algorithm for new Besu private networks**. It is a BFT-based protocol that provides strong guarantees even in the presence of malicious validators.

#### How QBFT Works

At a high level, each block goes through a multi-round voting protocol:

```
┌─────────────────────────────────────────────────────────────────┐
│  QBFT CONSENSUS ROUND FOR BLOCK N                               │
│                                                                 │
│  1. PROPOSE                                                     │
│     The designated proposer (rotates round-robin) builds        │
│     a block and broadcasts a PROPOSAL message                   │
│                                                                 │
│  2. PREPARE                                                     │
│     Each validator receives the proposal, validates it,         │
│     and broadcasts a PREPARE message (I received a valid block) │
│     Once a validator sees ≥2/3 PREPARE messages → move on      │
│                                                                 │
│  3. COMMIT                                                      │
│     Each validator broadcasts a COMMIT message                  │
│     (I am ready to commit this block)                           │
│     Once a validator sees ≥2/3 COMMIT messages → finalize       │
│                                                                 │
│  4. FINALIZE                                                    │
│     Block is added to the chain. COMMIT seals are embedded      │
│     in the block's extraData field as proof of consensus.       │
└─────────────────────────────────────────────────────────────────┘
```

If a round times out (e.g., the proposer is offline), validators enter a **round change** phase, elect a new proposer, and try again.

#### Fault Tolerance

- QBFT tolerates up to **⌊(n-1)/3⌋** faulty validators.
- With **4 validators**, it can tolerate **1 faulty** (malicious or offline) — the minimum useful setup.
- With **7 validators**, it can tolerate **2 faulty**.
- Network continues producing blocks as long as ≥2/3 of validators are honest and online.

```
Validators:   4   5   6   7   8   9   10
Max faulty:   1   1   1   2   2   2    3
```

#### Key Properties

- **Immediate finality:** Once a block is committed, it is final. No forks. No reorganizations.
- **No mining:** No computational puzzle. No wasted energy.
- **No staking:** Validators are approved by governance, not by depositing capital.
- **Minimum 4 validators:** Fewer than 4 cannot tolerate any faulty node and still maintain BFT guarantees.
- **Validator rotation:** New validators can be added or removed by vote of the existing validators.

#### Configuration in genesis.json

```json
"config": {
  "qbft": {
    "blockperiodseconds": 2,
    "epochlength": 30000,
    "requesttimeoutseconds": 4,
    "blockreward": "5000000000000000000"
  }
}
```

---

### 5.2 IBFT 2.0 (Istanbul Byzantine Fault Tolerant)

IBFT 2.0 is the predecessor to QBFT. It uses a very similar BFT protocol (three-phase: PRE-PREPARE → PREPARE → COMMIT) and provides the same **immediate finality** and **Byzantine fault tolerance** as QBFT.

#### IBFT 2.0 vs QBFT

| Property | IBFT 2.0 | QBFT |
|---|---|---|
| Finality | Immediate | Immediate |
| Fault tolerance | ⌊(n-1)/3⌋ | ⌊(n-1)/3⌋ |
| Min validators | 4 | 4 |
| Standard | Istanbul (EIP-650 era) | Improved / newer |
| Recommendation | Legacy | **Preferred for new networks** |

The main practical differences are minor protocol improvements in QBFT that make it more robust in edge cases and better aligned with evolving standards. For new Besu deployments, QBFT is always preferred.

#### Configuration in genesis.json

```json
"config": {
  "ibft2": {
    "blockperiodseconds": 2,
    "epochlength": 30000,
    "requesttimeoutseconds": 4
  }
}
```

---

### 5.3 Clique

Clique is the PoA algorithm originally designed for Ethereum's **Rinkeby testnet** (now deprecated). Unlike QBFT and IBFT 2.0, Clique does **not** implement BFT — it uses a simpler round-robin signing approach.

> ⚠️ **Clique was deprecated in Besu 25.12.0** and is no longer recommended for new networks. This section is included for historical awareness and to explain legacy setups.

#### How Clique Works

Validators (called **signers** in Clique) take turns signing blocks in a round-robin fashion:

```
Validators: [Alice, Bob, Carol, Dave]

Block 100: Alice signs  (her turn)
Block 101: Bob signs    (his turn)
Block 102: Carol signs  (her turn)
Block 103: Dave signs   (his turn)
Block 104: Alice signs  (round-robin restarts)
```

Validators that are out of turn can sign too, but in-turn signers get priority (their block has a higher "difficulty" value in Clique terms, so the network prefers it).

A signer is only allowed to sign **1 block every (n/2 + 1) blocks** (where n = number of signers). This prevents a single validator from monopolizing block production.

#### Fault Tolerance

Clique can tolerate up to **⌊(n-1)/2⌋** offline signers — significantly more fault tolerant than BFT algorithms in terms of liveness. However, this comes at the cost of not having Byzantine fault tolerance: Clique cannot handle **malicious** (actively lying) validators the way QBFT can.

```
Signers:      3    4    5    6    7
Max offline:  1    1    2    2    3
```

#### Finality in Clique

Clique does **not** provide immediate finality. Because there is no BFT voting round, a signed block could in theory be reorganized if a majority of validators went rogue. Finality is probabilistic — similar to PoW but much faster.

#### Why Clique Is Deprecated

- No Byzantine fault tolerance (cannot handle malicious validators).
- No immediate finality.
- QBFT and IBFT 2.0 are strictly superior for enterprise use cases.
- Rinkeby testnet (which used Clique) was shut down in September 2022.

---

## 6. Consensus Comparison Table

| Property | PoW (Ethash) | PoS (Ethereum) | QBFT | IBFT 2.0 | Clique |
|---|---|---|---|---|---|
| **Finality type** | Probabilistic | Checkpoint (~13 min) | Immediate | Immediate | Probabilistic |
| **Energy use** | Very high | Very low | Very low | Very low | Very low |
| **Trust model** | Trustless (hash power) | Trustless (economic stake) | Trusted validators | Trusted validators | Trusted validators |
| **Min validators** | 1 (any miner) | 1 validator (but network has ~1M) | 4 | 4 | 1 (but 3+ useful) |
| **Byzantine fault tolerance** | Yes (>50% honest) | Yes (>2/3 honest stake) | Yes (>2/3 honest) | Yes (>2/3 honest) | No |
| **Sybil resistance** | Computation cost | Stake cost (32 ETH) | Governance/whitelist | Governance/whitelist | Governance/whitelist |
| **Permissioned?** | No (public) | No (public) | Yes | Yes | Yes |
| **Besu support** | Yes (legacy/ETC) | Yes (mainnet/testnets) | Yes ✅ Recommended | Yes | Deprecated ⚠️ |
| **Best use case** | Public chains (historical) | Ethereum mainnet & testnets | Enterprise private nets | Legacy private nets | Legacy testnets only |
| **Block time** | ~13s (ETH legacy) | 12s (fixed slots) | Configurable (2–15s) | Configurable (2–15s) | Configurable |
| **Forking possible?** | Yes | Extremely rare | No | No | Yes (rare) |

---

## 7. Why Besu Supports Multiple Consensus Mechanisms

Besu is designed to be a **versatile Ethereum client** serving both the public Ethereum ecosystem and private enterprise networks. The different consensus mechanisms exist because different use cases have fundamentally different requirements:

### Public Ethereum Mainnet and Testnets

Besu fully supports Ethereum's PoS consensus (post-Merge) by integrating with consensus clients like **Teku**. For the public network, PoS provides the right balance of security, decentralization, and energy efficiency. Besu handles the **execution layer** — running transactions and the EVM — while Teku handles the **consensus layer** — managing validators and block finalization.

### Private Enterprise Networks

Enterprises building blockchain solutions typically need:

- **Known participants** — regulators, auditors, and business rules require knowing who is on the network.
- **Fast finality** — trade settlement, healthcare records, and supply chain events cannot wait 13 minutes.
- **High throughput** — enterprise workflows generate many transactions per second.
- **Energy efficiency** — running costly PoW infrastructure is not justifiable internally.

QBFT (and IBFT 2.0 for legacy setups) delivers all of these. A company can spin up a 4-node QBFT network in minutes and achieve sub-second finality with known validators.

### The Right Tool for the Right Job

```
Public blockchain, maximum decentralization, no trust?
  → Use Ethereum mainnet (PoS) with Besu + Teku

Testing on public infrastructure?
  → Use Sepolia or Hoodi testnet (PoS) with Besu + Teku

Private network, known validators, fast finality needed?
  → Use QBFT with Besu

Migrating a legacy private network from IBFT 2.0?
  → Stay on IBFT 2.0 or migrate to QBFT

Found an old Clique-based setup in the codebase?
  → Plan migration to QBFT; Clique is deprecated
```

---

## 8. Recap Checklist

After reading this guide, you should be able to confidently say:

- [ ] I understand what consensus is and why it is needed in a distributed system with untrusted participants
- [ ] I can explain the Byzantine Generals Problem and why it requires at least 3f+1 participants to tolerate f traitors
- [ ] I can describe how PoW mining works: miners iterate nonces until the block hash falls below a target
- [ ] I understand difficulty adjustment and why it is necessary in PoW
- [ ] I know what Ethash is and how it differs from Bitcoin's SHA-256 mining (memory-hard vs. compute-hard)
- [ ] I can explain what a 51% attack is and why it works in PoW
- [ ] I understand PoS: validators stake ETH, are pseudo-randomly selected to propose blocks, attest to others' blocks, and can be slashed for misbehavior
- [ ] I can explain slots, epochs, and checkpoints in Ethereum's PoS
- [ ] I understand that PoA uses approved, known validators and is suitable for private/consortium networks
- [ ] I can compare QBFT, IBFT 2.0, and Clique: QBFT is recommended, IBFT 2.0 is legacy, Clique is deprecated
- [ ] I understand why QBFT requires a minimum of 4 validators for Byzantine fault tolerance
- [ ] I can fill in the comparison table from memory for the main properties of each consensus mechanism

---

## 9. Check Your Understanding

Try to answer these without looking back:

1. **The Byzantine Generals Problem says you need at least 3f+1 nodes to tolerate f traitors. A QBFT network has 10 validators. How many can be offline or malicious before the network halts? Show your calculation.**

2. **A PoW miner finds a valid block. Another miner — working on the same problem — finds a different valid block at nearly the same time. Both are broadcast to the network. Describe exactly what happens next, step by step.**

3. **Why does Ethereum's PoS use slashing rather than simply ignoring misbehaving validators? What specific attack does slashing prevent that merely ejecting validators would not?**

4. **You are building a private Ethereum network for a consortium of 5 banks. They need sub-second block times, instant finality, and the network should keep running if one bank's nodes go offline. Which consensus mechanism do you choose and why? How many validators should you have?**

5. **Clique can tolerate more offline validators than QBFT (up to n/2 vs n/3). Yet QBFT is considered superior for enterprise use. Explain the trade-off: what does Clique sacrifice to achieve better liveness tolerance?**

6. **In Ethereum PoS, what is the relationship between a "slot," an "epoch," and a "checkpoint"? How long does it take for a transaction to achieve finality, and what event triggers finalization?**

---

## Next Up

**[05 — Ethereum: The Programmable Blockchain →](05_ethereum_introduction.md)**

With a solid understanding of consensus under your belt, the next guide explores Ethereum itself: what makes it different from Bitcoin, how smart contracts work, what the EVM is, and how the ecosystem of dApps, Layer 2s, and Ethereum's roadmap fit together.