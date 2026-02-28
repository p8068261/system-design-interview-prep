# Daily System Design Interview: February 27, 2026

---

## Question: Globally Distributed Multi-Master Transaction Engine

You are tasked with designing the core transaction engine for a global financial ledger system handling 10M+ TPS across 40+ regions. The system must support cross-region transactions (e.g., a user in Singapore initiates a transfer to a merchant in Brazil while both parties also have active trades in London). 

**Requirements:**
- Serializable isolation globally (no phantom reads, lost updates, or write skew)
- P99 latency <100ms for same-region transactions, <500ms for cross-region
- ACID guarantees even during inter-region network partitions
- Support for 2PC-like distributed transactions spanning arbitrary regions
- No single point of failure; tolerate full region outages
- Guarantee total order of transactions per-account (external consistency across regions)

**Constraints:**
- Clock sync via TrueTime-like API (bounded uncertainty ~7ms)
- Average cross-region RTT: 120-350ms
- Network partitions last minutes to hours in practice
- Must handle Byzantine failures at the infrastructure level
- Regulatory requirement: EU data stays in EU, China data in China

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           CLIENT LAYER                                   │
│            (Smart load balancers with topology awareness)                │
└─────────────────────────────────────────────────────────────────────────┘
                                   │
        ┌──────────────────────────┼──────────────────────────┐
        ▼                          ▼                          ▼
┌───────────────┐          ┌───────────────┐          ┌───────────────┐
│  Region Alpha │◄════════►│  Region Beta  │◄════════►│ Region Gamma  │
│   (Americas)  │   Paxos  │   (EMEA)      │   Paxos  │   (APAC)      │
└───────┬───────┘          └───────┬───────┘          └───────┬───────┘
        │                          │                          │
   ┌────┴────┐                ┌────┴────┐                ┌────┴────┐
   ▼         ▼                ▼         ▼                ▼         ▼
┌──────┐ ┌──────┐         ┌──────┐ ┌──────┐         ┌──────┐ ┌──────┐
│Shard │ │Shard │         │Shard │ │Shard │         │Shard │ │Shard │
│ A-N  │ │ O-Z  │         │ A-N  │ │ O-Z  │         │ A-N  │ │ O-Z  │
│Master│ │Master│         │Master│ │Master│         │Master│ │Master│
└──┬───┘ └──┬───┘         └──┬───┘ └──┬───┘         └──┬───┘ └──┬───┘
   │        │                │        │                │        │
┌──┴──┐  ┌──┴──┐          ┌──┴──┐  ┌──┴──┐          ┌──┴──┐  ┌──┴──┐
│ Paxos│  │Paxos│          │Paxos│  │Paxos│          │Paxos│  │Paxos│
│Group │  │Group│          │Group│  │Group│          │Group│  │Group│
└──────┘  └──────┘          └──────┘  └──────┘          └──────┘  └──────┘

Cross-Region Paxos Ring (5 nodes/region for resilience):
    ┌─────────────────────────────────────────────────────────────────┐
    ▼                                                                 ▼
Region_Alpha_1 ──► Region_Beta_1 ──► Region_Gamma_1 ──► Region_Alpha_2
     ▲                                                        │
     └─────────────────────────────────────────────────────────┘
```

**Core Components:**
1. **Spanner-like Timestamp Oracle** – TrueTime API for external consistency
2. **Geo-Partitioned Spans** – Shard groups co-located with data residency requirements
3. **Multi-Paxos Layers** – Intra-region consensus for shards, cross-region for global coordination
4. **Two-Phase Commit Manager** – Distributed transaction coordinator with starvation-free locking
5. **Conflict Resolution Layer** – Wait-die / wound-wait variants for deadlock handling

---

## Key Design Decisions

### 1. Timestamp Ordering vs Vector Clocks
**Decision:** Use Timestamp Ordering (TrueTime-based) for global consistency, not vector clocks.

**Trade-offs:**
- **Pro:** Enables external consistency (if T1 finishes before T2 starts, T1's timestamp < T2's). Easier reasoning for clients.
- **Pro:** Bounded clock uncertainty (7ms) allows for tight commit wait times.
- **Con:** Requires atomic clock/GPS infrastructure (costly, operational burden).
- **Con:** Commit-wait overhead adds ~7ms latency even for local transactions.

**Alternative:** Vector clocks would eliminate clock dependency but make "happens-before" queries O(num-regions) and complicate transaction ordering visibility.

### 2. Strong Consistency with Sync Cross-Region Replication
**Decision:** Synchronous Paxos replication to 2/3 regions before commit acknowledgment.

**Trade-offs:**
- **Pro:** Automatic failover with no data loss. Reads never see stale data.
- **Pro:** Serializable isolation without complex conflict reconciliation post-commit.
- **Con:** 100-350ms added latency for cross-region writes (mitigated via pipelining).
- **Con:** During partitions, region becomes unavailable for writes (vs. eventual consistency).

**Mitigation:** Transaction classification – same-region commits don't wait for cross-region sync; cross-region commits use async pipelining where possible.

### 3. Transaction Processing: 2PC with Deterministic Timestamp Assignment
**Decision:** Use 2PC with pre-assigned timestamps (from TrueTime) and optimistic execution.

**Trade-offs:**
- **Pro:** Deterministic timestamps allow read-only replicas to serve consistent snapshots without locking.
- **Pro:** "Pre-expression" of read/write sets allows early validation.
- **Con:** Higher abort rate under contention vs lock-based pessimistic concurrency.
- **Con:** Coordinator failure requires recovery protocol (Paxos-backed coordinator logs).

**Conflict overhead:** Wound-wait for multi-region conflicts; wait-die for single-region to minimize aborts.

---

## Deep Dive: Critical Components

### Component 1: Cross-Region PaxosRing — The "Global Brain"

The hardest problem: maintaining total order of cross-region transactions with sub-500ms latency.

**Approach:** Multi-coordinated Paxos with leader leasing

```
Structure:
- Each region has 3 replica groups (Raft/Paxos leaders) per shard
- Cross-region "consensus rings" for distributed transaction coordination
- Leader stickiness: A transaction chain (e.g., Singapore→Brazil) keeps routing to same coordinator

Latency Optimization:
1. Fast-path: Single-region reads/writes go through local Paxos only
2. Slow-path: Cross-region uses hierarchical consensus
   - Leader in initiating region proposes to cross-region PaxosRing
   - Ring commits to 2 other regions (quorum = 3/5 for safety)
   - Parallel commit to shard replicas in those regions

Anti-Deadlock:
- Timestamp-order locking: Txn with earlier timestamp "wins" in conflict
- Automatic txn migration: If coordinator region fails mid-commit,
  any region with PaxosRing quorum can complete the transaction
```

**Why this works for Staff+ context:** It separates the concerns of data storage (region-local) from transaction coordination (cross-region). The PaxosRing becomes a "meta-cluster" that only handles transaction metadata, not full data replication.

### Component 2: Timestamp Oracle with Bounded Uncertainty

**Mechanism:**

Similar to TrueTime: `TT.now()` returns an interval `[earliest, latest]`.

```
Transaction Lifecycle:
1. START: Get TT.now().latest as read timestamp
2. EXECUTE: Read at that timestamp (Spanner-like snapshot reads)
3. COMMIT: Get TT.now().earliest as commit timestamp
4. WAIT: Block until TT.now().latest > commit_timestamp
5. ACK: Return success to client

Commit Wait = ~2ε (twice clock uncertainty)
With GPS/atomic clocks: ε ≈ 3.5ms → wait ~7ms
```

**Critical insight:** The wait is the price for external consistency – no additional communication needed to establish ordering. Two transactions will have correctly ordered timestamps even if they commit concurrently across the globe.

**Staff-level consideration:** What if TrueTime infrastructure fails? Degrade to logical timestamps + vector clocks with explicit causal barriers. Requires clients to handle potential causality violations in degraded mode.

### Component 3: Distributed Transaction Scheduler — The Contention Killer

**Problem:** Serializable + distributed + high throughput = contention explosion

**Solution:** Three-tier conflict resolution

```
Tier 1: Same-Region Shortcuts
- Detect single-region transactions at coordinator
- Skip cross-region Paxos; commit locally with async replication
- Maintains serializability: all same-region reads see local commit

Tier 2: Contention-Weighted Locking
- Each shard tracks contention level (waits/sec)
- Hot keys trigger "contention scheduling": serialize conflicting txns
  at coordinator before dispatch rather than at shard
- Prevents thundering herds on popular records

Tier 3: Deterministic Reordering for Multi-Region
- Multi-region txns sorted by (timestamp, coordinator_region, txn_id)
- Ensures global ordering even across failure+cleanup paths
- Eliminates need for distributed deadlock detection
```

**Byzantine resilience:** Each shard runs BFT-SMaRt for consensus under malicious infrastructure assumptions. Adds 2-3x overhead but required for financial-grade correctness guarantees.

---

## Follow-up Questions

- **Q1: Clock Skew Defense:** You rely on GPS/atomic clocks for TrueTime. If an attacker compromises GPS signals in a region causing massive clock skew (>100ms), how does your system detect and respond? What consistency guarantees hold during recovery?

- **Q2: Regulatory Sharding:** EU transactions must stay in EU, China in China, yet a single transaction might span a user in EU and merchant in China. How do you structure the PaxosRing to satisfy data residency while maintaining the "2/3 sync regions" durability guarantee?

- **Q3: Hot Spot Mitigation:** A celebrity launches a cryptocurrency on your platform. Suddenly 50% of global write volume targets a single account (their token contract). Your wound-wait conflict resolution causes cascading aborts. How do you dynamically adapt the architecture to handle this hot spot while preserving serializability?

- **Q4: Coordinator Failure Mid-Commit:** A cross-region transaction has prepared successfully at all shards, but the coordinator crashes before issuing the final COMMIT/ABORT. Recovery discovers the coordinator's Paxos log was lost in a correlated disk failure. What guarantees can you provide, and how do you prevent indefinite blocking?

---

*Generated by Relay AI | System Design Daily | 2026-02-27*
