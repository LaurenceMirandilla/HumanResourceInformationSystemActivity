# Part 4 — Initial Architecture Diagram (HRIS)

**System:** Human Resource Information System (HRIS)
**Style:** Microservices — API Gateway, database-per-service, event-driven integration

---

## Rendered Diagram

![HRIS Microservices Architecture](./architecture-diagram.png)

---

## High-Level Architecture (Mermaid source)

Legend:

- **Solid arrow** = **SYNC** — synchronous communication (HTTP/REST or SQL, request/response)
- **Dotted arrow** = **ASYNC** — asynchronous communication (events via the message broker)

```mermaid
flowchart LR
    %% ================= Client Layer =================
    subgraph CLIENT_LAYER["Client Layer"]
        WEB["Web App<br/>(React)"]
        MOB["Mobile App<br/>(Android / iOS)"]
    end

    %% ================= API Gateway =================
    GW["API Gateway<br/><b>Single Entry Point</b><br/>routing · auth · rate limiting"]

    %% ================= Microservices =================
    subgraph SERVICES["HRIS Microservices — database per service"]
        EMP["<b>employee-service</b><br/>BC 1: Employee Records"]
        PAY["<b>payroll-service</b><br/>BC 2: Payroll"]
        LEA["<b>leave-service</b><br/>BC 3: Leave Management"]
    end

    %% ================= Databases =================
    subgraph DATA["Private Databases"]
        EMPDB[("employee_db<br/>PostgreSQL")]
        PAYDB[("payroll_db<br/>PostgreSQL")]
        LEADB[("leave_db<br/>PostgreSQL")]
    end

    %% ================= Message Broker =================
    MQ{{"<b>Message Broker</b><br/>RabbitMQ / Kafka<br/>topic: hris.events"}}

    %% ---------- SYNC: Client to Gateway ----------
    WEB -->|"SYNC · HTTPS/REST"| GW
    MOB -->|"SYNC · HTTPS/REST"| GW

    %% ---------- SYNC: Gateway to Services ----------
    GW -->|"SYNC · REST<br/>/api/employees"| EMP
    GW -->|"SYNC · REST<br/>/api/payroll"| PAY
    GW -->|"SYNC · REST<br/>/api/leaves"| LEA

    %% ---------- SYNC: Service to its own database ----------
    EMP -->|"SYNC · SQL"| EMPDB
    PAY -->|"SYNC · SQL"| PAYDB
    LEA -->|"SYNC · SQL"| LEADB

    %% ---------- SYNC: Service to Service ----------
    PAY -->|"SYNC · REST<br/>GET /employees/:id"| EMP
    LEA -->|"SYNC · REST<br/>GET /employees/:id"| EMP

    %% ---------- ASYNC: Publish / Subscribe ----------
    EMP -.->|"ASYNC pub · EmployeeHired<br/>EmployeeTerminated"| MQ
    LEA -.->|"ASYNC pub · LeaveApproved<br/>LeaveCancelled"| MQ
    PAY -.->|"ASYNC pub · PayrollProcessed"| MQ
    MQ -.->|"ASYNC sub · Employee*<br/>Leave*"| PAY
    MQ -.->|"ASYNC sub · Employee*"| LEA

    %% ================= Styling =================
    classDef client fill:#DBEAFE,stroke:#1E40AF,stroke-width:2px,color:#111827
    classDef gateway fill:#FDE68A,stroke:#B45309,stroke-width:3px,color:#111827
    classDef service fill:#D1FAE5,stroke:#047857,stroke-width:2px,color:#111827
    classDef database fill:#EDE9FE,stroke:#6D28D9,stroke-width:2px,color:#111827
    classDef broker fill:#FECACA,stroke:#B91C1C,stroke-width:3px,color:#111827

    class WEB,MOB client
    class GW gateway
    class EMP,PAY,LEA service
    class EMPDB,PAYDB,LEADB database
    class MQ broker
```

### Communication summary

| From | To | Type | Protocol / Mechanism | Purpose |
|------|----|------|----------------------|---------|
| Web / Mobile App | API Gateway | **SYNC** | HTTPS/REST | All external traffic |
| API Gateway | employee-service | **SYNC** | HTTP/REST `/api/employees` | Employee CRUD |
| API Gateway | payroll-service | **SYNC** | HTTP/REST `/api/payroll` | Pay runs, payslips |
| API Gateway | leave-service | **SYNC** | HTTP/REST `/api/leaves` | Leave applications |
| Each service | Its own database | **SYNC** | SQL (private, not shared) | Persistence |
| payroll-service | employee-service | **SYNC** | REST `GET /employees/:id` | Verify job grade before a pay run |
| leave-service | employee-service | **SYNC** | REST `GET /employees/:id` | Resolve the approving manager |
| employee-service | Broker | **ASYNC** | `EmployeeHired`, `EmployeeTerminated` | Announce workforce changes |
| leave-service | Broker | **ASYNC** | `LeaveApproved`, `LeaveCancelled` | Announce time-off outcomes |
| payroll-service | Broker | **ASYNC** | `PayrollProcessed` | Announce a completed pay run |
| Broker | payroll-service | **ASYNC** | subscribes to `Employee*`, `Leave*` | Open/close salary records, apply unpaid-leave deductions |
| Broker | leave-service | **ASYNC** | subscribes to `Employee*` | Provision / freeze leave balances |

### Architectural rules enforced by this diagram

1. **Single entry point.** Clients never talk to a microservice directly; every external request
   passes through the API Gateway, which handles authentication, routing, and rate limiting.
2. **Database per service.** Each microservice owns a private database. No service issues a query
   against another service's database — that is what makes the contexts independently deployable.
3. **Synchronous only where an answer is needed now.** REST calls are used for read-only lookups
   that must return before the caller can proceed (resolving an approver, verifying a job grade).
4. **Asynchronous for state propagation.** Anything that only needs to *eventually* be reflected in
   another context (a new hire, an approved leave, a completed pay run) travels as an event through
   the broker, so a slow or down service never blocks the publisher.

---

## Supporting View — Leave-to-Payroll Event Flow

This sequence diagram shows the headline integration: an approved unpaid leave becoming a payroll
deduction, without Leave Management ever calling Payroll directly.

```mermaid
sequenceDiagram
    autonumber
    actor EE as Employee (Web/Mobile)
    participant GW as API Gateway
    participant LEA as leave-service
    participant EMP as employee-service
    participant MQ as Message Broker
    participant PAY as payroll-service

    EE->>GW: POST /api/leaves (SYNC · REST)
    GW->>LEA: POST /leaves (SYNC · REST)
    LEA->>EMP: GET /employees/:id (SYNC · REST)
    EMP-->>LEA: employee + reporting line
    LEA-->>GW: 201 Created (status = PENDING)
    GW-->>EE: 201 Created

    Note over LEA: Manager approves the application

    LEA-)MQ: publish LeaveApproved (ASYNC)
    MQ-)PAY: deliver LeaveApproved (ASYNC)
    PAY->>PAY: record unpaid-day deduction
    PAY-)MQ: publish PayrollProcessed (ASYNC)
```

---

## Regenerating the diagram image

The raw source lives in [`architecture.mmd`](./architecture.mmd). `architecture-diagram.png` was
generated from it. To regenerate after an edit:

**Option A — Mermaid Live Editor (no install):**

1. Go to <https://mermaid.live>.
2. Paste the contents of `architecture.mmd`.
3. Click **Actions → PNG**.
4. Save the download as `docs/architecture-diagram.png`, overwriting the old file.

**Option B — Mermaid CLI (local):**

```bash
npm install -g @mermaid-js/mermaid-cli
mmdc -i docs/architecture.mmd -o docs/architecture-diagram.png -b white -w 2200
```
