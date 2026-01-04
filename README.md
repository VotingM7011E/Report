# VotingM7011E — Seminar Pre‑Submission Report (Draft)

**Last updated:** 2026-01-04 22:52  
**Status:** Initial submission for seminar (can be expanded after the seminar).

## GitHub links
- **Organization:** https://github.com/VotingM7011E
- **Repositories (microservices):** https://github.com/orgs/VotingM7011E/repositories
- **GitOps repo (Argo CD + Helm):** https://github.com/VotingM7011E/gitops

---

## 1) Motivation — Why this is a *dynamic web system*

A dynamic web system changes what it returns (data, UI state, permissions) based on **user input, stored data, and runtime state**, rather than serving static pages.

In our system:
- Users authenticate via **Keycloak (OIDC)** and receive tokens; backend APIs authorize requests based on meeting‑scoped roles (view/vote/manage).
- The system state changes over time (meeting agenda item index, nomination windows, voting in progress/completed), which changes what users can do and what is shown.
- Data is persisted in databases (MongoDB collections for meeting/motion state; SQL databases for elections and voting) and drives responses.
- The system is decomposed into multiple **microservices** communicating with both synchronous HTTP and asynchronous events (RabbitMQ), enabling evolving behavior.

---

## 2) High‑level architecture

### 2.1 Microservices and responsibilities (current code)

- **MeetingService** (Flask + MongoDB + Socket.IO)
  - Create meetings, manage agenda items, update `current_item`.
  - Publishes events:
    - `permission.create_meeting` when a meeting is created (so creator becomes manager/viewer)
    - `motion.create_motion_item` when moving to a motion agenda item
    - `motion.start_voting` when starting vote for a motion agenda item

- **PermissionService** (Flask + Keycloak Admin)
  - Meeting‑scoped roles stored as Keycloak realm roles: `z-<meeting_id>-<role>` where role ∈ {view, vote, manage}.
  - REST endpoints to read/write roles.
  - Consumes event `permission.create_meeting` to grant the meeting creator `view` + `manage`.

- **MotionService** (Flask + MongoDB + Socket.IO)
  - Stores motion items with motions and orchestrates sequential voting.
  - Consumes:
    - `motion.create_motion_item`
    - `motion.start_voting`
    - `voting.created`
    - `voting.completed`
  - Publishes:
    - `voting.create` (request VotingService to create a poll)

- **VotingService** (Flask + SQLAlchemy)
  - Stores polls, options, votes, vote selections.
  - Consumes:
    - `voting.create`
  - Publishes:
    - `voting.created` (poll created)
    - `voting.completed` (poll completed when expected voters reached)
  - Calls PermissionService to compute eligible voters (role `vote`) when needed.

- **ElectionService** (Flask + PostgreSQL)
  - Tables: `positions`, `nominations`.
  - Current implementation creates a poll by HTTP call to VotingService (planned: migrate to MQ like MotionService).

> Note: Some services are still in progress (ElectionService uses HTTP for poll creation today; the target design is fully event-driven).

### 2.2 Communication model

- **HTTP/REST:** user/client requests to services; ElectionService → VotingService poll creation currently via HTTP.
- **RabbitMQ events:** asynchronous orchestration of workflows (meeting creation → permissions; motion voting lifecycle; voting creation/completion callbacks).

### 2.3 Architecture diagram (Mermaid)

```mermaid
flowchart LR
  subgraph Client
    U["User Browser"]
    FE["Frontend"]
  end

  subgraph Identity
    KC["Keycloak"]
  end

  subgraph Messaging
    RMQ[("RabbitMQ")]
  end

  subgraph Services
    MS["MeetingService<br/>MongoDB + Socket.IO"]
    PS["PermissionService<br/>Keycloak Admin"]
    MOS["MotionService<br/>MongoDB + Socket.IO"]
    VS["VotingService<br/>SQL DB (SQLAlchemy)"]
    ES["ElectionService<br/>Postgres"]
  end

  U --> FE
  FE -->|"HTTPS + Bearer JWT"| MS
  FE -->|"HTTPS + Bearer JWT"| MOS
  FE -->|"HTTPS + Bearer JWT"| VS
  FE <--> KC

  %% Meeting -> Permission
  MS -->|"permission.create_meeting"| RMQ
  RMQ --> PS

  %% Meeting -> Motion
  MS -->|"motion.create_motion_item"| RMQ
  MS -->|"motion.start_voting"| RMQ
  RMQ --> MOS

  %% Motion -> Voting
  MOS -->|"voting.create"| RMQ
  RMQ --> VS

  %% Voting callbacks -> Motion
  VS -->|"voting.created"| RMQ
  VS -->|"voting.completed"| RMQ
  RMQ --> MOS

  %% Election currently via HTTP
  ES -->|"HTTP POST /polls/"| VS

  %% Permission roles stored in Keycloak
  PS --> KC
  VS -->|"HTTP GET eligible voters"| PS
```

---

## 3) GitOps CI/CD (high level)

We deploy using **GitOps**:
- **Argo CD** watches Git repositories for desired Kubernetes manifests and continuously reconciles the cluster to match Git.
- Services are deployed as Helm charts with environment-specific values.

### 3.1 GitOps pipeline diagram

```mermaid
flowchart LR
  DEV[Developers] -->|push code| REPO[Service Repositories]
  REPO -->|CI builds/tests| CI[GitHub Actions]
  CI -->|push images| REG[Container Registry]
  CI -->|update image tags/values| GITOPS[gitops repo]
  ARGO[Argo CD] -->|pull desired state| GITOPS
  ARGO -->|sync/apply| K8S[Kubernetes Cluster]
  K8S --> RUN[Running Services]
```

---

## 4) Security model (request flow)

### 4.1 Identity and tokens
- **Keycloak** acts as the OpenID Connect provider.
- Frontend uses an **Authorization Code** style login flow (redirect to Keycloak, exchange code for token).
- Backend APIs receive `Authorization: Bearer <access_token>` and enforce access based on meeting roles.

### 4.2 Meeting-scoped authorization
- PermissionService manages roles in Keycloak per meeting:
  - `z-<meeting_id>-view`
  - `z-<meeting_id>-vote`
  - `z-<meeting_id>-manage`
- Services use `check_role(user, meeting_id, action)` to gate routes.

### 4.3 Security flow diagram

```mermaid
sequenceDiagram
  participant User
  participant FE as Frontend
  participant KC as Keycloak
  participant API as Backend APIs

  User->>FE: Open app
  FE->>KC: Redirect to login (OIDC authorize)
  KC->>User: Login form
  User->>KC: Credentials
  KC->>FE: Authorization code
  FE->>KC: Exchange code for tokens
  KC-->>FE: access_token (Bearer JWT)
  FE->>API: Requests with Authorization: Bearer <JWT>
  API-->>FE: Response (depends on role & meeting state)
```

### 4.4 Secret handling (GitOps)
- Kubernetes Secrets are **base64-encoded** and should be treated as sensitive.

---

## 5) Monitoring (metrics/logs/traces)

### 5.1 Metrics
- **Prometheus + Grafana** used for metrics and dashboards.
- RabbitMQ exposes Prometheus metrics via `rabbitmq_prometheus` plugin at `/metrics` (default port 15692).
- Prometheus Operator discovers targets using **ServiceMonitor** resources.

### 5.2 Logs and traces
- Logs: currently via Kubernetes pod logs; centralized aggregation planned.
- Traces: not implemented yet (future extension).

### 5.3 Monitoring diagram

```mermaid
flowchart LR
  SVC[Services] -->|/metrics| PROM[Prometheus]
  RMQ[(RabbitMQ /metrics)] -->|/metrics| PROM
  PROM --> GRAF[Grafana]
```

---

## 6) Database schemas (graphical; draft)

> These schemas are inferred from code. Next step after seminar: export from live DBs for authoritative ERDs.

### 6.1 VotingService (SQLAlchemy)

```mermaid
erDiagram
    POLL ||--o{ POLL_OPTION : has
    POLL ||--o{ VOTE : receives
    VOTE ||--o{ VOTE_SELECTION : contains

    POLL {
      uuid uuid PK
      string meeting_id
      string poll_type
      int expected_voters
      bool completed
    }
    POLL_OPTION {
      int id PK
      int poll_id FK
      string option_value
      int option_order
    }
    VOTE {
      int id PK
      int poll_id FK
      string user_id
    }
    VOTE_SELECTION {
      int id PK
      int vote_id FK
      int poll_option_id FK
      int rank_order
    }
```

### 6.2 ElectionService (PostgreSQL)

```mermaid
erDiagram
    POSITIONS ||--o{ NOMINATIONS : has

    POSITIONS {
      int position_id PK
      string meeting_id
      string position_name
      bool is_open
      string poll_id
    }
    NOMINATIONS {
      int position_id FK
      string username PK
      bool accepted
    }
```

### 6.3 MeetingService (MongoDB; document shape)

```mermaid
erDiagram
    MEETINGS ||--o{ AGENDA_ITEMS : contains

    MEETINGS {
      string meeting_id PK
      string meeting_name
      int current_item
      string meeting_code
    }
    AGENDA_ITEMS {
      string meeting_id FK
      string type
      string title
      string motion_item_id
      bool motion_published
    }
```

### 6.4 MotionService (MongoDB; document shape)

```mermaid
erDiagram
    MOTION_ITEM ||--o{ MOTION : includes

    MOTION_ITEM {
      string motion_item_id PK
      string meeting_id
      string poll_uuid
      string poll_state
      string voting_state
      int current_index
    }
    MOTION {
      string motion_uuid PK
      string owner
      string motion
    }
```

### 6.5 PermissionService (Keycloak roles)

Meeting-scoped realm roles in Keycloak:
- `z-<meeting_id>-view`
- `z-<meeting_id>-vote`
- `z-<meeting_id>-manage`

---

## References (external docs)

- Argo CD (GitOps continuous delivery): https://argo-cd.readthedocs.io/
- Argo CD Application spec (Helm valueFiles, etc.): https://argo-cd.readthedocs.io/en/stable/user-guide/application-specification/
- Argo CD Secret Management guidance: https://argo-cd.readthedocs.io/en/stable/operator-manual/secret-management/
- Prometheus Operator (ServiceMonitor/PodMonitor): https://prometheus-operator.dev/docs/developer/getting-started/
- RabbitMQ Prometheus monitoring (`/metrics`): https://www.rabbitmq.com/docs/prometheus
- RabbitMQ URI spec: https://www.rabbitmq.com/docs/uri-spec
- Kubernetes DNS for Services: https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/
