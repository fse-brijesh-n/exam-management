Here is the corrected README.md file, updated with the proper business logic and workflows modeled after the standards of NTA and TCS iON.

```markdown
# NSEE India

**National Scholarship Entrance Exam Portal**

A government-grade online examination platform for conducting scholarship entrance exams across all Indian states — fully online, remote-proctored, and accessible from anywhere.

![Status](https://img.shields.io/badge/status-in%20development-orange)
![Version](https://img.shields.io/badge/version-1.0-blue)
![Java](https://img.shields.io/badge/Java-21-orange)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.3-green)
![React](https://img.shields.io/badge/React-18-blue)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-blue)
![Redis](https://img.shields.io/badge/Redis-7-red)

| Field | Value |
|---|---|
| Project | NSEE India |
| Domain | nseeindia.com |
| Mode | Online CBT (Remote Proctored) |
| Exam Levels | Class 5–12, BCA, B.Tech, MCA, M.Tech |
| Exam Fee | ₹99 per exam |
| Status | In Development |
| Version | 1.0 |

---

## Table of Contents

1. [Overview](#1-overview)
2. [Exam Levels](#2-exam-levels)
3. [Exam Mode](#3-exam-mode)
4. [Business Logic & Workflows](#4-business-logic--workflows)
5. [Feature Highlights](#5-feature-highlights)
6. [Technology Stack](#6-technology-stack)
7. [Architecture](#7-architecture)
8. [Repository Structure](#8-repository-structure)
9. [Roles & Permissions](#9-roles--permissions)
10. [Student Journey](#10-student-journey)
11. [Proctoring & Anti-Cheating](#11-proctoring--anti-cheating)
12. [Command Centre](#12-command-centre)
13. [Security & Compliance](#13-security--compliance)
14. [Design System](#14-design-system)
15. [Getting Started](#15-getting-started)
16. [Environment Variables](#16-environment-variables)
17. [Database](#17-database)
18. [API Conventions](#18-api-conventions)
19. [Testing](#19-testing)
20. [CI/CD](#20-cicd)
21. [Deployment](#21-deployment)
22. [Sprint Plan](#22-sprint-plan)
23. [Definition of Done](#23-definition-of-done)
24. [Branch & Release Strategy](#24-branch--release-strategy)
25. [Rules](#25-rules)
26. [Roadmap](#26-roadmap)
27. [Contact & License](#27-contact--license)

---

## 1. Overview

**NSEE India** is a production-grade, government-style online examination portal for conducting scholarship entrance exams for students across India.

- **Fully online** — Computer-Based Test (CBT), remote proctored
- **From anywhere** — no physical exam centres
- **AI + human proctoring** — for exam integrity
- **District and state-wise** — exam discovery and ranking
- **Government-grade** — security, compliance, accessibility

---

## 2. Exam Levels

| Level | Eligibility |
|---|---|
| Class 5 | Studying in 5th or passed 4th |
| Class 6 | Studying in 6th or passed 5th |
| Class 7 | Studying in 7th or passed 6th |
| Class 8 | Studying in 8th or passed 7th |
| Class 9 | Studying in 9th or passed 8th |
| Class 10 | Studying in 10th or passed 9th |
| Class 11 | Studying in 11th or passed 10th (stream-based) |
| Class 12 | Studying in 12th or passed 11th (stream-based) |
| BCA | 12th pass (any stream) |
| B.Tech | 12th with PCM |
| MCA | BCA / B.Sc. (IT / Maths) |
| M.Tech | B.Tech / B.E. |

---

## 3. Exam Mode

**Single mode: Online CBT — Remote Proctored**

- Student appears from home or any location
- No physical centres
- AI proctoring + human proctoring throughout
- Auto-submit on repeated violations
- Instant evaluation for MCQ

---

## 4. Business Logic & Workflows

This section defines the operational rules and process flows, modeled on the standards of NTA and TCS iON.

### 4.1 Application Process Logic

| Step | Rule |
|---|---|
| **Registration** | One candidate = one application per exam. Duplicate applications lead to cancellation. Candidate must use their own email and mobile number (cannot be changed later). |
| **Eligibility** | Candidates must ensure eligibility before filling the form. Applications of ineligible candidates are rejected at a later stage with no claim entertained. |
| **Application Correction** | A one-time correction window is opened (typically 48–72 hours) after the registration deadline. Only specific fields (e.g., name, photo, course selection) can be edited. |
| **Fee Payment** | Fee is non-refundable once paid. Application is incomplete until fee payment is successful. |
| **Application Withdrawal** | Once submitted successfully, applications cannot be withdrawn. |

### 4.2 Exam Centre Allocation Logic

| Rule | Description |
|---|---|
| **Automated Allocation** | The system performs rule-based allocation of examination centres based on: candidate preferences (if applicable), centre capacity, geographic proximity (district/state), and special category requirements. |
| **Aadhaar-Based (Future)** | Following NTA's 2026 policy, centres will be allocated strictly based on the address on the candidate's Aadhaar card to reduce impersonation and ensure fairness. |
| **No Overbooking** | The system prevents overbooking of centres and ensures optimal utilisation. Centres may be changed by the authority for logistic or administrative reasons. |
| **City Selection** | (Current NTA model) Candidates choose a city after fee payment on a first-come-first-serve basis. Once allotted, the city cannot be changed. |

### 4.3 Admit Card Release Logic

| Rule | Description |
|---|---|
| **Release Timing** | Admit cards are released typically 4–14 days before the exam date. |
| **Download Only** | Candidates must download and print the admit card from the official website. No hard copy is dispatched. |
| **Credentials** | Login requires Application Number and Date of Birth (or Password). |
| **Contents** | Admit card displays exam centre, date, shift, time, and candidate details. |
| **Re-issue** | Admit cards may be re-issued (e.g., for re-exam or corrections). |

### 4.4 Exam Day Logic (Remote Proctored)

| Step | Rule |
|---|---|
| **ID Verification** | Candidate must show Admit Card and ID card in front of the camera before exam starts. |
| **System Check** | Webcam, microphone, and internet speed tests are mandatory before the exam. |
| **Room Scan** | A 360-degree scan of the room is required. |
| **Identity Re-verification** | Continuous AI-based face matching with the registered photo occurs throughout the exam. |

### 4.5 Result Processing Logic

#### 4.5.1 Normalization (for Multi-Session Exams)

Since exams are conducted in multiple shifts with different difficulty levels, raw scores are not used directly. A **Normalization Procedure based on Percentile Score** is adopted.

| Concept | Rule |
|---|---|
| **Percentile Score** | Indicates the percentage of candidates who scored equal to or below a particular raw score in that session. |
| **Formula** | `100 X (Number of candidates in the session with raw score ≤ candidate's raw score) / Total number of candidates in that session` |
| **Precision** | Percentiles are calculated up to **7 decimal places** to reduce ties. |
| **Topper's Score** | The highest raw score in each session gets a percentile of 100. |
| **Final Score** | The Normalized (Percentile) Score is used for merit lists, **not** the raw marks. |

#### 4.5.2 Tie-Breaking Rules

If two or more candidates have the same Total NTA Score, the following hierarchy is used to determine ranks (example: JEE Main 2026):

| Priority | Criterion |
|---|---|
| 1 | Higher NTA Score in **Mathematics** |
| 2 | Higher NTA Score in **Physics** |
| 3 | Higher NTA Score in **Chemistry** |
| 4 | Lower incorrect-to-correct answer ratio (overall) |
| 5 | Lower incorrect-to-correct ratio in Mathematics |
| 6 | Lower incorrect-to-correct ratio in Physics |
| 7 | Lower incorrect-to-correct ratio in Chemistry |
| Final | If tie still persists, **same rank** is awarded. |

> **Note:** Age and application number have been removed as tie-breakers from 2025 onwards. Ranking is purely merit-driven.

### 4.6 Answer Key Challenge Logic

| Rule | Description |
|---|---|
| **Provisional Key** | A provisional answer key is released before results. |
| **Challenge Window** | Candidates can challenge the provisional key within a specified window. |
| **Fee** | A fee (e.g., ₹200) is charged per question challenged to discourage frivolous challenges. |
| **Refund** | The fee is refunded if the challenge is found correct by subject experts. |
| **Final Key** | If any challenge is accepted, the revised answer key applies to **all candidates** who appeared for the exam. |
| **Review** | Challenges are reviewed by subject experts before final results are declared. |

### 4.7 Scorecard & Result Download Logic

| Rule | Description |
|---|---|
| **Availability** | Scorecards are hosted on the official website after result declaration. |
| **Login** | Download requires Application Number, Date of Birth, and/or Password. |
| **DigiLocker** | Scorecards may also be made available via DigiLocker for secure access. |
| **Print** | Candidates must download and print the scorecard for future admission processes. |

---

## 5. Feature Highlights

### 5.1 Student
- Registration with OTP verification and single application rule
- Universal Scholarship ID (USID)
- Progressive profile with autosave and resume
- Exam discovery (state / district / eligibility)
- Apply with draft save and resume
- ₹99 payment via Razorpay (non-refundable)
- Admit card with QR + exam link (download only)
- Pre-exam system check and mock test
- Remote-proctored online CBT exam
- Instant result with normalization, ranking, and tie-breaking rules
- Answer key objection window with fee and refund logic
- Email / SMS / WhatsApp notifications
- Full exam history

### 5.2 Admin
- Super Admin with approval authority
- Exam Admin Setter for content and papers
- State and District admins
- Proctor Manager and Proctors
- Question bank with bulk upload
- Multiple paper sets (A/B/C/D)
- Notice Board with targeting
- Query and grievance management
- Reports and analytics
- Immutable audit log
- Command Centre for live monitoring

### 5.3 Platform
- 256-bit encrypted question papers
- AI proctoring (17+ checks)
- Command Centre for live monitoring
- Feature flags for staged rollout
- Government-style UI (GIGW / WCAG 2.1 AA)
- Multi-language support (English + Hindi + regional)

---

## 6. Technology Stack

| Layer | Technology |
|---|---|
| Backend | Java 21, Spring Boot 3.3 |
| Build | Maven |
| Architecture | Hexagonal (Ports & Adapters) |
| Persistence | Spring Data JPA, Hibernate, Flyway |
| Database | PostgreSQL 16 |
| Cache / Session | Redis 7 |
| Auth | Spring Security, JWT, OTP |
| Payments | Razorpay Java SDK |
| PDF | OpenPDF |
| WebSocket | Spring WebSocket (STOMP) |
| Frontend | React 18, TypeScript, Vite, Tailwind CSS, React Query |
| Container | Docker, Docker Compose |
| Reverse Proxy | Nginx |
| CI/CD | GitHub Actions |
| Hosting | VPS (Docker Compose) |
| Storage | S3-compatible (MinIO / AWS S3) |

> No Kafka. No Kubernetes (initially). Keep it simple, scale with evidence.

---

## 7. Architecture

### 7.1 Hexagonal (Ports & Adapters)

Every feature is a **vertical slice** with 4 layers:

```

domain/         → Pure business logic (no Spring, no JPA)
application/    → Use cases, orchestration
infrastructure/ → Adapters (JPA, Redis, external APIs)
presentation/   → REST controllers

```

### 7.2 Layer Rules

- `domain/` imports nothing outside the JDK
- `application/` imports only `domain/`
- `infrastructure/` implements `domain/port/out`
- `presentation/` calls `domain/port/in` only

### 7.3 Feature Module Template

```

features/exam/
├── domain/
│   ├── model/          # Aggregate roots, entities, VOs
│   ├── event/          # Domain events
│   ├── exception/      # Domain exceptions
│   └── port/
│       ├── in/         # Use case interfaces
│       └── out/        # Repository / external interfaces
├── application/
│   ├── service/        # Implements use cases
│   ├── dto/            # Commands, responses, mappers
│   └── event/          # Event publishers
├── infrastructure/
│   ├── persistence/    # JPA entities, repositories, adapters
│   ├── messaging/      # Redis / queue adapters
│   └── external/       # Third-party adapters
├── presentation/
│   ├── ExamController.java
│   ├── request/
│   └── response/
└── ExamModuleConfig.java   # Spring @Configuration

```

---

## 8. Repository Structure

### 8.1 Backend

```

nsee-backend/
├── pom.xml
├── Dockerfile
├── docker-compose.yml
├── .github/workflows/
│   ├── ci.yml
│   └── deploy.yml
├── src/main/java/com/nsee/
│   ├── NseeApplication.java
│   ├── core/
│   │   ├── domain/
│   │   ├── application/
│   │   ├── infrastructure/
│   │   └── shared/
│   ├── features/
│   │   ├── auth/
│   │   ├── student/
│   │   ├── profile/
│   │   ├── state/
│   │   ├── district/
│   │   ├── exam/
│   │   ├── questionbank/
│   │   ├── paper/
│   │   ├── application/
│   │   ├── payment/
│   │   ├── admitcard/
│   │   ├── examsession/
│   │   ├── proctoring/
│   │   ├── result/
│   │   ├── notice/
│   │   ├── query/
│   │   ├── notification/
│   │   ├── admin/
│   │   └── audit/
│   └── bootstrap/
├── src/main/resources/
│   ├── application.yml
│   ├── application-dev.yml
│   ├── application-prod.yml
│   └── db/migration/
└── src/test/java/com/nsee/
├── unit/
├── integration/
└── e2e/

```

### 8.2 Frontend

```

nsee-frontend/
├── package.json
├── vite.config.ts
├── tailwind.config.ts
├── Dockerfile
├── nginx.conf
├── .github/workflows/
│   ├── ci.yml
│   └── deploy.yml
└── src/
├── main.tsx
├── App.tsx
├── routes/
├── features/
│   ├── auth/
│   ├── student/
│   ├── profile/
│   ├── exam/
│   ├── application/
│   ├── payment/
│   ├── admitcard/
│   ├── exam-session/
│   ├── notice/
│   ├── query/
│   └── admin/
├── shared/
│   ├── components/
│   ├── hooks/
│   ├── api/
│   ├── utils/
│   └── types/
├── layouts/
│   ├── PublicLayout.tsx
│   ├── StudentLayout.tsx
│   └── AdminLayout.tsx
└── styles/

```

---

## 9. Roles & Permissions

| Role | Scope | Key Permissions |
|---|---|---|
| `SUPER_ADMIN` | Global | Create admins, approve exams/notices, global config, Command Centre |
| `EXAM_SETTER` | State / National | Create exams, question bank, generate papers, submit for approval |
| `STATE_ADMIN` | State | Manage districts, state reports, monitor sessions |
| `DISTRICT_ADMIN` | District | District reports, grievances |
| `PROCTOR_MANAGER` | Global | Assign proctors, manage shifts, audit decisions |
| `PROCTOR` | Session | Live monitoring, warn, terminate, log incidents |
| `STUDENT` | Self | Register, profile, apply, pay, exam, result |
| `SUPPORT` | Read-only | Queries, grievances, audit |

**Maker–Checker Rule:** Exam Setter creates → Super Admin approves → then publish.

---

## 10. Student Journey

```

REGISTER (name, email, phone, qualification, state, district, password)
↓
OTP VERIFY (email + SMS)
↓
USID GENERATED (NSEE/YYYY/STATE/DIST/000123)
↓
PROGRESSIVE PROFILE (personal, education, parents, bank, address, docs)
↓
EXAM DISCOVERY (state / district / eligibility-filtered)
↓
APPLY (draft save, resume, notify me)
↓
PAY ₹99 (Razorpay - non-refundable)
↓
ADMIT CARD (PDF + QR + exam link - download only)
↓
PRE-EXAM SYSTEM CHECK (webcam, mic, network, mock test)
↓
ONLINE CBT EXAM (remote proctored)
↓
RESULT (normalized score, rank with tie-breaking, answer key objection window)

```

---

## 11. Proctoring & Anti-Cheating

### 11.1 AI Proctoring (17+ Checks)

- Face detection (no face, multiple faces)
- Face match with registered photo
- Eye movement tracking
- Object detection (phone, book, second screen)
- Audio monitoring (voice detection)
- Tab switch detection
- Fullscreen exit detection
- Copy / paste blocking
- Right-click disable
- DevTools detection
- Background noise detection
- Screen recording detection
- Virtual machine detection
- Multiple monitor detection
- 360-degree room scan at start
- ID verification on camera
- Continuous identity re-verification

### 11.2 Human Proctoring

- Live proctor dashboard
- Random snapshot verification
- Live video monitoring
- Chat with candidate
- Warning issuance
- Exam termination
- Incident logging

### 11.3 Violation Handling

| Violation | Action |
|---|---|
| 1st | Warning modal |
| 2nd | Siren warning + proctor alert |
| 3rd | Auto-submit |
| Critical (face lost, phone detected) | Instant termination |

All violations logged immutably with evidence (screenshots, video clips).

---

## 12. Command Centre

- Central dashboard for all live exams
- Real-time candidate monitoring
- Drill-down: national → state → district → exam → candidate
- Live proctor assignment
- Alert queue for violations
- Candidate flags (high risk, suspicious)
- Live statistics (active, submitted, flagged, terminated)
- 24×7 operators during exam window

---

## 13. Security & Compliance

- 256-bit AES encryption for question papers
- Just-in-time paper release at exam start
- Role-based access control (`@PreAuthorize` on every controller)
- Multi-factor authentication for admins
- BCrypt password hashing
- JWT access (15 min) + refresh (7 days)
- Rate limiting via Redis + Bucket4j
- CORS restricted to frontend origin
- CSRF disabled (stateless JWT)
- Input validation with `@Valid`
- Immutable append-only audit log
- HTTPS via Nginx + Let's Encrypt
- Secrets via env vars only
- DPDP Act 2023 compliant
- CERT-In compliant
- GIGW and WCAG 2.1 AA accessible
- Data residency: India only
- ISO 27001 certified infrastructure
- Disaster recovery across two seismic zones
- NDA for all exam personnel

---

## 14. Design System

### 14.1 Colors

| Token | Hex | Usage |
|---|---|---|
| Navy Blue | `#0B3D91` | Header, primary buttons, links |
| Saffron | `#FF9933` | Accent, highlights |
| India Green | `#138808` | Success, verified |
| White | `#FFFFFF` | Background |
| Text | `#1A1A1A` | Body text |
| Muted | `#4A4A4A` | Secondary text |
| Border | `#E0E0E0` | Borders |
| Background | `#F7F9FC` | Page background |
| Error | `#D32F2F` | Errors |
| Warning | `#F5A623` | Warnings |

### 14.2 Typography

- **Headings:** Noto Serif / Merriweather
- **Body:** Noto Sans / Inter
- **Hindi / Regional:** Noto Sans Devanagari

### 14.3 Layout

```

[ Top Strip: Govt of India | Ministry of Education | A- A A+ | हिंदी ]
[ Header: Emblem | NSEE India | Login | Register ]
[ Nav: Home | About | Exams | Notices | Results | Contact ]
[ Urgent Notice Banner (if any) ]
[ Hero ]
[ Notice Board Widget ]
[ Exam List: Ongoing | Upcoming | Past ]
[ How it Works (4 steps) ]
[ Query Form ]
[ Footer: RTI | Privacy | Grievance | Contact ]

```

### 14.4 Accessibility (GIGW / WCAG 2.1 AA)

- Contrast ratio ≥ 4.5:1
- Skip to main content
- Keyboard navigable
- Alt text on all images
- Font size adjuster (A- A A+)
- Multi-language
- Screen reader friendly
- No color-only meaning

---

## 15. Getting Started

### 15.1 Prerequisites

- Java 21
- Maven 3.9+
- Node.js 20+
- Docker & Docker Compose
- PostgreSQL 16 (via Docker)
- Redis 7 (via Docker)

### 15.2 Clone

```bash
git clone https://github.com/yourorg/nsee-backend.git
git clone https://github.com/yourorg/nsee-frontend.git
```

15.3 Run Backend Locally

```bash
cd nsee-backend

# Start dependencies (Postgres + Redis)
docker compose up -d postgres redis

# Copy env
cp .env.example .env

# Build
mvn clean install

# Run
mvn spring-boot:run
```

Backend: http://localhost:8080
Swagger UI: http://localhost:8080/swagger-ui.html
Health: http://localhost:8080/actuator/health

15.4 Run Frontend Locally

```bash
cd nsee-frontend
npm install
npm run dev
```

Frontend: http://localhost:5173

15.5 Run Full Stack

```bash
docker compose up -d
```

Services:

· api → http://localhost:8080
· frontend → http://localhost:5173
· postgres → localhost:5432
· redis → localhost:6379
· nginx → http://localhost

---

16. Environment Variables

```env
# App
NODE_ENV=production
PORT=8080

# Database
DATABASE_URL=jdbc:postgresql://postgres:5432/nsee
DB_USER=nsee
DB_PASSWORD=change_me

# Redis
REDIS_URL=redis://redis:6379

# JWT
JWT_SECRET=change_me_strong_secret
JWT_ACCESS_EXPIRY=900
JWT_REFRESH_EXPIRY=604800

# Razorpay
RAZORPAY_KEY_ID=xxx
RAZORPAY_KEY_SECRET=xxx
RAZORPAY_WEBHOOK_SECRET=xxx

# Storage
S3_ENDPOINT=xxx
S3_BUCKET=nsee-docs
S3_ACCESS_KEY=xxx
S3_SECRET_KEY=xxx

# Email
SMTP_HOST=xxx
SMTP_USER=xxx
SMTP_PASS=xxx

# SMS / WhatsApp
SMS_API_KEY=xxx
WHATSAPP_API_KEY=xxx

# Monitoring
SENTRY_DSN=xxx

# Feature flags
FEATURE_EXAM=false
FEATURE_APPLICATION=false
FEATURE_PAYMENT=false
FEATURE_EXAMSESSION=false
FEATURE_PROCTORING=false
FEATURE_RESULT=false
```

---

17. Database

17.1 Core Tables

```sql
users, roles, user_roles, otp, refresh_tokens, login_audit
students, usid_sequence, student_profiles, profile_documents, profile_audit
states, districts
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

17.2 Migration Rules

· Flyway versioned: V1__init.sql, V2__auth.sql, ...
· Every migration reversible
· No ddl-auto=update in production
· Migrations run automatically on startup

---

18. API Conventions

· Base path: /api/v1
· JSON only
· JWT in Authorization: Bearer <token>
· Status codes: 200, 201, 204, 400, 401, 403, 404, 409, 422, 500

Error Format

```json
{
  "timestamp": "2026-10-04T10:00:00Z",
  "status": 400,
  "error": "VALIDATION_ERROR",
  "message": "Email already exists",
  "path": "/api/v1/auth/register"
}
```

Pagination

```
?page=0&size=20&sort=createdAt,desc
```

OpenAPI

Spec at /swagger-ui.html

---

19. Testing

19.1 Backend

Type Tool Location
Unit JUnit 5 + Mockito src/test/unit
Integration Testcontainers (Postgres + Redis) src/test/integration
E2E REST Assured src/test/e2e

Coverage gate: ≥ 85% (unit + integration).

19.2 Frontend

Type Tool
Component Vitest + React Testing Library
E2E Playwright

19.3 Run Tests

```bash
# Backend
mvn test                # unit
mvn verify              # integration + E2E

# Frontend
npm run test
npm run test:e2e
```

---

20. CI/CD

20.1 .github/workflows/ci.yml

Runs on push/PR to main, develop:

```
1. mvn spotless:check
2. mvn compile
3. mvn test                → fail if coverage < 85%
4. mvn verify              → Testcontainers Postgres + Redis
5. mvn failsafe:integration-test (E2E)
6. npm run lint
7. npm run test
8. npm run build
```

Merge is blocked if any step fails.

20.2 .github/workflows/deploy.yml

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

20.3 GitHub Secrets

```
VPS_HOST
VPS_USER
VPS_SSH_KEY
VPS_PORT
GHCR_TOKEN
```

---

21. Deployment

21.1 Docker Compose on VPS

Services: api, postgres, redis, nginx

21.2 First-time VPS Setup

```bash
ssh deploy@vps

mkdir -p /opt/nsee && cd /opt/nsee

# Copy docker-compose.yml + nginx.conf + .env
# Login to GHCR
echo $GHCR_TOKEN | docker login ghcr.io -u <user> --password-stdin

# Start
docker compose up -d
```

21.3 Rolling Update

Handled automatically by deploy.yml on every push to main.

21.4 Rollback

```bash
docker compose pull api:<previous-tag>
docker compose up -d api
```

---

22. Sprint Plan

# Sprint Duration Cumulative
0 Core Foundation 1 wk 1 wk
1 Auth 2 wk 3 wk
2 Student + USID 2 wk 5 wk
3 Profile 2 wk 7 wk
— PUBLIC LAUNCH — Week 7
4 State / District 1.5 wk 8.5 wk
5 Notice Board 1 wk 9.5 wk
6 Query 1 wk 10.5 wk
7 Exam (CRUD + Publish) 2 wk 12.5 wk
8 Question Bank 2 wk 14.5 wk
9 Paper Generation 2 wk 16.5 wk
10 Application 1.5 wk 18 wk
11 Payment 2 wk 20 wk
12 Admit Card 1.5 wk 21.5 wk
13 Exam Session 3 wk 24.5 wk
14 Proctoring 2 wk 26.5 wk
15 Result 1.5 wk 28 wk
16 Notification 1.5 wk 29.5 wk
17 Admin 2 wk 31.5 wk
18 Audit 1 wk 32.5 wk
19 Command Centre 2 wk 34.5 wk
20 Analytics 1.5 wk 36 wk

Public launch: Week 7
Full platform: ~9 months

---

23. Definition of Done

Per feature:

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

24. Branch & Release Strategy

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

25. Rules

1. No skipping tests — feature not done without green CI
2. No merging features — one feature per branch, per release tag
3. Freeze before next — no Sprint N+1 until Sprint N is signed off
4. Staging must work — every feature deployed to staging before freeze
5. Rollback ready — every migration reversible; every deploy can roll back
6. Feature flags — new features off by default; enable after QA
7. Documentation — OpenAPI + README updated per feature
8. Audit — every write action logged from Sprint 1 onward
9. Public site never breaks — after Week 7, public pages stay stable
10. No broken features visible — always use feature flags

---

26. Roadmap

Phase Timeline What Goes Live
Phase 1 Week 7 Public site, Registration, USID, Profile, Notices
Phase 2 Week 20 Apply + Payment
Phase 3 Week 24 Admit Card, Mock Test, Live Exam (remote proctored)
Phase 4 Week 28 Result, Answer Key Objection
Phase 5 Month 9 Command Centre, advanced analytics

---

27. Contact & License

NSEE India
🌐 https://nseeindia.com
📧 contact@nseeindia.com
📧 grievance@nseeindia.com

Copyright © 2026 NSEE India.
Content owned by Ministry of Education.
All rights reserved.

For licensing, partnership, or compliance queries:
📧 contact@nseeindia.com

---

Built for India. Built for students. Built to scale.

```

The README has been updated with a dedicated "Business Logic & Workflows" section. The key additions include:

*   **NTA-Modeled Rules**: Added explicit rules for application (single form, non-refundable fee, correction window), centre allocation (automated, Aadhaar-based future policy), admit card (download-only, release timing), and exam day (ID verification).
*   **Result Processing**: Included NTA's percentile-based normalization formula, detailed tie-breaking hierarchy (subject-wise scores and accuracy ratios), and the answer key challenge process with its fee and refund logic.
*   **Updated Student Journey**: The journey now reflects these corrected business rules at each stage.

This makes the document a more accurate blueprint for a government-grade portal. If you'd like, I can generate the corresponding **Flyway migration SQL** or a **sequence diagram** for any of these newly defined workflows.
