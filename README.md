
# NSEE INDIA — MASTER PROJECT BRIEF
**Handoff Document for Development Team**

| Field | Value |
|---|---|
| Version | 1.0 |
| Date | October 2026 |
| Prepared for | Development Agent / Team |
| Project | National Scholarship Entrance Exam Portal |
| Domain | nseeindia.com |
| Status | Ready for Development |

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Technology Stack](#2-technology-stack-final)
3. [Architecture](#3-architecture)
4. [Roles & Permissions](#4-roles--permissions)
5. [Public-First Launch](#5-public-first-launch-critical)
6. [Sprint Plan](#6-sprint-plan-incremental)
7. [Feature Specifications](#7-feature-specifications)
8. [Frontend Structure](#8-frontend-structure)
9. [Design System](#9-design-system-government-style-modern)
10. [Database](#10-database-key-tables)
11. [API Conventions](#11-api-conventions)
12. [Security](#12-security)
13. [CI/CD](#13-cicd)
14. [Branch & Release Strategy](#14-branch--release-strategy)
15. [Definition of Done](#15-definition-of-done-per-feature)
16. [Rules to Enforce](#16-rules-to-enforce)
17. [Immediate First Steps](#17-immediate-first-steps)
18. [Environment Variables](#18-environment-variables)
19. [Success Metrics](#19-success-metrics)
20. [Final Notes for Agent](#20-final-notes-for-agent)

---

## 1. PROJECT OVERVIEW

**NSEE India** is a government-grade online examination portal for conducting scholarship entrance exams across all Indian states, district-wise, in both **online** and **offline** modes.

### Exam Levels Supported
Class 5, 6, 7, 8, 9, 10, 11, 12, BCA, B.Tech, MCA, M.Tech

### Exam Fee
₹99 per exam (via Razorpay)

### Student Flow
```

Register → USID → Progressive Profile → Apply → Pay → Admit Card → Exam → Result

```

### Launch Strategy
Public-first. Launch registration + profile + notices in **7 weeks**. Unlock exam features one by one.

---

## 2. TECHNOLOGY STACK (FINAL)

| Layer | Technology |
|---|---|
| Backend | Java 21 + Spring Boot 3.3 |
| Build | Maven |
| Architecture | Hexagonal (Ports & Adapters) |
| Persistence | Spring Data JPA + Hibernate + Flyway |
| Database | PostgreSQL 16 |
| Cache/Session | Redis 7 |
| Auth | Spring Security + JWT + OTP |
| Payments | Razorpay Java SDK |
| PDF | OpenPDF |
| WebSocket | Spring WebSocket (STOMP) |
| Frontend | React 18 + TypeScript + Vite + Tailwind CSS + React Query |
| Container | Docker + Docker Compose |
| Reverse Proxy | Nginx |
| CI/CD | GitHub Actions |
| Hosting | VPS (Docker Compose) |
| Storage | S3-compatible (MinIO / AWS S3) |

> **No other frameworks. No Kafka. No Kubernetes (initially).**

---

## 3. ARCHITECTURE

### 3.1 Hexagonal (Ports & Adapters)

Each feature has 4 layers:

```

domain/         → Pure business logic (no Spring, no JPA)
application/    → Use cases, orchestration
infrastructure/ → Adapters (JPA, Redis, external APIs)
presentation/   → REST controllers

```

### 3.2 Feature-First Folder Structure

```

src/main/java/com/nsee/
├── core/                          # Shared kernel
│   ├── domain/
│   ├── application/
│   ├── infrastructure/
│   └── shared/
├── features/
│   ├── auth/
│   ├── student/
│   ├── profile/
│   ├── state/
│   ├── district/
│   ├── centre/
│   ├── exam/
│   ├── questionbank/
│   ├── paper/
│   ├── application/
│   ├── payment/
│   ├── admitcard/
│   ├── examsession/
│   ├── proctoring/
│   ├── result/
│   ├── notice/
│   ├── query/
│   ├── notification/
│   ├── admin/
│   └── audit/
└── bootstrap/

```

### 3.3 Feature Template (Repeat for Each)

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
└── ExamModuleConfig.java   # Spring @Configuration

```

**Rules:**
- `domain/` imports nothing outside JDK.
- `application/` imports only `domain/`.
- `infrastructure/` implements `domain/port/out`.
- `presentation/` calls `domain/port/in` only.

---

## 4. ROLES & PERMISSIONS

| Role | Scope | Key Permissions |
|---|---|---|
| `SUPER_ADMIN` | Global | Create admins, approve exams/notices, global config |
| `EXAM_SETTER` | State/National | Create exams, question bank, generate papers, publish |
| `STATE_ADMIN` | State | Manage districts, state reports |
| `DISTRICT_CONTROLLER` | District | Add centres, assign centre controllers |
| `CENTRE_CONTROLLER` | Centre | Conduct offline exam, verify students, unlock PCs |
| `INVIGILATOR` | Centre/Remote | Monitor live exam, warn, terminate |
| `STUDENT` | Self | Register, profile, apply, pay, exam, result |
| `SUPPORT` | Read-only | Queries, grievances, audit |

> **Maker–Checker Rule:** Exam Setter creates → Super Admin approves → then publish.

---

## 5. PUBLIC-FIRST LAUNCH (CRITICAL)

### 5.1 Public Launch = Week 7

Ship these features first to make the site live:

1. Core Foundation (Sprint 0)
2. Auth (Sprint 1)
3. Student Registration + USID (Sprint 2)
4. Progressive Profile (Sprint 3)

**Then LAUNCH** → public sees a working government portal.

### 5.2 Public Site Must Include

- Home page (government style)
- Notice Board (public)
- Exam catalog (read-only, static JSON initially)
- Public Query Form (no login)
- Registration + Login
- Progressive Profile
- "Notify Me" on each exam (instead of "Apply")
- About, RTI, Privacy, Terms, Grievance Officer, Contact, Sitemap

### 5.3 Locked Features

Show "Coming Soon" badge. Feature flags control visibility.

```yaml
features:
  auth: true
  student: true
  profile: true
  notice: true
  query: true
  exam: false          # read-only catalog via static JSON
  application: false
  payment: false
  examsession: false
  proctoring: false
  result: false
```

---

6. SPRINT PLAN (INCREMENTAL)

Rule: One feature at a time. Build → Test → Freeze → Next.

Sprint Workflow (per feature)

```
1.  Write OpenAPI contract
2.  Write Flyway migration
3.  Build domain (pure Java)
4.  Write domain unit tests           ✅ must pass
5.  Build application use cases
6.  Write application tests           ✅ must pass
7.  Build infra adapters (JPA/Redis)
8.  Write integration tests           ✅ must pass (Testcontainers)
9.  Build presentation (controller)
10. Write E2E tests                   ✅ must pass (REST Assured)
11. Build React UI
12. Write frontend tests              ✅ must pass (Vitest + RTL)
13. Run full CI
14. Deploy to staging
15. Manual QA + sign-off
16. Freeze feature, tag release
17. Next feature
```

Sprint Sequence

# Sprint Duration Cumulative
0 Core Foundation 1 wk 1 wk
1 Auth 2 wk 3 wk
2 Student + USID 2 wk 5 wk
3 Profile (progressive) 2 wk 7 wk
— PUBLIC LAUNCH — Week 7
4 State / District / Centre 1.5 wk 8.5 wk
5 Notice Board 1 wk 9.5 wk
6 Query (public) 1 wk 10.5 wk
7 Exam (CRUD + Publish) 2 wk 12.5 wk
8 Question Bank 2 wk 14.5 wk
9 Paper Generation 2 wk 16.5 wk
10 Application 1.5 wk 18 wk
11 Payment (Razorpay) 2 wk 20 wk
12 Admit Card 1.5 wk 21.5 wk
13 Exam Session 3 wk 24.5 wk
14 Proctoring 2 wk 26.5 wk
15 Result 1.5 wk 28 wk
16 Notification 1.5 wk 29.5 wk
17 Admin (Super Admin) 2 wk 31.5 wk
18 Audit 1 wk 32.5 wk
19 Offline Centre Module 2 wk 34.5 wk
20 Analytics / Reports 1.5 wk 36 wk

Public launch: Week 7
Full platform: ~9 months

---

7. FEATURE SPECIFICATIONS

7.1 Sprint 0 — Core Foundation

· Spring Boot scaffold
· Hexagonal folder structure
· PostgreSQL + Flyway + Redis setup
· Global exception handler
· OpenAPI (springdoc) config
· JWT infrastructure
· ci.yml + deploy.yml
· Docker Compose (api, postgres, redis, nginx)
· Health endpoint /actuator/health
· Freeze when: CI green, staging reachable

7.2 Sprint 1 — Auth

Endpoints:

```
POST /auth/register
POST /auth/verify-otp
POST /auth/login
POST /auth/refresh
POST /auth/logout
```

Tables: users, roles, user_roles, otp, refresh_tokens, login_audit

Tests: password hashing, JWT, OTP expiry, RBAC denial, wrong password

Freeze when: all roles log in, RBAC enforced.

7.3 Sprint 2 — Student + USID

USID Format: NSEE/YYYY/STATE/DIST/000123

Endpoints:

```
POST /students/register
GET  /students/usid/{id}
```

Tables: students, usid_sequence

Tests: USID uniqueness (100 parallel), format, resume partial registration

Freeze when: USID unique, traceable, audited.

7.4 Sprint 3 — Profile (Progressive)

Sections: Personal, Education, Parents, Bank, Documents, Address

Endpoints:

```
GET   /profiles/me
PATCH /profiles/me
GET   /profiles/me/completeness
```

Tables: student_profiles, profile_documents, profile_audit

Tests: completeness calculator, partial save → resume → complete, oversized uploads

Freeze when: 100% completeness required to apply.

7.5 Sprint 4 — Geography

· Tables: states, districts, centres, centre_staff
· Admin-only CRUD
· Hierarchy: State → District → Centre

7.6 Sprint 5 — Notice Board

Types: general, exam, admit_card, result, alert, circular

Targeting: state, district, level, exam, audience

Endpoints:

```
GET  /notices              (public)
GET  /notices/home         (top 5)
POST /notices              (admin)
POST /notices/{id}/submit
POST /notices/{id}/approve
POST /notices/{id}/publish
POST /notices/{id}/archive
```

Tables: notices, notice_audit

Auto-notices: admit card release, result declaration, exam schedule.

7.7 Sprint 6 — Public Query

· POST /queries (no login)
· CAPTCHA
· Ticket ID + status tracking
· Support dashboard

7.8 Sprint 7 — Exam

Levels: Class 5–12, BCA, B.Tech, MCA, M.Tech

State machine: DRAFT → PENDING → APPROVED → PUBLISHED → ARCHIVED

Endpoints:

```
POST /exams
PUT  /exams/{id}
POST /exams/{id}/submit      (to approval)
POST /exams/{id}/approve     (Super Admin only)
POST /exams/{id}/publish
```

Tests: Setter cannot approve own exam

7.9 Sprint 8 — Question Bank

· MCQ CRUD, bulk CSV upload
· Tags: level, subject, topic, difficulty, language
· Version control

7.10 Sprint 9 — Paper Generation

· Sets A/B/C/D per district/centre
· Randomize questions + options
· AES-256 encryption
· Release only at exam start

7.11 Sprint 10 — Application

· Eligibility engine (per level)
· Draft save, resume
· Centre preference
· Moves to PAYMENT_PENDING

7.12 Sprint 11 — Payment

· Razorpay order server-side
· Webhook verification
· ₹99 per exam
· Idempotency, retry, reconciliation
· Application → APPLIED

7.13 Sprint 12 — Admit Card

· PDF with QR code
· Online + offline variants
· Release date gating

7.14 Sprint 13 — Exam Session

· Start exam, timer, navigation
· Autosave every 5s
· Submit + auto-submit on timeout
· WebSocket /ws/session/{id}
· Load test: 1000 concurrent

7.15 Sprint 14 — Proctoring

· Tab switch, fullscreen exit, copy/paste block
· Face detection, multiple faces
· Siren warning + modal
· Auto-submit after 3 violations
· Proctor dashboard /ws/proctor/{examId}

7.16 Sprint 15 — Result

· Auto-evaluation MCQ
· Normalization across sets
· Ranks: district, state, national
· Publish result

7.17 Sprint 16 — Notification

· Email, SMS, WhatsApp, push
· Templates
· Event-driven
· Retry + DLQ

7.18 Sprint 17 — Admin

· Super Admin dashboard
· Create/approve State Admins, Exam Setters, District Controllers
· Approval queue

7.19 Sprint 18 — Audit

· Immutable append-only log
· @Auditable annotation
· Every write action logged

7.20 Sprint 19 — Offline Centre Module

· District Controller adds centres
· Centre Controller sees registered students
· Verify, allot seat/PC, unlock exam
· Post-exam sync

7.21 Sprint 20 — Analytics

· Dashboards: state, district, centre
· Exam stats, pass %, violations
· Export CSV/PDF
· Materialized views

---

8. FRONTEND STRUCTURE

```
nsee-frontend/
├── src/
│   ├── main.tsx
│   ├── App.tsx
│   ├── routes/
│   ├── features/          # mirrors backend
│   │   ├── auth/
│   │   ├── student/
│   │   ├── profile/
│   │   ├── exam/
│   │   ├── application/
│   │   ├── payment/
│   │   ├── admitcard/
│   │   ├── exam-session/
│   │   ├── notice/
│   │   ├── query/
│   │   └── admin/
│   ├── shared/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── api/
│   │   ├── utils/
│   │   └── types/
│   ├── layouts/
│   │   ├── PublicLayout.tsx
│   │   ├── StudentLayout.tsx
│   │   └── AdminLayout.tsx
│   └── styles/
├── Dockerfile
├── nginx.conf
└── .github/workflows/
```

---

9. DESIGN SYSTEM (GOVERNMENT STYLE MODERN)

9.1 Colors

Token Hex Usage
Navy Blue #0B3D91 Header, primary buttons, links
Saffron #FF9933 Accent, highlights
India Green #138808 Success, verified
White #FFFFFF Background
Text #1A1A1A Body text
Muted #4A4A4A Secondary text
Border #E0E0E0 Borders
Background #F7F9FC Page background
Error #D32F2F Errors
Warning #F5A623 Warnings

9.2 Typography

· Headings: Noto Serif / Merriweather
· Body: Noto Sans / Inter
· Hindi/Regional: Noto Sans Devanagari

9.3 Layout

```
[ Top Strip: Govt of India | Ministry of Education | A- A A+ | हिंदी ]
[ Header: Emblem | NSEE India | Login | Register ]
[ Nav: Home | About | Exams | Notices | Centres | Results | Contact ]
[ Urgent Notice Banner (if any) ]
[ Hero ]
[ Notice Board widget ]
[ Exam List: Ongoing | Upcoming | Past ]
[ How it Works (4 steps) ]
[ Query Form ]
[ Footer: RTI | Privacy | Grievance | Contact ]
```

9.4 Accessibility (GIGW / WCAG 2.1 AA)

· Contrast ≥ 4.5:1
· Skip to main content
· Keyboard navigable
· Alt text
· Font size adjuster
· Multi-language
· Screen reader friendly

---

10. DATABASE (KEY TABLES)

```sql
users, roles, user_roles, otp, refresh_tokens, login_audit
students, usid_sequence, student_profiles, profile_documents, profile_audit
states, districts, centres, centre_staff, centre_seats, centre_attendance
exams, exam_approvals, exam_audit
questions, question_options, question_tags
exam_papers, paper_questions, paper_keys
exam_applications, application_drafts
payments, payment_webhooks
admit_cards
exam_sessions, session_responses, session_events
proctoring_events, violations
results, ranks
notices, notice_audit
queries, query_responses
notifications, notification_templates, notification_logs
audit_logs (append-only)
```

Migrations: Flyway, versioned V1__init.sql, V2__auth.sql, etc.
Rule: Every migration reversible. No ddl-auto=update in prod.

---

11. API CONVENTIONS

· Base path: /api/v1
· JSON only
· JWT in Authorization: Bearer <token>
· HTTP status codes: 200, 201, 204, 400, 401, 403, 404, 409, 422, 500
· Error format:

```json
{
  "timestamp": "2026-10-04T10:00:00Z",
  "status": 400,
  "error": "VALIDATION_ERROR",
  "message": "Email already exists",
  "path": "/api/v1/auth/register"
}
```

· Pagination: ?page=0&size=20&sort=createdAt,desc
· OpenAPI spec at /swagger-ui.html

---

12. SECURITY

· Spring Security + JWT (access 15m, refresh 7d)
· BCrypt password hashing
· @PreAuthorize on every controller
· Rate limiting via Redis + Bucket4j
· CORS restricted to frontend origin
· CSRF disabled (stateless JWT)
· Input validation (@Valid)
· AES-256 encryption for papers
· Immutable audit log
· HTTPS via Nginx + Let's Encrypt
· Secrets via env vars only
· DPDP Act 2023, CERT-In, GIGW compliance
· Data residency: India only

---

13. CI/CD

13.1 .github/workflows/ci.yml

Runs on push/PR to main, develop:

```
1. mvn spotless:check
2. mvn compile
3. mvn test                → fail if coverage < 85%
4. mvn verify              → Testcontainers Postgres + Redis
5. mvn failsafe:integration-test (E2E)
6. npm run lint (frontend)
7. npm run test (Vitest)
8. npm run build
```

Merge blocked if any step fails.

13.2 .github/workflows/deploy.yml

Runs on push to main:

```
1. Build Docker image (multi-stage)
2. Push to ghcr.io
3. SSH into VPS
4. docker compose pull
5. docker compose up -d
6. Flyway migrations auto-run on startup
7. Health check /actuator/health
```

13.3 GitHub Secrets

```
VPS_HOST
VPS_USER
VPS_SSH_KEY
VPS_PORT
GHCR_TOKEN
```

13.4 Docker Compose on VPS

Services: api, postgres, redis, nginx

---

14. BRANCH & RELEASE STRATEGY

```
main         ← production
  ↑
develop      ← staging (integration)
  ↑
feature/auth
feature/student
feature/profile
feature/exam
...
```

· One branch per feature
· PR → develop → CI → review → merge
· QA sign-off → develop → main → deploy

Release Tags

```
v0.1.0-core
v0.2.0-auth
v0.3.0-student
v0.4.0-profile
v0.5.0-geography
v0.6.0-notice
v0.7.0-query
v0.8.0-exam
...
v1.0.0-production
```

---

15. DEFINITION OF DONE (PER FEATURE)

☐ OpenAPI spec merged
☐ Flyway migration applied in dev/staging
☐ Domain unit tests ≥ 90% coverage
☐ Integration tests pass (Testcontainers)
☐ E2E API tests pass
☐ React UI functional with API
☐ Frontend tests pass
☐ CI green on main
☐ Deployed to staging
☐ QA sign-off
☐ Audit log entries verified
☐ Docs updated
☐ Feature flag enabled
☐ Release tagged

---

16. RULES TO ENFORCE

1. No skipping tests — feature not done without green CI
2. No merging features — one feature per branch, per release tag
3. Freeze before next — no Sprint N+1 until Sprint N is signed off
4. Staging must work — every feature deployed to staging before freeze
5. Rollback ready — every migration reversible; every deploy can roll back
6. Feature flags — new features off by default; enable after QA
7. Documentation — OpenAPI + README updated per feature
8. Audit — every write action logged from Sprint 1 onward
9. Public site never breaks — after Week 7, public pages must stay stable
10. No broken features visible — always use feature flags

---

17. IMMEDIATE FIRST STEPS

```bash
# 1. Sprint 0 scaffold
spring init --dependencies=web,data-jpa,security,data-redis,flyway,validation,websocket \
  --build=maven --java-version=21 --groupId=com.nsee --artifactId=nsee-backend

# 2. Create hexagonal folders
mkdir -p src/main/java/com/nsee/{core,features,bootstrap}

# 3. Add ci.yml + deploy.yml
# 4. Add docker-compose.yml (api, postgres, redis, nginx)
# 5. Add /actuator/health
# 6. Push → CI green → deploy staging

# 7. Sprint 1 (Auth)
# 8. Sprint 2 (Student + USID)
# 9. Sprint 3 (Profile)
# 10. PUBLIC LAUNCH — Week 7
```

---

18. ENVIRONMENT VARIABLES

```env
NODE_ENV=production
PORT=8080

DATABASE_URL=jdbc:postgresql://postgres:5432/nsee
DB_USER=nsee
DB_PASSWORD=xxx

REDIS_URL=redis://redis:6379

JWT_SECRET=xxx
JWT_ACCESS_EXPIRY=900
JWT_REFRESH_EXPIRY=604800

RAZORPAY_KEY_ID=xxx
RAZORPAY_KEY_SECRET=xxx
RAZORPAY_WEBHOOK_SECRET=xxx

S3_ENDPOINT=xxx
S3_BUCKET=nsee-docs
S3_ACCESS_KEY=xxx
S3_SECRET_KEY=xxx

SMTP_HOST=xxx
SMTP_USER=xxx
SMTP_PASS=xxx

SMS_API_KEY=xxx
WHATSAPP_API_KEY=xxx

SENTRY_DSN=xxx
```

---

19. SUCCESS METRICS

Metric Target
Public launch Week 7
Registration flow < 2 min
Profile completion 80%+
API p95 latency < 300ms
Exam session uptime 99.9%
Concurrent exam users 10,000+
Test coverage (backend) ≥ 85%
Accessibility WCAG 2.1 AA
Load time < 2s

---

20. FINAL NOTES FOR AGENT

1. Follow hexagon strictly — domain must be framework-free.
2. One feature at a time — no parallel feature branches merging.
3. Test before move — no sprint starts until previous is frozen.
4. Public launch is priority — Week 7 is non-negotiable.
5. Government look matters — trust is the product.
6. Every action audited — from Sprint 1.
7. Feature flags everywhere — no broken features visible.
8. Document as you build — OpenAPI + README per feature.
9. Rollback always ready — migrations reversible.
10. Ask before assuming — clarify with stakeholder on ambiguity.

---

END OF MASTER BRIEF

This document is the single source of truth.
Any change must be versioned and approved.

Next action: Start Sprint 0 immediately.
Public launch target: Week 7.
Full platform target: Month 9.

```

---

Save the above as **`NSEE_MASTER_BRIEF.md`** in the root of your repository. Hand this file directly to your agent — it contains everything needed to start development.
