# Distributed Sync & Conflict Resolution: The Offline-First Handbook

This repository provides a rigorous architectural framework for building robust **Offline-Asset-Synchronization** systems. In distributed environments, the core tension lies between local autonomy and global consistency. This handbook dissects the strategies required to manage state across disconnected nodes using an **AP-focused (Availability and Partition Tolerance)** approach under the CAP theorem.

---

## 1. The Core Tension: AP Over Consistency

In an offline-first architecture, systems must prioritize **Availability** (allowing users to work without a connection) and **Partition Tolerance** (surviving network drops). This necessitates the sacrifice of immediate consistency in favor of **Eventual Consistency**.

### The Five-Question Design Framework

Before selecting an architecture, every system must be audited against these first principles:

1. **Data Shape:** Is it structured, document-based, or binary?
2. **Conflict Semantics:** What is the business cost of a lost update?
3. **Connectivity Patterns:** Is it "occasionally offline" or "mostly offline"?
4. **Data Volume:** Are we syncing deltas or full state?
5. **Ordering Requirements:** Is causality preservation required?

---

## 2. System Architecture

A resilient sync system is organized into a four-layer mental model.

| Layer | Responsibility | Key Components |
| --- | --- | --- |
| **Presentation** | User feedback & manual resolution | Conflict UI, sync status indicators |
| **Application** | Business logic & sync triggers | Change detection logic, data validation |
| **Sync Engine** | The "Brain" | Batching, queuing, conflict detection |
| **Storage** | Data persistence | SQLite/IndexedDB, Version Vectors, UUIDs |

### The Sync State Machine

Sync operations must follow a strict state machine to prevent race conditions and data corruption.

---

## 3. Conflict Resolution Strategies

The choice of strategy determines the system's integrity and user experience.

* **Last-Write-Wins (LWW):** Uses the highest timestamp. Simple, but risks silent data loss due to clock skew.
* **First-Write-Wins (FWW):** The server rejects any update not based on the current authoritative version.
* **Field-Level Merge:** Merges changes if they affect different attributes within the same record.
* **CRDTs (Conflict-free Replicated Data Types):** Mathematically guaranteed convergence (e.g., G-Counters, OR-Sets) without a central coordinator.
* **Operational Transformation (OT):** Complex transformation of operations, typically for real-time collaborative text editing.

---

## 4. Sync Algorithms & Resilience

### Delta Sync vs. Full Sync

Systems should utilize **Delta Sync** via **Sync Tokens** (opaque cursors) to minimize bandwidth. Full sync should remain strictly as a recovery fallback for corrupted local states.

### Idempotency and Retries

All sync endpoints **must** be idempotent. Network failures during a sync response often lead to duplicate requests. Use client-generated request IDs to deduplicate operations on the server.

### Failure Recovery: Exponential Backoff

To prevent "thundering herd" failures when service resumes, implement exponential backoff with jitter:

### The Circuit Breaker Pattern

If the failure rate exceeds a defined threshold (e.g., 5% over 1 minute), the sync engine must enter the **OPEN** state, immediately rejecting requests to allow the backend to recover.

---

## 5. Observability & Operations

A sync system without metrics is a black box waiting to fail. Track these four pillars:

1. **Sync Lag:** Time delta between local change and global visibility ( hour is critical).
2. **Conflict Rate:** Percentage of syncs requiring resolution logic ( indicates UI/UX friction).
3. **Failure Rate:** Percentage of 4xx/5xx responses.
4. **Queue Depth:** Number of pending local changes.

### Dead Letter Queue (DLQ)

Permanent failures (e.g., schema mismatches or unresolvable business rule violations) must be moved to a DLQ for manual intervention or administrative audit, preventing them from blocking the primary sync queue.

---

## 6. Strategic Interview Framework (SCAR)

When defending this architecture in a technical review or interview:

* **S (Scope):** Define the data volume and connectivity constraints.
* **C (Core Architecture):** Explain the 4-layer model and UUID strategy.
* **A (Algorithms):** Justify your choice of LWW vs. CRDT based on business cost.
* **R (Resilience):** Detail the circuit breaker and idempotency strategy.

---
