Kidora --- Backend Architecture & System Workflows
Status: Architecture draft · Architecture: Modular Monolith +
Background Workers
Proposed stack: NestJS, TypeScript, PostgreSQL, Prisma, Redis,
BullMQ

This is a living design document consolidating Kidora's product flows,
backend architecture, Mermaid diagrams, workflow inventory, data-model
draft, security requirements, and implementation plan. It does not mean
every feature belongs in the MVP.
1. Product overview
Kidora is an adaptive educational platform for children aged 4--10. It
combines structured lessons, educational games, an AI Learning Buddy,
and a parent dashboard.

Main actors - Parent/Guardian: account owner, manages child
profiles and permissions, views progress, manages billing. - Child:
uses lessons, games, and AI support inside Child Mode. -
Teacher/School Admin: optional school features; manages classrooms,
assignments, and authorized student progress. - Platform Admin:
content, moderation, support, and operations. - Workers/Providers:
asynchronous jobs, email, push, payments, AI, and object storage.
Key identity decisions
- A child does not have independent email/password login in the
  initial design.
- A parent logs in and selects a linked child profile to enter Child
  Mode.
- Every child-scoped request must validate the child context and
  guardian-child relationship on the server. A client-provided
  childId is not proof of authorization.
- Parent registration creates a PENDING_VERIFICATION account. Login
  and token issuance are blocked until email verification.
  Verification tokens are random, hashed in storage, short-lived, and
  single-use.
- Email verification does not replace legally appropriate guardian
  consent for processing children's data.
2. Product-level user flow
```mermaid
flowchart TD
    A[Open Kidora] --> B{User type}
    B -->|Parent| C[Register or Login]
    C --> D{Email verified?}
    D -->|No| E[Verify email]
    E --> F[Parent onboarding]
    D -->|Yes| F
    F --> G[Manage child profiles]
    G --> H[Select child]
    H --> I[Enter Child Mode]
    I --> J[Child learning home]
    J --> K[Lessons]
    J --> L[Educational games]
    J --> M[AI Learning Buddy]
    K --> N[Progress updated]
    L --> N
    M --> N
    N --> J
    G --> O[Parent dashboard]
    O --> P[Reports, goals and controls]
    O --> Q[Subscription and settings]
    B -->|Teacher / School| R[School dashboard]
    R --> S[Classrooms and assignments]
    R --> T[Authorized student progress]
    B -->|Platform Admin| U[Admin panel]
    U --> V[Content, safety and support]
```
3. High-level system architecture
```mermaid
flowchart TB
    subgraph Clients
      P[Parent Dashboard]
      C[Child App]
      S[School Dashboard - Optional]
      A[Admin Panel]
    end
    subgraph API["NestJS API Entry"]
      G[REST API]
      V[DTO Validation]
      AU[Authentication]
      AZ[Authorization + Child Context]
      RL[Rate Limiting]
    end
    subgraph Modules["Modular Monolith"]
      ID[Identity]
      PC[Parents and Children]
      CO[Consent and Privacy]
      LE[Learning]
      GA[Games and Gamification]
      AI[AI Learning Buddy]
      AN[Analytics and Reports]
      NO[Notifications]
      BI[Billing]
      CM[Content Management]
      SC[School Management - Optional]
      SU[Support]
    end
    subgraph Infra["Data and Async"]
      DB[(PostgreSQL)]
      R[(Redis)]
      OB[(Outbox Events)]
      D[Outbox Dispatcher]
      Q[BullMQ]
      W[Background Workers]
      OS[(Object Storage)]
    end
    subgraph Providers
      EM[Email / Push]
      PAY[Payment Gateway]
      LLM[LLM Provider]
      OBS[Logs / Metrics / Traces]
    end
    P --> G
    C --> G
    S --> G
    A --> G
    G --> V --> AU --> AZ --> RL --> Modules
    Modules --> DB
    Modules --> R
    Modules --> OB
    OB --> D --> Q --> W
    W --> DB
    W --> EM
    BI <--> PAY
    AI <--> LLM
    CM --> OS
    G --> OBS
    W --> OBS
```
Architecture principles 1. Start with a Modular Monolith, not
microservices. 2. PostgreSQL is the transactional source of truth. 3.
Redis supports caching, rate limits, ephemeral state, and BullMQ---not
primary business storage. 4. Use workers for non-interactive tasks. 5.
Write business data and its Outbox event in the same database
transaction. 6. Keep AI behind an application boundary so providers can
change. 7. School functionality remains optional until the launch scope
is decided. 8. Avoid Kafka/Kubernetes until real scale or operational
needs justify them.
4. Backend request lifecycle
```mermaid
flowchart TD
    A[HTTP Request] --> B[Security Headers / CORS / Request ID]
    B --> C[Rate Limiter]
    C --> D[Parse and Validate DTO]
    D --> E{Public endpoint?}
    E -->|Yes| H[Controller]
    E -->|No| F[Validate Access Token / Session]
    F --> G[Load User and Permissions]
    G --> I{Child-scoped request?}
    I -->|Yes| J[Validate Child Context and Guardian Link]
    I -->|No| K[Validate Ownership / Tenant Scope]
    J --> L[Check Consent and Entitlements if required]
    K --> L
    L --> M[Controller]
    H --> N[Application Use Case]
    M --> N
    N --> O[Domain Rules]
    O --> P[Repository / Prisma]
    P --> Q[(PostgreSQL)]
    N --> R[Response Mapper]
    R --> S[HTTP Response]
    N -. same transaction .-> T[(Outbox Event)]
```
5. Main data flow and Outbox pattern
```mermaid
flowchart LR
    A[User Action] --> B[NestJS API]
    B --> C[Authentication and Authorization]
    C --> D[Domain Module]
    D --> E[(PostgreSQL Transaction)]
    E --> F[Business Data]
    E --> G[Outbox Event]
    G --> H[Outbox Dispatcher]
    H --> I[BullMQ]
    I --> J[Worker]
    J --> K[Analytics / Reports / Notifications / Rewards]
    K --> L[(Read Models)]
    L --> M[Authorized Dashboard Query]
```
The domain transaction writes both business data and the Outbox event
atomically. A dispatcher later publishes pending events to BullMQ.
Consumers must be idempotent because delivery can happen more than once.
6. Core sequence diagrams
6.1 Parent registration and email verification
```mermaid
sequenceDiagram
    autonumber
    actor Parent
    participant Client
    participant API
    participant Auth as Identity Module
    participant DB as PostgreSQL
    participant Outbox
    participant Worker
    participant Email as Email Provider
    Parent->>Client: Submit registration
    Client->>API: POST /auth/register
    API->>Auth: Register parent
    Auth->>DB: Check email uniqueness
    Auth->>Auth: Hash password and verification token
    Auth->>DB: Create PENDING_VERIFICATION user + token hash
    Auth->>DB: Insert VerificationEmailRequested Outbox event
    Auth-->>Client: Registration accepted; check email
    Outbox->>Worker: Dispatch email job
    Worker->>Email: Send verification link
    Parent->>Client: Open verification link
    Client->>API: POST /auth/verify-email
    API->>Auth: Verify token
    Auth->>DB: Validate hash, expiry, unused state
    Auth->>DB: Mark token used and user ACTIVE
    Auth-->>Client: Email verified
```
6.2 Parent login and refresh token rotation
```mermaid
sequenceDiagram
    autonumber
    actor Parent
    participant Client
    participant API
    participant Auth
    participant DB as PostgreSQL
    Parent->>Client: Enter credentials
    Client->>API: POST /auth/login
    API->>Auth: Authenticate
    Auth->>DB: Load account and status
    Auth->>Auth: Verify password
    alt Email not verified
      Auth-->>Client: EMAIL_VERIFICATION_REQUIRED
    else Invalid credentials
      Auth-->>Client: Generic authentication failure
    else Active account
      Auth->>DB: Create refresh session record
      Auth-->>Client: Access token + secure refresh cookie
    end
    Client->>API: POST /auth/refresh
    API->>Auth: Validate refresh token/session
    Auth->>DB: Check session, expiry and token hash
    Auth->>DB: Rotate refresh token
    Auth-->>Client: New access token + refresh cookie
```
6.3 Parent selects child / enters Child Mode
```mermaid
sequenceDiagram
    autonumber
    actor Parent
    participant Client
    participant API
    participant Child as Child Module
    participant Consent
    participant DB as PostgreSQL
    Parent->>Client: Select child profile
    Client->>API: POST /children/{childId}/select
    API->>Child: Validate selection
    Child->>DB: Check child is active
    Child->>DB: Verify guardian-child relationship
    Child->>Consent: Check required consent and purpose
    Consent->>DB: Load consent status
    alt Relationship or required consent invalid
      Child-->>API: Reject child context
      API-->>Client: Access denied / consent required
    else Valid
      Child->>DB: Create or rotate child context
      Child-->>API: Child context established
      API-->>Client: Child Mode context
    end
```
6.4 Learning session and answer evaluation
```mermaid
sequenceDiagram
    autonumber
    actor Child
    participant Client
    participant API
    participant Learning
    participant DB as PostgreSQL
    participant Outbox
    participant Queue as BullMQ
    participant Worker
    Child->>Client: Open lesson
    Client->>API: POST /learning/sessions
    API->>Learning: Start session
    Learning->>Learning: Validate child context and lesson
    Learning->>DB: Create learning session
    Learning-->>Client: Session and lesson content
    loop Each question
      Child->>Client: Submit answer
      Client->>API: POST /learning/sessions/{id}/answers
      API->>Learning: Evaluate answer
      Learning->>DB: Validate session and persist attempt
      Learning-->>Client: Result and feedback
    end
    Child->>Client: Complete lesson
    Client->>API: POST /learning/sessions/{id}/complete
    API->>Learning: Complete session
    Learning->>DB: Update session state
    Learning->>DB: Insert LessonCompleted Outbox event
    Learning-->>Client: Completion result
    Outbox->>Queue: Dispatcher publishes job
    Queue->>Worker: Process completion
    Worker->>DB: Update mastery, analytics, missions and rewards idempotently
```
6.5 Game session and reward ledger
```mermaid
sequenceDiagram
    autonumber
    actor Child
    participant Client
    participant API
    participant Games
    participant DB as PostgreSQL
    participant Outbox
    participant Worker
    Child->>Client: Start game level
    Client->>API: POST /games/{gameId}/sessions
    API->>Games: Start game
    Games->>Games: Validate child context, game and level
    Games->>DB: Create game session
    Games-->>Client: Session configuration
    Child->>Client: Submit answer
    Client->>API: POST /game-sessions/{id}/answers
    API->>Games: Evaluate submission
    Games->>DB: Validate and store answer
    Games-->>Client: Server-evaluated result
    Child->>Client: Complete level
    Client->>API: POST /game-sessions/{id}/complete
    API->>Games: Complete session
    Games->>DB: Persist score and completion
    Games->>DB: Insert GameCompleted Outbox event
    Games-->>Client: Completion summary
    Outbox->>Worker: Dispatch event
    Worker->>DB: Insert idempotent reward ledger entry
    Worker->>DB: Update achievements and missions
```
6.6 AI Learning Buddy
```mermaid
sequenceDiagram
    autonumber
    actor Child
    participant Client
    participant API
    participant AI as AI Application Service
    participant Safety
    participant RAG as Approved Content Retrieval
    participant LLM as LLM Provider
    participant DB as PostgreSQL
    Child->>Client: Ask question
    Client->>API: POST /ai/conversations/{id}/messages
    API->>AI: Send message with child context
    AI->>AI: Check consent, entitlement and usage limit
    AI->>Safety: Moderate input
    alt Input blocked
      Safety-->>AI: Block category
      AI-->>Client: Safe fallback
    else Input allowed
      AI->>RAG: Retrieve approved age-appropriate content
      RAG-->>AI: Relevant context
      AI->>LLM: Generate explanation or hint
      LLM-->>AI: Draft response
      AI->>Safety: Review output
      alt Output rejected or provider fails
        Safety-->>AI: Reject / fallback
        AI-->>Client: Safe fallback or retry-later response
      else Output approved
        AI->>DB: Persist messages and usage
        AI-->>Client: Age-appropriate explanation
      end
    end
```
6.7 Parent report generation
```mermaid
flowchart TD
    A[Scheduled Job or Parent Request] --> B[Authorize Guardian]
    B --> C[Load Child and Period]
    C --> D[Analytics Processor]
    D --> E[Calculate Deterministic Metrics]
    E --> F[Rules Engine]
    F --> G[Evidence-Based Insights]
    G --> H{AI wording needed?}
    H -->|No| I[Build Report]
    H -->|Yes| J[Generate Parent-Friendly Wording]
    J --> K[Safety and Evidence Check]
    K --> I
    I --> L[Save Versioned Report Snapshot]
    L --> M[Outbox ReportReady]
    M --> N[Notification Worker]
    N --> O[Notify Guardian]
    O --> P[Parent Dashboard]
```
Report content: overall progress, repeated strengths, areas for
practice, trends, parent guidance, next goals, and achievements. Metrics
must come from actual data. Compare the child with their own previous
performance, not harmful public rankings. AI must not diagnose a child
or invent evidence.
6.8 Subscription and payment webhook
```mermaid
sequenceDiagram
    autonumber
    actor Parent
    participant Client
    participant API
    participant Billing
    participant DB as PostgreSQL
    participant Pay as Payment Gateway
    participant Outbox
    participant Worker
    Parent->>Client: Choose plan
    Client->>API: POST /billing/checkout
    API->>Billing: Create checkout
    Billing->>DB: Create pending subscription/payment
    Billing->>Pay: Create payment session
    Pay-->>Client: Hosted checkout URL
    Parent->>Pay: Complete payment
    Pay->>API: Payment webhook
    API->>Billing: Verify signature and event
    Billing->>DB: Check provider event ID for idempotency
    Billing->>DB: Update payment/subscription transactionally
    Billing->>DB: Insert SubscriptionActivated Outbox event
    API-->>Pay: Acknowledge webhook
    Outbox->>Worker: Dispatch event
    Worker->>DB: Update entitlements/read models
```
Never trust the browser redirect as proof of payment. Verify webhook
signatures and process provider event IDs idempotently. Renewal, retry,
refund, grace period, and cancellation timing need product decisions.
6.9 Outbox dispatcher and worker
```mermaid
sequenceDiagram
    autonumber
    participant Module as Domain Module
    participant DB as PostgreSQL
    participant Dispatcher
    participant Queue as BullMQ
    participant Worker
    Module->>DB: Begin transaction
    Module->>DB: Write business data
    Module->>DB: Insert Outbox event
    Module->>DB: Commit
    loop Poll unpublished events
      Dispatcher->>DB: Read pending events
      Dispatcher->>Queue: Publish job with event ID
      Dispatcher->>DB: Mark event dispatched
    end
    Queue->>Worker: Deliver job
    Worker->>DB: Check processed-event/idempotency record
    alt Already processed
      Worker-->>Queue: Acknowledge duplicate
    else New event
      Worker->>DB: Apply database changes
      Worker->>DB: Store processed-event record
      Worker-->>Queue: Acknowledge success
    end
```
Use bounded exponential backoff with jitter, idempotent handlers,
failed-job handling, and alerts for old unpublished events or repeated
failures.
7. Workflow inventory
This is a planning inventory, not a promise that all items are MVP
scope.
  Domain                              Workflows
  Identity                            Register, verify email, resend
verification, login, refresh,
logout, forgot/reset password,
revoke all sessions

  Parent                              Onboarding, update profile, manage
settings

  Child                               Add/view/update/deactivate child,
select child, switch child

  Consent & Privacy                   Record consent, withdraw consent,
export data, delete account/data,
apply retention policy

  Learning                            Browse catalog, start/resume
session, submit answer, complete
lesson, assess skill, update
mastery

  Games                               Browse games, start session, submit
answer, complete game

  Gamification                        Award points, unlock achievement,
update mission

  AI                                  Start conversation, send question,
generate hint, moderate
input/output, provider
failure/fallback

  Parent Reports                      View dashboard/history, generate
report/insights, set and track
goals

  Notifications                       Create, deliver, track, retry
failed delivery

  Billing                             Subscribe, webhook, renew, payment
failure, cancel, refund

  School (optional)                   Create tenant, invite teacher,
manage classroom, enroll child,
assign lesson, view authorized
progress

  Content                             Create/update content, review
AI-generated content,
publish/reject

  Support                             Create ticket, assign/handle
ticket, close ticket

  Operations                          Process Outbox, retry failed jobs,
scheduled analytics, data retention
Security                            Audit sensitive action, rate-limit
and abuse handling
8. Suggested backend module structure
src/
├── main.ts
├── app.module.ts
├── common/
│   ├── decorators/
│   ├── guards/
│   ├── interceptors/
│   ├── filters/
│   ├── pipes/
│   ├── middleware/
│   ├── exceptions/
│   ├── constants/
│   ├── interfaces/
│   ├── utils/
│   └── types/
├── core/
│   ├── config/
│   ├── database/
│   ├── prisma/
│   ├── redis/
│   ├── queue/
│   ├── logger/
│   ├── events/
│   └── observability/
├── modules/
│   ├── identity/
│   ├── parents/
│   ├── children/
│   ├── consent/
│   ├── learning/
│   ├── games/
│   ├── gamification/
│   ├── ai-learning-buddy/
│   ├── analytics/
│   ├── parent-reports/
│   ├── notifications/
│   ├── billing/
│   ├── content/
│   ├── schools/       # Optional / phase-dependent
│   └── support/
└── workers/
    ├── email/
    ├── notifications/
    ├── analytics/
    ├── reports/
    ├── gamification/
    └── ai/
Internal structure of a module
learning/
├── presentation/       # Controllers, DTOs, guards, mappers
├── application/        # Use cases, commands, queries, services
├── domain/             # Entities, value objects, rules, events
├── infrastructure/     # Repositories, Prisma, external providers
└── learning.module.ts
Create folders when needed; avoid empty architecture scaffolding.
9. Initial data model --- draft
Not a finalized schema. Confirm scope, constraints, retention, and
authorization before migrations.
  Domain                              Candidate tables
  Identity                            users, auth_identities,
refresh_sessions, roles,
user_role_assignments,
permissions

  Parent/Child                        parent_profiles,
child_profiles,
guardian_child_links,
child_contexts

  Consent                             consents, consent_versions or
consent evidence fields

  Learning                            subjects, skills,
skill_prerequisites, lessons,
lesson_skills, questions,
learning_sessions, attempts,
skill_assessments,
skill_mastery

  Games                               games, game_levels,
game_skills, game_sessions,
game_answers, game_events

  Gamification                        achievements,
child_achievements, missions,
mission_progress, reward_ledger

  AI                                  ai_conversations, ai_messages,
ai_usage_records,
ai_content_reviews

  Recommendations                     recommendations,
recommendation_feedback

  Analytics/Reports                   learning_analytics,
learning_goals, parent_reports,
report_skill_insights,
report_recommendations,
parent_action_items

  Notifications                       notifications,
notification_preferences,
notification_deliveries

  Billing                             plans, plan_features,
subscriptions, invoices,
payment_transactions,
payment_provider_events

  School (optional)                   tenants, tenant_memberships,
schools, classrooms,
classroom_teachers,
classroom_enrollments,
lesson_assignments
Operations                          outbox_events,
processed_events,
job_executions, audit_logs,
support_tickets
Data design notes - Enforce tenant and guardian-child isolation on
every relevant query. Composite keys and PostgreSQL RLS can provide
defense in depth. - entity_type + entity_id polymorphic references are
not real foreign keys; use typed references or explicit integrity rules
where needed. - Keep immutable historical assessments separate from the
latest skill_mastery projection. - Use JSONB for flexible
configuration only, not as a substitute for relational modeling. -
Decide whether subjects/skills are global catalog data and who owns
lessons. - Keep reward ledger append-only, with idempotency keys and
reversal entries. - Define analytics granularity, timezone handling, and
unique aggregation keys. - Do not create all proposed tables in the MVP.
10. Security, privacy and child safety
- Obtain and record legally appropriate guardian consent for required
  purposes.
- Minimize child data collection and define retention/deletion rules.
- Enforce guardian-child checks on every parent read/write.
- Enforce child context on every child-scoped operation.
- Enforce tenant scope for school-owned resources.
- Hash passwords and one-time tokens; never store raw
  reset/verification tokens.
- Use secure HttpOnly cookies and refresh-token rotation.
- Rate-limit login, registration, resend, reset, AI, and abuse-prone
  endpoints.
- Audit sensitive actions: consent changes, child-profile access,
  export, deletion.
- Moderate AI inputs and outputs; retrieve only approved
  age-appropriate content.
- Never expose raw child data to teachers or other guardians without
  authorization.
- AI insights must be evidence-grounded and must not diagnose
  children.
11. Observability and reliability
Proposed tools: Pino, Sentry (or equivalent), OpenTelemetry,
Prometheus/Grafana, k6, Jest, Supertest.

Monitor: - API p50/p95/p99 latency, request rate, error rate. - Database
query latency and connection-pool saturation. - Redis health and
latency. - Queue depth, oldest waiting job, retries, failed jobs. -
Outbox backlog and oldest unpublished event. - Email/push delivery
success. - AI latency, failures, token usage, and cost. - Payment
webhook verification failures and processing delay.

Use timeouts for external calls, bounded retries with exponential
backoff and jitter, idempotency keys, database transactions, graceful
degradation, backups, and tested restore procedures.
12. Example API endpoints
Illustrative only; finalize contracts during API design.
  Method                  Endpoint                                       Purpose
  POST                    /auth/register                               Register parent

  POST                    /auth/verify-email                           Verify parent email

  POST                    /auth/resend-verification                    Resend verification

  POST                    /auth/login                                  Parent login

  POST                    /auth/refresh                                Rotate refresh token

  POST                    /auth/logout                                 Revoke current session

  POST                    /auth/forgot-password                        Start reset

  POST                    /auth/reset-password                         Complete reset

  GET                     /children                                    List guardian's
children

  POST                    /children                                    Create child profile

  PATCH                   /children/:childId                           Update child

  POST                    /children/:childId/select                    Enter Child Mode

  GET                     /subjects                                    Browse subjects

  GET                     /lessons/:lessonId                           View lesson

  POST                    /learning/sessions                           Start session

  POST                    /learning/sessions/:sessionId/answers        Submit answer

  POST                    /learning/sessions/:sessionId/complete       Complete lesson

  GET                     /games                                       Browse games

  POST                    /games/:gameId/sessions                      Start game

  POST                    /game-sessions/:sessionId/answers            Submit game answer

  POST                    /game-sessions/:sessionId/complete           Complete game

  POST                    /ai/conversations                            Start AI conversation

  POST                    /ai/conversations/:conversationId/messages   Send AI message

  GET                     /parent/children/:childId/dashboard          Authorized child
dashboard

  GET                     /parent/children/:childId/reports            List reports

  POST                    /billing/checkout                            Create checkout
POST                    /webhooks/payments/:provider                 Payment webhook
For every endpoint define authentication, permissions, child/tenant
scope, consent, DTOs, response, status codes, idempotency, rate limits,
and audit requirements.
13. Testing strategy
- Unit: domain rules, answer evaluation, mastery calculation,
  rewards, consent, entitlements, AI fallback.
- Integration: repositories, transactions, Outbox, guardian-child
  authorization, tenant isolation, token rotation/replay, payment
  webhook idempotency, worker handlers.
- E2E: register → verify → login; create/select child; complete
  lesson; complete game and award once; parent reads only own child's
  report; AI safe response/fallback; payment webhook activates
  entitlement once.
- Load/resilience: catalog reads, answer submissions, dashboards,
  AI limits/timeouts, queue backlog, worker recovery, DB saturation.
14. Suggested implementation phases
1. Foundation: identity, verification, parent, child,
   guardian-child link, consent, authorization, audit basics.
2. Core learning: catalog, lessons, questions, sessions, attempts,
   evaluation, basic parent dashboard.
3. Engagement: built-in games, server-side scoring, reward ledger,
   achievements, missions.
4. AI support: conversations, approved-content retrieval,
   input/output safety, usage limits, fallback, cost monitoring.
5. Reports and operations: analytics, reports, goals,
   notifications, workers, operational dashboards.
6. Expansion: subscriptions/payments and school/tenant features if
   validated.
Do not implement every module just because it appears in this document.
Choose MVP scope based on validation, legal requirements, delivery
capacity, and business priorities.
15. Open product decisions
1. Are schools/teachers in the first MVP, or is launch family-only?
2. What guardian consent purposes are required, and what happens when
   consent is withdrawn?
3. Which child profile fields are essential?
4. Which features are free vs paid, and are limits per family, child,
   or usage?
5. Does cancellation take effect immediately or at the end of the paid
   period?
6. What payment retry schedule or grace period applies?
7. What refund policy must be supported?
8. What AI usage/session limits apply?
9. Which content is authored, AI-generated, reviewed, and published?
10. Which report metrics are essential for the first release?
11. Can multiple guardians access a child, and what can each do?
12. What is the data export/deletion policy?
16. Workflow specification template
Use this before implementing each workflow:
Workflow Name:

1. Business Goal
2. Trigger
3. Actors
4. Preconditions
5. Main Flow
6. Alternative Flows
7. Failure Scenarios
8. Business Rules
9. Components Involved
10. Database Operations and Transaction Boundaries
11. Events and Background Jobs
12. Security and Authorization
13. API Contracts
14. Sequence Diagram
15. Test Scenarios
For every feature ask: Who? What? Why? When? Where? How? What if?

Recommended engineering order:

Business Goal → Actors & Permissions → Business Rules → Data
Requirements → System Flow → Failure Scenarios → API Design →
Implementation → Testing → Observability
17. Working notes
- Diagrams use Mermaid and can be rendered directly by GitHub
  Markdown.
- Keep this file updated as decisions change.
- Mark features MVP, Later, or Out of Scope after scope is
  agreed.
- Convert each workflow into a detailed specification b
