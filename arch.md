# NSEE India — Architecture & Flow Diagrams

> Single source of truth for system flows. Diagrams use Mermaid (renders on GitHub/GitLab/VS Code).
> Version 1.1 · October 2026 · Owner: Dev Team

## Table of Contents

1. [Student End-to-End Journey](#1-student-end-to-end-journey)
2. [Hexagonal Request Flow](#2-hexagonal-request-flow)
3. [Auth Flow](#3-auth-flow)
4. [USID Generation](#4-usid-generation)
5. [Maker–Checker Approval](#5-makerchecker-approval)
6. [Payment Flow](#6-payment-flow)
7. [Exam Session Flow](#7-exam-session-flow)
8. [Sprint Development Loop](#8-sprint-development-loop)
9. [Release & Deployment](#9-release--deployment)
10. [Feature Flag Flow](#10-feature-flag-flow)
11. [Role Hierarchy](#11-role-hierarchy)
12. [Database ER](#12-database-er)
13. [Architecture Layers](#13-architecture-layers)

---

## 1. Student End-to-End Journey

```mermaid
flowchart TD
    A([Student visits nseeindia.com]) --> B[Register<br/>mobile + password]
    B --> C{OTP verify}
    C -->|fail| B
    C -->|success| D[Account ACTIVE]
    D --> E["Generate USID<br/>NSEE/2026/MH/PUN/000123"]
    E --> F["Progressive Profile<br/>6 sections"]
    F --> G{Completeness 100%?}
    G -->|no| F
    G -->|yes| H[Browse Exams]
    H --> I[Apply<br/>eligibility check]
    I --> J{Payment 99<br/>Razorpay}
    J -->|fail| J
    J -->|success| K[Application APPLIED]
    K --> L[Admit Card<br/>PDF + QR]
    L --> M[Live Exam Session<br/>+ Proctoring]
    M --> N[Auto Evaluation]
    N --> O[Result + Rank<br/>District/State/National]

    style A fill:#0B3D91,color:#fff
    style O fill:#138808,color:#fff
    style J fill:#FF9933,color:#000
    style C fill:#F5A623,color:#000
```

### Stage Reference

| # | Stage | Trigger | Tables | Endpoint |
|---|-------|---------|--------|----------|
| 1 | Register | Mobile + password | `users`, `otp` | `POST /auth/register` |
| 1b | Verify OTP | 6-digit code | `otp`, `users` | `POST /auth/verify-otp` |
| 2 | USID issue | Profile start | `students`, `usid_sequence` | `POST /students/register` |
| 3 | Profile | Section-wise save | `student_profiles`, `profile_documents` | `PATCH /profiles/me` |
| 4 | Apply | Eligibility pass | `exam_applications` | `POST /applications` |
| 5 | Pay | 99 INR order | `payments`, `payment_webhooks` | `POST /payments/order` |
| 6 | Admit card | Release date reached | `admit_cards` | `GET /admit-cards/me` |
| 7 | Exam | Start button | `exam_sessions`, `session_responses`, `proctoring_events` | `POST /sessions/start` |
| 8 | Result | Admin publishes | `results`, `ranks` | `GET /results/me` |

---

## 2. Hexagonal Request Flow

```mermaid
flowchart LR
    subgraph Client
        C[React SPA]
    end

    subgraph Presentation["presentation/"]
        Ctrl["StudentController<br/>@Valid @PreAuthorize"]
    end

    subgraph Application["application/"]
        Svc["RegisterStudentService<br/>@Transactional"]
    end

    subgraph Domain["domain/"]
        PortIn["RegisterStudentUseCase (port-in)"]
        PortOut["StudentRepositoryPort (port-out)"]
        Model[Student aggregate]
    end

    subgraph Infra["infrastructure/"]
        Adapter["StudentRepositoryAdapter<br/>@Repository"]
        Jpa["StudentJpaRepository<br/>Spring Data"]
        Mapper[StudentJpaMapper]
    end

    DB[(PostgreSQL)]

    C -->|HTTP| Ctrl
    Ctrl --> PortIn
    PortIn -.implements.-> Svc
    Svc --> PortOut
    Svc --> Model
    PortOut -.implements.-> Adapter
    Adapter --> Mapper
    Adapter --> Jpa
    Jpa --> DB

    style Domain fill:#e8f4ff,stroke:#0B3D91
    style Application fill:#fff4e6,stroke:#FF9933
    style Infra fill:#e8f8e8,stroke:#138808
    style Presentation fill:#f0f0f0,stroke:#4A4A4A
```

**Dependency rule (ArchUnit-enforced):**

```
presentation -> application -> domain
                    ^
              infrastructure  (implements domain ports)
```

`domain/` imports nothing but `java.*`.

---

## 3. Auth Flow

### 3a. Register + OTP + JWT

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant API as Backend
    participant R as Redis
    participant DB as PostgreSQL
    participant SMS as SMS Gateway

    C->>API: POST /auth/register mobile+password
    API->>API: validate + BCrypt hash
    API->>DB: INSERT users status=PENDING_OTP
    API->>API: generate 6-digit OTP
    API->>R: SET otp:{mobile} TTL=5min
    API->>DB: INSERT otp hash purpose=REGISTER
    API->>SMS: send OTP
    SMS-->>C: SMS delivered
    API-->>C: 200 userId

    C->>API: POST /auth/verify-otp mobile+otp
    API->>R: GET otp:{mobile}
    R-->>API: hash
    API->>API: compare + check attempts
    API->>DB: UPDATE users SET status=ACTIVE
    API->>DB: INSERT refresh_tokens hash+family_id
    API->>DB: INSERT login_audit success=true
    API-->>C: 200 accessToken + refreshToken
```

### 3b. JWT Refresh (Rotating)

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant API as Backend
    participant DB as PostgreSQL

    Note over C,API: 15 min later

    C->>API: POST /auth/refresh refreshToken
    API->>DB: find refresh_tokens WHERE hash=?
    alt token valid
        API->>DB: UPDATE revoked=true, INSERT new token
        API-->>C: 200 new pair
    else token reused
        API->>DB: revoke entire family_id
        API-->>C: 401 TOKEN_REUSE
    end
```

**Rate limits (Redis + Bucket4j):**
- 3 OTP requests / mobile / 10 min
- 5 verify attempts / OTP (then invalidate)
- 10 login attempts / IP / min

---

## 4. USID Generation

**Format:** `NSEE/YYYY/STATE/DIST/000123`

```mermaid
sequenceDiagram
    autonumber
    participant S1 as Student 1
    participant S2 as Student 2
    participant API as Backend
    participant DB as PostgreSQL

    par 100 parallel requests
        S1->>API: POST /students/register
        S2->>API: POST /students/register
    end

    API->>DB: BEGIN
    API->>DB: SELECT next_usid MH PUN 2026
    Note over DB: INSERT ON CONFLICT<br/>DO UPDATE RETURNING last_value<br/>atomic
    DB-->>API: 47
    API->>DB: INSERT students usid=NSEE/2026/MH/PUN/000047
    API->>DB: INSERT audit_logs
    API->>DB: COMMIT
    API-->>S1: 201 usid

    API->>DB: BEGIN
    API->>DB: SELECT next_usid MH PUN 2026
    DB-->>API: 48
    API->>DB: INSERT students usid=NSEE/2026/MH/PUN/000048
    API->>DB: COMMIT
    API-->>S2: 201 usid

    Note over DB: 100 parallel gives 100 unique USIDs<br/>no lock contention
```

**Atomic function:**

```sql
CREATE OR REPLACE FUNCTION next_usid(p_state VARCHAR(4), p_district VARCHAR(8), p_year SMALLINT)
RETURNS TEXT AS $$
DECLARE v_seq INTEGER;
BEGIN
    INSERT INTO usid_sequence (state_code, district_code, year, last_value)
    VALUES (p_state, p_district, p_year, 1)
    ON CONFLICT (state_code, district_code, year)
    DO UPDATE SET last_value = usid_sequence.last_value + 1
    RETURNING last_value INTO v_seq;

    RETURN format('NSEE/%s/%s/%s/%s', p_year, p_state, p_district, lpad(v_seq::text, 6, '0'));
END;
$$ LANGUAGE plpgsql;
```

---

## 5. Maker–Checker Approval

```mermaid
stateDiagram-v2
    [*] --> DRAFT: Exam Setter creates
    DRAFT --> DRAFT: edit details
    DRAFT --> PENDING: submit for approval
    PENDING --> APPROVED: Super Admin approves
    PENDING --> DRAFT: Super Admin rejects
    APPROVED --> PUBLISHED: Super Admin publishes
    PUBLISHED --> ARCHIVED: auto after result
    ARCHIVED --> [*]

    note right of PENDING
        Setter CANNOT approve own exam
        DB check + PreAuthorize
    end note

    note right of PUBLISHED
        Visible on public site
        Notices auto-generated
        Applications open
    end note
```

```mermaid
sequenceDiagram
    autonumber
    participant ES as Exam Setter
    participant SA as Super Admin
    participant PUB as Public

    ES->>ES: create exam DRAFT
    ES->>SA: submit for approval PENDING
    alt approve
        SA->>SA: status=APPROVED
        SA->>SA: status=PUBLISHED
        SA->>PUB: exam visible
        SA->>PUB: notices auto-generated
    else reject
        SA->>ES: status=DRAFT + remarks
    end
```

Every transition writes to `exam_approvals`, `exam_audit`, and `audit_logs`.

---

## 6. Payment Flow

```mermaid
sequenceDiagram
    autonumber
    participant St as Student
    participant API as Backend
    participant RZ as Razorpay
    participant DB as PostgreSQL
    participant N as Notification

    St->>API: POST /payments/order applicationId
    API->>API: check eligibility
    API->>API: idempotency key = appId
    API->>RZ: create order 9900 paise
    RZ-->>API: order_id + key_id
    API->>DB: INSERT payments status=CREATED idempotency_key
    API-->>St: 200 orderId + keyId

    St->>RZ: Razorpay Checkout UPI/Card
    RZ-->>St: payment success
    St->>API: POST /payments/verify paymentId + signature

    par Webhook server-to-server
        RZ->>API: POST /webhooks/razorpay event_id + payload
        API->>API: verify HMAC signature
        API->>DB: INSERT payment_webhooks event_id UNIQUE
        alt duplicate event
            API-->>RZ: 200 ignored
        else new event
            API->>DB: UPDATE payments status=CAPTURED
            API->>DB: UPDATE exam_applications status=APPLIED
            API->>N: queue notification
            API-->>RZ: 200 OK
        end
    and Client verify
        API->>API: verify signature
        API-->>St: 200 status=APPLIED
    end
```

**Idempotency guarantees:**
- `payments.idempotency_key` UNIQUE — double-tap cannot create 2 orders
- `payment_webhooks.event_id` UNIQUE — Razorpay retry cannot double-apply
- Webhook handler is transactional + uses `ON CONFLICT DO NOTHING`

**Daily reconciliation:**

```mermaid
flowchart LR
    A[03:00 IST cron] --> B[Fetch Razorpay settlement]
    B --> C[Compare with payments table]
    C --> D{Mismatch?}
    D -->|no| E[Log OK]
    D -->|yes| F[Flag to DLQ]
    F --> G[Notify finance]

    style F fill:#D32F2F,color:#fff
```

---

## 7. Exam Session Flow

```mermaid
sequenceDiagram
    autonumber
    participant St as Student Browser
    participant API as Backend
    participant WS as WebSocket
    participant DB as PostgreSQL
    participant R as Redis
    participant Pr as Proctor Dashboard

    St->>API: POST /sessions/start
    API->>API: verify application + admit card
    API->>API: decrypt paper AES-256
    API->>DB: INSERT exam_sessions IN_PROGRESS + ends_at
    API->>DB: INSERT session_responses xN
    API-->>St: 200 questions + timer

    St->>WS: CONNECT /ws/session/{id}
    WS-->>St: type TIMER remaining 5400

    loop every 5 seconds
        St->>WS: type SAVE_BATCH answers list
        WS->>DB: UPSERT session_responses
        WS->>DB: INSERT session_events
        WS-->>St: type SAVED
    end

    loop heartbeat
        St->>WS: PING
        WS->>R: SET heartbeat:{sessionId} TTL=60
        WS-->>St: PONG
    end

    alt proctoring violation
        St->>WS: type VIOLATION kind TAB_SWITCH
        WS->>DB: INSERT proctoring_events
        WS->>WS: count violations
        WS->>Pr: push live alert
        alt count >= 3
            WS->>DB: UPDATE session TERMINATED
            WS-->>St: type TERMINATED
        else
            WS-->>St: type WARN modal true
        end
    end

    St->>API: POST /sessions/{id}/submit
    API->>DB: UPDATE session SUBMITTED
    API->>DB: lock session_responses
    API->>R: PUBLISH SessionSubmitted
    API-->>St: 200 submitted

    Note over API,DB: async evaluation writes results table
```

**Failure handling:**
- Browser crash — reconnect WS, resume from server state (Redis holds last 60s)
- Network drop — client buffers locally, resends on reconnect
- Server restart — session state in Postgres, heartbeat resumes

---

## 8. Sprint Development Loop

```mermaid
flowchart TD
    Start([Sprint N begins]) --> A[1 OpenAPI contract]
    A --> B[2 Flyway migration]
    B --> C[3 Domain pure Java]
    C --> D[4 Domain unit tests >= 90pct]
    D --> E{pass?}
    E -->|no| C
    E -->|yes| F[5 Application use cases]
    F --> G[6 Application tests]
    G --> H[7 Infra adapters]
    H --> I[8 Integration tests<br/>Testcontainers]
    I --> J[9 REST controller]
    J --> K[10 E2E tests<br/>REST Assured]
    K --> L[11 React UI]
    L --> M[12 Frontend tests<br/>Vitest + RTL]
    M --> N[13 Full CI]
    N --> O{green?}
    O -->|no| Fix[Fix]
    Fix --> N
    O -->|yes| P[14 Deploy staging]
    P --> Q[15 Manual QA + sign-off]
    Q --> R[16 Freeze + tag]
    R --> S[17 Next sprint]

    style D fill:#138808,color:#fff
    style N fill:#138808,color:#fff
    style R fill:#0B3D91,color:#fff
```

**Gate:** Step N+1 blocked until step N passes. Enforced by branch protection + CI.

---

## 9. Release & Deployment

```mermaid
gitGraph
    commit id: "sprint0"
    branch develop
    checkout develop
    commit id: "v0.1.0-core"
    branch feature/auth
    checkout feature/auth
    commit id: "auth-wip"
    commit id: "auth-tests"
    checkout develop
    merge feature/auth tag: "v0.2.0-auth"
    branch feature/student
    checkout feature/student
    commit id: "student"
    checkout develop
    merge feature/student tag: "v0.3.0-student"
    branch feature/profile
    checkout feature/profile
    commit id: "profile"
    checkout develop
    merge feature/profile tag: "v0.4.0-profile"
    checkout main
    merge develop tag: "PUBLIC-LAUNCH-Week-7"
    checkout develop
    commit id: "sprint4"
```

### Environment Pipeline

```mermaid
flowchart LR
    F[feature/*] -->|PR| D[develop]
    D -->|QA sign-off| M[main]
    D -->|auto-deploy| S["STAGING<br/>VPS 1"]
    M -->|tag + deploy| P["PRODUCTION<br/>VPS 2"]

    style S fill:#FF9933,color:#000
    style P fill:#138808,color:#fff
```

### CI on PR to develop

```
1. mvn spotless:check
2. mvn compile
3. mvn test                    coverage < 85% = FAIL
4. mvn verify                  Testcontainers
5. mvn failsafe:integration-test
6. npm run lint
7. npm run test
8. npm run build
9. gitleaks detect             secret scan
```

### CD on push to main

```
1. Build multi-stage Docker image
2. Push to ghcr.io/nsee/api:sha-<short>
3. SSH to VPS
4. docker compose pull
5. docker compose up -d
6. Flyway migrations run on startup
7. healthcheck /actuator/health returns 200 OK
8. If healthcheck fails -> auto-rollback to previous image
```

### Release Tags

| Tag | Sprint |
|-----|--------|
| `v0.1.0-core` | Sprint 0 |
| `v0.2.0-auth` | Sprint 1 |
| `v0.3.0-student` | Sprint 2 |
| `v0.4.0-profile` | Sprint 3 — **PUBLIC LAUNCH (Week 7)** |
| `v0.5.0-geography` | Sprint 4 |
| `v0.6.0-notice` | Sprint 5 |
| `v0.7.0-query` | Sprint 6 |
| `v0.8.0-exam` | Sprint 7 |
| ... | ... |
| `v1.0.0-production` | Sprint 20 |

---

## 10. Feature Flag Flow

```mermaid
flowchart LR
    A[Build feature<br/>flag false] --> B[Deploy staging<br/>flag OFF]
    B --> C{QA + stakeholder<br/>sign-off}
    C -->|reject| A
    C -->|approve| D[UPDATE feature_flags<br/>SET enabled true]
    D --> E[Redis PUBLISH<br/>flag-changed]
    E --> F[API cache invalidated]
    E --> G[Frontend refetches on nav]
    F --> H[Endpoint active]
    G --> I[UI unlocked]
    H --> J[Live feature]
    I --> J

    style J fill:#138808,color:#fff
    style A fill:#F5A623
```

**Backend gate:**

```java
@GetMapping("/applications")
@PreAuthorize("@flags.isEnabled('application')")
public List<ApplicationDto> list() { ... }
```

**Frontend gate:**

```tsx
{flags.application ? <ApplyButton /> : <ComingSoonBadge />}
```

---

## 11. Role Hierarchy

```mermaid
flowchart TB
    SA["SUPER_ADMIN<br/>Global"]
    ES["EXAM_SETTER<br/>State/National"]
    STA["STATE_ADMIN<br/>State"]
    DC["DISTRICT_CONTROLLER<br/>District"]
    CC["CENTRE_CONTROLLER<br/>Centre"]
    INV["INVIGILATOR<br/>Centre"]
    STU["STUDENT<br/>Self"]
    SUP["SUPPORT<br/>Read-only"]

    SA --> ES
    SA --> STA
    SA --> SUP
    STA --> DC
    DC --> CC
    CC --> INV
    STU -->|applies_to| ES

    style SA fill:#0B3D91,color:#fff
    style STU fill:#138808,color:#fff
    style ES fill:#FF9933,color:#000
```

| Role | Scope | Key Permissions |
|------|-------|-----------------|
| `SUPER_ADMIN` | Global | Create admins, approve exams/notices, global config |
| `EXAM_SETTER` | State/National | Create exams, question bank, generate papers, publish |
| `STATE_ADMIN` | State | Manage districts, state reports |
| `DISTRICT_CONTROLLER` | District | Add centres, assign centre controllers |
| `CENTRE_CONTROLLER` | Centre | Conduct offline exam, verify students, unlock PCs |
| `INVIGILATOR` | Centre/Remote | Monitor live exam, warn, terminate |
| `STUDENT` | Self | Register, profile, apply, pay, exam, result |
| `SUPPORT` | Read-only | Queries, grievances, audit |

---

## 12. Database ER

```mermaid
erDiagram
    USERS ||--o{ USER_ROLES : has
    ROLES ||--o{ USER_ROLES : assigned
    USERS ||--o| STUDENTS : "is_a"
    STUDENTS ||--|| STUDENT_PROFILES : has
    STUDENTS ||--o{ PROFILE_DOCUMENTS : uploads
    STUDENTS ||--o{ STUDENT_EDUCATION : has
    STUDENTS ||--o{ EXAM_APPLICATIONS : applies
    EXAMS ||--o{ EXAM_APPLICATIONS : receives
    EXAMS ||--o{ EXAM_PAPERS : generates
    EXAMS ||--o{ QUESTIONS : contains
    QUESTIONS ||--o{ QUESTION_OPTIONS : has
    EXAM_PAPERS ||--o{ PAPER_QUESTIONS : includes
    EXAM_PAPERS ||--o{ PAPER_KEYS : has
    EXAM_APPLICATIONS ||--o| PAYMENTS : paid_by
    EXAM_APPLICATIONS ||--o| ADMIT_CARDS : issued
    EXAM_APPLICATIONS ||--o| EXAM_SESSIONS : runs
    EXAM_SESSIONS ||--o{ SESSION_RESPONSES : records
    EXAM_SESSIONS ||--o{ PROCTORING_EVENTS : logs
    EXAM_SESSIONS ||--o| RESULTS : produces
    RESULTS ||--o{ RANKS : has
    STATES ||--o{ DISTRICTS : contains
    DISTRICTS ||--o{ CENTRES : contains
    CENTRES ||--o{ CENTRE_STAFF : staffed
    EXAMS ||--o{ NOTICES : about
    QUERIES ||--o{ QUERY_RESPONSES : answered

    USERS {
        bigint id PK
        citext email UK
        varchar mobile UK
        varchar password_hash
        varchar status
    }
    STUDENTS {
        bigint id PK
        varchar usid UK
        bigint user_id FK
        varchar full_name
        date date_of_birth
    }
    EXAMS {
        bigint id PK
        varchar code UK
        varchar title
        varchar status
        int fee_paise
    }
    EXAM_APPLICATIONS {
        bigint id PK
        varchar application_no UK
        bigint student_id FK
        bigint exam_id FK
        varchar status
    }
    PAYMENTS {
        bigint id PK
        varchar order_id UK
        varchar payment_id UK
        varchar idempotency_key UK
        int amount_paise
        varchar status
    }
```

Full DDL: see `nsee-backend/src/main/resources/db/migration/`.

---

## 13. Architecture Layers

```mermaid
flowchart TB
    subgraph P["presentation/"]
        P1[Controllers]
        P2[Request DTOs]
        P3[Response DTOs]
    end

    subgraph A["application/"]
        A1[Services]
        A2[Commands]
        A3[Mappers]
    end

    subgraph D["domain/ (no framework)"]
        D1[Models]
        D2[Ports in]
        D3[Ports out]
        D4[Events]
        D5[Exceptions]
    end

    subgraph I["infrastructure/"]
        I1[JPA entities]
        I2[Adapters]
        I3[External APIs]
        I4[Redis]
    end

    P1 --> D2
    A1 -.implements.-> D2
    A1 --> D3
    I2 -.implements.-> D3
    I2 --> I1
    P1 --> A1
    A1 --> D1

    style D fill:#e8f4ff,stroke:#0B3D91,stroke-width:2px
```

**Rule:** arrows only point **inward** toward `domain/`. ArchUnit test fails the build if violated.

### Feature Folder Template

```
features/exam/
├── domain/
│   ├── model/          # Aggregate roots, entities, VOs
│   ├── event/          # Domain events
│   ├── exception/      # Domain exceptions
│   └── port/
│       ├── in/         # Use case interfaces
│       └── out/        # Repository/external interfaces
├── application/
│   ├── service/        # Implements use cases
│   ├── dto/            # Commands, responses, mappers
│   └── event/          # Event publishers
├── infrastructure/
│   ├── persistence/    # JPA entities, repositories, adapters
│   ├── messaging/      # Redis/queue adapters
│   └── external/       # Third-party adapters
├── presentation/
│   ├── ExamController.java
│   ├── request/
│   └── response/
└── ExamModuleConfig.java
```

---

## Appendix — Rendering the Diagrams

### On GitHub / GitLab
Commit this file as `architecture.md`. Mermaid renders automatically in the Markdown view.

### Locally (PNG/SVG export)
```bash
npm install -g @mermaid-js/mermaid-cli

# extract each block to its own .mmd file, then:
mmdc -i docs/diagrams/01-journey.mmd -o docs/diagrams/01-journey.svg
```

### CI auto-render
```yaml
- name: Render diagrams
  run: |
    npm i -g @mermaid-js/mermaid-cli
    for f in docs/diagrams/*.mmd; do
      mmdc -i "$f" -o "${f%.mmd}.svg"
    done
```

### VS Code
Install **"Markdown Preview Mermaid Support"** → `Ctrl+Shift+V` renders live.

---

*End of document. Update version on every change.*
