# Project Boundary Document
**Project Name:** Enterprise Employee Travel & Expense Management System (ETEP)
**Version:** 1.0 | **Classification:** Internal – Confidential
**References:** PRD v1.0, KPI Document v1.0

---

## 1. Project Overview Summary

ETEP replaces a fully manual, email-and-Excel-driven Travel & Expense workflow for an organisation of **10,000+ employees across multiple locations**. The platform digitalises four operational domains — Travel Requests, Expense Claims, Reimbursements, and Reporting — into a single, governed, cloud-native system.

| Dimension | Current State | Target State |
|-----------|--------------|--------------|
| Travel approval cycle | 4–7 business days (email) | < 4 hours (95th percentile) |
| Expense claim processing | 10–15 days (Excel + paper) | < 3 business days |
| Reimbursement settlement | 15–30 days | 3 business days post-approval |
| Policy compliance rate | ~50 % (estimated) | ≥ 98 % |
| Finance FTE effort on T&E | ~40 hrs/week (10 FTEs) | < 10 hrs/week |
| Audit trail | None (email archives) | Immutable temporal tables + append-only audit log |
| System uptime SLA | N/A | 99.9 % |

**Technology Boundary:**

| Layer | Technology Chosen |
|-------|------------------|
| Backend | ASP.NET Core Web API (.NET 8 LTS), Clean Architecture, CQRS via MediatR |
| Frontend | Angular 17 SPA, Standalone Components, NgRx, SignalR, ngx-translate |
| Database | SQL Server 2022 (Azure SQL Managed Instance), EF Core 8, Always Encrypted, TDE |
| Cloud | Azure Kubernetes Service, Azure Service Bus, Azure Blob Storage, Azure Cache for Redis, Azure APIM, Azure Key Vault |
| DevOps | Azure DevOps, SonarQube, OWASP Dependency-Check, Helm, Docker, Trivy |
| Security | Azure AD / MSAL JWT, RBAC, Row-Level Security, AES-256, TLS 1.3 |

---

## 2. System Boundary Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        AZURE FRONT DOOR (WAF + CDN)                     │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │ HTTPS / TLS 1.3
┌───────────────────────────────▼─────────────────────────────────────────┐
│                    AZURE API MANAGEMENT (APIM)                          │
│           OAuth 2.0 JWT validation · Rate limiting · WAF policy         │
└──────┬────────────────────┬────────────────────┬────────────────────────┘
       │                    │                    │
┌──────▼──────┐  ┌──────────▼──────────┐  ┌─────▼───────────────────────┐
│ Angular 17  │  │  Microservices on   │  │   Background Workers (AKS)  │
│   SPA       │  │  AKS (Namespaces)   │  │   Hangfire + Service Bus    │
│ (Azure CDN) │  │  Auth · Travel ·    │  │   consumers                 │
│ SignalR Hub │  │  Expense · Reimb ·  │  └─────────────────────────────┘
│ ngx-translate│  │  Notify · Docs ·   │
└─────────────┘  │  Reports · ML/Fraud │
                 └──────────┬──────────┘
                            │
       ┌────────────────────┼────────────────────┐
       │                    │                    │
┌──────▼──────┐  ┌──────────▼──────────┐  ┌─────▼───────────────────────┐
│ Azure SQL MI │  │ Azure Cache (Redis) │  │  Azure Service Bus          │
│ SQL Svr 2022 │  │ Session · RefData · │  │  4 Topics + DLQ             │
│ Always Enc. │  │ Fraud score cache   │  │  travel/expense/reimb/notify │
│ TDE · RLS   │  └─────────────────────┘  └─────────────────────────────┘
│ Temporal Tbls│
└─────────────┘
       │
┌──────▼──────────────────────────────────────────────────────────────────┐
│                        EXTERNAL INTEGRATIONS                            │
│  Azure AD/MSAL · HRMS (SAP SF) · Payroll (SAP/Oracle) · SendGrid/Twilio│
│  Azure AI Document Intelligence (OCR) · RBI Forex API · GST Portal API │
└─────────────────────────────────────────────────────────────────────────┘
       │
┌──────▼──────────────────────────────────────────────────────────────────┐
│                         OBSERVABILITY PLANE                             │
│     Azure Application Insights · Log Analytics · Azure Monitor         │
│     Custom KQL Dashboards · PagerDuty Alerts · Distributed Tracing     │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Directory / Folder Structure

The repository follows a **monorepo** strategy managed via **Azure DevOps** with a single Git repository containing all services, the frontend, infrastructure-as-code, and documentation.

```
etep/                                         ← Repository root
│
├── .azuredevops/                             ← Azure DevOps pipeline definitions
│   ├── pipelines/
│   │   ├── pr-validation.yml                 ← PR: lint + unit tests + SonarQube
│   │   ├── build-backend.yml                 ← Build .NET 8 services → Docker images
│   │   ├── build-frontend.yml                ← Build Angular 17 SPA → Docker image
│   │   ├── deploy-dev.yml
│   │   ├── deploy-qa.yml
│   │   ├── deploy-staging.yml
│   │   └── deploy-prod.yml                   ← Blue-green with rollback gates
│   └── templates/
│       ├── sonarqube-scan.yml
│       ├── trivy-scan.yml
│       ├── owasp-dependency-check.yml
│       └── helm-deploy.yml
│
├── src/                                      ← All application source code
│   │
│   ├── backend/                              ← ASP.NET Core (.NET 8) services
│   │   │
│   │   ├── ETEP.Domain/                      ← Domain layer (no external deps)
│   │   │   ├── Entities/
│   │   │   │   ├── Employee.cs
│   │   │   │   ├── TravelRequest.cs
│   │   │   │   ├── ExpenseClaim.cs
│   │   │   │   ├── ExpenseLineItem.cs
│   │   │   │   ├── Approval.cs
│   │   │   │   ├── Reimbursement.cs
│   │   │   │   ├── ReimbursementBatch.cs
│   │   │   │   ├── AuditLog.cs
│   │   │   │   └── Budget.cs
│   │   │   ├── Enums/
│   │   │   │   ├── TravelStatus.cs
│   │   │   │   ├── ClaimStatus.cs
│   │   │   │   ├── ApprovalAction.cs
│   │   │   │   └── GradeBand.cs
│   │   │   ├── ValueObjects/
│   │   │   │   ├── Money.cs
│   │   │   │   ├── TravelAuthorizationNumber.cs
│   │   │   │   └── ClaimNumber.cs
│   │   │   ├── Events/                       ← Domain events (MediatR INotification)
│   │   │   │   ├── TravelRequestApproved.cs
│   │   │   │   ├── ExpenseClaimSubmitted.cs
│   │   │   │   └── ReimbursementProcessed.cs
│   │   │   └── Exceptions/
│   │   │       ├── PolicyViolationException.cs
│   │   │       ├── DuplicateReceiptException.cs
│   │   │       └── BudgetExceededException.cs
│   │   │
│   │   ├── ETEP.Application/                 ← Application layer (CQRS, use cases)
│   │   │   ├── Common/
│   │   │   │   ├── Behaviours/
│   │   │   │   │   ├── ValidationBehaviour.cs      ← FluentValidation pipeline
│   │   │   │   │   ├── LoggingBehaviour.cs
│   │   │   │   │   ├── PerformanceBehaviour.cs
│   │   │   │   │   └── AuditBehaviour.cs
│   │   │   │   ├── Interfaces/
│   │   │   │   │   ├── ITravelRepository.cs
│   │   │   │   │   ├── IExpenseRepository.cs
│   │   │   │   │   ├── IReimbursementRepository.cs
│   │   │   │   │   ├── IHrmsService.cs
│   │   │   │   │   ├── IOcrService.cs
│   │   │   │   │   ├── IPayrollService.cs
│   │   │   │   │   ├── INotificationService.cs
│   │   │   │   │   ├── IBlobStorageService.cs
│   │   │   │   │   └── IFraudScoringService.cs
│   │   │   │   └── Models/
│   │   │   │       ├── PolicyCheckResult.cs
│   │   │   │       └── OcrExtractionResult.cs
│   │   │   ├── Features/
│   │   │   │   ├── TravelRequests/
│   │   │   │   │   ├── Commands/
│   │   │   │   │   │   ├── CreateTravelRequest/
│   │   │   │   │   │   │   ├── CreateTravelRequestCommand.cs
│   │   │   │   │   │   │   ├── CreateTravelRequestCommandHandler.cs
│   │   │   │   │   │   │   └── CreateTravelRequestCommandValidator.cs
│   │   │   │   │   │   ├── ApproveTravelRequest/
│   │   │   │   │   │   ├── RejectTravelRequest/
│   │   │   │   │   │   └── CancelTravelRequest/
│   │   │   │   │   └── Queries/
│   │   │   │   │       ├── GetTravelRequestById/
│   │   │   │   │       ├── GetMyTravelRequests/
│   │   │   │   │       └── GetPendingApprovals/
│   │   │   │   ├── ExpenseClaims/
│   │   │   │   │   ├── Commands/
│   │   │   │   │   │   ├── CreateExpenseClaim/
│   │   │   │   │   │   ├── AddLineItem/
│   │   │   │   │   │   ├── SubmitExpenseClaim/
│   │   │   │   │   │   ├── ApproveExpenseClaim/
│   │   │   │   │   │   └── UploadReceipt/
│   │   │   │   │   └── Queries/
│   │   │   │   │       ├── GetExpenseClaimById/
│   │   │   │   │       ├── GetMyExpenseClaims/
│   │   │   │   │       └── ValidateClaimPolicy/
│   │   │   │   ├── Reimbursements/
│   │   │   │   │   ├── Commands/
│   │   │   │   │   │   ├── ProcessReimbursementBatch/
│   │   │   │   │   │   ├── ApproveReimbursement/
│   │   │   │   │   │   └── HandlePaymentCallback/
│   │   │   │   │   └── Queries/
│   │   │   │   │       ├── GetReimbursementById/
│   │   │   │   │       └── GetBatchStatus/
│   │   │   │   ├── Budget/
│   │   │   │   │   ├── Commands/
│   │   │   │   │   └── Queries/
│   │   │   │   └── Reports/
│   │   │   │       └── Queries/
│   │   │   │           ├── GetTravelSpendReport/
│   │   │   │           ├── GetPolicyViolationReport/
│   │   │   │           └── GetAdvanceOutstandingReport/
│   │   │   └── Services/
│   │   │       ├── PolicyEngineService.cs     ← 14-point policy validation
│   │   │       ├── FraudDetectionService.cs
│   │   │       └── BudgetCalculationService.cs
│   │   │
│   │   ├── ETEP.Infrastructure/              ← Infrastructure layer
│   │   │   ├── Persistence/
│   │   │   │   ├── ApplicationDbContext.cs   ← EF Core DbContext
│   │   │   │   ├── Configurations/           ← Fluent API entity configs
│   │   │   │   │   ├── EmployeeConfiguration.cs
│   │   │   │   │   ├── TravelRequestConfiguration.cs
│   │   │   │   │   ├── ExpenseClaimConfiguration.cs
│   │   │   │   │   ├── ApprovalConfiguration.cs
│   │   │   │   │   └── ReimbursementConfiguration.cs
│   │   │   │   ├── Migrations/               ← EF Core code-first migrations
│   │   │   │   ├── Repositories/
│   │   │   │   │   ├── TravelRepository.cs
│   │   │   │   │   ├── ExpenseRepository.cs
│   │   │   │   │   └── ReimbursementRepository.cs
│   │   │   │   └── CompiledQueries/          ← EF Core compiled queries
│   │   │   │       ├── TravelCompiledQueries.cs
│   │   │   │       └── ExpenseCompiledQueries.cs
│   │   │   ├── Services/
│   │   │   │   ├── HrmsService.cs            ← SAP SuccessFactors/Workday adapter
│   │   │   │   ├── PayrollService.cs         ← SAP Payroll/Oracle HCM adapter
│   │   │   │   ├── OcrService.cs             ← Azure AI Document Intelligence
│   │   │   │   ├── BlobStorageService.cs     ← Azure Blob Storage
│   │   │   │   ├── NotificationService.cs    ← SendGrid + Twilio + SignalR
│   │   │   │   ├── FraudScoringService.cs    ← HTTP client → ML microservice
│   │   │   │   ├── ForexRateService.cs       ← RBI / Open Exchange Rates API
│   │   │   │   └── GstVerificationService.cs ← GST Portal API
│   │   │   ├── Messaging/
│   │   │   │   ├── Publishers/
│   │   │   │   │   ├── TravelEventPublisher.cs
│   │   │   │   │   ├── ExpenseEventPublisher.cs
│   │   │   │   │   └── ReimbursementEventPublisher.cs
│   │   │   │   └── Consumers/
│   │   │   │       ├── HrmsEmployeeChangedConsumer.cs
│   │   │   │       └── PaymentCallbackConsumer.cs
│   │   │   ├── Jobs/                         ← Hangfire background jobs
│   │   │   │   ├── ReimbursementBatchJob.cs
│   │   │   │   ├── PurgeStaleDraftsJob.cs
│   │   │   │   ├── ReconciliationReminderJob.cs
│   │   │   │   ├── BudgetAlertJob.cs
│   │   │   │   └── HrmsSyncJob.cs
│   │   │   └── Security/
│   │   │       ├── RateLimitingMiddleware.cs
│   │   │       ├── AuditMiddleware.cs
│   │   │       └── EncryptionHelper.cs       ← Azure Key Vault CMK wrapper
│   │   │
│   │   ├── ETEP.API/                         ← Presentation layer (ASP.NET Core)
│   │   │   ├── Controllers/
│   │   │   │   ├── TravelController.cs
│   │   │   │   ├── ExpenseController.cs
│   │   │   │   ├── ReimbursementController.cs
│   │   │   │   ├── ReportController.cs
│   │   │   │   ├── BudgetController.cs
│   │   │   │   ├── AdminController.cs
│   │   │   │   └── HealthController.cs
│   │   │   ├── Hubs/
│   │   │   │   └── NotificationHub.cs        ← SignalR hub
│   │   │   ├── Middleware/
│   │   │   │   ├── ExceptionHandlingMiddleware.cs  ← RFC 7807 ProblemDetails
│   │   │   │   ├── CorrelationIdMiddleware.cs
│   │   │   │   └── SecurityHeadersMiddleware.cs
│   │   │   ├── Filters/
│   │   │   │   └── AntiForgeryFilter.cs
│   │   │   ├── Program.cs                    ← Minimal API bootstrapping
│   │   │   ├── appsettings.json
│   │   │   ├── appsettings.Development.json
│   │   │   └── Dockerfile
│   │   │
│   │   └── ETEP.ML/                          ← Python FastAPI fraud scoring service
│   │       ├── main.py
│   │       ├── model/
│   │       │   ├── fraud_model.pkl
│   │       │   └── feature_engineering.py
│   │       ├── requirements.txt
│   │       └── Dockerfile
│   │
│   ├── frontend/                             ← Angular 17 SPA
│   │   ├── src/
│   │   │   ├── app/
│   │   │   │   ├── core/                     ← Singleton services, guards, interceptors
│   │   │   │   │   ├── auth/
│   │   │   │   │   │   ├── auth.guard.ts
│   │   │   │   │   │   ├── auth.interceptor.ts   ← JWT attach + refresh
│   │   │   │   │   │   └── msal.config.ts
│   │   │   │   │   ├── services/
│   │   │   │   │   │   ├── signalr.service.ts
│   │   │   │   │   │   ├── notification.service.ts
│   │   │   │   │   │   └── error-handler.service.ts
│   │   │   │   │   └── store/                ← Root NgRx setup
│   │   │   │   │       └── app.state.ts
│   │   │   │   ├── shared/                   ← Reusable standalone components
│   │   │   │   │   ├── components/
│   │   │   │   │   │   ├── data-table/
│   │   │   │   │   │   ├── budget-gauge/
│   │   │   │   │   │   ├── receipt-uploader/
│   │   │   │   │   │   ├── approval-chain/
│   │   │   │   │   │   ├── toast/
│   │   │   │   │   │   ├── confirm-dialog/
│   │   │   │   │   │   └── status-badge/
│   │   │   │   │   ├── directives/
│   │   │   │   │   │   └── has-permission.directive.ts
│   │   │   │   │   └── pipes/
│   │   │   │   │       ├── currency-inr.pipe.ts
│   │   │   │   │       └── grade-band-label.pipe.ts
│   │   │   │   ├── features/
│   │   │   │   │   ├── auth/
│   │   │   │   │   │   ├── login/
│   │   │   │   │   │   └── session-timeout/
│   │   │   │   │   ├── travel/
│   │   │   │   │   │   ├── travel-request-form/
│   │   │   │   │   │   ├── travel-request-list/
│   │   │   │   │   │   ├── travel-request-detail/
│   │   │   │   │   │   ├── approval-queue/
│   │   │   │   │   │   └── store/
│   │   │   │   │   │       ├── travel.actions.ts
│   │   │   │   │   │       ├── travel.effects.ts
│   │   │   │   │   │       ├── travel.reducer.ts
│   │   │   │   │   │       └── travel.selectors.ts
│   │   │   │   │   ├── expense/
│   │   │   │   │   │   ├── claim-form/
│   │   │   │   │   │   ├── line-item-editor/
│   │   │   │   │   │   ├── receipt-capture/   ← Camera + file upload + OCR preview
│   │   │   │   │   │   ├── policy-check-panel/
│   │   │   │   │   │   ├── claim-list/
│   │   │   │   │   │   └── store/
│   │   │   │   │   ├── reimbursement/
│   │   │   │   │   │   ├── payment-status/
│   │   │   │   │   │   ├── batch-view/
│   │   │   │   │   │   ├── settlement-history/
│   │   │   │   │   │   └── store/
│   │   │   │   │   ├── reports/
│   │   │   │   │   │   ├── dashboard/
│   │   │   │   │   │   ├── report-builder/
│   │   │   │   │   │   ├── export-progress/
│   │   │   │   │   │   └── store/
│   │   │   │   │   └── admin/
│   │   │   │   │       ├── policy-config/
│   │   │   │   │       ├── budget-setup/
│   │   │   │   │       ├── user-management/
│   │   │   │   │       ├── blackout-calendar/
│   │   │   │   │       └── store/
│   │   │   │   ├── app.component.ts           ← Standalone root component
│   │   │   │   ├── app.config.ts              ← provideRouter, provideStore, etc.
│   │   │   │   └── app.routes.ts              ← Lazy-loaded feature routes
│   │   │   ├── assets/
│   │   │   │   ├── i18n/
│   │   │   │   │   ├── en.json
│   │   │   │   │   └── hi.json
│   │   │   │   └── images/
│   │   │   ├── styles/
│   │   │   │   ├── _tokens.scss               ← Design tokens (colors, spacing, typography)
│   │   │   │   ├── _material-theme.scss       ← Angular Material theme override
│   │   │   │   └── styles.scss
│   │   │   └── environments/
│   │   │       ├── environment.ts
│   │   │       └── environment.prod.ts
│   │   ├── angular.json
│   │   ├── tsconfig.json
│   │   ├── karma.conf.js                      ← Jasmine/Karma unit test config
│   │   ├── playwright.config.ts               ← E2E test config
│   │   └── Dockerfile
│   │
│   └── tests/                                 ← Shared test projects
│       ├── ETEP.UnitTests/                    ← xUnit (.NET 8)
│       │   ├── Application/
│       │   │   ├── TravelRequests/
│       │   │   ├── ExpenseClaims/
│       │   │   └── Reimbursements/
│       │   └── Domain/
│       ├── ETEP.IntegrationTests/             ← xUnit + Testcontainers + WebApplicationFactory
│       │   ├── Api/
│       │   │   ├── TravelControllerTests.cs
│       │   │   ├── ExpenseControllerTests.cs
│       │   │   └── ReimbursementControllerTests.cs
│       │   └── Infrastructure/
│       │       └── PayrollServiceTests.cs
│       └── ETEP.E2ETests/                     ← Playwright (TypeScript)
│           ├── specs/
│           │   ├── employee-travel-journey.spec.ts
│           │   ├── manager-approval-journey.spec.ts
│           │   ├── finance-claim-journey.spec.ts
│           │   └── hr-admin-journey.spec.ts
│           └── page-objects/
│               ├── TravelRequestPage.ts
│               ├── ExpenseClaimPage.ts
│               └── ApprovalQueuePage.ts
│
├── infrastructure/                            ← Infrastructure as Code
│   ├── terraform/                             ← Azure resource provisioning
│   │   ├── modules/
│   │   │   ├── aks/
│   │   │   │   └── main.tf                    ← AKS cluster + node pools
│   │   │   ├── sql/
│   │   │   │   └── main.tf                    ← Azure SQL MI + Always Encrypted
│   │   │   ├── redis/
│   │   │   │   └── main.tf                    ← Azure Cache for Redis cluster
│   │   │   ├── servicebus/
│   │   │   │   └── main.tf                    ← Service Bus namespace + topics + DLQ
│   │   │   ├── blob/
│   │   │   │   └── main.tf                    ← Storage accounts + containers
│   │   │   ├── apim/
│   │   │   │   └── main.tf                    ← API Management + WAF policy
│   │   │   ├── keyvault/
│   │   │   │   └── main.tf                    ← Key Vault + CMK + managed identity
│   │   │   └── monitoring/
│   │   │       └── main.tf                    ← App Insights + Log Analytics + alerts
│   │   ├── environments/
│   │   │   ├── dev.tfvars
│   │   │   ├── qa.tfvars
│   │   │   ├── staging.tfvars
│   │   │   └── prod.tfvars
│   │   └── main.tf
│   │
│   └── helm/                                  ← Kubernetes Helm charts
│       ├── etep-backend/
│       │   ├── Chart.yaml
│       │   ├── values.yaml
│       │   ├── values-dev.yaml
│       │   ├── values-prod.yaml
│       │   └── templates/
│       │       ├── deployment.yaml
│       │       ├── service.yaml
│       │       ├── ingress.yaml               ← NGINX Ingress + TLS
│       │       ├── hpa.yaml                   ← HPA: CPU ≥ 70 % → scale
│       │       ├── configmap.yaml
│       │       └── secret.yaml                ← Refs to Azure Key Vault
│       ├── etep-frontend/
│       │   └── templates/
│       ├── etep-ml/
│       │   └── templates/
│       └── etep-hangfire/
│           └── templates/
│
├── database/                                  ← Database scripts & documentation
│   ├── migrations/                            ← EF Core auto-generated (tracked here)
│   ├── seed/
│   │   ├── GradeBandEntitlements.sql
│   │   ├── PolicyRules.sql
│   │   ├── CityTierMaster.sql
│   │   └── ProjectCodes.sql
│   ├── procedures/
│   │   ├── sp_GenerateTAN.sql
│   │   └── sp_ArchiveExpiredRecords.sql
│   ├── rls/
│   │   └── fn_securitypredicate.sql           ← Row-Level Security predicate
│   └── schema-diagram.png                     ← ERD exported from SSMS
│
├── docs/                                      ← Project documentation
│   ├── PRD_v1.0.md                            ← This PRD document
│   ├── project_scope.md                       ← Project Scope Document
│   ├── project_boundary.md                    ← This document
│   ├── KPI_v1.0.md                            ← KPI tracking document
│   ├── adr/                                   ← Architecture Decision Records
│   │   ├── ADR-001-clean-architecture.md
│   │   ├── ADR-002-cqrs-mediatr.md
│   │   ├── ADR-003-azure-sql-mi.md
│   │   ├── ADR-004-angular-ngrx.md
│   │   └── ADR-005-aks-helm.md
│   ├── api/
│   │   └── openapi.yaml                       ← OpenAPI 3.1 spec (auto-generated)
│   └── runbooks/
│       ├── rollback-procedure.md
│       ├── dr-failover.md
│       └── incident-response.md
│
├── scripts/                                   ← Developer & ops utility scripts
│   ├── setup-local-dev.ps1                    ← Docker Compose + local secrets
│   ├── run-migrations.ps1
│   ├── seed-dev-data.ps1
│   └── generate-openapi.ps1
│
├── docker-compose.yml                         ← Local development stack
├── docker-compose.override.yml                ← Dev overrides (ports, volumes)
├── .editorconfig
├── .gitignore
├── global.json                                ← .NET 8 SDK pin
├── Directory.Build.props                      ← Shared MSBuild properties
├── Directory.Packages.props                   ← Central NuGet package versioning
└── README.md
```

---

## 4. Service-to-Service Communication Boundary

| Producer Service | Event / Message | Consumer Service | Bus / Mechanism |
|-----------------|-----------------|-----------------|-----------------|
| Travel Service | `travel.request.approved` | Notification Service | Azure Service Bus |
| Travel Service | `travel.request.approved` | Reimbursement Service (advance) | Azure Service Bus |
| Expense Service | `expense.claim.submitted` | Notification Service | Azure Service Bus |
| Expense Service | `expense.claim.approved` | Reimbursement Service | Azure Service Bus |
| Expense Service | `expense.fraud.flagged` | Notification Service (Finance alert) | Azure Service Bus |
| Reimbursement Service | `reimbursement.payment.pushed` | Notification Service | Azure Service Bus |
| Reimbursement Service | `reimbursement.payment.failed` | Notification Service + Finance alert | Azure Service Bus |
| HRMS External | `hrms.employee.changed` | Auth Service + Travel + Expense | Azure Service Bus (inbound) |
| Payroll External | `payment.status.callback` | Reimbursement Service | Webhook → Service Bus |
| Hangfire Worker | Budget alert events | Notification Service | In-process via MediatR |
| All Services → | Structured logs | Azure Log Analytics | App Insights SDK |
| All Services → | Distributed traces | Azure Application Insights | App Insights SDK |

---

## 5. Environment Boundary

| Environment | Purpose | Azure Resources | Access |
|-------------|---------|-----------------|--------|
| Development | Local + shared feature branch dev | Docker Compose (local); shared AKS dev namespace | Engineering team |
| QA | Automated test execution; integration tests | AKS qa namespace; Azure SQL dev tier; Redis Basic | QA team + CI/CD |
| Staging | UAT; performance testing; pre-prod validation | AKS staging; Azure SQL Business Critical (30% capacity); Redis Standard | QA + Business users (UAT) |
| Pre-Production | Final gate; DR testing; load test at full scale | Production-mirror; 100 % capacity; same SKUs as prod | DevOps + Architect |
| Production | Live system | AKS prod (multi-AZ); Azure SQL MI Business Critical; Redis Premium | Operations only |
| DR (Passive) | Warm standby; automated failover | Secondary Azure region; geo-replicated SQL; Redis geo-replication | Auto-failover; manual DR drill |

---

## 6. Security Boundary

```
Internet
    │
    ▼
Azure Front Door (WAF + DDoS protection)
    │
    ▼
Azure APIM (OAuth 2.0 JWT validation · rate limiting 1,000 req/min/user)
    │
    ▼
AKS Ingress (NGINX · TLS 1.3 · HSTS)
    │ (Internal cluster network — no direct internet access)
    ▼
Microservices (managed identity — no stored credentials)
    │
    ▼
Azure SQL MI (Always Encrypted · TDE · RLS · Private endpoint only)
Azure Key Vault (CMK · RBAC · Private endpoint only)
Azure Blob Storage (Private endpoint · signed URLs 15-min TTL)
```

**Role Boundary Summary:**

| Role | Travel | Expense | Reimbursement | Reports | Admin |
|------|--------|---------|---------------|---------|-------|
| Employee | Own only | Own only | Own status | Own history | None |
| Line Manager | Team read + approve | Team read + approve | Team status | Team reports | None |
| Dept Head (L2) | Dept + approve | Dept + approve high-value | Dept status | Dept reports | None |
| Finance Reviewer | All read | All + annotate | Process batches | All financial | None |
| Finance Controller | All read + override | Final approve + override | Trigger payroll | All + config | Budget config |
| HR Admin | All read | Read only | Read only | HR analytics | Full admin |
| System Admin | Read only | Read only | Read only | System config | User management |
| Internal Audit | All read (immutable) | All + download | All read | All + audit logs | None |

---

## 7. KPI Pass/Fail Validation Boundary

All 20 KPI sections must reach **Pass** before Production deployment is authorised. The following table maps each KPI to its technical validation owner and blocking gate:

| KPI Ref | Section | Validation Owner | Blocking Gate |
|---------|---------|-----------------|---------------|
| K1 | Functional Requirements | QA Lead + Business Analyst | UAT sign-off |
| K2 | Non-Functional Requirements | DevOps + Security | Load test report + VAPT report |
| K3 | System Architecture | Architect | Integration smoke tests green |
| K4 | Data Model | DBA | EF migration applied to staging; RI verified |
| K5 | API Design | QA Lead | Postman/Newman collection 100 % green |
| K6 | User Flow | BA + Pilot users | All 4 persona journeys in UAT |
| K7 | UI/UX | Frontend Lead | axe-core: 0 AA violations |
| K8 | Integrations | Integration Engineer | HRMS < 30 min; notifications < 60 s |
| K9 | Security | Security Officer | VAPT: 0 Critical/High |
| K10 | Performance | DevOps | Redis hit > 80 %; auto-scale verified |
| K11 | Observability | DevOps | All alerts firing; dashboards live |
| K12 | Testing | QA Lead | SonarQube ≥ 80 %; Playwright 100 % |
| K13 | Deployment | DevOps | Pipeline green; rollback tested |
| K14 | Risk Mitigation | PM | Training complete; dry-run done |
| K15 | Success Metrics | PM + Finance | 30-day post-launch report |
| K16 | Timeline | PM | Phase milestone dates met |
| K17 | Future Enhancements | Architect | Roadmap in backlog; extensibility confirmed |
| K18 | Compliance | Compliance Officer | Internal Audit sign-off |
| K19 | Audit Framework | DBA + Compliance | Temporal tables active; immutability tested |
| K20 | Data Retention | DBA + Legal | Archival job verified; retrieval < 5 min |
