# Project_scope.md — Enterprise Travel & Expense Management System (ETEMS)

## Goal & Overview
Deliver a cloud-native, centralized Enterprise Travel & Expense Management System automating travel request → approval → expense claim → receipt capture → policy validation → reimbursement for 10,000+ employees, eliminating manual processes and providing real-time spend visibility.

---

## Tech Stack Details
- **Backend:** .NET Core 8 / C# 12, Entity Framework Core 8
- **Frontend:** Angular 17, TypeScript 5, RxJS 7, NgRx 17
- **Database:** Azure SQL Database (PostgreSQL 14 compatible)
- **Cloud (Azure):** App Service, Blob Storage, Service Bus, Key Vault, Application Insights, Front Door
- **AI Services:** Azure Form Recognizer (OCR)
- **Auth:** Azure Active Directory (AAD) SSO, JWT
- **Testing:** xUnit, contract tests (OpenAPI/AsyncAPI)
- **Monitoring:** Serilog (structured JSON), Azure Monitor

---

## Core Features & Requirements
1. **Travel Request Management** — structured digital requests with multi-level approval routing, budget validation, and booking confirmation attachment
2. **Expense Claim Management** — itemized claims with receipt upload, OCR auto-population, duplicate detection, and policy validation
3. **Automated Approval Workflow Engine** — configurable multi-level chains (by amount/department/cost center), SLA timers, auto-escalation, out-of-office delegation
4. **Policy Engine** — configurable rules (hotel caps, per-diem rates, travel class, meal allowances) with versioning and inline violation warnings
5. **Finance & Reimbursement Module** — batch/on-demand reimbursement processing, ERP export (CSV/XML), partial approvals
6. **Reporting & Analytics** — real-time dashboards (employee, manager, finance, executive) with Excel/CSV export
7. **Notifications** — email + in-app alerts at every lifecycle stage, SLA reminders, action links for approval
8. **Audit Trail** — immutable append-only log of all state changes, config changes, and user actions

---

## Design System & UX Standards
- **Design System:** Material Design 3 (Angular Material 17)
- **Responsive:** Desktop-first with progressive enhancement for tablets (PWA for mobile)
- **Accessibility:** WCAG 2.1 AA compliant (keyboard navigable, screen reader compatible)
- **UX Patterns:** Stepped forms (travel/expense), dashboard cards, inline validation with badges, modal confirmations for destructive actions
- **Feedback:** Toast notifications for async operations, skeleton loaders for reports, progress indicators for OCR/batch jobs

---

## In-Scope Development Portion

### Backend (API + Services)
- Authentication/authorization middleware (AAD + role-based guards)
- Travel request CRUD + approval/rejection/send-back endpoints
- Expense claim CRUD + line items + receipt upload + OCR integration
- Policy engine (rule configuration + inline validation endpoint)
- Approval workflow resolver (hierarchy + escalation + delegation)
- Duplicate detection service (90-day rolling window)
- Reimbursement batch processor (scheduled + on-demand) + ERP formatter
- Audit log service (immutable + searchable export)
- Reporting endpoints (spend summary, policy violations, budget utilization, audit log)
- Notification dispatcher (Service Bus + email/in-app delivery)
- Health check endpoints (`/health/live`, `/health/ready`)

### Frontend (Angular Modules)
- Authentication guard + MSAL integration + role-based routing
- Travel request module (new request, list, detail, approval actions)
- Expense claim module (new claim, line item entry, receipt upload with OCR preview, duplicate warning modal)
- Approval queue component (team + department views, action via email deep-link)
- Finance module (reimbursement queue, batch trigger, export download)
- Admin module (policy rule config, approval hierarchy drag-drop, employee profile)
- Reporting module (spend dashboard charts, export buttons)
- Notification center (in-app + email action links)

### Azure Infrastructure (IaC-ready)
- App Service (auto-scale: 2–10 instances, 70% CPU trigger)
- SQL Database (private endpoint, hourly backups, geo-redundant optional)
- Blob Storage (receipts + reimbursement exports containers, managed identity access)
- Service Bus (topics: travel-request-events, expense-claim-events)
- Key Vault (secrets: DB strings, Service Bus keys, Form Recognizer keys)
- Application Insights (distributed tracing, availability tests, P1 alerts)
- WebJob (reimbursement batch runner, isolated from API)

---

## Functional + Non-Functional Requirements

### Functional (Core)
| ID | Requirement |
|---|---|
| FR-01 | AAD SSO authentication + role-based access (Employee, Manager, Dept Head, Finance Reviewer, Finance Admin, HR Admin, Exec, Sys Admin) |
| FR-02 | Travel request: destination, dates, purpose, type, estimated cost, cost center; budget validation at submission |
| FR-03 | Multi-level approval routing (configurable by dept + cost threshold) with approve/reject/return actions |
| FR-04 | Expense claim: line items (category, date, currency, amount, description, receipt) + optional travel request linkage |
| FR-05 | Receipt OCR via Form Recognizer (JPEG, PNG, PDF, ≤10MB); auto-populates amount/date/vendor |
| FR-06 | Duplicate detection: same employee + date + vendor + amount within 90 days → warning + reviewer override with justification |
| FR-07 | Policy engine: hotel caps, per-diem rates, travel class, meal allowances; violations flagged inline; approver acknowledgment required |
| FR-08 | SLA timers per approval level (default 48 hrs) → auto-escalation to deputy |
| FR-09 | Reimbursement batch (daily/weekly/on-demand) → export file (CSV/XML) → Blob Storage → ERP pickup |
| FR-10 | Audit trail: immutable log with actor, action, timestamp, prev/new state, IP address; searchable + exportable by admin roles |

### Non-Functional (with KPI constraints)
| ID | Requirement | KPI Constraint |
|---|---|---|
| NFR-01 | API response time (P95) | ≤ 300ms (KPI-PF-01) |
| NFR-02 | Page load time | < 2 seconds (KPI-PF-02) |
| NFR-03 | Approval workflow execution | < 5 seconds (KPI-PF-03) |
| NFR-04 | Concurrent user support | ≥ 5,000 concurrent (KPI-PF-04) |
| NFR-05 | Transaction throughput | ≥ 100 TPS (KPI-PF-05) |
| NFR-06 | Platform availability | ≥ 99.95% (KPI-OP-03) |
| NFR-07 | Travel approval SLA | ≤ 24 hours (KPI-OP-01) |
| NFR-08 | Expense settlement time | ≤ 3 business days (KPI-OP-02) |
| NFR-09 | Workflow success rate (no manual intervention) | ≥ 99% (KPI-OP-04) |
| NFR-10 | User adoption rate | ≥ 90% (KPI-OP-05) |
| NFR-11 | MTTD | < 48 hours (KPI-RL-01) |
| NFR-12 | MTTR | < 4 hours (KPI-RL-02) |
| NFR-13 | Incident resolution time | < 24 hours (KPI-RL-03) |
| NFR-14 | RPO | ≤ 15 minutes (KPI-DR-01) |
| NFR-15 | RTO | ≤ 60 minutes (KPI-DR-02) |
| NFR-16 | Code test coverage | ≥ 80% (KPI-QT-01) |
| NFR-17 | Contract test compliance | 100% APIs covered (KPI-QT-02) |
| NFR-18 | Security review completion | 100% before release (KPI-QT-03) |

**Governance Constraints (Zero tolerance):**
- Critical decisions without ADR: 0 (KPI-GV-01)
- Undocumented API contracts: 0 (KPI-GV-02)
- NFR compliance before release: 100% (KPI-GV-03)
- Architecture review gate compliance: 100% releases (KPI-GV-04)
- Production change approval rate: 100% (KPI-GV-05)

---

## KPIs Points with Constraints
All operational, governance, reliability, performance, DR, and quality KPIs from `KPI.md` are accepted as binding success criteria. Key constraints enforced at release gates:
- API P95 < 300ms (failure blocks release)
- Availability ≥ 99.95% (monthly SLA)
- Zero tolerance for undocumented API contracts or missing ADRs
- Load test must validate ≥ 5,000 concurrent users before production deployment
- DR drill quarterly validating RPO ≤ 15 min, RTO ≤ 60 min
- Code coverage < 80% fails CI pipeline

---

## Stopping Point
**Stop condition:** Completion and sign-off of v1.0 MVP delivering all functional requirements (FR-01 to FR-10) with:
- All governance KPIs (GV-01 to GV-05) satisfied
- Performance KPIs (PF-01 to PF-05) validated via load test report
- Security review (QT-03) signed off by Architecture Review Board
- DR drill executed with RPO/RTO within limits
- No outstanding P1 or P2 bugs
- User acceptance testing completed with ≥ 90% success rate on core flows (travel request → approval → expense claim → reimbursement)

**Deferred to v2.0:** GDS integration, native mobile apps, ML fraud detection, corporate credit card feed, real-time currency rates, multi-tenant support.

---

## Out of Scope (v1.0)
- Direct airline/hotel booking (GDS integration)
- Native iOS/Android mobile apps (PWA only)
- Automated bank transfer (export file only; payment execution external)
- Multi-tenant / multi-company support
- ML-based anomaly/fraud detection (rule-based only)
- Corporate credit card feed / automated transaction import
- Real-time live currency rate feed (manual rate configuration by Finance Admin)
- Automatic HR system org chart sync (manual hierarchy maintenance)
- Cross-region geo-redundancy (single Azure region, upgrade path available)
- Real-time two-way ERP sync (export-file-based with optional webhook acknowledgment)