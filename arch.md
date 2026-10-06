```markdown
# NSEE India — Architecture

**National Scholarship Entrance Exam Portal**

System architecture document for a government-grade, fully online, remote-proctored examination platform.

| Field | Value |
|---|---|
| Document | Architecture |
| Project | NSEE India |
| Domain | nseeindia.com |
| Mode | Online CBT (Remote Proctored) |
| Version | 1.0 |
| Last Updated | October 2026 |

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [High-Level System Architecture](#2-high-level-system-architecture)
3. [Hexagonal Architecture](#3-hexagonal-architecture)
4. [Feature Module Architecture](#4-feature-module-architecture)
5. [Component Architecture](#5-component-architecture)
6. [Data Architecture](#6-data-architecture)
7. [Business Logic Architecture](#7-business-logic-architecture)
8. [Exam Session Architecture](#8-exam-session-architecture)
9. [Proctoring Architecture](#9-proctoring-architecture)
10. [Command Centre Architecture](#10-command-centre-architecture)
11. [Question Paper Security Architecture](#11-question-paper-security-architecture)
12. [Payment Architecture](#12-payment-architecture)
13. [Notification Architecture](#13-notification-architecture)
14. [Authentication & Authorization](#14-authentication--authorization)
15. [Integration Architecture](#15-integration-architecture)
16. [Deployment Architecture](#16-deployment-architecture)
17. [Security Architecture](#17-security-architecture)
18. [Observability & Monitoring](#18-observability--monitoring)
19. [Scalability & Performance](#19-scalability--performance)
20. [Disaster Recovery](#20-disaster-recovery)
21. [Technology Decisions](#21-technology-decisions)
22. [Architecture Decision Records](#22-architecture-decision-records)
23. [Constraints & Assumptions](#23-constraints--assumptions)

---

## 1. Architecture Overview

### 1.1 Goals

| Goal | Description |
|---|---|
| **Correctness** | NTA-grade business logic for exams, results, normalization, tie-breaking |
| **Integrity** | AI + human proctoring, encrypted papers, immutable audit |
| **Scalability** | Support 100,000+ concurrent exam sessions |
| **Availability** | 99.9% uptime during exam windows |
| **Security** | DPDP Act 2023, CERT-In, GIGW compliant |
| **Maintainability** | Hexagonal, feature-first, testable, documented |
| **Observability** | Full tracing, metrics, logs, alerts |

### 1.2 Architecture Style

- **Modular monolith** (not microservices initially)
- **Hexagonal (Ports & Adapters)** per feature
- **Feature-first** folder structure
- **Event-driven** for internal async operations
- **Stateless** services where possible
- **Database per aggregate** (logical, single physical DB)

### 1.3 Architectural Principles

1. Domain logic is pure — no framework imports
2. Every feature is a vertical slice
3. Dependencies point inward (domain has no outward deps)
4. Ports define contracts, adapters implement them
5. Feature flags for staged rollout
6. Every write action is audited
7. Papers encrypted until exam start
8. Results follow NTA normalization and tie-breaking rules

---

## 2. High-Level System Architecture

```

┌─────────────────────────────────────────────────────────────────────┐
│                          CLIENT LAYER                               │
│                                                                     │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐            │
│   │ Student  │  │ Proctor  │  │  Admin   │  │ Command  │            │
│   │ Browser  │  │ Browser  │  │ Browser  │  │  Centre  │            │
│   └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘            │
└────────┼─────────────┼─────────────┼─────────────┼──────────────────┘
│             │             │             │
└─────────────┴──────┬──────┴─────────────┘
│
┌─────────▼─────────┐
│   Cloudflare      │
│   CDN + WAF +     │
│   DDoS Protection │
└─────────┬─────────┘
│
┌─────────▼─────────┐
│   Nginx           │
│   Reverse Proxy   │
│   TLS Termination │
└─────────┬─────────┘
│
┌────────────────────┼────────────────────┐
│                    │                    │
┌─────▼─────┐        ┌─────▼─────┐        ┌─────▼─────┐
│  React    │        │  Spring   │        │ WebSocket │
│  SPA      │        │  Boot API │        │  Server   │
│  (Vite)   │        │           │        │  (STOMP)  │
└───────────┘        └─────┬─────┘        └─────┬─────┘
│                    │
┌────────────────────┼────────────────────┘
│                    │
┌─────▼─────┐        ┌─────▼─────┐        ┌───────────┐
│PostgreSQL │        │  Redis    │        │  S3/MinIO │
│   16      │        │   7       │        │  Storage  │
│           │        │           │        │           │
│ Primary   │        │ Session   │        │ Docs      │
│ Read      │        │ Cache     │        │ Papers    │
│ Replicas  │        │ Pub/Sub   │        │ PDFs      │
└───────────┘        │ Streams   │        └───────────┘
└───────────┘
│
┌─────────┴─────────┐
│                   │
┌─────▼─────┐      ┌─────▼─────┐
│ Razorpay  │      │  SMS /    │
│ Payment   │      │  Email /  │
│ Gateway   │      │ WhatsApp  │
└───────────┘      └───────────┘

```

---

## 3. Hexagonal Architecture

### 3.1 Core Concept

Every feature is organized around a **domain core** surrounded by **ports** and **adapters**.

```

```

### 3.2 Layer Responsibilities

| Layer | Responsibility | Dependencies |
|---|---|---|
| `domain` | Business rules, entities, VOs, events | JDK only |
| `application` | Use cases, orchestration, DTOs | `domain` |
| `infrastructure` | Persistence, messaging, external APIs | `domain.port.out` |
| `presentation` | HTTP controllers, WebSocket, request/response | `domain.port.in` |

### 3.3 Dependency Rule

```

presentation  →  application  →  domain  ←  infrastructure
↑
│
(implements ports)

```

**Nothing depends on `presentation`. `domain` depends on nothing.**

### 3.4 Port Types

| Port | Location | Purpose |
|---|---|---|
| **Input Port** (Driving) | `domain/port/in` | Use case interface — called by adapters |
| **Output Port** (Driven) | `domain/port/out` | Repository/external interface — implemented by adapters |

### 3.5 Example — Exam Feature

```

features/exam/
├── domain/
│   ├── model/
│   │   ├── Exam.java                    (Aggregate Root)
│   │   ├── ExamStatus.java              (Enum)
│   │   └── ExamLevel.java               (Value Object)
│   ├── event/
│   │   ├── ExamCreatedEvent.java
│   │   ├── ExamApprovedEvent.java
│   │   └── ExamPublishedEvent.java
│   ├── exception/
│   │   ├── ExamNotFoundException.java
│   │   └── InvalidExamStateException.java
│   └── port/
│       ├── in/
│       │   ├── CreateExamUseCase.java
│       │   ├── ApproveExamUseCase.java
│       │   └── PublishExamUseCase.java
│       └── out/
│           ├── ExamRepositoryPort.java
│           └── ExamEventPublisherPort.java
├── application/
│   ├── service/
│   │   ├── CreateExamService.java
│   │   ├── ApproveExamService.java
│   │   └── PublishExamService.java
│   ├── dto/
│   │   ├── CreateExamCommand.java
│   │   ├── ExamResponse.java
│   │   └── ExamMapper.java
│   └── event/
│       └── ExamEventPublisher.java
├── infrastructure/
│   ├── persistence/
│   │   ├── ExamJpaEntity.java
│   │   ├── ExamJpaRepository.java
│   │   └── ExamRepositoryAdapter.java
│   └── messaging/
│       └── ExamRedisEventPublisher.java
├── presentation/
│   ├── ExamController.java
│   ├── request/
│   │   └── CreateExamRequest.java
│   └── response/
│       └── ExamResponse.java
└── ExamModuleConfig.java

```

---

## 4. Feature Module Architecture

### 4.1 Feature List & Dependencies

```

core (kernel)
│
├── auth ──────────┐
│                  │
├── student ───────┤
│                  │
├── profile ───────┤
│                  │
├── state ─────────┤
│                  │
├── district ──────┤
│                  │
├── exam ──────────┼──┐
│                  │  │
├── questionbank ──┤  │
│                  │  │
├── paper ─────────┘  │
│                     │
├── application ──────┤
│                     │
├── payment ──────────┤
│                     │
├── admitcard ────────┤
│                     │
├── examsession ──────┤
│                     │
├── proctoring ───────┤
│                     │
├── result ───────────┤
│                     │
├── notice ───────────┤
│                     │
├── query ────────────┤
│                     │
├── notification ─────┤
│                     │
├── admin ────────────┤
│                     │
├── audit ────────────┤
│                     │
├── commandcentre ────┤
│                     │
└── analytics ────────┘

```

### 4.2 Module Communication Rules

| Rule | Description |
|---|---|
| **No direct domain access** | Features communicate via **use cases** or **events** |
| **Shared kernel only** | Cross-cutting concerns live in `core` |
| **Events for async** | Long-running work is event-driven |
| **No circular deps** | Feature graph is acyclic |
| **Ports over direct calls** | If Feature A needs Feature B, it calls B's input port |

### 4.3 Cross-Feature Communication

```

┌──────────────┐                     ┌──────────────┐
│  Application │  publishes event    │              │
│   Feature    │ ──────────────────► │  Event Bus   │
└──────────────┘                     │  (Redis      │
│   Streams)   │
└──────┬───────┘
│
┌─────────────────┼─────────────────┐
│                 │                 │
┌─────▼─────┐     ┌─────▼─────┐     ┌─────▼─────┐
│ Payment   │     │Notification│    │  Audit    │
│ Feature   │     │  Feature  │     │  Feature  │
└───────────┘     └───────────┘     └───────────┘

```

---

## 5. Component Architecture

### 5.1 Backend Components

```

┌─────────────────────────────────────────────────────────────┐
│                    Spring Boot Application                  │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │            Presentation Layer (Adapters In)           │  │
│  │  REST Controllers │ WebSocket │ Filters │ Interceptors│  │
│  └───────────────────────────┬───────────────────────────┘  │
│                              │                              │
│  ┌───────────────────────────▼───────────────────────────┐  │
│  │              Application Layer                        │  │
│  │  Use Case Services │ Event Handlers │ Schedulers      │  │
│  └───────────────────────────┬───────────────────────────┘  │
│                              │                              │
│  ┌───────────────────────────▼───────────────────────────┐  │
│  │                 Domain Layer                          │  │
│  │  Aggregates │ Entities │ VOs │ Domain Services        │  │
│  │  Domain Events │ Business Rules │ Specifications      │  │
│  └───────────────────────────┬───────────────────────────┘  │
│                              │                              │
│  ┌───────────────────────────▼───────────────────────────┐  │
│  │            Infrastructure Layer (Adapters Out)        │  │
│  │  JPA Repos │ Redis Adapters │ S3 Client │ Razorpay    │  │
│  │  SMTP Client │ SMS Client │ WebSocket Gateway         │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              Cross-Cutting Concerns                   │  │
│  │  Security │ Logging │ Metrics │ Tracing │ Audit       │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘

```

### 5.2 Frontend Components

```

┌─────────────────────────────────────────────────────────────┐
│                    React SPA (Vite)                         │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                    Routing (React Router)             │  │
│  │  Public │ Student │ Admin │ Proctor │ Command Centre  │  │
│  └───────────────────────────┬───────────────────────────┘  │
│                              │                              │
│  ┌───────────────────────────▼───────────────────────────┐  │
│  │                 Layouts & Guards                      │  │
│  │  PublicLayout │ StudentLayout │ AdminLayout           │  │
│  │  AuthGuard │ RoleGuard │ FeatureFlagGuard             │  │
│  └───────────────────────────┬───────────────────────────┘  │
│                              │                              │
│  ┌───────────────────────────▼───────────────────────────┐  │
│  │                Feature Modules                        │  │
│  │  auth │ student │ profile │ exam │ application │ ...  │  │
│  └───────────────────────────┬───────────────────────────┘  │
│                              │                              │
│  ┌───────────────────────────▼───────────────────────────┐  │
│  │              Shared Layer                             │  │
│  │  Components │ Hooks │ API Client │ Utils │ Types      │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              State Management                         │  │
│  │  React Query (server state) │ Zustand (client state)  │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘

```

---

## 6. Data Architecture

### 6.1 Data Stores

| Store | Purpose | Data |
|---|---|---|
| **PostgreSQL 16** | Primary transactional data | Users, students, exams, applications, payments, results |
| **Redis 7** | Cache, session, pub/sub, streams | Sessions, OTP, timers, live proctoring events, event bus |
| **S3 / MinIO** | Object storage | Documents, photos, admit cards, scorecards, question papers |
| **Elasticsearch** *(optional)* | Full-text search, logs | Audit logs, query search |

### 6.2 Data Flow

```

Student Action
│
▼
React SPA  ──HTTP──►  Nginx  ──►  Spring Boot API
│
▼
┌─────────┴─────────┐
│                   │
Write path          Read path
│                   │
▼                   ▼
PostgreSQL           Redis Cache
│                   │
▼                   │
Publish Event             │
│                   │
▼                   │
Redis Streams             │
│                   │
┌─────────┼─────────┐         │
▼         ▼         ▼         │
Notification  Audit   Analytics     │
│
┌────────────────────┘
│
▼
Response to Client

```

### 6.3 Database Schema Strategy

| Aspect | Strategy |
|---|---|
| **Migrations** | Flyway, versioned `V1__init.sql`, `V2__auth.sql` |
| **Naming** | snake_case tables and columns |
| **Primary Keys** | UUID v7 (time-ordered) |
| **Timestamps** | `created_at`, `updated_at` on every table |
| **Soft Deletes** | `deleted_at` where applicable |
| **Audit Columns** | `created_by`, `updated_by` on mutable tables |
| **Append-only** | Audit, payments, violations, session responses |
| **Encryption** | Question papers, sensitive PII fields |
| **Partitioning** | Session responses, audit logs, proctoring events (by month) |

### 6.4 Core Tables (Grouped)

```

Identity & Access
users, roles, user_roles, otp, refresh_tokens, login_audit

Student
students, usid_sequence, student_profiles, profile_documents, profile_audit

Geography
states, districts

Exam Content
exams, exam_approvals, exam_audit
questions, question_options, question_tags
exam_papers, paper_questions, paper_keys

Application Lifecycle
exam_applications, application_drafts
payments, payment_webhooks
admit_cards

Exam Execution
exam_sessions, session_responses, session_events
proctoring_events, violations

Results
results, ranks, answer_key_challenges

Communication
notices, notice_audit
queries, query_responses
notifications, notification_templates, notification_logs

Compliance
audit_logs (append-only)
consent_records (DPDP)

```

### 6.5 Aggregates & Boundaries

| Aggregate Root | Contains | Consistency Boundary |
|---|---|---|
| `Student` | Profile, Documents | Per-student transaction |
| `Exam` | Approvals, Metadata | Per-exam transaction |
| `Question` | Options, Tags | Per-question transaction |
| `ExamPaper` | PaperQuestions, PaperKeys | Per-paper transaction |
| `Application` | Draft, Payment status | Per-application transaction |
| `Payment` | Webhooks | Per-payment transaction |
| `ExamSession` | Responses, Events | Per-session transaction |
| `Result` | Ranks, Challenges | Per-result transaction |

---

## 7. Business Logic Architecture

This section defines the operational rules modeled on **NTA / TCS iON** standards.

### 7.1 Application Lifecycle State Machine

```

DRAFT ──submit──► SUBMITTED ──pay──► PAYMENT_PENDING ──success──► APPLIED
│                  │                        │
│                  │                        └──failure──► PAYMENT_FAILED
│                  │
└──abandon──► EXPIRED                  CORRECTION_WINDOW (48–72 hrs)
│
▼
CORRECTED
│
▼
ADMIT_CARD_READY
│
▼
EXAM_ATTENDED
│
▼
RESULT_PUBLISHED

```

**Rules:**

| Rule | Enforcement |
|---|---|
| One candidate = one application per exam | Unique constraint `(student_id, exam_id)` |
| Email and mobile are immutable after registration | Update blocked in domain layer |
| Fee non-refundable once paid | Payment domain rule |
| Correction window is time-bound (48–72 hrs) | Scheduler + feature flag |
| Ineligible candidates rejected at verification | Eligibility engine in domain |
| Application cannot be withdrawn after submit | State machine blocks transition |

### 7.2 Eligibility Engine

```

Input:  Student profile + Exam eligibility rules
Output: ELIGIBLE | NOT_ELIGIBLE | NEEDS_REVIEW

Rules per level:
Class 5–10  → current class or previous class passed
Class 11–12 → stream matches exam paper (PCM / PCB / Commerce / Arts)
BCA         → 12th pass (any stream)
B.Tech      → 12th with PCM
MCA         → BCA / B.Sc. (IT / Maths)
M.Tech      → B.Tech / B.E.

Additional:
• State / district match (if exam is state-scoped)
• Age criteria (if applicable)
• Document completeness = 100%

```

**Implementation:** `EligibilityService` in `application` layer, calling `EligibilityRules` in `domain`.

### 7.3 Exam Centre Allocation (Logical)

Even though exams are remote-proctored, the concept of "district" is retained for **reporting, ranking, and proctor assignment**.

```

Input:  Application + Student district + Exam scope
Output: Assigned district (for reporting) + Proctor pool

Rules:

1. District derived from student's registered district
2. If exam is state-scoped, district must be within state
3. Proctor assigned from same state (language alignment)
4. Load balancing across proctor pool
5. No overbooking of proctors (max sessions per proctor)

```

> **Note:** No physical centre allocation. NTA's Aadhaar-based centre allocation rule does not apply to remote-proctored exams. The logic is adapted for **proctor allocation** instead.

### 7.4 Admit Card Release Logic

| Rule | Description |
|---|---|
| Release timing | 4–14 days before exam |
| Download only | No hard copy dispatched |
| Auth | Login with Application Number + DOB |
| Contents | Exam date, shift, time, proctor link, instructions |
| Re-issue | Allowed for corrections or re-exam |
| Watermark | Candidate photo + USID watermark |

### 7.5 Result Processing — NTA-Grade Logic

#### 7.5.1 Normalization (Multi-Session Exams)

```

Raw Score (per session)  →  Percentile Score  →  Normalized Score

Percentile Formula:
P = 100 × (Number of candidates in session with raw score ≤ candidate's raw score)
─────────────────────────────────────────────────────────────────────
Total number of candidates in that session

Precision: 7 decimal places (to reduce ties)
Topper:     Highest raw score in session → percentile = 100
Merit:      Based on normalized percentile, NOT raw marks

```

**Architecture:**

```

ExamSession (raw)  ──►  NormalizationService  ──►  Result (normalized)
│
├── per-session stats
├── percentile computation
└── normalized score

```

#### 7.5.2 Tie-Breaking Hierarchy

```

If two candidates have equal normalized score:

1. Higher score in Mathematics (or primary subject)
2. Higher score in Physics (or secondary subject)
3. Higher score in Chemistry (or tertiary subject)
4. Lower incorrect-to-correct ratio (overall)
5. Lower incorrect-to-correct ratio in Mathematics
6. Lower incorrect-to-correct ratio in Physics
7. Lower incorrect-to-correct ratio in Chemistry
8. Same rank awarded (final tie)

Removed from tie-breaking (from 2025 onwards):
✗ Age
✗ Application number

```

**Implementation:** `TieBreakerService` in `domain`, with pluggable strategy per exam.

#### 7.5.3 Rank Computation

```

Rank types:
• National rank
• State rank
• District rank
• Category rank (if applicable)

Rank assigned after:

1. Normalization
2. Tie-breaking
3. Category reservation (if applicable)

Rank is dense (no gaps) and shared on ties.

```

### 7.6 Answer Key Challenge Logic

```

Provisional Key Released
│
▼
Challenge Window (e.g., 3 days)
│
▼
Candidate challenges question(s) ──► Fee ₹200 per question
│
▼
Subject Expert Review
│
├── Accepted ──► Fee refunded + revised key applies to ALL candidates
│
└── Rejected ──► Fee forfeited + original key retained
│
▼
Final Key Released
│
▼
Results Recomputed (if key changed)

```

**Rules:**

| Rule | Description |
|---|---|
| Fee | ₹200 per question challenged |
| Refund | Only if challenge is accepted |
| Impact | Revised key applies to all candidates |
| Review | By subject experts, not automated |
| Audit | Every challenge logged immutably |

### 7.7 Scorecard & Result Download

| Rule | Description |
|---|---|
| Availability | After final result declaration |
| Auth | Application Number + DOB |
| DigiLocker | Scorecards pushed to DigiLocker |
| Format | PDF with QR verification |
| Retention | Available for 5 years |

---

## 8. Exam Session Architecture

### 8.1 Session Lifecycle

```

PRE_EXAM_CHECK ──► IDENTITY_VERIFIED ──► ROOM_SCANNED ──► EXAM_ACTIVE
│
├── autosave every 5s
├── proctoring active
├── timer synced
│
▼
SUBMITTED
│
├── auto (timeout)
├── manual (candidate)
└── forced (violation)
│
▼
EVALUATED

```

### 8.2 Session State Management

| Component | Responsibility |
|---|---|
| **Redis** | Live session state, timer, current question, autosave buffer |
| **PostgreSQL** | Persistent responses, session events, final submission |
| **WebSocket** | Real-time sync between client and server |
| **Scheduler** | Auto-submit on timeout, session cleanup |

### 8.3 Autosave Strategy

```

Client                      Server                     Redis              Postgres
│                           │                          │                   │
│─── answer (every 5s) ────►│                          │                   │
│                           │─── write buffer ────────►│                   │
│                           │                          │                   │
│                           │─── flush (every 30s) ────┼──────────────────►│
│                           │                          │                   │
│◄── ack ───────────────────│                          │                   │

```

**Rules:**

- Autosave every 5 seconds to Redis
- Flush to PostgreSQL every 30 seconds
- Immediate flush on question navigation
- Guaranteed persistence on submit
- Replay from Redis on reconnect

### 8.4 Timer Architecture

- Server-authoritative timer (not client)
- Stored in Redis with TTL
- Synced to client every 10 seconds
- Cannot be paused by candidate
- Auto-submit triggered by server on expiry
- Grace period: 30 seconds for network recovery

### 8.5 Reconnection & Recovery

```

Client disconnects
│
▼
Session state preserved in Redis (TTL: 2 hours)
│
▼
Client reconnects with session token
│
▼
Server validates token + session window
│
▼
State restored from Redis + Postgres
│
▼
Exam resumes with correct timer

```

---

## 9. Proctoring Architecture

### 9.1 Proctoring Stack

```

┌─────────────────────────────────────────────────────────────┐
│                    Client (Browser)                         │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Webcam │ Mic │ Screen Share │ Browser Events         │  │
│  └───────────────────────────┬───────────────────────────┘  │
│                              │                              │
│  ┌───────────────────────────▼───────────────────────────┐  │
│  │           TensorFlow.js / MediaPipe (on-device)       │  │
│  │  • Face detection                                     │  │
│  │  • Eye tracking                                       │  │
│  │  • Object detection (phone, book)                     │  │
│  │  • Multiple face detection                            │  │
│  └───────────────────────────┬───────────────────────────┘  │
│                              │                              │
│                              │ events (not video)           │
│                              ▼                              │
│                     WebSocket to server                     │
└─────────────────────────────┬───────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────────┐
│                    Server (Spring Boot)                     │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │            Proctoring Event Processor                 │  │
│  │  • Validate event                                    │  │
│  │  • Score severity                                    │  │
│  │  • Apply violation rules                             │  │
│  │  • Trigger actions (warning, siren, auto-submit)     │  │
│  └───────────────────────────┬───────────────────────────┘  │
│                              │                              │
│                              ▼                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              Violation Engine                         │  │
│  │  • Thresholds per violation type                      │  │
│  │  • Escalation rules                                   │  │
│  │  • Auto-submit trigger                                │  │
│  └───────────────────────────┬───────────────────────────┘  │
│                              │                              │
│                              ▼                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │           Evidence Storage                            │  │
│  │  • Snapshots (S3)                                     │  │
│  │  • Video clips (S3)                                   │  │
│  │  • Event log (Postgres)                               │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────────┐
│                    Proctor Dashboard                        │
│                                                             │
│  • Live video feed                                          │
│  • Violation alerts                                         │
│  • Chat with candidate                                      │
│  • Warn / Terminate buttons                                 │
│  • Incident log                                             │
└─────────────────────────────────────────────────────────────┘

```

### 9.2 Violation Types & Severity

| Violation | Severity | Auto Action |
|---|---|---|
| Tab switch | Low | Warning |
| Fullscreen exit | Low | Warning |
| Copy/paste | Medium | Warning |
| Right-click | Low | Block |
| DevTools open | High | Warning + proctor alert |
| No face detected (5s+) | Medium | Warning |
| Multiple faces | High | Warning + proctor alert |
| Phone detected | Critical | Instant termination |
| Second screen | Critical | Instant termination |
| Audio anomaly | Medium | Warning |
| Face mismatch | Critical | Proctor review |
| Network anomaly | Low | Log only |

### 9.3 Escalation Rules

```

Violation count per session:
1st  → Warning modal
2nd  → Siren warning + proctor alert
3rd  → Auto-submit + session terminated
Critical violation at any time → Instant termination

```

### 9.4 Proctoring Data Flow

```

Client event → WebSocket → Proctoring Service → Rule Engine
│
┌─────────────────┼─────────────────┐
▼                 ▼                 ▼
Store event      Trigger action    Alert proctor
(Postgres)       (Redis Pub/Sub)   (WebSocket)

```

### 9.5 Evidence Storage

| Type | Storage | Retention |
|---|---|---|
| Snapshots | S3 | 1 year |
| Video clips | S3 | 1 year |
| Event log | Postgres (partitioned) | 5 years |
| Violation summary | Postgres | 5 years |

---

## 10. Command Centre Architecture

### 10.1 Purpose

Central monitoring of all live exams — modeled on NTA's Okhla command centre and TCS iON's monitoring dashboard.

### 10.2 Architecture

```

┌─────────────────────────────────────────────────────────────┐
│                    Command Centre UI                        │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              National View                            │  │
│  │  Total active │ Submitted │ Flagged │ Terminated      │  │
│  └───────────────────────────┬───────────────────────────┘  │
│                              │                              │
│  ┌───────────────────────────▼───────────────────────────┐  │
│  │              State Drill-Down                         │  │
│  │  State-wise stats │ Alerts │ Proctor load             │  │
│  └───────────────────────────┬───────────────────────────┘  │
│                              │                              │
│  ┌───────────────────────────▼───────────────────────────┐  │
│  │              District Drill-Down                      │  │
│  └───────────────────────────┬───────────────────────────┘  │
│                              │                              │
│  ┌───────────────────────────▼───────────────────────────┐  │
│  │              Exam Drill-Down                          │  │
│  └───────────────────────────┬───────────────────────────┘  │
│                              │                              │
│  ┌───────────────────────────▼───────────────────────────┐  │
│  │              Candidate Drill-Down                     │  │
│  │  Live video │ Events │ Violations │ Actions           │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
▲
│ WebSocket (real-time)
│
┌─────────────────────────────┴───────────────────────────────┐
│                    Backend Services                         │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │           Redis Pub/Sub (live events)                 │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │           Aggregation Service (stats)                 │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │           Alert Service (priority queue)              │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘

```

### 10.3 Real-Time Data Pipeline

```

Exam Session Events
│
▼
Redis Pub/Sub  ──►  Aggregation Service  ──►  Command Centre UI
│                    │
│                    ▼
│              Redis Cache (stats)
│
▼
PostgreSQL (persistent)

```

### 10.4 Command Centre Metrics

| Metric | Update Frequency |
|---|---|
| Active sessions | Real-time |
| Submitted sessions | Real-time |
| Flagged sessions | Real-time |
| Terminated sessions | Real-time |
| Violation rate | Every 30s |
| Proctor load | Every 30s |
| System health | Every 10s |

---

## 11. Question Paper Security Architecture

### 11.1 Paper Lifecycle

```

CREATED (plaintext in memory)
│
▼
ENCRYPTED (AES-256)
│
▼
STORED (S3, encrypted at rest)
│
▼
RELEASED (at exam start, just-in-time)
│
▼
DECRYPTED (in memory, per session)
│
▼
DISCARDED (after exam window)

```

### 11.2 Encryption Architecture

```

┌─────────────────────────────────────────────────────────────┐
│                    Key Management                           │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │           Master Key (KMS / HSM)                      │  │
│  └───────────────────────────┬───────────────────────────┘  │
│                              │                              │
│                              ▼                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │           Paper Key (per paper)                       │  │
│  │           Encrypted with Master Key                   │  │
│  └───────────────────────────┬───────────────────────────┘  │
│                              │                              │
│                              ▼                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │           Paper Payload                               │  │
│  │           Encrypted with Paper Key                    │  │
│  │           AES-256-GCM                                 │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘

```

### 11.3 Release Rules

| Rule | Description |
|---|---|
| Just-in-time | Papers released only at exam start |
| Per-session | Each session gets a unique decryption context |
| Time-bound | Access expires at exam end |
| Audit | Every access logged |
| No pre-fetch | Papers not loaded before exam window |

### 11.4 Paper Distribution Flow

```

Exam Start Time
│
▼
Scheduler triggers paper release
│
▼
Paper Service loads encrypted paper from S3
│
▼
Decrypts with Paper Key (from KMS)
│
▼
In-memory only (never written to disk)
│
▼
Serves to session via authenticated API
│
▼
Discarded after session ends

```

---

## 12. Payment Architecture

### 12.1 Razorpay Integration

```

┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│   Student    │         │   Backend    │         │  Razorpay    │
│   Browser    │         │              │         │              │
└──────┬───────┘         └──────┬───────┘         └──────┬───────┘
│                        │                        │
│─── apply exam ────────►│                        │
│                        │─── create order ──────►│
│                        │◄── order_id ───────────│
│◄── order_id + key ─────│                        │
│                        │                        │
│─── checkout ───────────┼───────────────────────►│
│                        │                        │
│◄── payment_id ─────────┼────────────────────────│
│                        │                        │
│─── verify signature ──►│                        │
│                        │─── verify ────────────►│
│                        │◄── valid ──────────────│
│                        │                        │
│                        │─── mark APPLIED        │
│◄── success ────────────│                        │
│                        │                        │
│                        │◄── webhook ────────────│
│                        │─── reconcile ──────────│

```

### 12.2 Idempotency

| Rule | Description |
|---|---|
| Order ID | Unique per application attempt |
| Payment ID | Unique per Razorpay transaction |
| Idempotency Key | Prevents duplicate order creation |
| Webhook Retry | Handled idempotently |
| Reconciliation | Daily job reconciles orders vs payments |

### 12.3 Payment States

```

CREATED ──► PENDING ──► SUCCESS ──► APPLICATION_APPLIED
│
└──► FAILED ──► RETRY_ALLOWED
│
└──► ABANDONED ──► EXPIRED

```

### 12.4 Refund Logic

| Rule | Description |
|---|---|
| Non-refundable | Exam fee is non-refundable once paid |
| Exception | Refund only if exam is cancelled by NSEE |
| Processing | Via Razorpay refund API |
| Timeline | 5–7 business days |

---

## 13. Notification Architecture

### 13.1 Event-Driven Notifications

```

Domain Event  ──►  Event Bus  ──►  Notification Service
│
├── Template Engine
│
├── Channel Router
│
├── Email (SMTP)
├── SMS (Provider API)
├── WhatsApp (Provider API)
└── Push (FCM)
│
▼
Delivery Log

```

### 13.2 Notification Events

| Event | Channels |
|---|---|
| Registration success | Email, SMS |
| OTP | Email, SMS |
| Application submitted | Email, SMS |
| Payment success | Email, SMS |
| Admit card released | Email, SMS, WhatsApp |
| Exam reminder (24h, 1h) | Email, SMS, WhatsApp |
| Result declared | Email, SMS, WhatsApp |
| Notice published | Email (subscribers) |
| Query response | Email |

### 13.3 Retry & DLQ

```

Notification Job
│
▼
Attempt 1 ──► fail ──► Retry (5s)
│
▼
Attempt 2 ──► fail ──► Retry (30s)
│
▼
Attempt 3 ──► fail ──► Retry (5m)
│
▼
Attempt 4 ──► fail ──► Dead Letter Queue
│
▼
Manual review / alert

```

---

## 14. Authentication & Authorization

### 14.1 Authentication Flow

```

Login Request
│
▼
Validate credentials (BCrypt)
│
▼
Check MFA (for admins)
│
▼
Issue Access Token (JWT, 15m)
│
▼
Issue Refresh Token (JWT, 7d, stored in Redis)
│
▼
Return tokens to client

```

### 14.2 JWT Structure

```json
{
  "sub": "user-uuid",
  "role": "STUDENT",
  "scope": ["exam:read", "application:write"],
  "iat": 1234567890,
  "exp": 1234568790,
  "jti": "token-uuid"
}
```

14.3 Authorization Layers

Layer Mechanism
Route-level @PreAuthorize("hasRole('ADMIN')")
Method-level @PreAuthorize("@authz.canPublishExam(#examId)")
Data-level Row-level security (student can only access own data)
Feature-level Feature flags per role

14.4 Role Hierarchy

```
SUPER_ADMIN
    │
    ├── EXAM_SETTER
    ├── STATE_ADMIN
    │       │
    │       └── DISTRICT_ADMIN
    │
    ├── PROCTOR_MANAGER
    │       │
    │       └── PROCTOR
    │
    ├── SUPPORT
    │
    └── STUDENT
```

14.5 Session Management

Token Storage TTL
Access Token Client (memory) 15 min
Refresh Token Redis + client 7 days
Session Token (exam) Redis 2 hours
OTP Redis 5 min

---

15. Integration Architecture

15.1 External Integrations

Integration Purpose Protocol
Razorpay Payments REST + Webhook
SMTP Provider Email SMTP
SMS Provider SMS REST
WhatsApp Provider WhatsApp REST
FCM Push notifications REST
DigiLocker Scorecard delivery REST
S3 / MinIO Object storage S3 API
KMS / HSM Key management REST

15.2 Integration Pattern

```
Feature  ──►  Output Port  ──►  Adapter  ──►  External Service
                                    │
                                    ├── Circuit Breaker
                                    ├── Retry
                                    ├── Timeout
                                    └── Fallback
```

15.3 Resilience

Pattern Implementation
Circuit Breaker Resilience4j
Retry Exponential backoff
Timeout Per-integration config
Bulkhead Thread pool isolation
Fallback Graceful degradation

---

16. Deployment Architecture

16.1 Production Topology

```
                    ┌─────────────────┐
                    │   Cloudflare    │
                    │   CDN + WAF     │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │   VPS (Primary) │
                    │                 │
                    │  ┌───────────┐  │
                    │  │  Nginx    │  │
                    │  └─────┬─────┘  │
                    │        │        │
                    │  ┌─────▼─────┐  │
                    │  │  API      │  │
                    │  │ (Spring)  │  │
                    │  └─────┬─────┘  │
                    │        │        │
                    │  ┌─────▼─────┐  │
                    │  │PostgreSQL │  │
                    │  └───────────┘  │
                    │                 │
                    │  ┌───────────┐  │
                    │  │  Redis    │  │
                    │  └───────────┘  │
                    └─────────────────┘
                             │
                    ┌────────▼────────┐
                    │   VPS (Backup)  │
                    │   (Standby)     │
                    └─────────────────┘
```

16.2 Docker Compose Services

```
services:
  api          → Spring Boot (port 8080)
  frontend     → React SPA (port 5173)
  postgres     → PostgreSQL 16 (port 5432)
  redis        → Redis 7 (port 6379)
  nginx        → Reverse proxy (port 80, 443)
  minio        → S3-compatible storage (port 9000)
```

16.3 Environments

Environment Purpose Deploy Trigger
local Developer machine Manual
dev Shared dev Push to develop
staging QA / UAT Push to develop (after CI)
production Live Push to main (after CI)

16.4 CI/CD Pipeline

```
Push to main
      │
      ▼
┌─────────────────┐
│  CI Workflow    │
│  • Lint         │
│  • Compile      │
│  • Unit tests   │
│  • Integration  │
│  • E2E tests    │
│  • Build        │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  CD Workflow    │
│  • Docker build │
│  • Push to GHCR │
│  • SSH to VPS   │
│  • docker pull  │
│  • docker up    │
│  • Migrate      │
│  • Health check │
└─────────────────┘
```

---

17. Security Architecture

17.1 Defense in Depth

```
Layer 1: Network        → Cloudflare WAF, DDoS, IP reputation
Layer 2: Transport      → TLS 1.3, HSTS
Layer 3: Application    → Spring Security, JWT, RBAC
Layer 4: Data           → AES-256 encryption, KMS
Layer 5: Database       → Row-level security, encrypted columns
Layer 6: Audit          → Immutable append-only logs
Layer 7: Monitoring     → SIEM, alerts, anomaly detection
```

17.2 Threat Model

Threat Mitigation
Credential stuffing Rate limiting, CAPTCHA, MFA
Token theft Short-lived tokens, refresh rotation
SQL injection Parameterized queries (JPA)
XSS Content Security Policy, input sanitization
CSRF Stateless JWT, SameSite cookies
DDoS Cloudflare, rate limiting
Paper leak AES-256, JIT release, audit
Impersonation Face match, ID verification, MFA
Cheating AI proctoring, browser lockdown
Data breach Encryption, RBAC, audit, DR

17.3 Compliance

Regulation Compliance
DPDP Act 2023 Consent, data minimization, retention, grievance officer
CERT-In Log retention, incident reporting
GIGW Accessibility, government standards
WCAG 2.1 AA Accessibility
ISO 27001 Infrastructure (via cloud provider)
Data Residency India only

17.4 Secrets Management

· All secrets via environment variables
· No secrets in code or repo
· Rotated quarterly
· KMS for encryption keys
· Audit for secret access

---

18. Observability & Monitoring

18.1 Three Pillars

Pillar Tool Purpose
Logs ELK / Loki Structured application logs
Metrics Prometheus + Grafana System and business metrics
Traces OpenTelemetry + Jaeger Distributed tracing

18.2 Key Metrics

Category Metric
API Request rate, latency (p50, p95, p99), error rate
Exam Active sessions, submissions, violations
Payment Success rate, failure rate, reconciliation
Proctoring Violation rate, auto-submit rate
System CPU, memory, disk, network
Business Registrations, applications, results

18.3 Alerts

Alert Severity Action
API error rate > 5% Critical Page on-call
Exam session failure Critical Page on-call
Payment failure > 10% Critical Page on-call
DB replication lag > 30s High Notify
Redis memory > 80% High Notify
Disk usage > 80% Medium Notify
Violation spike Medium Notify proctor manager

18.4 Dashboards

· System Health — CPU, memory, disk, network
· API Performance — latency, errors, throughput
· Exam Operations — active sessions, violations, submissions
· Payment — success rate, failures, reconciliation
· Business — registrations, applications, results

---

19. Scalability & Performance

19.1 Scaling Strategy

Component Strategy
API Horizontal (stateless)
WebSocket Horizontal with Redis pub/sub
PostgreSQL Vertical + read replicas
Redis Cluster mode
S3 Native scaling
Nginx Load balancing

19.2 Performance Targets

Metric Target
API p95 latency < 300ms
API p99 latency < 1s
Exam autosave < 100ms
WebSocket latency < 200ms
Page load < 2s
Concurrent exam sessions 100,000+
Concurrent API requests 10,000+

19.3 Load Testing

Scenario Tool Frequency
API load k6 / Gatling Weekly
Exam concurrency k6 Before each exam
Payment flow Custom Monthly
Proctoring events Custom Before each exam

19.4 Caching Strategy

Data Cache TTL
Session state Redis 2 hours
OTP Redis 5 min
Exam catalog Redis 5 min
Question bank Redis 15 min
Static content CDN 24 hours
User profile Redis 10 min

---

20. Disaster Recovery

20.1 RPO / RTO

Metric Target
RPO (Recovery Point Objective) 15 minutes
RTO (Recovery Time Objective) 1 hour

20.2 Backup Strategy

Data Frequency Retention Location
PostgreSQL Continuous WAL + daily full 30 days S3 (cross-region)
Redis Daily snapshot 7 days S3
S3 objects Versioned 1 year Cross-region
Config Git Forever GitHub

20.3 DR Drills

· Quarterly DR drill
· Simulate primary region failure
· Measure RTO / RPO
· Document learnings

20.4 Failover

```
Primary VPS fails
        │
        ▼
Health check detects failure (30s)
        │
        ▼
DNS failover to backup VPS (Cloudflare)
        │
        ▼
Backup VPS starts services
        │
        ▼
Restore from latest backup
        │
        ▼
Service restored (target: 1 hour)
```

---

21. Technology Decisions

Decision Choice Rationale
Language Java 21 Mature, performant, government-grade
Framework Spring Boot 3.3 Industry standard, security, ecosystem
Architecture Hexagonal Testable, maintainable, swappable
Database PostgreSQL 16 ACID, JSONB, partitioning, reliable
Cache Redis 7 Fast, pub/sub, streams, session
Frontend React 18 + Vite Modern, fast, component-based
Styling Tailwind CSS Utility-first, consistent
State React Query + Zustand Server + client state separation
Payments Razorpay India-first, reliable, RBI-compliant
Containers Docker Portable, reproducible
Orchestration Docker Compose Simple, sufficient for VPS
CI/CD GitHub Actions Integrated, free for public repos
Reverse Proxy Nginx Proven, performant
Storage S3 / MinIO Scalable, cost-effective
Monitoring Prometheus + Grafana Standard, extensible
Tracing OpenTelemetry Vendor-neutral

Explicitly NOT used (initial):

· Kafka (Redis Streams sufficient)
· Kubernetes (Docker Compose sufficient)
· Microservices (modular monolith first)

---

22. Architecture Decision Records

ADR-001: Hexagonal Architecture

Decision: Use hexagonal architecture per feature.

Context: Government-grade platform, long-term maintainability, need to swap adapters.

Consequences:

· ✅ Domain logic testable in isolation
· ✅ Adapters swappable
· ✅ Clear boundaries
· ❌ More boilerplate
· ❌ Steeper learning curve

ADR-002: Modular Monolith over Microservices

Decision: Start with modular monolith.

Context: Small team, need fast iteration, no proven scale requirement yet.

Consequences:

· ✅ Simpler deployment
· ✅ Easier debugging
· ✅ Faster development
· ❌ Requires discipline to keep modules separate
· ❌ Single point of failure

ADR-003: Redis Streams over Kafka

Decision: Use Redis Streams for internal events.

Context: Need async event bus, already using Redis, small scale initially.

Consequences:

· ✅ Simpler infrastructure
· ✅ Lower operational cost
· ✅ Sufficient for current scale
· ❌ Migration to Kafka may be needed later

ADR-004: Remote Proctoring Only

Decision: All exams are remote-proctored. No physical centres.

Context: Cost, reach, government directive to modernize.

Consequences:

· ✅ Wider reach
· ✅ Lower infrastructure cost
· ✅ No centre management
· ❌ Heavier reliance on AI proctoring
· ❌ Internet dependency for candidates

ADR-005: NTA-Grade Result Logic

Decision: Adopt NTA normalization and tie-breaking rules.

Context: Multi-session exams, fairness, government-standard.

Consequences:

· ✅ Fair across sessions
· ✅ Government-grade standard
· ✅ Recognized by stakeholders
· ❌ Complex computation
· ❌ Requires per-session stats

ADR-006: Feature Flags for Staged Rollout

Decision: Every feature behind a flag.

Context: Public-first launch, incremental delivery.

Consequences:

· ✅ No broken features visible
· ✅ Safe rollout
· ✅ Easy rollback
· ❌ Extra config management

ADR-007: JIT Paper Release

Decision: Papers released just-in-time, encrypted at rest.

Context: Paper leak prevention, government-grade security.

Consequences:

· ✅ Prevents leaks
· ✅ Auditable
· ❌ Requires reliable scheduler
· ❌ Complex key management

---

23. Constraints & Assumptions

23.1 Constraints

Constraint Impact
Single VPS initially Vertical scaling limits
No Kubernetes Manual orchestration
No Kafka Redis Streams used
India data residency Cloud provider must support
Government compliance DPDP, CERT-In, GIGW
Budget Open-source preferred

23.2 Assumptions

Assumption Risk if Wrong
Students have stable internet Exam disruption
Students have webcam + mic Cannot proctor
Razorpay stays available Payment failure
Cloud provider stays in India Compliance breach
Load stays under 100k concurrent Scaling required

23.3 Open Questions

1. Should papers be released per-session or per-exam?
2. How long should violation evidence be retained?
3. Should proctors be in-house or outsourced?
4. What is the SLA for support queries?
5. How many sessions per exam (for normalization)?

---

Appendix A — Feature-to-Layer Mapping

Feature Domain Application Infra Presentation
auth User, Role, OTP Login, Register JPA, Redis, JWT REST
student Student, USID Register, USID JPA, Redis REST
profile Profile, Docs Save, Complete JPA, S3 REST
exam Exam, Approval CRUD, Publish JPA REST
questionbank Question, Option CRUD, Bulk JPA, S3 REST
paper Paper, Key Generate, Release JPA, S3, KMS REST
application Application Apply, Submit JPA REST
payment Payment Order, Webhook Razorpay REST
admitcard AdmitCard Generate JPA, S3 REST
examsession Session, Response Start, Submit JPA, Redis, WS REST + WS
proctoring Violation Detect, Act JPA, Redis, WS REST + WS
result Result, Rank Evaluate, Rank JPA REST
notice Notice CRUD, Target JPA REST
query Query Submit, Reply JPA REST
notification Notification Send SMTP, SMS Async
admin User, Role Create, Approve JPA REST
audit AuditLog Log, Query JPA REST
commandcentre — Aggregate Redis, WS REST + WS
analytics — Aggregate JPA, Redis REST

---

Appendix B — Data Flow Diagrams

B.1 Student Registration Flow

```
Student → Register Form → API → Validate → Save Student → Generate USID
                                                    │
                                                    ▼
                                            Send OTP (Email + SMS)
                                                    │
                                                    ▼
                                            Student verifies OTP
                                                    │
                                                    ▼
                                            Account activated
```

B.2 Exam Application Flow

```
Student → Select Exam → Check Eligibility → Draft Application
                                                    │
                                                    ▼
                                            Select District
                                                    │
                                                    ▼
                                            Submit Application
                                                    │
                                                    ▼
                                            Payment (Razorpay)
                                                    │
                                                    ▼
                                            Application APPLIED
                                                    │
                                                    ▼
                                            Admit Card queued
```

B.3 Exam Session Flow

```
Candidate → Login → Pre-Exam Check → Identity Verify → Room Scan
                                                            │
                                                            ▼
                                                    Paper Released (JIT)
                                                            │
                                                            ▼
                                                    Exam Starts
                                                            │
                                            ┌───────────────┼───────────────┐
                                            │               │               │
                                        Autosave       Proctoring       Timer
                                            │               │               │
                                            └───────────────┼───────────────┘
                                                            │
                                                            ▼
                                                    Submit / Auto-Submit
                                                            │
                                                            ▼
                                                    Evaluate → Result
```

B.4 Result Processing Flow

```
Session Responses → Raw Score → Percentile (per session)
                                        │
                                        ▼
                                Normalized Score
                                        │
                                        ▼
                                Tie-Breaking
                                        │
                                        ▼
                                Rank Assignment
                                        │
                                        ▼
                                Answer Key Challenge (if any)
                                        │
                                        ▼
                                Final Result Published
```

---

Appendix C — Sequence Diagrams

C.1 Login Sequence

```
Client → API: POST /auth/login
API → AuthService: validate credentials
AuthService → UserRepository: findByEmail
UserRepository → DB: SELECT
DB → UserRepository: user
UserRepository → AuthService: user
AuthService → PasswordEncoder: matches
AuthService → JwtService: generate tokens
JwtService → AuthService: access + refresh
AuthService → Redis: store refresh token
AuthService → API: tokens
API → Client: 200 OK + tokens
```

C.2 Exam Autosave Sequence

```
Client → WebSocket: answer event
WS → ProctoringService: process
ProctoringService → Redis: buffer answer
Redis → ProctoringService: ack
ProctoringService → WS: ack
WS → Client: ack
(Every 30s)
Scheduler → ProctoringService: flush
ProctoringService → Redis: read buffer
ProctoringService → DB: bulk insert
DB → ProctoringService: ok
ProctoringService → Redis: clear buffer
```

C.3 Payment Sequence

```
Client → API: POST /payments/order
API → PaymentService: createOrder
PaymentService → Razorpay: createOrder
Razorpay → PaymentService: orderId
PaymentService → DB: save payment (CREATED)
PaymentService → API: orderId + key
API → Client: orderId + key
Client → Razorpay: checkout
Razorpay → Client: paymentId + signature
Client → API: POST /payments/verify
API → PaymentService: verify
PaymentService → Razorpay: verifySignature
Razorpay → PaymentService: valid
PaymentService → DB: update (SUCCESS)
PaymentService → ApplicationService: mark APPLIED
ApplicationService → DB: update
PaymentService → API: success
API → Client: 200 OK
```

---

Appendix D — Glossary

Term Meaning
CBT Computer-Based Test
USID Universal Scholarship ID
NTA National Testing Agency
JIT Just-In-Time (paper release)
RPO Recovery Point Objective
RTO Recovery Time Objective
DPDP Digital Personal Data Protection Act
GIGW Guidelines for Indian Government Websites
WCAG Web Content Accessibility Guidelines
UFM Unfair Means
PwD Persons with Disabilities
KMS Key Management Service
HSM Hardware Security Module
DLQ Dead Letter Queue

---

Document Control

Version Date Author Changes
1.0 Oct 2026 Architecture Team Initial release with NTA-grade business logic

---

END OF ARCHITECTURE DOCUMENT

This document is the authoritative architecture reference for NSEE India.
All development must conform to the patterns, rules, and decisions described here.
Any deviation requires an Architecture Decision Record (ADR) and approval.

```

---

Save the above as **`ARCHITECTURE.md`** in the root of your repository, alongside `README.md`.

This architecture document includes:

- **Updated business logic** aligned with NTA / TCS iON standards (normalization, tie-breaking, answer key challenge, application lifecycle)
- **Remote-proctored architecture** — no physical centres
- **Hexagonal architecture** with feature-first modules
- **Exam session, proctoring, and command centre architecture**
- **Question paper security** with JIT release and KMS
- **Payment, notification, and integration architecture**
- **Deployment, security, observability, and DR**
- **Architecture Decision Records** for every major choice
- **Appendices** with data flows, sequence diagrams, and glossary

This is production-ready and can be handed directly to your development agent.
