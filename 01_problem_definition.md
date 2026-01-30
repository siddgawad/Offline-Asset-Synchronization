# Phase 1: Problem Definition - Cargo Ship Inventory Sync

## System Overview

```mermaid
mindmap
  root((Offline Asset Sync))
    Entities
      Containers
      Goods
      Documents
    Users
      Captain
      Cargo Officer
      Shore Manager
    Constraints
      21 days offline
      128kbps satellite
      $10/MB bandwidth
    Risks
      $50K customs fines
      Cargo seizure
      Safety incidents
```

---

## Data Model Overview

```mermaid
erDiagram
    VESSEL ||--o{ CONTAINER : carries
    CONTAINER ||--o{ GOODS : contains
    CONTAINER ||--o{ DOCUMENT : has
    CONTAINER ||--o{ STATUS_CHANGE : logs
    
    VESSEL {
        uuid vessel_id PK
        string name
        string imo_number
        string current_route
        timestamp last_sync
    }
    
    CONTAINER {
        uuid container_id PK
        uuid vessel_id FK
        string iso_size_code
        string seal_number
        enum status
        point gps_location
        int version
        timestamp updated_at
    }
    
    GOODS {
        uuid goods_id PK
        uuid container_id FK
        string description
        decimal quantity
        decimal weight_kg
        decimal value_usd
        enum status
    }
    
    DOCUMENT {
        uuid doc_id PK
        uuid container_id FK
        enum doc_type
        blob content
        enum clearance_status
    }
```

---

## User Roles & Permissions

```mermaid
flowchart LR
    subgraph VESSEL["🚢 Vessel Side"]
        CAP["👨‍✈️ Captain"]
        CO["👷 Cargo Officer"]
    end
    
    subgraph SHORE["🏢 Shore Side"]
        SM["👔 Shore Manager"]
        CUSTOMS["📋 Customs Broker"]
    end
    
    subgraph DATA["Data Ownership"]
        PHYS["Physical Status<br/>(loaded, damaged, location)"]
        DOCS["Documentation<br/>(customs, compliance)"]
    end
    
    CAP -->|"✅ Full Authority"| PHYS
    CO -->|"✅ Write Access"| PHYS
    CAP -->|"👁️ Read Only"| DOCS
    
    SM -->|"✅ Full Authority"| DOCS
    CUSTOMS -->|"✅ Write Access"| DOCS
    SM -->|"👁️ Read Only"| PHYS
```

---

## Conflict Resolution Rules

```mermaid
flowchart TD
    CONFLICT["⚔️ Conflict Detected<br/>Same record modified on<br/>vessel AND shore"]
    
    CONFLICT --> CHECK{"What field<br/>was modified?"}
    
    CHECK -->|"Physical Status<br/>(location, damage, loaded)"| VESSEL_WINS["🚢 Vessel Wins<br/>(Captain has eyes on cargo)"]
    CHECK -->|"Customs/Documents<br/>(clearance, compliance)"| SHORE_WINS["🏢 Shore Wins<br/>(Compliance team owns this)"]
    CHECK -->|"Other fields"| STORE_BOTH["📦 Store Both<br/>as siblings"]
    
    VESSEL_WINS --> AUDIT["📝 Store both versions<br/>for audit trail"]
    SHORE_WINS --> AUDIT
    STORE_BOTH --> USER["👤 User chooses<br/>on next access"]
    
    USER --> AUDIT
```

---

## System Constraints

```mermaid
flowchart TB
    subgraph CONNECTIVITY["📡 Connectivity"]
        SAT["Satellite: 128kbps"]
        COST["Cost: $10/MB"]
        WINDOW["Sync Window: Port arrival"]
    end
    
    subgraph SCALE["📊 Scale"]
        FLEET["100 vessels"]
        CONTAINERS["2,000 containers/vessel"]
        CHANGES["500 changes/day/vessel"]
    end
    
    subgraph TIMING["⏱️ Timing"]
        OFFLINE["Offline: Up to 21 days"]
        SYNC["Sync time: < 5 min"]
        DATA["Daily data: 50MB → 1.2MB"]
    end
```

---

## Risk Matrix

```mermaid
quadrantChart
    title Risk Assessment: Data Loss Scenarios
    x-axis Low Impact --> High Impact
    y-axis Low Probability --> High Probability
    quadrant-1 Monitor
    quadrant-2 Critical Priority
    quadrant-3 Accept
    quadrant-4 Mitigate
    
    "Lost customs docs": [0.85, 0.3]
    "Sync corruption": [0.6, 0.2]
    "Container miscount": [0.4, 0.5]
    "Delayed sync": [0.3, 0.7]
    "Network timeout": [0.2, 0.8]
```

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| Lost customs docs | $50K fine + cargo held | Low | Checksums, redundant storage |
| Sync corruption | Data integrity loss | Low | Transaction logs, rollback |
| Container miscount | Operational delays | Medium | Reconciliation checks |
| Network timeout | Sync failure | High | Retry with exponential backoff |

---

## Offline Timeline Scenario

```mermaid
gantt
    title Vessel Voyage: Shanghai → Los Angeles (14 days)
    dateFormat  YYYY-MM-DD
    
    section Vessel Activity
    Depart Shanghai     :milestone, m1, 2024-01-01, 0d
    At Sea (Offline)    :active, offline, 2024-01-01, 14d
    Arrive Los Angeles  :milestone, m2, after offline, 0d
    
    section Data Activity
    Local edits accumulate    :crit, edits, 2024-01-01, 14d
    Sync on port arrival      :done, sync, after edits, 1d
    
    section Shore Activity
    Remote updates made       :shore, 2024-01-03, 10d
    Conflicts resolved        :resolve, after sync, 1d
```

---

## Summary Table

| Parameter | Value |
|-----------|-------|
| **Primary Entity** | Containers with Goods inside |
| **Data Fields** | container_id, type, location, status, docs; goods_id, description, qty, weight, value, status, docs |
| **Conflict Model** | Multi-version storage (siblings) with domain-aware defaults |
| **Connectivity** | Satellite (128kbps, $10/MB) |
| **Offline Duration** | Up to 21 days (transpacific worst case) |
| **Fleet Size** | 100 vessels |
| **Containers per Vessel** | 2,000 average |
| **Records per Day** | ~500 status changes per vessel |
| **Estimated Daily Data** | ~50MB raw → ~1.2MB compressed |
| **Critical Failure** | Lost customs docs = $50K fine + cargo held |

---

## Conflict Resolution Summary

**Rule**: Since captain is on vessel, they have authority on physical status. Shore manager has authority on customs clearance and documents.

**Implementation**: 
1. Store both values as concurrent siblings
2. Apply domain-aware default (vessel wins physical, shore wins docs)
3. User can override and manually choose winner
4. Full audit trail preserved regardless of resolution method
