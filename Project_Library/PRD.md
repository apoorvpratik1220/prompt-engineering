# Product Requirements Document (PRD)
## Enterprise Employee Travel & Expense Management System (ETEMS)

**Version:** 2.0
**Author:** Senior Technical Architect
**Persona Reference:** Senior_Architect_Personas.md
**Date:** June 2026
**Status:** Draft — Pending Stakeholder Review

---

## 1. Problem Statement

### Organization Context
* Enterprise with 10,000+ employees operating across multiple geographic locations
* Employees travel for client meetings, training programs, audits, conferences, and internal business activities
* No centralized digital system exists for travel or expense lifecycle management

### Current State — Root Causes
* Travel requests raised via email — no structured submission, no audit trail, no SLA
* Approvals managed over phone calls and email threads — approvers miss or lose requests
* Expense claims submitted as Excel sheets — no validation, no policy enforcement, no duplicate check
* Receipts submitted as physical paper — lost, damaged, or missing at reconciliation time
* Reimbursements batch-processed manually — employees wait weeks with no visibility

### Business Pain Points
* **Zero visibility** — management has no real-time view of travel spend, request pipeline, or budget consumption
* **Policy non-compliance** — no automated enforcement of hotel caps, per-diem rates, or travel class rules
* **Audit exposure** — no immutable record of who approved what, when, and under which policy version
* **Reconciliation overhead** — finance teams spend disproportionate effort reconciling paper receipts against spreadsheets
* **Reimbursement delays** — manual validation and batch processing create multi-week payout cycles
* **Duplicate claims** — no system-level detection of duplicate submissions across dates, vendors, or amounts
* **Budget overruns** — no real-time departmental or cost-center spend tracking against approved budgets
* **Scalability ceiling** — manual process cannot scale with organizational growth beyond current headcount

### Impact
* Operational inefficiency compounding across 10,000+ employee workforce
* Financial risk from uncontrolled, un-audited travel spend at enterprise scale
* Employee dissatisfaction driving policy workarounds and shadow processes
* Compliance and regulatory exposure during internal and external audits

---

## 2. Solution Overview

### Product Vision
* Deliver a centralized, cloud-native **Enterprise Employee Travel & Expense Management System (ETEMS)**
* Automate the full lifecycle: travel request → multi-level approval → expense claim → receipt capture → policy validation → reimbursement
* Enforce policy programmatically — eliminate manual policy checking from every workflow step
* Provide real-time dashboards for employees, approvers, finance, and executive stakeholders

### Solution Pillars
* **Travel Request Management** — structured digital requests with multi-level configurable approval routing
* **Expense Claim Management** — itemized claims with receipt upload, OCR auto-extraction, and inline policy validation
* **Automated Approval Workflows** — configurable chains based on amount, department, travel type, and cost center
* **Policy Engine** — enforce travel class limits, hotel caps, per-diem rates, advance booking windows, and budget thresholds
* **Finance & Reimbursement Module** — batch or on-demand processing with ERP/payroll export hooks
* **Reporting & Analytics** — real-time spend dashboards, budget utilization, policy violation reports, and compliance audit trail
* **Notifications** — email and in-app alerts at every lifecycle stage with approver action links

---

### 2A. Functional Requirements

#### FR-01: User Authentication & Authorization
* Authenticate via Azure Active Directory (AAD) Single Sign-On — no separate credential store
* Role-based access: Employee, Line Manager, Department Head, Finance Reviewer, Finance Admin, HR Admin, Executive, System Admin
* Session timeout enforced after 30 minutes of inactivity
* Role assignments managed via Azure AD groups — no in-app role assignment for core roles
* All protected routes guarded at both API and Angular UI layers

#### FR-02: Travel Request Management
* Employee creates travel request with: destination, travel dates, purpose, travel type, estimated cost, cost center
* System validates budget availability against cost center at submission time
* Multi-level approval routing configurable per department and cost threshold
* Approver actions: approve, reject, or return-with-comments at each routing level
* Travel reference number auto-generated on full approval chain completion
* Employee attaches booking confirmation document post-approval for record linkage

#### FR-03: Expense Claim Management
* Employee raises expense claim — linked to approved travel request or standalone for local/ad-hoc expenses
* Claim contains one or more line items: category, date, currency, amount, description, and receipt
* Receipt upload supports JPEG, PNG, PDF; max 10MB per file; stored in Azure Blob Storage
* OCR auto-populates amount, date, and vendor from receipt image via Azure Form Recognizer
* Employee saves draft claims and returns to complete — locked after submission
* Duplicate detection runs at submission: same employee + date + vendor + amount flags for reviewer override

#### FR-04: Policy Engine
* Finance Admin configures rules: hotel nightly limits by city tier, per-diem rates by destination, meal allowances, travel class rules by trip duration, advance booking windows
* Policy validation runs on every line item at save and at final submission
* Violations displayed as inline warnings — does not block submission but requires explicit approver acknowledgment
* Policy version history maintained — each claim validated against the policy active at time of submission

#### FR-05: Approval Workflow Engine
* Configurable multi-level approval chains per department, cost center, and amount range
* SLA timers per approval level — auto-escalation to deputy approver on breach
* Out-of-office delegation — approver nominates deputy; routing switches automatically during delegate period
* Approvers act via email action links (approve/reject) without requiring portal login for simple cases
* Full approval history with actor, timestamp, and comments visible on every request and claim record

#### FR-06: Notifications
* Email notifications triggered at every lifecycle event: submission, approval, rejection, return, reimbursement payout
* In-app notification center with unread count badge and direct action links
* Configurable SLA reminder notifications dispatched to approvers at defined interval before breach
* Finance Admin broadcasts system-wide announcements via notification center

#### FR-07: Finance & Reimbursement
* Finance reviewer validates approved claims before triggering reimbursement batch
* Partial approval: individual line items approved or rejected independently within a single claim
* Reimbursement batch run configurable: daily, weekly, or on-demand trigger by Finance Admin
* Reimbursement export file generated in standard format for ERP / payroll system ingestion
* Employee views reimbursement status, expected payout date, and line-item breakdown in self-service portal

#### FR-08: Reporting & Analytics
* Employee view: personal spend history, open request/claim pipeline, reimbursement tracker
* Manager view: team spend summary, pending approvals, budget utilization by cost center
* Finance view: spend by department/period, policy violation report, reimbursement queue, audit export
* Executive view: organization-wide spend dashboard, top cost centers, trend analysis by period
* All reports exportable to Excel/CSV with applied filter state preserved in export

#### FR-09: Audit Trail
* Immutable append-only log of every state change, user action, approval, and configuration change
* Every audit entry: actor (userId + name), action type, timestamp, previous state, new state, IP address
* Audit trail searchable by: date range, actor, request/claim ID, action type, and cost center
* Audit export restricted to HR Admin and Finance Admin roles — not accessible to general employees

#### FR-10: Administration
* System Admin manages: user roles, approval hierarchy mappings, department configurations
* Finance Admin manages: policy rules, exchange rates, cost center budgets, reimbursement batch schedules
* HR Admin manages: employee profiles, delegation rules, org structure updates, data export/anonymization requests
* All admin actions logged in audit trail with same immutability guarantees as transactional records

---

### 2B. Non-Functional Requirements

#### NFR-01: Performance
* API read endpoint P95 response time ≤ 500ms under normal load (up to 2,000 concurrent users)
* Dashboard report generation ≤ 3 seconds for datasets up to 100,000 records
* Receipt OCR processing completion ≤ 10 seconds per upload
* Batch reimbursement run completes within 30 minutes for up to 5,000 claims per cycle

#### NFR-02: Availability
* System uptime: 99.9% monthly SLA — planned maintenance excluded
* Planned maintenance only during off-peak windows: weekends, 2 AM–5 AM local time
* Azure Availability Zones for critical services — no single point of failure on API or database tier
* Recovery Point Objective (RPO): ≤ 1 hour — automated Azure SQL backups every hour
* Recovery Time Objective (RTO): ≤ 4 hours — documented and quarterly-tested restore runbook

#### NFR-03: Scalability
* Horizontal scaling via Azure App Service auto-scale — triggered at 70% CPU or 80% memory utilization
* Stateless API layer — no server-side session state; all session context carried in JWT claims
* Azure Blob Storage for receipt documents — scales independently from application and database tiers
* Database connection pooling via Entity Framework Core — pool size tuned per environment tier

#### NFR-04: Security
* All data in transit encrypted via TLS 1.2+ — enforced at Azure Front Door and App Service level
* All data at rest encrypted via Azure Storage Service Encryption and SQL Server Transparent Data Encryption (TDE)
* JWT access tokens expire in 1 hour — refresh token rotation enforced on every use
* No PII in URL query parameters or server-side log entries — redaction enforced in logging middleware
* Role enforcement at API gateway and service layer — defense in depth; not controller-only
* SQL injection prevention via EF Core parameterized queries — no raw dynamic SQL in application code
* All secrets, connection strings, and API keys stored in Azure Key Vault — zero config-file secrets
* Threat model review required at architecture phase before each major feature set — not post-implementation

#### NFR-05: Observability (per Architect Persona — first-class requirement)
* Structured JSON logging via Serilog — every log entry includes: traceId, userId, correlationId, timestamp, severity
* Distributed tracing via Azure Application Insights — end-to-end request visibility across API and background job layers
* Health check endpoints: `/health/live` and `/health/ready` — monitored by Azure Monitor availability tests
* P1 alerts (system down, batch failure, auth spike) routed to on-call within 2 minutes via Azure Monitor alert rules
* Real-time Azure Monitor dashboard: API latency percentiles, error rate, active user count, queue depth

#### NFR-06: Maintainability
* Code coverage ≥ 80% on business logic and service layers enforced via xUnit in CI pipeline gate
* All PRs require passing build + unit + integration tests before merge — no exceptions
* ADR documented for every architectural decision — stored in `/docs/adr/` within the repository
* All API endpoints documented in OpenAPI spec — auto-generated via Swashbuckle, verified in CI
* No hardcoded configuration values — all environment-specific config via Azure App Configuration and Key Vault

#### NFR-07: Compliance & Data Governance
* Financial records retained for 7 years — soft delete only; no hard deletes on claim or request data
* Soft delete pattern: `IsDeleted`, `DeletedAt`, `DeletedBy` on all financial entity tables
* GDPR-compatible: employee data export and anonymization on HR request within 30-day SLA
* All PII fields identified, tagged, and documented in the system data dictionary

#### NFR-08: Accessibility
* Angular frontend compliant with WCAG 2.1 AA standards across all employee-facing flows
* All forms fully keyboard navigable — no mouse-only interaction patterns permitted
* Screen reader compatibility verified on: raise claim, submit claim, approval action, reimbursement status flows

---

## 3. User Flow

### 3.1 Travel Request Flow

#### Steps
* Employee authenticates via Azure AD SSO — role resolved from AAD group membership
* Employee navigates to Travel Request module — clicks "New Request"
* Employee completes structured form: destination, travel dates, purpose, travel type, estimated cost, cost center
* On form save — Policy Engine validates estimated cost and travel type against active rules; violations shown inline
* Employee reviews flagged violations, acknowledges, and submits
* System creates TravelRequest record — status: `Submitted`; approval chain resolved from workflow config
* Level-1 approver (Line Manager) receives email notification with approve/reject action links
* Approver acts via email link or portal — logs action with comments; record updated to `L1-Approved` or `Rejected`
* If cost exceeds threshold → routed to Level-2 (Department Head); same notification and action pattern
* If cost exceeds executive threshold → routed to Level-3 (Finance Head)
* On full chain approval — status set to `Approved`; travel reference number generated and emailed to employee
* Employee proceeds with external booking; uploads booking confirmation document against the request
* If rejected at any level — employee notified with rejection reason; can revise and resubmit

---

#### Technical Specification — Travel Request Flow

##### Backend (.NET Core / C# / Entity Framework Core / SQL Server)
* `TravelRequestsController` — POST `/api/v1/travel-requests`; PUT `/api/v1/travel-requests/{id}/approve|reject|send-back`
* `TravelRequestService` — orchestrates: validate policy → persist entity → resolve approval chain → dispatch notification event
* `PolicyValidationService` — loads active `PolicyRule` entities for cost center and travel type; returns violation list
* `ApprovalChainResolver` — queries `ApprovalHierarchy` config table by department + cost threshold; returns ordered approver list
* `TravelRequest` entity: `Id`, `EmployeeId`, `Destination`, `TravelDates` (value object), `Purpose`, `TravelType` (enum), `EstimatedCost`, `CostCenterId`, `Status` (enum), `ReferenceNumber`, `RowVersion` (optimistic concurrency)
* EF Core migration: indexes on `EmployeeId`, `Status`, `CostCenterId`; FK on `CostCenterId` indexed; `RowVersion` column for concurrency
* `AuditLogService` — called on every status transition; writes immutable `AuditLog` record with actor + previous/new state
* Azure Service Bus topic — `travel-request-events` — published on submit and every status change; notification worker subscribes

##### Frontend (Angular)
* `TravelRequestModule` — lazy-loaded; route-guarded by `AuthGuard` + `RoleGuard(['Employee', 'Manager'])`
* `NewTravelRequestComponent` — Angular Reactive Form; validators: required fields, date range (departure before return), positive cost
* `PolicyValidationService` (Angular) — calls `POST /api/v1/policies/validate` on form blur for estimated cost field; displays inline violation badges
* `TravelRequestFacade` (NgRx) — actions: `submitRequest`, `loadRequestById`, `approveRequest`; effects call API; state updates trigger UI
* Approver email action links — deep-link to `ApprovalActionComponent`; MSAL silent token refresh on landing; one-click approve/reject with mandatory comment for reject

##### Azure Infrastructure
* Azure App Service hosts API — auto-scale configured; min 2 instances for availability
* Azure SQL Database — TravelRequests table with private endpoint; no public access
* Azure Service Bus — Standard tier; `travel-request-events` topic with `notification-subscription` and `audit-subscription`
* Azure Key Vault — DB connection string, Service Bus connection string stored as secrets; App Service uses Managed Identity to access

---

### 3.2 Expense Claim Flow

#### Steps
* Employee navigates to Expense Claim module — selects "New Claim" — optionally links to an approved Travel Request
* Employee adds line items one by one: category, date, currency, amount, description
* Employee uploads receipt per line item — file validated (type, size); OCR auto-extracts amount, date, vendor; pre-fills form fields
* Employee reviews OCR-extracted values — corrects if needed — saves line item
* Policy Engine validates each line item on save — flags violations inline; employee sees warning badge per flagged item
* On submission — system runs duplicate detection across employee's prior claims for same date + vendor + amount
* Duplicate detected → employee shown warning; must confirm unique before proceeding
* Claim submitted — status: `Submitted`; routed to approval chain per same hierarchy as travel requests
* Approver reviews claim, line items, receipts, and violation flags — must explicitly acknowledge each flagged violation
* Finance Reviewer validates supporting documents and amounts before marking `Finance-Approved`
* Reimbursement record created — status: `Pending-Payment`; included in next batch run
* Employee notified of approval, rejection, or partial approval at each stage

---

#### Technical Specification — Expense Claim Flow

##### Backend (.NET Core / C# / Entity Framework Core / SQL Server)
* `ExpenseClaimsController` — POST `/api/v1/expense-claims`; POST `/api/v1/expense-claims/{id}/line-items`; POST `/api/v1/expense-claims/{id}/receipts`; PUT `/api/v1/expense-claims/{id}/submit|approve|reject`
* `ExpenseClaimService` — orchestrates: validate line items → run duplicate check → persist → resolve approval chain → publish event
* `DuplicateDetectionService` — queries `ExpenseLineItem` table: same `EmployeeId` + `Date` + `Amount` + `VendorName` within rolling 90-day window; returns match list for reviewer
* `ReceiptProcessingService` — uploads blob to Azure Blob Storage; calls Azure Form Recognizer; maps extracted fields to `ReceiptExtractionResult`; stores result against line item
* `ExpenseClaim` entity: `Id`, `TravelRequestId` (nullable FK), `EmployeeId`, `TotalAmount`, `Currency`, `Status`, `SubmittedAt`, `IsDeleted`, `RowVersion`
* `ExpenseLineItem` entity: `Id`, `ClaimId` (FK), `Category` (enum), `Amount`, `Currency`, `Date`, `Description`, `ReceiptBlobUri`, `PolicyViolationFlag`, `VendorName`
* EF Core: composite index on (`EmployeeId`, `Date`, `Amount`) for duplicate detection query performance
* `ReimbursementService` — creates `Reimbursement` record on Finance approval; status: `Pending-Payment`; added to batch queue

##### Frontend (Angular)
* `ExpenseClaimModule` — lazy-loaded; `NewClaimComponent` renders line item entry panel with receipt upload slot
* `ReceiptUploadComponent` — drag-and-drop zone; file type/size validation client-side before API call; shows OCR-extracted values in editable preview card
* `LineItemFormComponent` — reactive form per line item; policy violation badge shown inline if `policyViolationFlag` returned from save response
* `DuplicateWarningComponent` — modal shown on submit if duplicate detected; employee must check confirmation checkbox before submit proceeds
* `ClaimDetailComponent` — approver view shows each line item with receipt thumbnail, amount, category, and violation badge; acknowledge checkbox per flagged item required before approve action enabled

##### Azure Infrastructure
* Azure Blob Storage — `receipts` container (private); App Service accesses via Managed Identity; receipts referenced by URI stored in `ExpenseLineItem`
* Azure Cognitive Services (Form Recognizer) — called synchronously from `ReceiptProcessingService`; timeout: 15 seconds; failure falls back to manual entry flag
* Azure Service Bus — `expense-claim-events` topic; notification worker and audit worker subscribe independently

---

### 3.3 Finance & Reimbursement Flow

#### Steps
* Finance Reviewer accesses reimbursement queue dashboard — filtered by status: `Finance-Approved` and `Pending-Payment`
* Finance Reviewer validates claim documents — approves individual line items or rejects with reason
* Partial rejection: approved line items proceed; rejected line items returned to employee for re-raise as separate claim
* Finance Admin triggers reimbursement batch run — on-demand or scheduled (daily/weekly)
* System generates reimbursement export file (CSV/XML) per configured ERP format
* Export file pushed to Azure Blob Storage — ERP system picks up via SFTP or API integration hook
* Employee notified of payout with amount breakdown and expected payment date
* Reimbursement record updated to `Paid` on ERP confirmation callback or manual Finance Admin confirmation

---

#### Technical Specification — Finance & Reimbursement Flow

##### Backend (.NET Core / C# / Entity Framework Core / SQL Server)
* `ReimbursementsController` — GET `/api/v1/reimbursements`; POST `/api/v1/reimbursements/process`; GET `/api/v1/reimbursements/{id}`
* `ReimbursementBatchService` — queries all `Reimbursement` records in `Pending-Payment` status; groups by employee; generates export payload; writes to Blob Storage; updates status to `Exported`
* `ExportFormatterService` — pluggable formatter per ERP format (CSV, XML, JSON); format configured per environment via Azure App Configuration
* Azure WebJob (or Hangfire job) — `ReimbursementBatchJob`; triggered on schedule or on-demand via Service Bus message from Finance Admin action
* `Reimbursement` entity: `Id`, `ClaimId` (FK), `EmployeeId`, `TotalAmount`, `Status`, `BatchRunId`, `ExportedAt`, `PaidAt`

##### Frontend (Angular)
* `FinanceModule` — lazy-loaded; role-guarded for `FinanceReviewer` and `FinanceAdmin` roles only
* `ReimbursementQueueComponent` — tabbed view: Pending Review / Pending Payment / Exported / Paid; sortable by amount, date, department
* `BatchRunComponent` — Finance Admin initiates batch; shows last run timestamp, record count, and export download link
* `SpendDashboardComponent` — ngx-charts bar/line charts for spend by department and period; data loaded from `/api/v1/reports/spend-summary`

##### Azure Infrastructure
* Azure WebJobs — hosts `ReimbursementBatchJob`; isolated from API process; triggered via Azure Service Bus queue message
* Azure Blob Storage — `reimbursement-exports` container; ERP integration reads from here via SFTP adapter or Azure Logic App trigger

---

### 3.4 Admin Configuration Flow

#### Steps
* Finance Admin logs in — navigates to Policy Configuration module
* Creates or edits policy rules: per-diem rate by destination, hotel cap by city tier, travel class by duration, meal allowance by category
* System validates rule set for conflicts — overlapping rules flagged before save
* Policy saved with effective date — existing in-flight claims continue to validate against policy active at their submission timestamp
* System Admin configures approval hierarchy — maps departments to approver roles, sets cost thresholds per level, assigns deputies
* HR Admin updates employee profile — sets manager mapping, cost center, delegation rules

---

#### Technical Specification — Admin Flow

##### Backend
* `PoliciesController` — GET `/api/v1/policies`; POST `/api/v1/policies`; POST `/api/v1/policies/validate`
* `ApprovalHierarchyController` — GET/POST `/api/v1/approval-hierarchy`
* `PolicyRule` entity: `Id`, `RuleType` (enum), `Scope`, `Limit`, `Currency`, `EffectiveFrom`, `EffectiveTo`; versioned by date range
* `ApprovalHierarchy` entity: `Id`, `DepartmentId`, `Level`, `ApproverId`, `CostThresholdMin`, `CostThresholdMax`, `DeputyApproverId`
* Admin actions publish to audit log immediately — same `AuditLogService` used across all domains

##### Frontend (Angular)
* `AdminModule` — lazy-loaded; route-guarded to `SystemAdmin`, `FinanceAdmin`, `HRAdmin` roles
* `PolicyConfigComponent` — table view of active rules; inline edit mode; conflict detection shown as warning before save
* `ApprovalHierarchyComponent` — drag-and-reorder approval levels per department; deputy assignment dropdown per level

---

## 4. API Design

### Design Principles (per Architect Persona — ADR-002)
* RESTful, versioned APIs — all routes prefixed `/api/v1/`
* OpenAPI / Swagger spec mandatory and auto-generated before any endpoint is merged
* JWT Bearer token (Azure AD) — every endpoint authenticated; no anonymous endpoints except health checks
* Standardized response envelope: `{ success: bool, data: T, errors: [], traceId: string }`
* Idempotent POST operations via client-generated `X-Idempotency-Key` header on claim and request submission
* Rate limiting: 100 requests/minute per authenticated user via ASP.NET Core rate limiting middleware
* Global exception middleware: catches unhandled exceptions; returns standardized error; logs with traceId; never leaks stack traces

### Travel Request Endpoints
* `POST /api/v1/travel-requests` — create and submit travel request
* `GET /api/v1/travel-requests/{id}` — fetch by ID with current status and approval history
* `GET /api/v1/travel-requests?employeeId=&status=&page=&pageSize=` — paginated, filterable list
* `PUT /api/v1/travel-requests/{id}/approve` — approver approves with optional comments
* `PUT /api/v1/travel-requests/{id}/reject` — approver rejects with mandatory reason field
* `PUT /api/v1/travel-requests/{id}/send-back` — return to employee for correction with comments
* `POST /api/v1/travel-requests/{id}/booking-confirmation` — attach booking document (multipart)

### Expense Claim Endpoints
* `POST /api/v1/expense-claims` — create new claim (linked or standalone)
* `GET /api/v1/expense-claims/{id}` — fetch with line items, receipts, and violation flags
* `POST /api/v1/expense-claims/{id}/line-items` — add line item
* `DELETE /api/v1/expense-claims/{id}/line-items/{lineItemId}` — remove line item (draft only)
* `PUT /api/v1/expense-claims/{id}/submit` — finalize and submit for approval
* `PUT /api/v1/expense-claims/{id}/approve` — approver/finance approve
* `PUT /api/v1/expense-claims/{id}/reject` — reject with mandatory reason
* `POST /api/v1/expense-claims/{id}/receipts` — upload receipt file (multipart); returns OCR result

### Policy Endpoints (Finance Admin)
* `GET /api/v1/policies` — list all active rules with effective date ranges
* `POST /api/v1/policies` — create or update policy rule; validated for conflicts before persist
* `POST /api/v1/policies/validate` — validate a draft claim/request payload against current active policy; used by Angular for inline pre-validation

### Finance / Reimbursement Endpoints
* `GET /api/v1/reimbursements?status=&fromDate=&toDate=&department=` — reimbursement queue with filters
* `POST /api/v1/reimbursements/process` — trigger batch reimbursement run; returns batch job ID
* `GET /api/v1/reimbursements/{id}` — reimbursement record with status and line breakdown
* `GET /api/v1/reimbursements/batch/{batchId}` — batch run status and export download link

### Reporting Endpoints
* `GET /api/v1/reports/spend-summary?department=&period=&costCenter=` — aggregated spend data
* `GET /api/v1/reports/policy-violations?fromDate=&toDate=&employeeId=` — all flagged violation records
* `GET /api/v1/reports/budget-utilization?costCenter=&period=` — budget consumed vs. approved
* `GET /api/v1/reports/audit-log?fromDate=&toDate=&actorId=&entityType=` — audit trail export (Admin roles only)

---

## 5. Edge Cases

### Travel Request Edge Cases
* **Overlapping dates** — employee submits request with dates overlapping an existing approved request; system warns but does not block; reviewer sees overlap badge
* **Approver on leave** — no action within configured SLA window (default 48 hours); auto-escalates to deputy; deputy notified with context of original request
* **Cancelled post-approval** — employee cancels approved request; audit trail preserved; any linked expense claims locked and flagged for Finance Admin review
* **Multi-currency international travel** — amounts stored in original submission currency; converted at claim submission time using exchange rate configured for the trip date; missing rate returns explicit error prompting Finance Admin to configure
* **Submission on behalf** — EA or admin raises request for another employee; both submitter ID and traveler ID tracked as separate fields in the request record

### Expense Claim Edge Cases
* **Standalone claim** — claim raised without linked travel request (local/ad-hoc expense); allowed with standalone category flag; routes to lower approval threshold
* **OCR failure** — uploaded receipt is unreadable or returns low-confidence extraction; system falls back to manual entry mode; claim flagged with `receipt-manual-entry` indicator for reviewer attention
* **Duplicate submission** — same employee submits identical date + vendor + amount within 90-day window; flagged as probable duplicate; reviewer must explicitly override with written justification to process
* **Foreign currency — rate missing** — line item in currency with no rate configured for trip date; line item saved with `rate-missing` flag; claim blocked from submission until Finance Admin configures rate
* **Line item over policy limit** — flagged inline; does not block submission; approver sees violation badge per item; must check acknowledgment checkbox per flagged item before approve action is enabled
* **Employee terminated mid-workflow** — claim frozen in current state; HR Admin notified; claim ownership transferred to departing employee's last known manager for resolution

### Approval Workflow Edge Cases
* **Orphan approver** — no manager mapped for employee in approval hierarchy config; claim escalates to Finance Admin queue with `hierarchy-misconfigured` flag
* **Retroactive conflict of interest** — approver approved but is later identified as having a conflict; audit trail remains immutable; Finance Admin re-routes for secondary independent review; original approval record preserved
* **System downtime during SLA window** — SLA clock paused at detection of outage via health check failure; resumes on recovery; notification re-sent to approvers with updated deadline
* **Auto-approval** — claims below micro-transaction threshold (configurable by Finance Admin) bypass manual approval; logged with `auto-approved` actor and threshold value in audit trail
* **Partial approval** — finance rejects specific line items while approving others; approved items create reimbursement record; rejected items returned to employee as a new standalone draft claim

### Finance & Reimbursement Edge Cases
* **Missing bank details** — claim approved but employee has no payment details on file; reimbursement held in `pending-bank-details` status; employee prompted via notification to update profile before payout
* **ERP sync failure** — export file generated but ERP acknowledgment not received within timeout window; reimbursement batch marked `sync-failed`; alert raised to Finance Admin; automatic retry on next batch cycle
* **Batch run overlap** — concurrent batch trigger blocked by advisory lock at database level; second trigger returns `409 Conflict` with active batch job ID
* **Currency conversion variance** — exchange rate changes between claim submission and reimbursement processing; reimbursement calculated using rate at claim submission date; no retroactive recalculation

---

## 6. KPIs (Success Metrics & Acceptance Criteria)

### Business KPIs
* Travel request approval cycle time: ≤ 48 hours for standard requests — measured from submission to final approval
* Expense claim reimbursement cycle time: ≤ 5 business days from Finance approval to payout export
* Policy compliance rate: ≥ 98% of submitted claims pass without undetected violations — validated at 90-day post go-live audit
* Finance reconciliation effort: ≥ 70% reduction in manual hours vs. pre-system baseline — measured via finance team time-tracking at 90 days
* Employee satisfaction score: ≥ 4.0 / 5.0 on post-deployment T&E process survey

### System KPIs
* API P95 response time: ≤ 500ms for all read endpoints under 2,000 concurrent users
* System availability: ≥ 99.9% uptime per calendar month — verified via Azure Monitor availability tests
* Receipt OCR extraction accuracy: ≥ 90% field-level accuracy on legible uploads
* Reimbursement batch processing: 0 errors per batch run across 5,000 claims in production
* Audit trail completeness: 100% of state transitions logged — verified via automated audit integrity check run weekly

### Sprint Acceptance Criteria (per feature)
* **Travel Request:** Employee raises, submits, and tracks request end-to-end; approver receives email with action link within 60 seconds of submission; approval or rejection reflected in portal within 5 seconds of approver action
* **Expense Claim:** Employee adds line items, uploads receipts, and submits; OCR pre-fills fields within 10 seconds of upload; policy violations flagged before submit button is activated
* **Duplicate Detection:** Second submission with identical date + vendor + amount within 90 days flagged automatically; reviewer override requires written justification field — cannot be blank
* **Policy Engine:** Finance Admin configures per-diem rates and hotel caps; changes effective for new claim submissions within 5 minutes of save; in-flight claims validated against policy at submission date
* **Approval Workflow:** Approver acts via email link without navigating to portal; action logged with actor ID, timestamp, and comments; SLA escalation fires within 5 minutes of breach detection
* **Reporting:** Finance Manager spend summary dashboard loads 100,000-record dataset within 3 seconds; export to Excel completes within 10 seconds for same dataset
* **Reimbursement Batch:** Batch of 5,000 claims processed and export file generated within 30 minutes; Finance Admin receives completion notification with record count and export download link

---

## 7. Limitations

### Scope Exclusions — v1.0
* **No GDS integration** — no direct airline or hotel booking via the system; employees book externally and upload confirmation
* **No native mobile app** — Angular PWA delivered in v1.0; native iOS/Android apps deferred to v2.0 roadmap
* **No automated bank transfer** — reimbursement records exported to payroll/ERP; payment execution is external to this system
* **No multi-tenant support** — single enterprise deployment only; multi-company capability deferred
* **No ML fraud detection** — rule-based policy engine and duplicate detection only; ML anomaly detection deferred to v2.0
* **No corporate credit card feed** — manual expense claim entry only; automated card transaction import deferred
* **No real-time currency rates** — exchange rates manually configured by Finance Admin per trip date range; no live rate feed integration in v1.0

### Technical Limitations
* OCR accuracy dependent on upload image quality — handwritten or severely damaged receipts require manual entry fallback
* Exchange rate dependency — if Finance Admin has not configured a rate for a given currency and date, claim submission is blocked for that line item
* Approval hierarchy requires manual maintenance — no automatic sync with HR system org chart in v1.0; HR Admin updates hierarchy manually
* Concurrent user load tested and scaled to 2,000 simultaneous users — Azure App Service auto-scale handles burst above this; load test required before go-live to validate scaling behavior
* ERP integration is export-file-based — no real-time two-way sync; ERP acknowledgment is manual or webhook-based depending on ERP capability

### Compliance Limitations
* System generates a comprehensive audit trail but does not perform regulatory classification — legal and compliance team maps audit exports to specific regulatory frameworks (SOX, GDPR, local tax law)
* Data residency locked to Azure region selected at deployment — cross-region data replication and geo-redundancy not included in v1.0 scope; available as an upgrade path via Azure SQL geo-replication

