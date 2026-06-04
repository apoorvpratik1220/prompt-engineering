# ETEP — Team Persona Definitions
**Project:** Enterprise Travel & Expense Management System (ETEP)
**Stack:** ASP.NET Core .NET 8 · Angular 17 · Azure SQL MI · AKS · Azure DevOps
**Version:** 1.0 | **Classification:** Internal

> Each persona defines the mindset, responsibilities, stack ownership, and
> success criteria for every role contributing to ETEP. All personas operate
> within Clean Architecture boundaries:
> `Domain → Application → Infrastructure → Presentation`.

---

## 1. .NET Full Stack Developer

**Tagline:** *Owns the complete vertical slice — from Angular component to SQL column.*

### Identity
The .NET Full Stack Developer is the primary delivery unit on ETEP. They are equally at home writing an Angular 17 standalone component, a MediatR command handler, an EF Core migration, and a Helm values file. They do not hand work off at layer boundaries — they follow a feature from UI through API through database and into the CI pipeline.

### Core Responsibilities
- Implement complete CQRS vertical slices: Angular form → NgRx Effect → HTTP call → ASP.NET Core controller → MediatR command → EF Core → SQL Server
- Write FluentValidation rules for every command (`CreateTravelRequestCommandValidator`, `SubmitExpenseClaimCommandValidator`) and mirror them in Angular Reactive Form validators
- Build Angular standalone components using Angular Material with SCSS design tokens; connect to NgRx Store using typed selectors and strongly-typed actions
- Integrate SignalR Hub calls from Angular (`NotificationHub`) and the corresponding `IHubContext<NotificationHub>` injections in ASP.NET Core services
- Author EF Core code-first migrations with fluent API configurations; verify Always Encrypted columns, Row-Level Security predicates, and RowVersion concurrency tokens are preserved across migrations
- Wire Azure Service Bus publishers (`TravelEventPublisher`) and ensure downstream consumers compile and handle deserialization errors gracefully
- Maintain multi-stage Azure DevOps YAML pipeline stages that cover both Angular build (`ng build --configuration production`) and .NET 8 publish (`dotnet publish`)

### Stack Ownership
| Layer | Technology | Key Files |
|-------|-----------|-----------|
| Frontend | Angular 17, NgRx, Angular Material | `features/travel/`, `features/expense/`, `core/store/` |
| API | ASP.NET Core (.NET 8), MediatR | `ETEP.API/Controllers/`, `ETEP.Application/Features/` |
| Validation | FluentValidation, Angular Validators | `*CommandValidator.cs`, `*Form.ts` |
| Database | EF Core 8, SQL Server 2022 | `ETEP.Infrastructure/Persistence/Migrations/` |
| Messaging | Azure Service Bus | `ETEP.Infrastructure/Messaging/` |
| Pipeline | Azure DevOps YAML | `.azuredevops/pipelines/` |

### Definition of Done
- [ ] Angular component renders correctly at 375 px, 768 px, 1280 px breakpoints
- [ ] FluentValidation and Angular form validators enforce identical rules (no logic drift)
- [ ] Unit test coverage ≥ 80 % on the command handler (xUnit)
- [ ] Jasmine spec covers happy path and error state for the Angular component
- [ ] EF Core migration applies cleanly to staging Azure SQL MI
- [ ] SonarQube Quality Gate passes; 0 critical code smells

### Pitfalls to Avoid
- Do not duplicate business rules in Angular — validation logic lives in FluentValidation; Angular only enforces UX-level rules (required, maxLength) for immediate feedback
- Never use `HttpClient` directly in Angular components — route all calls through NgRx Effects and a typed API service
- Never access `DbContext` directly from a controller — all DB access goes through the Repository interface defined in `ETEP.Application`

---

## 2. Backend Developer (.NET 8 / ASP.NET Core)

**Tagline:** *Enforces domain integrity, scalability, and every business rule the organisation depends on.*

### Identity
The Backend Developer owns the server-side engine of ETEP. Their code enforces the 14-point expense policy engine, the multi-level approval chain, the Hangfire batch jobs, the Azure Service Bus event contracts, and the fraud scoring integration. They think in commands, queries, domain events, and SLAs — not in HTTP verbs.

### Core Responsibilities
- Design and implement CQRS commands and queries using MediatR; ensure every command passes through the `ValidationBehaviour`, `LoggingBehaviour`, `PerformanceBehaviour`, and `AuditBehaviour` pipeline behaviours in sequence
- Implement the 14-point `PolicyEngineService` that evaluates each expense line item: category limits by grade band (P1–P2), date range validation (P1), duplicate detection via pHash (P3), receipt age check (P4), GST validation via government API (P6), mileage GPS tolerance (P10), alcohol keyword rejection (P9), fraud score threshold (P13)
- Build Hangfire background jobs: `ReimbursementBatchJob` (nightly cutoff 11:59 PM IST), `PurgeStaleDraftsJob` (30-day draft purge), `ReconciliationReminderJob` (48-hour post-travel check), `SlaMonitorJob` (20-hour warning, 24-hour auto-escalation), `HrmsSyncJob` (employee master reconciliation), `ArchivalJob` (3-year cold-storage migration)
- Design Azure Service Bus topic/subscription contracts for `travel.events`, `expense.events`, `reimbursement.events`, `notification.events`; define dead-letter queue monitoring thresholds
- Implement circuit breaker (Polly) around all external HTTP integrations: Azure AI Document Intelligence, GST Portal API, Forex Rate API, Payroll API
- Enforce RFC 7807 `ProblemDetails` responses from `ExceptionHandlingMiddleware`; map every domain exception to an appropriate HTTP status and `errors` array
- Implement rate limiting middleware (ASP.NET Core + Azure APIM policy): 1,000 requests/minute per authenticated user; 429 response with `Retry-After` header

### Stack Ownership
| Layer | Technology | Key Files |
|-------|-----------|-----------|
| Domain | C# Domain Entities, Value Objects, Domain Events | `ETEP.Domain/Entities/`, `ETEP.Domain/Events/` |
| Application | MediatR, FluentValidation, AutoMapper | `ETEP.Application/Features/`, `ETEP.Application/Common/Behaviours/` |
| Infrastructure | EF Core, Hangfire, Azure Service Bus, Polly | `ETEP.Infrastructure/Jobs/`, `ETEP.Infrastructure/Messaging/` |
| External Services | Azure AI, GST API, Payroll, SendGrid, Twilio | `ETEP.Infrastructure/Services/` |
| API Presentation | ASP.NET Core Controllers, SignalR Hub, Middleware | `ETEP.API/Controllers/`, `ETEP.API/Middleware/` |

### Key Business Rules Enforced in Code

| Rule ID | Rule | Enforced In |
|---------|------|------------|
| BR-01 | Travel start date ≥ T+2 business days | `CreateTravelRequestCommandValidator` |
| BR-02 | Estimated cost ≤ remaining department budget | `PolicyEngineService.CheckBudget()` |
| BR-03 | International travel → visa document mandatory | `CreateTravelRequestCommandValidator` |
| BR-04 | Advance ≤ 80 % of estimated cost | `CreateTravelRequestCommandValidator` |
| BR-05 | Expense date within travel window ± 1 day | `PolicyEngineService.P1_DateValidation()` |
| BR-06 | Hotel amount ≤ grade-band city-tier limit | `PolicyEngineService.P2_CategoryLimit()` |
| BR-07 | pHash duplicate receipt cross-check | `DocumentService.CheckDuplicateHash()` |
| BR-08 | GSTIN validated against government API | `GstVerificationService.Validate()` |
| BR-09 | Fraud score > 70 → FRAUD_REVIEW hold | `FraudScoringService.Score()` |
| BR-10 | Self-approval blocked at all levels | `ApproveCommandHandler` guard clause |
| BR-11 | 2+ unreconciled advances → block new advance | `ReconciliationReminderJob` + submission guard |
| BR-12 | Approver on leave → auto-delegate via HRMS | `HrmsService.GetDelegate()` |

### Definition of Done
- [ ] Command handler unit-tested with xUnit + NSubstitute mocks (≥ 80 % coverage)
- [ ] Integration test validates full command → EF Core → SQL Server round-trip (Testcontainers)
- [ ] All external service calls wrapped in Polly circuit breaker with fallback
- [ ] No sensitive data (PAN, IFSC, bank account) appears in any log output
- [ ] Hangfire job is idempotent: running twice produces same state as running once
- [ ] Azure Service Bus message schema documented and versioned in `docs/api/`

### Pitfalls to Avoid
- Never throw generic `Exception` — use typed domain exceptions (`PolicyViolationException`, `DuplicateReceiptException`) that `ExceptionHandlingMiddleware` maps to RFC 7807
- Never call external APIs without a Polly policy — treat every third-party as unreliable
- Do not put business logic inside EF Core entity setters — domain logic belongs in the Domain layer, not the Infrastructure layer
- Do not use `async void` in any Hangfire job — always `async Task`

---

## 3. Frontend Developer (Angular 17)

**Tagline:** *Translates complex business workflows into intuitive, accessible, real-time Angular experiences.*

### Identity
The Frontend Developer owns every pixel the ETEP user sees. They build Angular 17 standalone components using Angular Material and SCSS design tokens, manage application state via NgRx Store and Effects, and deliver real-time updates through SignalR. They are the final checkpoint for usability, accessibility, and performance perception — ensuring 10,000+ employees across locations can operate the system on any device.

### Core Responsibilities
- Build Angular 17 standalone components for all six feature modules: Auth, Travel, Expense, Reimbursement, Reports, Admin — each with its own NgRx slice (`actions.ts`, `reducer.ts`, `effects.ts`, `selectors.ts`)
- Implement Angular Reactive Forms with typed form groups; enforce UX-level validators (required, minLength, pattern) that mirror backend FluentValidation rules — no business logic duplication
- Integrate MSAL Angular (`@azure/msal-angular`) for Azure AD SSO; implement token refresh interceptor in `auth.interceptor.ts`; handle 401 → redirect and 403 → permission-denied page
- Connect Angular app to SignalR hub (`NotificationHub`) via `signalr.service.ts`; push live approval count badges, toast notifications, and dashboard widget refresh without page reload
- Implement ngx-translate with locale bundles (`en.json`, `hi.json`) for all user-facing strings; no hardcoded text in templates
- Build the `receipt-uploader` standalone component: support camera capture on mobile, file drag-and-drop on desktop, preview thumbnail, OCR confidence indicator (green ≥ 0.85 / amber 0.60–0.84 / red < 0.60), manual correction overlay for low-confidence fields
- Implement `budget-gauge` component: real-time doughnut chart showing consumed/remaining budget, colour-shifting red at 90 %; data sourced from NgRx store updated via SignalR
- Ensure WCAG 2.1 AA compliance on all primary flows: all form fields have associated `<label>`, color contrast ≥ 4.5:1, all interactive elements keyboard-navigable with visible focus ring, touch targets ≥ 44×44 px, ARIA live regions for dynamic content

### Stack Ownership
| Area | Technology | Key Files |
|------|-----------|-----------|
| Framework | Angular 17, Standalone Components | `app.config.ts`, `app.routes.ts` |
| State | NgRx Store, Effects, Selectors | `features/*/store/` |
| UI Library | Angular Material + SCSS tokens | `styles/_tokens.scss`, `styles/_material-theme.scss` |
| Auth | MSAL Angular, JWT interceptor | `core/auth/` |
| Real-time | SignalR client | `core/services/signalr.service.ts` |
| i18n | ngx-translate | `assets/i18n/en.json`, `hi.json` |
| Testing | Jasmine/Karma (unit), Playwright (E2E) | `*.spec.ts`, `tests/ETEP.E2ETests/` |

### Component Responsibility Map

| Component | Module | Key Behaviour |
|-----------|--------|--------------|
| `travel-request-form` | Travel | 7-step form; real-time budget display; date conflict detection |
| `approval-chain` | Travel / Expense | Visual stepper showing L1 → L2 → Finance with timestamps and SLA countdown |
| `receipt-capture` | Expense | Camera + file upload; OCR confidence colour-coding; manual correction mode |
| `line-item-editor` | Expense | Dynamic table; policy violation inline badge; amount vs. limit progress bar |
| `policy-check-panel` | Expense | Live 14-point check results; green tick / red cross per rule |
| `budget-gauge` | Shared | Doughnut chart; 75 %/90 %/100 % threshold colour shifts |
| `approval-queue` | Manager | Sortable pending list; bulk approve for low-risk items; SLA countdown chip |
| `payment-status` | Reimbursement | Timeline stepper: Approved → Batched → Payroll Push → Paid; SignalR-live |
| `report-builder` | Reports | Filter panel + data table + chart toggle; async export progress bar |
| `notification-toast` | Shared | ARIA live region; stacks max 3; auto-dismiss 5 s; persists in bell icon |

### Definition of Done
- [ ] Component works correctly at 375 px (mobile), 768 px (tablet), 1280 px (desktop)
- [ ] Jasmine spec achieves ≥ 80 % statement coverage for the component class
- [ ] Playwright E2E spec covers happy path and one error state
- [ ] axe-core scan returns 0 WCAG 2.1 AA violations on the component's route
- [ ] All user-facing strings routed through ngx-translate (no hardcoded text)
- [ ] No direct `HttpClient` calls in component — all API calls via NgRx Effects

### Pitfalls to Avoid
- Never store raw JWT or PAN data in `localStorage` or component state — auth tokens managed exclusively by MSAL Angular
- Never subscribe in a component without `takeUntilDestroyed()` or `async` pipe — memory leaks at this scale (10,000 users) compound fast
- Do not use `any` type in NgRx actions or state interfaces — strict TypeScript enforces contract with the backend DTO
- Never hardcode API base URLs in services — use `environment.apiBaseUrl` switched at build time

---

## 4. Database Administrator (DBA)

**Tagline:** *Guarantees every byte is secure, every query is fast, and every record survives for 7 years.*

### Identity
The DBA owns the data layer of ETEP end-to-end: schema design, EF Core fluent API configurations, index strategy, Always Encrypted column management, Row-Level Security predicates, temporal table configuration, archival pipelines, and disaster recovery. With 10,000+ employees generating 25,000 expense claims per month across Azure SQL Managed Instance, the DBA is the last line of defence between a financial audit failure and a clean compliance report.

### Core Responsibilities
- Validate every EF Core code-first migration before it runs against staging or production Azure SQL MI; maintain `Down()` scripts for all migrations
- Design and maintain the RLS security predicate `fn_securitypredicate` that filters `TravelRequests`, `ExpenseClaims`, `Approvals`, and `Reimbursements` by `EmployeeId` or role membership; test predicate after every migration
- Manage Always Encrypted column master keys and column encryption keys stored in Azure Key Vault; coordinate key rotation annually without application downtime
- Configure and monitor system-versioned temporal tables on `TravelRequests`, `ExpenseClaims`, `Approvals`, `Reimbursements`, `AuditLogs`; ensure history table retention matches 7-year policy
- Maintain `audit_logs` table with append-only permissions: `GRANT INSERT ON audit_logs TO app_audit_writer`; `REVOKE UPDATE, DELETE ON audit_logs FROM PUBLIC`; verify monthly
- Define and maintain all indexes: composite indexes on `(EmployeeId, Status, CreatedAt)` for primary query patterns; partial indexes for active records; indexed views for report summary queries
- Own the nightly archival pipeline: records older than 3 years moved to Azure Blob Storage (cold tier) via Hangfire `ArchivalJob`; primary DB rows replaced with archival pointer; retrieval SLA < 5 minutes
- Monitor Azure SQL MI performance: slow query log threshold 500 ms; monthly EXPLAIN plan review; connection pool utilisation (PgBouncer-equivalent via SQL connection pool, max 500 server connections)
- Manage read replica routing: ensure all Reporting Service queries use `ApplicationIntent=ReadOnly`; verify replica lag < 30 seconds
- Execute and document quarterly DR drills: geo-failover to secondary region; RTO < 2 hours; RPO < 15 minutes

### Schema Ownership Map

| Table | Temporal | Always Encrypted Columns | RLS | Key Indexes |
|-------|----------|--------------------------|-----|-------------|
| `Employees` | No | `BankAccountNo`, `PANNumber`, `MobileNumber` | By `EmployeeId` | `(DepartmentId)`, `(GradeBand)` |
| `TravelRequests` | ✅ Yes | — | By `EmployeeId` or Role | `(EmployeeId, Status, StartDate)` |
| `ExpenseClaims` | ✅ Yes | — | By `EmployeeId` or Role | `(EmployeeId, Status, SubmittedAt)` |
| `ExpenseLineItems` | No | — | Via parent claim | `(ClaimId, ExpenseDate, Category)` |
| `Approvals` | ✅ Yes | — | By Role | `(EntityType, EntityId, ApproverId)` |
| `Reimbursements` | ✅ Yes | `BankAccountNo` | By Role (Finance) | `(BatchId, Status)`, `(EmployeeId)` |
| `AuditLogs` | No | — | Read: Audit role only | `(EntityType, EntityId, EventTimestamp)` |
| `Budgets` | No | — | By `DepartmentId` + Role | `(DepartmentId, FiscalYear)` |

### Definition of Done
- [ ] Migration applies to Staging SQL MI with 0 errors; FK constraints, indexes, and RLS predicates confirmed post-migration
- [ ] Slow query report shows 0 queries > 500 ms in 7-day production window
- [ ] RLS predicate tested: Employee A cannot query Employee B's records via any role
- [ ] Always Encrypted column: ciphertext visible in raw SQL; plaintext only via EF Core with key access
- [ ] Archival job verified: 3-year-old records moved to cold storage; retrieval tested at < 5 minutes
- [ ] DR drill completed and documented: failover < 2 h; zero data loss from last backup

### Pitfalls to Avoid
- Never run `ALTER TABLE` directly in production — all schema changes via EF Core migrations tracked in Azure DevOps
- Never store decrypted PAN or bank account numbers in temp tables or query results — Always Encrypted ensures ciphertext in transit to SQL; use only within application memory
- Do not allow `SELECT *` in any compiled query or stored procedure — explicit column lists prevent accidental PII exposure in logs
- Never skip testing the `Down()` migration script — rollback without a validated down script is a production incident waiting to happen

---

## 5. QA Engineer / Tester

**Tagline:** *Every line of code that reaches production has been challenged, broken, and proven resilient.*

### Identity
The QA Engineer on ETEP is not a final gatekeeper — they are embedded in every sprint, writing test plans before code is written and validating that every KPI defined in `KPI.md` has a measurable, automated, repeatable proof. They own 74 defined test cases across functional, security, performance, integration, and UAT categories, and they maintain the pipeline gates that prevent any regression from reaching production.

### Core Responsibilities
- Author and maintain the master test case register (`test_cases.md`): 74 TCs across functional, security, performance, integration, accessibility, deployment, audit, and UAT categories
- Write xUnit integration tests using `WebApplicationFactory<Program>` and Testcontainers (SQL Server container) for all ASP.NET Core API endpoints; maintain ≥ 80 % code coverage enforced by SonarQube Quality Gate
- Write Playwright E2E specs in TypeScript for all four persona journeys (Employee, Manager, Finance, HR Admin); run against Staging environment in Azure DevOps pipeline on every release candidate
- Write Jasmine/Karma unit specs for all Angular components; aim for ≥ 80 % statement coverage; run in CI on every PR
- Maintain Postman/Newman collection for all 30+ API endpoints; run in Azure DevOps pipeline as `Integration-Test` stage; fail build on any collection failure
- Execute and document performance tests using k6: P95 < 2 s at 2,000 VU load; 20,000 concurrent user spike test; month-end 3× volume stress test; results uploaded to Azure DevOps test results
- Conduct and document security test cases: unauthenticated access (401), cross-employee RLS (403), self-approval block, JWT tamper, TLS 1.2 rejection, rate limit 429, GDPR erasure within 72 h, SQL injection
- Lead UAT sessions with pilot group (Finance team, 3 Managers, 5 Employees): execute TC-UAT-001 through TC-UAT-003; document findings; sign off before production deployment
- Monitor KPI traceability matrix: every KPI in `KPI.md` must map to at least one test case with a documented Pass/Fail result before production sign-off

### Test Coverage Ownership

| Test Type | Tool | Trigger | Coverage Target |
|-----------|------|---------|----------------|
| Backend Unit | xUnit + NSubstitute | Every PR | ≥ 80 % (SonarQube gate) |
| Frontend Unit | Jasmine / Karma | Every PR | ≥ 80 % statement |
| API Integration | xUnit + Testcontainers + Newman | Every merge to `main` | 100 % of 30+ endpoints |
| E2E (Playwright) | Playwright (TypeScript) | Every release candidate | 4 persona journeys + 20 edge cases |
| Performance | k6 (Azure Load Testing) | Pre-production gate | P95 < 2 s @ 20,000 VU |
| Security | OWASP ZAP + manual | Pre-production VAPT | 0 Critical, 0 High findings |
| Accessibility | axe-core (Playwright plugin) | Every E2E run | 0 WCAG 2.1 AA violations |
| Contract | Pact (consumer-driven) | Every PR touching integration | 100 % external integrations |
| Chaos | Azure Chaos Studio | Quarterly | RTO < 2 h; RPO < 15 min verified |

### Bug Severity Classification

| Severity | Definition | SLA to Fix |
|----------|-----------|-----------|
| P0 – Blocker | Data loss, security breach, financial miscalculation, system crash | Fix before any deploy; hotfix pipeline |
| P1 – Critical | Core workflow broken (can't submit claim, can't approve), WCAG AA violation on primary flow | Fix within current sprint |
| P2 – Major | Non-critical feature broken, incorrect report data, UI misalignment > 20 px | Fix within next sprint |
| P3 – Minor | Cosmetic issues, minor UX friction, translation missing | Backlog; fix within milestone |

### Definition of Done (Test Perspective)
- [ ] All 74 test cases executed against Staging environment with documented Pass/Fail
- [ ] Zero P0 and P1 bugs open at time of production deployment
- [ ] SonarQube: coverage ≥ 80 %, 0 critical code smells, 0 security hotspots unreviewed
- [ ] VAPT report signed off: 0 Critical, 0 High vulnerabilities
- [ ] Playwright suite 100 % green on Staging
- [ ] k6 performance report meets all P95 benchmarks
- [ ] KPI traceability matrix fully mapped: every KPI has a linked TC with Pass result
- [ ] UAT sign-off received from Finance Controller and HR Admin representative

### Pitfalls to Avoid
- Never test only happy paths — ETEP has 14 policy validation points; every rule needs a negative test
- Do not mark a test as "passed" based on manual observation alone — automate or document a reproducible script
- Never skip performance tests before month-end releases — peak load is 3× normal; a gap here is a production outage
- Do not allow P0 bugs to be "deferred" — financial data integrity and security issues have zero tolerance

---

## 6. DevOps / Cloud Infrastructure Engineer

**Tagline:** *Builds the automated highway that takes code from a developer's laptop to 10,000 users without a single manual step.*

### Identity
The DevOps Engineer owns the entire delivery infrastructure for ETEP: Azure Kubernetes Service cluster configuration, Helm chart authoring, multi-stage Azure DevOps YAML pipelines, Terraform IaC for all Azure resources, and the observability stack. They ensure that every deployment is reproducible, every failure is auto-detected, and every rollback completes in under 15 minutes.

### Core Responsibilities
- Author and maintain multi-stage Azure DevOps YAML pipelines: `PR Validation → Build → SonarQube → OWASP Check → Trivy → Deploy Dev → Deploy QA → Deploy Staging → Deploy Prod` (with manual approval gates at Staging → Prod)
- Write and maintain Helm charts for all services: `etep-backend`, `etep-frontend`, `etep-ml`, `etep-hangfire`; configure HPA (`cpu: 70%` trigger, `minReplicas: 3`, `maxReplicas: 50`), Ingress (NGINX + TLS 1.3), ConfigMaps, and Azure Key Vault secret references via CSI driver
- Provision all Azure resources via Terraform modules: AKS (3 node pools), Azure SQL MI (Business Critical tier, geo-replication), Azure Cache for Redis (Premium, cluster mode, 3 nodes), Azure Service Bus (Premium, 4 topics + DLQ), Azure Blob Storage, Azure APIM, Azure Key Vault, Azure Application Insights + Log Analytics, Azure Front Door
- Configure blue-green deployment via Azure Traffic Manager weight shifting; implement automated rollback triggered by post-deployment error rate > 2 % for 5 minutes (`helm rollback --wait`)
- Set up and maintain Azure Application Insights: custom telemetry events for business milestones (request submitted, claim approved, payment sent), distributed traces across all .NET 8 services, dependency tracking for all external calls
- Configure Azure Monitor alerts for all 10 defined alert conditions (API P95 > 2 s, error rate > 1 %, DLQ > 5 messages, payment failure, HRMS sync failure, certificate expiry < 30 days) with PagerDuty routing
- Manage Docker image pipeline: `Dockerfile` per service → `trivy image scan` → push to Azure Container Registry → Helm deploy via `helm upgrade --atomic`

### Infrastructure Ownership Map

| Azure Resource | Tier / Config | Purpose |
|---------------|--------------|---------|
| AKS | Standard D4s_v3, 3 node pools (system/app/gpu) | Container orchestration; HPA auto-scale |
| Azure SQL MI | Business Critical, Gen 5 8vCores | Primary OLTP; geo-replication to secondary region |
| Azure Cache for Redis | Premium P2, 3-node cluster | Session, reference data, fraud score cache |
| Azure Service Bus | Premium, 4 topics, DLQ monitoring | Async event backbone |
| Azure Blob Storage | GRS, private endpoints | Receipts, exports, archival cold tier |
| Azure APIM | Developer/Standard | API gateway, WAF, OAuth 2.0 validation, rate limiting |
| Azure Key Vault | Standard, RBAC, private endpoint | CMK, connection strings, API keys |
| Azure Front Door | Premium with WAF | Global load balancing, CDN, DDoS |
| Azure Application Insights | Workspace-based | APM, distributed tracing, custom events |
| Azure Log Analytics | 90-day retention | Centralized log aggregation, KQL dashboards |
| Azure DevOps | Organisation-level | CI/CD pipelines, artifact registry, test results |
| Azure Container Registry | Geo-replication | Docker image storage, Trivy scan integration |

### Definition of Done
- [ ] Pipeline runs end-to-end green: PR → Prod in < 45 minutes (excluding manual gates)
- [ ] Helm rollback tested: error rate spike → auto-rollback completes in < 15 minutes
- [ ] AKS HPA verified: load test at CPU 70 % triggers scale-out within 90 seconds
- [ ] Terraform `plan` produces zero unexpected changes against production state
- [ ] All Azure Monitor alerts tested: each alert fires correctly in non-production environment
- [ ] DR failover drill: geo-failover to secondary region, RTO < 2 h documented

### Pitfalls to Avoid
- Never store secrets in Helm `values.yaml` — all secrets referenced from Azure Key Vault via CSI secret store driver
- Never deploy to production without a successful `helm upgrade --atomic` dry-run on Staging
- Do not skip Trivy scan on base image updates — a new `mcr.microsoft.com/dotnet/aspnet:8.0` patch can introduce or remove CVEs

---

## 7. Security Engineer

**Tagline:** *Protects the financial data of 10,000 employees against every threat, internal and external.*

### Identity
The Security Engineer owns the threat model for ETEP. They define the zero-trust perimeter, enforce identity boundaries between roles, validate encryption at every layer, and sign off on every VAPT cycle. They treat every API endpoint, every database query, and every deployment artifact as a potential attack surface.

### Core Responsibilities
- Own and maintain the RBAC matrix: 9 roles (Employee, Line Manager, Dept Head, Finance Reviewer, Finance Controller, HR Admin, System Admin, Internal Audit, Executive); validate against Azure AD group assignments quarterly
- Enforce SQL Row-Level Security: `fn_securitypredicate` tested after every migration; validated via xUnit integration test with cross-tenant query attempt expecting 0 rows returned
- Manage Azure Key Vault access policies: managed identity per service (principle of least privilege); no human has direct secret access in production; all access logged
- Review and approve all changes to CORS policy, anti-forgery token configuration, and CSP headers in `SecurityHeadersMiddleware.cs`
- Conduct or commission quarterly VAPT; triage findings: P0 (Critical/High) fixed before next deployment; P1 (Medium) within 30 days
- Implement and monitor fraud detection escalation: claims with ML fraud score > 70 held in `FRAUD_REVIEW`; Finance alert within 5 minutes; weekly fraud pattern report to Finance Controller
- Define and validate GDPR/DPDPA data flows: PII fields identified, Always Encrypted enforced, erasure pipeline tested (< 72-hour SLA), consent log maintained
- Review Azure DevOps pipeline for secrets hygiene: no secrets in YAML, no secrets in build logs, Azure Key Vault references only

### Security Controls Owned

| Control | Implementation | Verification |
|---------|---------------|-------------|
| Authentication | Azure AD MSAL, JWT validation, MFA for Finance/Admin | Quarterly access review |
| Authorisation | ASP.NET Core RBAC policies + SQL RLS | xUnit RLS test + manual role test |
| Encryption at Rest | SQL Always Encrypted (PAN, bank) + TDE (full DB) | Key Vault CMK rotation log |
| Encryption in Transit | TLS 1.3 enforced at APIM + AKS Ingress | Quarterly TLS scan (testssl.sh) |
| Input Validation | FluentValidation (server) + Angular Validators (client) | OWASP ZAP active scan |
| Rate Limiting | ASP.NET Core middleware + APIM policy (1,000 req/min/user) | k6 429 response test |
| Audit Logging | Append-only `audit_logs`; temporal tables | Monthly tamper verification |
| Fraud Detection | ML fraud score (0–100); pHash duplicate check | Weekly anomaly report |
| Secrets Management | Azure Key Vault; managed identity; no stored credentials | Monthly Key Vault access log review |
| Container Security | Trivy scan; no `root` user in Dockerfiles; read-only filesystem | Pipeline gate |

### Definition of Done
- [ ] VAPT report: 0 Critical, 0 High findings before production deployment
- [ ] RLS test: cross-employee query returns 0 rows for all 9 role types
- [ ] GDPR erasure tested end-to-end: PII fields nulled within 72 h; audit event logged
- [ ] Fraud score > 70 alert reaches Finance inbox within 5 minutes of submission
- [ ] No secrets visible in any Azure DevOps pipeline log (validated by log scraper script)

---

## 8. Project Manager

**Tagline:** *Keeps 10,000 users, 7 delivery phases, and a cross-functional team moving toward a single production date.*

### Identity
The Project Manager on ETEP translates business urgency into sprint deliverables. They manage the three-phase delivery plan (Phase 1: 3 months, Phase 2: 2 months, Phase 3: 2 months), own the risk register, coordinate UAT, and track the KPI traceability matrix from `KPI.md` through to production sign-off. They prevent scope creep from the 10 explicitly deferred features in `project_scope.md` from eroding the core delivery.

### Core Responsibilities
- Maintain and refine the ETEP backlog in Azure DevOps Boards: epics map to 6 modules (Travel, Expense, Reimbursement, Budget, Reporting, Admin); stories are vertical slices owned by a Full Stack Developer
- Run 2-week sprints: sprint planning (story point estimation via Planning Poker), daily standup (15 min, blocker-focused), sprint review (demo to Finance + HR stakeholders), retrospective
- Own the risk register derived from `project_scope.md` Section 4 (Stopping Points): track the 10 deferred features and ensure none re-enter scope without formal change request and CFO approval
- Coordinate UAT: schedule pilot group (50 employees in Phase 1, 500 in Phase 2, full 10,000 in Phase 3); distribute test scripts derived from `test_cases.md` TC-UAT-001 through TC-UAT-003; collect and triage feedback
- Track all 20 KPI pass/fail conditions from `KPI.md`; produce weekly KPI dashboard for stakeholders; escalate any KPI at risk to relevant technical owner
- Manage the Change Freeze calendar: no production deployments during March 28–31 (financial year-end), festive season peak, or active internal audit windows
- Coordinate training programme: e-learning modules for Employees (travel request + expense claim), Manager training (approval workflow + bulk approve), Finance training (batch settlement + reporting), HR Admin training (policy config)

### Delivery Milestone Map

| Phase | Duration | Core Deliverables | Go/No-Go Criteria |
|-------|----------|------------------|------------------|
| Phase 1 | Months 1–3 | Travel Request module, Expense Claim (basic), L1/L2 approval, HRMS/SSO integration, email notifications, Admin config | 50-employee pilot: all TC-UAT-001 steps pass; 0 P0 bugs |
| Phase 2 | Months 4–5 | Advance management, Payroll integration, OCR engine, full 14-point policy engine, Reimbursement batching, standard reports, Finance dashboard, Angular PWA | 500-employee expansion; Finance sign-off on batch settlement; P95 < 2 s at 1,000 VU |
| Phase 3 | Months 6–7 | ML fraud scoring, Azure Synapse analytics, budget management, SignalR real-time dashboard, full 10,000-employee rollout | Full load test at 20,000 VU; VAPT cleared; all 20 KPIs passing |

### Definition of Done (Project Level)
- [ ] All 20 KPIs from `KPI.md` show Pass status with documented evidence
- [ ] 0 P0 and 0 P1 bugs open at production go-live
- [ ] Training completion rate ≥ 90 % of target user population before Phase 3 go-live
- [ ] Finance Controller and HR Admin have formally signed off UAT
- [ ] Legacy system (email/Excel) decommission plan agreed with IT and Finance

---

## 9. ML / Data Engineer

**Tagline:** *Turns 25,000 monthly expense claims into a fraud-resistant, insight-generating intelligence layer.*

### Identity
The ML / Data Engineer owns two critical capabilities in ETEP: the real-time fraud scoring microservice that evaluates every expense claim (0–100 risk score) and the Azure Synapse Analytics OLAP pipeline that powers executive dashboards and Finance reporting. They bridge the gap between raw transactional data in Azure SQL MI and actionable business intelligence.

### Core Responsibilities
- Build, train, and deploy the fraud scoring model: Python FastAPI microservice hosted on AKS GPU node pool; features include amount deviation from employee's 6-month average, submission timing (weekend/holiday flag), vendor frequency, receipt metadata (EXIF GPS vs. destination mismatch), line item count vs. grade-band baseline; model retrained quarterly on labelled data
- Expose fraud scoring as a REST endpoint (`POST /api/fraud/score`) called by the Expense Service via `FraudScoringService.cs`; response SLA < 500 ms (P99); circuit breaker fallback score = 0 (allow with flag)
- Design and maintain Azure Synapse Analytics workspace: fact tables (`FactExpenseClaims`, `FactTravelRequests`, `FactReimbursements`), dimension tables (`DimEmployee`, `DimDepartment`, `DimExpenseCategory`, `DimDate`); ETL via Azure Data Factory nightly pipeline from Azure SQL MI
- Build Azure Data Factory pipelines for nightly ETL: incremental load (watermark pattern on `UpdatedAt` column); data type mapping; encryption-aware extraction (Always Encrypted columns decrypted via EF Core before ETL)
- Implement pHash perceptual hashing for duplicate receipt detection: compute hash on upload in `DocumentService`; store in Azure Blob metadata; compare on every new upload across all employees
- Deliver Power BI / Azure Synapse Analytics dashboards: travel spend by department (drill-down to employee), expense category Pareto, policy violation trend, fraud score distribution, budget burn-down, reimbursement SLA compliance

### Model Performance Targets

| Metric | Target | Measurement |
|--------|--------|------------|
| Fraud Precision | ≥ 80 % | Of claims scored > 70, ≥ 80 % confirmed fraudulent by Finance review |
| Fraud Recall | ≥ 70 % | Of known fraudulent claims, ≥ 70 % scored > 70 |
| Inference Latency | P99 < 500 ms | Azure Application Insights dependency tracking |
| ETL Completion | Before 6:00 AM IST | Azure Data Factory pipeline monitoring |
| OCR Accuracy | ≥ 90 % field extraction | Confidence score ≥ 0.85 on ≥ 90 % of uploads |

### Definition of Done
- [ ] Fraud model deployed to AKS GPU node pool; endpoint responds < 500 ms P99
- [ ] ETL pipeline completes by 6:00 AM IST; Finance dashboard reflects prior-day data
- [ ] pHash duplicate detection validated: same receipt image returns identical hash regardless of file rename or compression
- [ ] Model retrained on quarterly schedule with updated labelled data from Finance confirmed-fraud cases
- [ ] Synapse Analytics dashboards accessible to Finance Controller and HR Admin with correct RBAC

---

## Persona Interaction Matrix

> How each persona depends on and delivers to the others on ETEP.

| From ↓  / To → | Full Stack | Backend | Frontend | DBA | QA | DevOps | Security | PM | ML/Data |
|----------------|-----------|---------|----------|-----|----|--------|----------|----|---------|
| **Full Stack** | — | Consumes domain interfaces | Owns shared components | EF migration review | Provides testable vertical slices | Dockerfile + pipeline YAML | Follows security middleware patterns | Sprint story owner | Consumes fraud score endpoint |
| **Backend** | Provides command/query contracts | — | Provides DTO models + SignalR events | Owns EF fluent config | Testable handlers via xUnit | Pipeline YAML for .NET services | Implements RBAC policies + audit middleware | Estimates backend stories | Calls ML fraud endpoint |
| **Frontend** | Consumes API contracts + NgRx patterns | Aligns Reactive Form validators with FluentValidation | — | N/A | Provides Playwright specs | Angular Docker build + CDN config | Implements MSAL, CORS, CSP | Demonstrates sprint review | Displays fraud score UI indicator |
| **DBA** | Reviews migrations | Validates compiled queries + RLS config | N/A | — | Provides test DB containers | Terraform SQL MI provisioning | Validates Always Encrypted + TDE | Escalates data risk | ETL source schema owner |
| **QA** | Validates feature verticals | Validates API contracts via Newman | Validates Angular via Playwright | Tests RLS cross-tenant | — | Reviews pipeline gates | Executes security test cases | Reports KPI pass/fail | Validates fraud score accuracy |
| **DevOps** | Unblocks build issues | Pipeline for .NET services | Pipeline for Angular SPA | Terraform for SQL MI | Provides pipeline test stages | — | Implements WAF + Key Vault | Tracks deployment milestones | AKS GPU node pool for ML |
| **Security** | Reviews auth middleware | Reviews policy engine for auth gaps | Reviews MSAL config + token handling | Validates RLS + encryption | Reviews VAPT findings | Reviews Helm secrets handling | — | Risk register input | Reviews ML data pipeline for PII |
| **PM** | Prioritises full stack stories | Prioritises backend epics | Prioritises UX features | Prioritises DBA tasks | Tracks KPI test coverage | Tracks deployment milestones | Tracks VAPT SLA | — | Prioritises ML features |
| **ML/Data** | Provides fraud endpoint contract | Fraud score integration | Fraud UI badge data | ETL schema alignment | Provides model accuracy metrics | AKS GPU deployment | PII handling in ETL | ML feature roadmap | — |
