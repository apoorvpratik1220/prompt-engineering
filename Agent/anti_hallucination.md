# Anti-Hallucination Protocol
**Project:** Enterprise Travel & Expense Management System (ETEP)
**Version:** 1.0 | **Classification:** Internal – Mandatory Compliance
**Applies To:** All team members — Developers, QA, DBA, DevOps, PM, ML/Data, Security

---

## What This Document Is

This protocol defines a strict, zero-tolerance standard for **evidence-based work** across the
entire ETEP project. No team member — human or AI-assisted — may assert, implement, document,
or commit anything that is not backed by a cited, verifiable source from the project's own
approved reference materials.

Hallucination in a software project is not a AI-only risk. It is any act of:
- Assuming a requirement without reading the PRD
- Guessing a schema without checking the data model
- Estimating a KPI without reading `KPI.md`
- Writing a business rule without a documented source
- Closing a test case without a reproducible result

Every claim must have a **proof tag**. No proof tag — no merge.

---

## 1. The Core Rule

> **Nothing is real until it has a source.**

Before any team member writes, speaks, commits, or decides — they must answer:

```
Where is this written?
Which document, which section, which line?
```

If that question cannot be answered in under 60 seconds by pointing to a file
in this repository — **stop. Do not proceed.**

---

## 2. Approved Source Documents

Only the following documents are considered ground truth on ETEP.
All claims must trace back to one of these.

| Source ID | Document | Location | What It Governs |
|-----------|----------|----------|----------------|
| `SRC-PRD` | Product Requirements Document | `docs/PRD_v1.0.md` | Functional requirements, modules, workflows, business rules |
| `SRC-KPI` | KPI Document | `docs/KPI_v1.0.md` | Pass/Fail acceptance criteria, performance targets, compliance benchmarks |
| `SRC-SCOPE` | Project Scope Document | `docs/project_scope.md` | In-scope features, stopping points, KPI constraints, acceptance criteria |
| `SRC-BOUNDARY` | Project Boundary Document | `docs/project_boundary.md` | Folder structure, service map, environment boundaries, RBAC matrix |
| `SRC-TC` | Test Cases Document | `docs/test_cases.md` | All 74 test cases, KPI-to-TC traceability map |
| `SRC-PERSONA` | Persona Document | `docs/persona.md` | Role responsibilities, DoD checklists, ownership maps |
| `SRC-ADR` | Architecture Decision Records | `docs/adr/ADR-*.md` | Technology choices and rationale |
| `SRC-SCHEMA` | Database Schema | `database/schema-diagram.png` + EF Core migrations | Table structures, column types, indexes, RLS predicates |
| `SRC-API` | OpenAPI Specification | `docs/api/openapi.yaml` | Endpoint contracts, request/response schemas, HTTP status codes |
| `SRC-TICKET` | Azure DevOps Work Item | Azure DevOps Boards (linked ticket number) | Sprint-level acceptance criteria, story points, assignee |

---

## 3. Proof Tag Standard

Every piece of work that produces an output — code, comment, test, document, decision —
must carry a **proof tag** linking it to its source.

### 3.1 Format

```
[SOURCE_ID § Section/Line — "exact quoted claim or rule"]
```

### 3.2 Examples

#### In Code (C# — Command Validator)
```csharp
// [SRC-PRD § 1.1.2 — "Travel start date: >= T+2 business days from today"]
RuleFor(x => x.StartDate)
    .Must(date => date >= DateTime.Today.AddBusinessDays(2))
    .WithMessage("start_date must be at least T+2 business days from today.");
```

#### In Code (Angular — Form Validator)
```typescript
// [SRC-PRD § 1.1.2 — "Business Justification: Min 50 chars"]
businessJustification: ['', [Validators.required, Validators.minLength(50)]]
```

#### In a Test Case
```
// TC-TR-002 [SRC-TC § Module 1 — TC-TR-002] [SRC-PRD § 1.1.2 — "Mandatory Fields"]
// [SRC-KPI § 2 — "Mandatory Field Enforcement: Pass"]
```

#### In a SQL Migration
```sql
-- [SRC-SCHEMA § employees table — "BankAccountNo: Always Encrypted, AES-256"]
-- [SRC-PRD § 9.3 — "Encryption: sensitive fields encrypted with AES-256 at application layer"]
ALTER TABLE Employees
    ADD BankAccountNo NVARCHAR(20) ENCRYPTED WITH (
        COLUMN_ENCRYPTION_KEY = CEK_ETEP,
        ENCRYPTION_TYPE = Deterministic,
        ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256'
    );
```

#### In a PR Description
```
## Changes
- Added 90% budget alert logic

## Proof
- [SRC-PRD § 1.5 — "Automated alerts at 75%, 90%, and 100% consumption thresholds"]
- [SRC-KPI § 15 — "Processing Time Reduction: Pass"]
- Implements TC-BG-001 [SRC-TC § Module 5 — TC-BG-001]
```

#### In a Meeting / Decision
```
Decision: Set advance cap at 80% of estimated cost.
Proof: [SRC-PRD § 1.3 — "Maximum advance: 80% of total estimated travel cost approved in TAN"]
Raised by: [Name] | Date: [Date] | Ticket: ETEP-421
```

---

## 4. Triggers That Require a Proof Tag

The following actions are **blocked** without a proof tag.
The blocker is enforced at PR review, standup challenge, or QA gate.

| Action | Proof Required | Blocked By |
|--------|---------------|-----------|
| Writing a FluentValidation rule | `SRC-PRD` section citing the business rule | PR reviewer |
| Writing an Angular form validator | `SRC-PRD` section for the same rule | PR reviewer |
| Adding a new API endpoint | `SRC-API` or `SRC-PRD` defining the endpoint | PR reviewer + API spec |
| Creating a DB column | `SRC-SCHEMA` or `SRC-PRD` citing the field | DBA review |
| Writing a test case | `SRC-TC` test case ID + `SRC-KPI` or `SRC-PRD` | QA lead |
| Closing a test case as Pass | Reproducible evidence (screenshot/log/run ID) | QA lead |
| Deferring a feature | `SRC-SCOPE § 4` — Stopping Points list | PM sign-off |
| Changing a policy limit (e.g., per-diem) | `SRC-PRD` + Finance Controller approval email | PM + Finance |
| Configuring an AKS HPA threshold | `SRC-SCOPE § 3.4` or `SRC-KPI § 10` | DevOps review |
| Quoting a KPI target in a meeting | `SRC-KPI` section number | Standup challenge |
| Raising a risk | `SRC-PRD § 14` or `SRC-SCOPE § 1.2` citing the risk | PM review |
| Declaring a bug P0 | `SRC-TC` severity definition | QA lead |
| Merging to `main` | SonarQube gate + all proof tags on PR | Azure DevOps pipeline gate |

---

## 5. The Five Hallucination Patterns to Eliminate

### H1 — Assumed Requirement
> *"I think the hotel limit for Grade A is ₹10,000."*

**Why it fails:** "I think" is not a source. The limit is defined in `SRC-PRD § 1.2.2`
(Grade A Metro: ₹8,000/night). The code will be wrong. The test will pass the wrong number.
The employee will be over-reimbursed or under-reimbursed.

**Correct behaviour:**
1. Open `docs/PRD_v1.0.md`
2. Navigate to Section 1.2.2 — Expense Category Policy Matrix
3. Read: `Grade Band A (Sr. Mgmt) | Hotel (Metro Cities) | ₹8,000/night`
4. Write the validator with the proof tag
5. Proceed

---

### H2 — Invented API Contract
> *"The endpoint probably returns a 200 with the claim ID in the body."*

**Why it fails:** "Probably" is not a contract. If the frontend assumes 200 and the backend
returns 201, Angular's success handler may silently swallow the response. If the body shape
is assumed, NgRx reducer will assign `undefined` to state fields.

**Correct behaviour:**
1. Open `docs/api/openapi.yaml`
2. Find `POST /api/v1/expenses/claims`
3. Read the defined response schema: `201 Created`, body: `{ claim_id, claim_number, status }`
4. Write the NgRx effect against the documented schema
5. Add proof tag: `[SRC-API § POST /api/v1/expenses/claims — 201 response schema]`

---

### H3 — Guessed Test Result
> *"That test should pass — I ran it locally and it looked fine."*

**Why it fails:** "Looked fine" is not a test result. A test case is only closed as Pass
when it produces a documented, reproducible artefact: a pipeline run ID, a screenshot,
a Newman JSON report, a k6 summary. "Locally" is not reproducible by the QA lead,
the PM, or the auditor.

**Correct behaviour:**
1. Run the test in the Azure DevOps pipeline (not locally)
2. Link the pipeline run URL to the test case in Azure DevOps
3. Attach the output (Newman report, Playwright trace, k6 summary) as a test result artefact
4. Mark Pass only after the QA lead reviews the artefact

---

### H4 — Undocumented Stopping Point
> *"We can add the airline booking integration — it won't take long."*

**Why it fails:** Airline booking integration is explicitly listed as a Stopping Point
in `SRC-SCOPE § 4 — S1`. Adding it without a formal change request re-opens scope,
risks the Phase 3 deadline, and bypasses the vendor API contract agreements that are
the documented reason for deferral.

**Correct behaviour:**
1. Check `docs/project_scope.md § 4` before starting any feature
2. If the feature is listed (S1–S10), stop
3. Raise a formal change request with PM
4. PM gets CFO approval and updates `SRC-SCOPE`
5. Only then does the feature enter the backlog

---

### H5 — Role Assumption Without RBAC Check
> *"Finance users can see all employee data — they need it for reporting."*

**Why it fails:** Finance Reviewer access is defined in `SRC-BOUNDARY § 6 — Security
Boundary` and enforced by SQL Row-Level Security. "They need it" is not a documented
permission. Granting unauthorised access to PAN or bank account data is a GDPR violation
and a SOX audit finding.

**Correct behaviour:**
1. Open `docs/project_boundary.md § 6`
2. Locate Finance Reviewer row in the RBAC matrix
3. Read the defined access: `Expense Claims: All read + annotate`
4. Check the RLS predicate in `database/rls/fn_securitypredicate.sql`
5. If the access is not defined, raise a Security Change Request to the Security Engineer

---

## 6. PR Review Checklist (Anti-Hallucination Gates)

Every pull request must pass this checklist before a reviewer approves it.
If any item is unchecked, the PR is **returned without review**.

```markdown
## Anti-Hallucination PR Checklist

### Source Traceability
- [ ] Every new business rule has a [SRC-PRD § X.X] proof tag in the code comment
- [ ] Every new API endpoint matches a definition in `docs/api/openapi.yaml` [SRC-API]
- [ ] Every new DB column matches `SRC-SCHEMA` or a cited PRD field
- [ ] Every new test case references a TC-ID from `docs/test_cases.md` [SRC-TC]
- [ ] Every KPI-related change references the KPI section from `docs/KPI_v1.0.md` [SRC-KPI]

### No Assumed Behaviour
- [ ] No TODO comments that defer a business rule decision ("// TODO: check limit")
- [ ] No hardcoded values without a PRD citation (e.g., `50000` must cite SRC-PRD § 1.1)
- [ ] No "assumed" HTTP status codes — all verified against `openapi.yaml`
- [ ] No cross-layer contract assumptions — DTOs match OpenAPI spec exactly

### Stopping Points
- [ ] This PR does not implement any feature listed in `SRC-SCOPE § 4` (S1–S10)
- [ ] If a deferred feature is touched, a PM-approved change request ticket is linked

### Test Evidence
- [ ] All new test cases have a linked Azure DevOps pipeline run (not local only)
- [ ] No test case is marked Pass without an attached artefact (report / trace / screenshot)
- [ ] Performance test numbers are from a k6 run, not estimated

### Security & Data
- [ ] No new data field is stored without citing its encryption/masking rule from SRC-PRD § 9
- [ ] No new role permission is granted without citing the RBAC matrix from SRC-BOUNDARY § 6
```

---

## 7. Meeting & Communication Protocol

### In Standups
When a team member makes a claim about a requirement, limit, or timeline:

| Trigger Phrase | Required Response |
|---------------|------------------|
| *"I think the rule is..."* | "Which section of the PRD?" |
| *"It should work like..."* | "Is that in the OpenAPI spec?" |
| *"The limit is around..."* | "What does KPI.md say exactly?" |
| *"We can probably ship..."* | "What does the Phase milestone say in project_scope.md?" |
| *"The test passed..."* | "Which pipeline run ID?" |

These are not hostile questions. They are the standard of professional accuracy
on a system that handles ₹15–20 Cr in annual financial transactions.

### In Design Sessions
Before proposing any technical decision:
1. State which source document supports the decision
2. Quote the relevant section
3. Record the decision in an ADR (`docs/adr/ADR-NNN-title.md`) with source citations
4. Link the ADR to the Azure DevOps epic

### In Demos / Sprint Reviews
All demo data must come from the Staging environment.
No demo may use hardcoded or mocked data to illustrate a feature.
If Staging is unavailable, the demo is rescheduled — not faked.

---

## 8. AI-Assisted Work Protocol

When any team member uses an AI assistant (GitHub Copilot, ChatGPT, Claude, etc.)
to generate code, documentation, or test cases for ETEP:

| Step | Action |
|------|--------|
| **Before prompting** | Paste the relevant PRD section or schema into the prompt. Never ask the AI to "guess" or "assume" requirements |
| **After receiving output** | Verify every business rule, limit, field name, and endpoint against the approved source documents |
| **Before committing** | Replace any AI-generated assumption with a proof-tagged, source-verified statement |
| **Never accept** | AI-generated database column names, API field names, or limit values without checking SRC-SCHEMA and SRC-PRD |
| **Never trust** | AI recall of version numbers, library APIs, or Azure service configurations — verify against official docs |

**The AI is a drafting tool. The source documents are the truth.**

---

## 9. Violation Handling

### What Counts as a Violation

| Violation | Example | Consequence |
|-----------|---------|------------|
| **V1 — Untagged Rule** | FluentValidation rule with no SRC-PRD citation | PR returned; tag required before re-review |
| **V2 — Assumed Limit** | Hardcoded `₹10000` without PRD reference | PR returned; correct value + proof tag required |
| **V3 — Fabricated Test Pass** | Test marked Pass with no pipeline artefact | Test reverted to Open; pipeline run required |
| **V4 — Scope Creep** | Implementing a deferred feature (S1–S10) without change request | Work reverted; PM escalation; change request mandatory |
| **V5 — Role Assumption** | Granting a permission not in the RBAC matrix | Code reverted; Security Engineer review required |
| **V6 — Undocumented Decision** | Architecture change with no ADR | PR blocked; ADR must be authored and merged first |

### Process
1. Reviewer or QA lead flags the violation with the violation code (V1–V6)
2. Author acknowledges in PR / standup — no defensiveness
3. Author corrects and adds proof tags within the same sprint
4. Repeat violations in the same sprint are escalated to PM
5. Violations that reach production are treated as incidents with a post-mortem

---

## 10. Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│              ETEP ANTI-HALLUCINATION QUICK REFERENCE            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Before you write ANY code, test, or decision — ask:           │
│                                                                 │
│    1. WHERE is this requirement written?                        │
│       → docs/PRD_v1.0.md  [SRC-PRD]                           │
│                                                                 │
│    2. WHAT does the KPI document say about this?               │
│       → docs/KPI_v1.0.md  [SRC-KPI]                           │
│                                                                 │
│    3. IS this feature in scope?                                 │
│       → docs/project_scope.md § 4 (Stopping Points)           │
│         [SRC-SCOPE]                                             │
│                                                                 │
│    4. WHAT does the API contract say?                           │
│       → docs/api/openapi.yaml  [SRC-API]                       │
│                                                                 │
│    5. WHAT does the schema say?                                 │
│       → database/ + EF Core migrations  [SRC-SCHEMA]           │
│                                                                 │
│  If you cannot answer all 5 in 60 seconds: STOP AND READ.     │
│                                                                 │
│  Proof tag format:                                              │
│  [SRC-PRD § 1.2.2 — "Hotel Metro Grade-A: ₹8,000/night"]      │
│                                                                 │
│  No proof tag = No merge. No exceptions.                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 11. Document Maintenance

| Responsibility | Owner | Frequency |
|---------------|-------|-----------|
| Update approved source list when new docs are added | Project Manager | On every new baseline document |
| Review violation log and report to team | QA Lead | Weekly in sprint review |
| Audit PR history for untagged rules | Security Engineer | Monthly |
| Retrain team on this protocol | Project Manager | Start of each phase (Phase 1, 2, 3) |
| Update Quick Reference Card if sources change | Full Stack Lead | On every document version bump |
