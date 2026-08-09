---
aliases: []
tags: [system-design/design]
status: draft
date:
source:
repo:
---

# NNN — <System Name>

> One-line: what this system does and for whom.

## 1. Requirements

**Functional**
-

**Non-functional**
- Scale:
- Latency:
- Availability / consistency target:

**Explicitly out of scope**
-

## 2. Back-of-envelope

| Quantity | Estimate | Working |
|---|---|---|
| DAU | | |
| QPS (avg / peak) | | |
| Storage / yr | | |
| Bandwidth | | |

## 3. API

```
POST /resource
GET  /resource/{id}
```

## 4. Data model

```mermaid
erDiagram
    USER ||--o{ THING : owns
    USER {
        uuid id PK
        text email
    }
```

Store choice + why:

## 5. High-level design

```mermaid
flowchart LR
    C[Client] --> LB[Load Balancer]
    LB --> API[API Servers]
    API --> CACHE[(Cache)]
    API --> DB[(Primary DB)]
```

## 6. Deep dives
<!-- the 2-3 places this design is actually interesting -->

###

```mermaid
sequenceDiagram
    participant C as Client
    participant A as API
    participant D as DB
    C->>A: request
    A->>D: write
    D-->>A: ack
    A-->>C: 201
```

## 7. Bottlenecks & scaling

| Bottleneck | Symptom | Fix | Cost |
|---|---|---|---|

## 8. Trade-offs made
<!-- what you chose, what you gave up. this section is the whole point -->
-

## 9. Failure modes
- What breaks when X dies:
- Recovery:

## Concepts used
- [[ ]]

## Implementation
- Repo:
- Commit:
- What I actually got working vs. skipped:

## Open questions
-
