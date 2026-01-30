# Offline Asset Synchronization System

System design for synchronizing cargo ship inventory data between vessels and shore operations, handling extended offline periods, satellite bandwidth constraints, and domain-aware conflict resolution.

## 🎯 Problem Statement

Cargo vessels operate offline for **14-21 days** during transpacific voyages. During this time, both vessel crews and shore managers need to track and update container inventory. When vessels reconnect via expensive satellite links ($10/MB), the system must:

- Sync thousands of container status changes efficiently
- Resolve conflicts when both sides edited the same record
- Prioritize safety-critical data over routine logs
- Handle network failures gracefully

## 📋 Design Documents

| Document | Description |
|----------|-------------|
| [01_problem_definition.md](./01_problem_definition.md) | Requirements, constraints, user roles, conflict scenarios |
| [02_Architecture.md](./02_Architecture.md) | System components, data flows, Mermaid diagrams, design decisions |
| 03_Data_Model.md | Database schemas, sync metadata, version tracking *(coming soon)* |
| 04_Sync_Protocol.md | API contracts, payload formats, sync algorithm *(coming soon)* |
| 05_Failure_Handling.md | Retry logic, circuit breakers, dead letter queues *(coming soon)* |
| 06_Operations.md | Monitoring, alerting, runbooks, cost analysis *(coming soon)* |

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                         VESSEL SIDE                                  │
│   📱 Tablet UI  →  ⚙️ Sync Engine  →  💾 SQLite  →  📋 Change Queue │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                │ 🛰️ Satellite (128kbps, $10/MB)
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         SHORE SIDE                                   │
│   ⚖️ Load Balancer  →  🔌 REST API  →  🐘 PostgreSQL  →  ⚔️ Resolver│
└─────────────────────────────────────────────────────────────────────┘
```

## 🔑 Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Vessel Database | SQLite | Lightweight, zero-config, works offline |
| Transport Protocol | REST/HTTPS | Stateless, retry-friendly for unreliable satellite |
| Sync Strategy | Delta + Compression | 97% bandwidth reduction ($500 → $12/day) |
| Conflict Resolution | Domain-aware rules | Physical status → vessel wins, Docs → shore wins |

## 📊 Bandwidth Optimization

| Technique | Reduction |
|-----------|-----------|
| Delta sync (changes only) | 80% |
| Gzip compression | 85% |
| Priority queuing | 20% |
| **Total: 50MB → 1.2MB** | **97.6%** |

## ⚔️ Conflict Resolution Strategy

```
IF field is physical_status:
    → Vessel authority (captain is on-site)
ELSE IF field is customs_clearance:
    → Shore authority (compliance team owns this)
ELSE:
    → Last-write-wins with full audit trail
    
ALWAYS: Store both versions for audit compliance
```

## 🔄 Sync State Machine

```
IDLE → CHECKING → PREPARING → PUSHING → PROCESSING → COMPLETE
                                ↓
                            RETRYING (exponential backoff)
                                ↓
                             FAILED (dead letter queue)
```

## 📈 Scale Specifications

| Parameter | Value |
|-----------|-------|
| Fleet size | 100 vessels |
| Containers per vessel | 2,000 average |
| Offline duration | Up to 21 days |
| Daily data per vessel | ~50MB (raw), ~1.2MB (optimized) |
| Sync time | < 5 minutes over satellite |

## 🛡️ Failure Handling

- **Network drops**: Checkpoint + resume from last successful record
- **Corrupt payload**: SHA-256 checksum validation
- **Version conflicts**: Domain-aware resolution with manual override
- **Shore outage**: Circuit breaker with exponential backoff
- **Disk full**: Prune old sync logs, alert user

## 🧠 Concepts Demonstrated

- **CAP Theorem**: AP system with eventual consistency
- **CRDTs**: State-based conflict-free data types
- **Vector Clocks**: Logical ordering for distributed events
- **Circuit Breaker**: Failure isolation pattern
- **Dead Letter Queue**: Permanent failure handling
- **Exponential Backoff**: Retry strategy with jitter


