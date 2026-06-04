# Project Scope Document
**Project Name:** Enterprise Employee Travel & Expense Management System (ETEP)
**Version:** 1.0 | **Classification:** Internal – Confidential
**Stack:** ASP.NET Core (.NET 8) · Angular 17 · Azure SQL MI · AKS

---

## 1. KPI Points with Constraints

### 1.1 Functional KPIs & Constraints

| # | KPI | Target / Constraint | Enforcement Point |
|---|-----|---------------------|-------------------|
| F1 | Workflow Accuracy | Travel → Expense → Reimbursement → Reporting execute without deviation | MediatR pipeline + FluentValidation on every command |
| F2 | Mandatory Field Enforcement | All required fields validated before submission; drafts cannot transition to `SUBMITTED` with missing fields | FluentValidation rules per CQRS command; Angular Reactive Forms with `Validators.required` |
| F3 | Exception Handling | Invalid submissions return inline error messages; draft records retained for 30 days before auto-purge | Hangfire job: `PurgeStaleDraftsJob` runs nightly; API returns RFC 7807 `ProblemDetails` |
| F4 | OCR Accuracy | ≥ 90 % field-extraction accuracy on receipt images | Azure AI Document Intelligence confidence threshold ≥ 0.85; low-confidence fields flagged for manual correction |
| F5 | Advance Reconciliation | Advance vs. actual reconciliation must complete within 48 hours of travel end | Hangfire `ReconciliationReminderJob`; blocks new advance requests if prior unreconciled |
| F6 | Duplicate Receipt Detection | 100 % of re-submitted identical receipt images flagged before Finance review | Perceptual hash (pHash) stored on Azure Blob metadata; compared on every upload |

### 1.2 Non-Functional KPIs & Constraints

| # | KPI | Target | Constraint / Stack Binding |
|---|-----|--------|---------------------------|
| NF1 | Response Time | 95th-percentile API response < 2 s | Azure APIM throttling; Redis cache for reference data; compiled EF Core queries |
| NF2 | Concurrency | 20,000 simultaneous users without degradation | AKS HPA (CPU ≥ 70 % → scale-out); Azure SQL MI Business Critical tier; PgBouncer-equivalent via SQL connection pool |
| NF3 | Encryption at Rest | AES-256 via SQL Always Encrypted + TDE | Column-level Always Encrypted for PAN, bank account; TDE for full database; Azure Key Vault CMK |
| NF4 | Encryption in Transit | TLS 1.3 minimum on all endpoints | Azure APIM + AKS Ingress (NGINX) enforce TLS 1.3; HSTS header on all responses |
| NF5 | Uptime SLA | 99.9 % (≤ 8.7 h downtime/year) | Multi-AZ AKS; Azure SQL Business Critical with auto-failover; Azure Front Door health probes |
| NF6 | SOX Compliance | Full immutable audit trail for all financial actions | SQL Server temporal tables (system-versioned); audit_logs table append-only (no UPDATE/DELETE grants) |
| NF7 | ISO 27001 | Annual ISMS audit; quarterly VAPT | OWASP Dependency-Check in CI; Trivy container scanning; SonarQube Quality Gate in Azure DevOps |
| NF8 | GDPR / DPDPA | Data minimisation; right to erasure within 72 h | Anonymisation pipeline (Hangfire job); all PII stored in Always Encrypted columns; consent log table |
| NF9 | HRMS Sync Latency | Employee master data synced within 30 minutes of org change | Azure Service Bus topic `hrms.employee.changed`; consumer SLA monitored via Application Insights |
| NF10 | Notification Delivery | Email/SMS delivered within 60 seconds of trigger event | Azure Service Bus + SendGrid/Twilio; dead-letter queue alarm at > 5 undelivered messages |
| NF11 | Rollback Time | Blue-green rollback completed in < 15 minutes | Azure DevOps release gates; Helm `helm rollback` with pre-configured revision; automated smoke tests post-switch |
| NF12 | Unit Test Coverage | ≥ 80 % on all backend services | SonarQube Quality Gate blocks merge if coverage drops below threshold |

---

## 2. Functional Requirements (In-Scope)

### 2.1 Module: Travel Requests
- Employee initiates travel request (purpose, dates, origin, destination, mode, estimated cost, advance)
- Real-time budget balance displayed via Redis-cached budget service
- Multi-level approval chain: L1 (Line Manager) → L2 (Dept Head, if > ₹50,000 or international) → Finance (if > ₹1,00,000)
- Approver absence detection via HRMS leave API → auto-skip to delegate
- Travel Authorization Number (TAN) generated on full approval
- Blackout date enforcement (HR-configured calendar)
- International travel: visa document upload mandatory; FEMA forex limit enforcement

### 2.2 Module: Expense Claims
- Claim linked to approved TAN or standalone (local/adhoc)
- Azure AI Document Intelligence OCR: auto-extract amount, vendor, date, GSTIN from receipt
- 14-point policy validation engine (category limits by grade band, duplicate detection, GST validation, mileage GPS check, etc.)
- Advance adjustment auto-computed: `NetReimbursable = TotalClaimed − AdvancePaid`
- Fraud score (0–100) computed by ML microservice; score > 70 → mandatory Finance hold
- Line-item-level approval; Finance can approve/reject individual lines

### 2.3 Module: Reimbursement Processing
- Nightly Hangfire batch: groups Finance-approved claims into settlement batch
- Payroll system integration: pushes NEFT/IMPS/RTGS payment file via Azure Service Bus
- Payment status webhook received from payroll; employee notified via SignalR (in-app) + email/SMS
- Failed payments: 3 auto-retries (exponential backoff 1 h, 4 h, 8 h); Finance alert on exhaustion

### 2.4 Module: Budget Management
- Annual budgets defined per department + cost centre + grade band
- Real-time consumption tracker; alerts at 75 %, 90 %, 100 % via Azure Service Bus events
- Budget reallocation workflow with CFO approval; freeze capability

### 2.5 Module: Reporting & Analytics
- Standard reports: travel spend by dept, expense category breakdown, policy violations, advance outstanding, reimbursement settlement, vendor spend
- ClickHouse-equivalent: Azure Synapse Analytics for OLAP; ETL via Azure Data Factory (nightly)
- Export to XLSX/PDF via async job; download via signed Azure Blob URL (15-min TTL)
- Real-time dashboard widgets via SignalR push

### 2.6 Module: Administration
- HR Admin: policy rules config (no hardcoding), per-diem tables, blackout calendar, grade-band entitlements
- User management: HRMS-driven onboard/offboard; role assignment synced from Azure AD groups
- Approval hierarchy override and delegate assignment

---

## 3. Non-Functional Requirements (In-Scope)

| Area | Requirement | Implementation |
|------|-------------|----------------|
| Performance | P95 API < 2 s; dashboard load < 1.5 s | Redis cache (reference data TTL 1 h); compiled EF Core queries; Azure CDN for Angular SPA |
| Scalability | 20,000 concurrent; auto-scale to 3× on month-end peak | AKS HPA; Azure SQL read replicas for reporting; Azure Service Bus partitioned queues |
| Security | Zero-trust; field-level encryption; RBAC | Azure AD MSAL JWT; Always Encrypted; Row-Level Security on SQL; APIM rate limiting |
| Availability | 99.9 % uptime | Multi-AZ AKS; Azure SQL Business Critical; Azure Front Door |
| DR | RTO < 2 h; RPO < 15 min | Geo-redundant Azure SQL; automated failover; weekly DR drill |
| Observability | Full distributed tracing | Azure Application Insights SDK in .NET 8 services + Angular; Log Analytics workspace; custom KQL dashboards |
| Accessibility | WCAG 2.1 AA | Angular Material + axe-core in Playwright suite |
| i18n | Multi-language UI | ngx-translate with en/hi/regional locale bundles |

---

## 4. Stopping Points (Out-of-Scope for v1.0)

The following are **explicitly excluded** from the current delivery and deferred to post-launch roadmap:

| # | Stopped Feature | Reason / Next Phase |
|---|-----------------|---------------------|
| S1 | Direct airline/hotel booking integration | Phase 5 – requires vendor API contracts; complex PNR sync |
| S2 | AI-driven predictive budget forecasting | Phase 5 – needs 12 months of historical data to train models |
| S3 | WhatsApp / Microsoft Teams bot for approvals | Phase 5 – channel partnership agreements pending |
| S4 | Blockchain immutable audit (permissioned) | Phase 5 – regulatory clarity on admissibility pending |
| S5 | Multi-entity / group consolidation reporting | Phase 5 – subsidiary HRMS integrations not yet defined |
| S6 | Carbon footprint / ESG travel tracking | Phase 5 – ESG framework definition in progress |
| S7 | Native iOS/Android mobile app | Phase 3 – Angular PWA delivered in Phase 2; native in Phase 3 |
| S8 | External auditor self-service portal login | Phase 4 – security review for external access ongoing |
| S9 | Corporate card auto-reconciliation (bank feed) | Phase 5 – bank API agreements not finalised |
| S10 | Real-time forex hedging recommendations | Phase 5 – treasury system integration not scoped |

---

## 5. In-Scope Development Portions

### 5.1 Backend (.NET 8 / ASP.NET Core)

| Service | Responsibility | Key Libraries |
|---------|---------------|---------------|
| Auth Service | Azure AD/MSAL JWT validation, role claims | Microsoft.Identity.Web, MSAL |
| Travel Service | TravelRequest CQRS (Commands + Queries), TAN lifecycle | MediatR, FluentValidation, AutoMapper |
| Expense Service | ExpenseClaim CQRS, OCR orchestration, policy engine | MediatR, Azure.AI.FormRecognizer |
| Reimbursement Service | Settlement batch, payroll push, webhook handler | Hangfire, Azure.Messaging.ServiceBus |
| Notification Service | Email/SMS/push dispatch, template engine | SendGrid SDK, Twilio SDK, SignalR Hub |
| Document Service | Blob upload/download, virus scan, signed URL | Azure.Storage.Blobs, VirusTotal API |
| Reporting Service | Report generation, OLAP query, export | Azure Synapse, EPPlus (XLSX), iTextSharp (PDF) |
| ML/Fraud Service | Fraud score inference endpoint (Python FastAPI) | scikit-learn, Azure ML Managed Endpoint |

### 5.2 Frontend (Angular 17)

| Module | Components | State |
|--------|-----------|-------|
| Auth Module | Login, MFA prompt, session timeout | NgRx AuthState |
| Travel Module | RequestForm, RequestList, RequestDetail, ApprovalChain | NgRx TravelState |
| Expense Module | ClaimForm, LineItemEditor, ReceiptCapture, PolicyCheck | NgRx ExpenseState |
| Reimbursement Module | PaymentStatus, BatchView, SettlementHistory | NgRx ReimbState |
| Reports Module | Dashboard, ReportBuilder, ExportProgress | NgRx ReportState |
| Admin Module | PolicyConfig, BudgetSetup, UserManagement, Calendar | NgRx AdminState |
| Shared | ConfirmDialog, Toast, DataTable, BudgetGauge, SignalR service | – |

### 5.3 Database (SQL Server 2022 / Azure SQL MI)

- EF Core 8 code-first with fluent API; all migrations tracked in Azure DevOps
- Always Encrypted columns: `BankAccountNo`, `PANNumber`, `GSTIN`, `MobileNumber`
- Transparent Data Encryption: enabled database-wide
- Row-Level Security: `fn_securitypredicate` filters `TravelRequests`, `ExpenseClaims` by `EmployeeId` or role membership
- System-versioned temporal tables: `TravelRequests`, `ExpenseClaims`, `Approvals`, `Reimbursements`
- Compiled queries registered in DI container for high-frequency reads (e.g., budget balance, pending approvals count)
- Azure SQL read replica: reporting queries routed via `ApplicationIntent=ReadOnly`

### 5.4 Infrastructure (Azure)

- AKS cluster (3 node pools: system, app, gpu-for-ml); Helm charts per service
- Azure APIM: API gateway, OAuth 2.0 validation, rate limits (1,000 req/min/user), WAF policy
- Azure Cache for Redis: cluster mode (3 nodes); session cache, reference data cache, fraud score cache
- Azure Service Bus: 4 topics (`travel.events`, `expense.events`, `reimbursement.events`, `notification.events`); DLQ monitoring
- Azure Blob Storage: `receipts` container (encrypted, private); `exports` container (signed URLs)
- Azure Key Vault: all connection strings, API keys, encryption keys; RBAC access from managed identity
- Azure Application Insights: SDK in all .NET 8 services + Angular; custom telemetry for business events
- Azure Front Door: global load balancing, WAF, CDN for Angular SPA

### 5.5 DevOps (Azure DevOps)

- Multi-stage YAML pipelines: PR → Build → Test → SonarQube → Security Scan → Docker Build → Trivy → Deploy (Dev → QA → Staging → Prod)
- SonarQube Quality Gate: coverage ≥ 80 %, 0 critical code smells blocks merge
- OWASP Dependency-Check: High/Critical CVEs block pipeline
- Trivy: container image scan; Critical CVEs block Docker push
- Helm deployments with `helm upgrade --atomic`; automated `helm rollback` on health-check failure
- Blue-green deployment via Azure Traffic Manager weight shifting; rollback < 15 min

---

## 6. Acceptance Criteria Summary

All 20 KPI sections from `KPI.md` must achieve **Pass** status before Production sign-off:

| Ref | Section | Pass Condition |
|-----|---------|----------------|
| K1 | Functional Requirements | Workflows execute without deviation in UAT across all 4 modules |
| K2 | Non-Functional Requirements | Load test confirms P95 < 2 s at 20,000 concurrent; encryption validated by security audit |
| K3 | System Architecture | All services deployed independently; integration smoke tests green |
| K4 | Data Model | Migration scripts applied to Staging; referential integrity constraints verified |
| K5 | API Design | 100 % endpoint availability in Postman/Newman suite; 0 schema validation failures |
| K6 | User Flow | All 4 persona journeys completed end-to-end in UAT without error |
| K7 | UI/UX | axe-core scan: 0 WCAG 2.1 AA violations on primary flows |
| K8 | Integrations | HRMS sync < 30 min; notifications < 60 s; SSO login < 3 s |
| K9 | Security | VAPT report: 0 Critical, 0 High findings before Go-Live |
| K10 | Performance | Redis cache hit rate > 80 %; auto-scale verified under load test |
| K11 | Observability | All alerts firing correctly in Application Insights; dashboards live |
| K12 | Testing | SonarQube coverage ≥ 80 %; Playwright E2E suite 100 % green |
| K13 | Deployment | Azure DevOps pipeline end-to-end green; blue-green rollback tested |
| K14 | Risk Mitigation | Training completed for pilot group; dry-run migration validated |
| K15 | Success Metrics | Post-launch 30-day report shows approval cycle < 4 h (P95) |
| K16 | Timeline | Phase 1 delivered in Month 3; Phase 2 in Month 5; Phase 3 in Month 7 |
| K17 | Future Enhancements | Roadmap items documented in backlog; architecture extensibility confirmed |
| K18 | Compliance | GST, FEMA, GDPR, SOX controls verified by Internal Audit sign-off |
| K19 | Audit Framework | Temporal tables active; append-only audit_logs confirmed; immutability tested |
| K20 | Data Retention | Archival job verified; 7-year retention confirmed in policy; retrieval tested < 5 min |
