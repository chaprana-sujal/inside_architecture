# Campus-Scoped Real-Time Messaging — Backend Architecture

> **Proposed by:** Sujal Chaprana  
> **Context:** Low-latency, horizontally scalable, campus-scoped anonymous messaging system  
> **Compliance:** IT Rules 2021 (traceability at storage layer, masked at API layer)

---

## 1. High-Level System Overview

```mermaid
flowchart TD
    subgraph Client["📱 Client Layer"]
        A1[Mobile App - Campus A]
        A2[Mobile App - Campus B]
    end

    subgraph Gateway["🔌 Socket Gateway Layer (Per-Campus Tenant)"]
        G1[WebSocket Gateway\nCampus A Shard]
        G2[WebSocket Gateway\nCampus B Shard]
    end

    subgraph Broker["⚡ Async Message Broker (Redis Pub/Sub)"]
        R1[Redis Channel\nCampus A]
        R2[Redis Channel\nCampus B]
        RS[Redis Sentinel\n3-Node HA Cluster]
    end

    subgraph Persistence["🗄️ Persistence Layer"]
        DB[(Primary DB\nPostgreSQL / MongoDB)]
        NW[Notification Worker\nFCM / APNs Push]
    end

    A1 -- Sticky WebSocket\nConsistent Hash --> G1
    A2 -- Sticky WebSocket\nConsistent Hash --> G2

    G1 -- Pub/Sub --> R1
    G2 -- Pub/Sub --> R2

    R1 -. Sentinel Managed .-> RS
    R2 -. Sentinel Managed .-> RS

    G1 -- Write-Ahead\nDB Write First --> DB
    G2 -- Write-Ahead\nDB Write First --> DB

    R1 -- Undelivered\nMessage Watch --> NW
    NW -- Push Notification --> A1
    NW -- Push Notification --> A2
```

---

## 2. Message Flow — End to End

```mermaid
sequenceDiagram
    autonumber
    participant App as 📱 Mobile App
    participant GW as 🔌 WebSocket Gateway<br/>(Campus Shard)
    participant DB as 🗄️ Database
    participant Redis as ⚡ Redis Pub/Sub
    participant Sub as 📱 Subscriber App(s)
    participant Push as 🔔 Notification Worker

    App->>GW: Send message (channel/circle)
    GW->>DB: Write-Ahead: persist message<br/>with monotonic seq_id
    DB-->>GW: ACK (write confirmed)
    GW->>Redis: Publish to campus channel
    Redis-->>Sub: Deliver to online subscribers
    Redis-->>Push: Trigger for offline users
    Push->>App: FCM / APNs push notification
```

> **Key Principle:** DB write always happens **before** Redis publish. No message is silently dropped.

---

## 3. Offline User Delivery (Push + Pull Hybrid)

```mermaid
sequenceDiagram
    autonumber
    participant App as 📱 App (was offline)
    participant GW as 🔌 WebSocket Gateway
    participant DB as 🗄️ Database
    participant Push as 🔔 Push Worker

    Note over App: User reconnects
    App->>GW: Connect + send last_seq_id
    GW->>DB: Query: messages where seq_id > last_seq_id
    DB-->>GW: Return missed messages (gap replay)
    GW-->>App: Deliver missed messages in order

    Note over App: User truly offline (app closed)
    Push->>DB: Poll for undelivered messages
    Push->>App: FCM / APNs push notification
    Note over App: App opens → reconnects → gap replay as above
```

---

## 4. Gateway Scaling Strategy

```mermaid
flowchart LR
    subgraph LB["🌐 Load Balancer"]
        direction TB
        LB1[Consistent Hash Router\nCampus + User ID]
    end

    subgraph CampusA["Campus A — Burst Scaling"]
        GA1[Gateway Pod 1\n~30k connections]
        GA2[Gateway Pod 2\n~30k connections]
        GA3[Gateway Pod 3\n~30k connections]
    end

    subgraph CampusB["Campus B — Baseline"]
        GB1[Gateway Pod 1\n~30k connections]
    end

    Client1[Clients] --> LB1
    LB1 -- Campus A --> CampusA
    LB1 -- Campus B --> CampusB

    note1["⚠️ Scale per-tenant, not globally\nFest/exam bursts only affect one campus shard"]
```

| Parameter | Value |
|-----------|-------|
| Connections per Gateway Pod | ~30,000 |
| Session Strategy | Sticky (Consistent Hash: campus + user) |
| Scaling Unit | Per-campus tenant (not global) |
| Health Check | Heartbeats + idle timeouts |
| Recovery | Jittered exponential backoff (thundering herd prevention) |

---

## 5. Failure Handling

```mermaid
flowchart TD
    F1{Redis Available?}
    F2[Publish message\nto Redis Pub/Sub]
    F3[Retry Worker:\nre-publish from\nDB queue]
    F4{Gateway Crashes?}
    F5[Client: Jittered\nExponential Backoff]
    F6[Reconnect →\ngap replay from DB]
    F7{Redis Node Fails?}
    F8[Redis Sentinel\nautomatically promotes\nnew primary]

    F1 -- Yes --> F2
    F1 -- No --> F3
    F3 --> F2

    F4 -- Yes --> F5
    F5 --> F6

    F7 -- Yes --> F8
    F8 --> F2
```

### Failure Matrix

| Failure Scenario | Mitigation | Message Loss? |
|---|---|---|
| Redis unavailable | Retry worker re-publishes from DB queue | ❌ Never |
| Redis node failure | Redis Sentinel (3-node) auto-failover | ❌ Never |
| Gateway crash | Client jittered backoff → reconnect → gap replay | ❌ Never |
| User offline | Write-ahead DB + Push (FCM/APNs) + gap replay on reconnect | ❌ Never |
| Broker overload | Horizontal scale per-campus shard | ❌ Never |

---

## 6. Identity & Compliance Layer

```mermaid
flowchart LR
    App[📱 Client App] -- Anonymous UX --> API[API Layer\nIdentity Mask]
    API -- Auditable Author Record --> DB[(Database\nReal Identity Stored)]
    DB -. IT Rules 2021\nTraceability .-> LEGAL[Legal / Law Enforcement\nAudit Trail]
```

- **API Layer:** Strips identity before delivering content to clients (anonymous UX)
- **Storage Layer:** Retains auditable author records with full traceability
- **Compliance:** Satisfies IT (Intermediary Guidelines) Rules 2021 traceability requirements without compromising anonymous user experience

---

## 7. Trade-offs Summary

| Concern | Trade-off | Mitigation |
|---|---|---|
| Redis fire-and-forget | Messages not stored by default | Write-ahead to DB before Redis publish |
| Tenant isolation | Routing complexity increases | Offset by fault isolation and targeted scaling |
| Schema migrations | Must run across all campus shards | Additive-only migrations; blue-green rollouts per shard |
| Campus-edge alternative | Per-campus auth infra + cert mgmt balloons cost | Rejected in favour of centralized cloud + logical sharding |

---

## 8. Tech Stack Summary

| Component | Technology |
|---|---|
| Client | Mobile App (iOS / Android) |
| WebSocket Gateway | Node.js / Go (per-campus pods) |
| Message Broker | **Redis Pub/Sub** (Kafka if scale demands) |
| High Availability | Redis Sentinel (3-node) |
| Database | PostgreSQL / MongoDB |
| Push Notifications | FCM (Android) + APNs (iOS) |
| Orchestration | Kubernetes (HPA per campus tenant) |
| Session Routing | Consistent hash (campus + user ID) |

---

*This document reflects the architecture discussed and proposed for a secure, scalable, campus-scoped real-time messaging backend.*
