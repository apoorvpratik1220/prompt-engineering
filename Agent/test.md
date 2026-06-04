# Test Cases — Enterprise Travel & Expense Management System (ETEP)
**Version:** 1.0 | **Stack:** ASP.NET Core .NET 8 · Angular 17 · Azure SQL MI · AKS
**References:** PRD v1.0 · KPI Document v1.0
**Test Framework:** xUnit (.NET 8) · Jasmine/Karma · Playwright · Postman/Newman

---

## Legend

| Symbol | Meaning |
|--------|---------|
| ✅ | Expected Pass |
| ❌ | Expected Fail / System must reject |
| 🔒 | Security / Auth test |
| ⚡ | Performance test |
| 🔄 | Integration test |
| 🧩 | Edge case |

---

## Module 1 — Travel Request Management
> **KPI Ref:** F1 (Workflow Accuracy), F2 (Mandatory Field Enforcement), F3 (Exception Handling)

---

### TC-TR-001 · Submit valid domestic travel request
| Field | Value |
|-------|-------|
| **Type** | Functional – Happy Path |
| **Priority** | P0 |
| **Precondition** | Employee logged in via Azure AD SSO; active grade band B; department budget ₹2,00,000 remaining |
| **Steps** | 1. Navigate to Travel → New Request 2. Select Purpose = `CLIENT_MEETING` 3. Set Origin = `Mumbai`, Destination = `Delhi` 4. Set Start = T+3 days, End = T+5 days 5. Select Mode = `Air`, `Cab` 6. Enter Estimated Cost = ₹32,000 7. Set Advance Required = ₹25,000 8. Enter Project Code = `PROJ-2025-042` 9. Enter Justification (60 chars) 10. Click Submit |
| **Expected Result** | HTTP 201; status = `PENDING_L1`; TAN = null; L1 approver notified via email + push within 60 s; real-time budget gauge decreases by ₹32,000 |
| **KPI Validated** | F1, F2, NF10 |

---

### TC-TR-002 · Submit request with missing mandatory fields
| Field | Value |
|-------|-------|
| **Type** | Functional – Negative |
| **Priority** | P0 |
| **Precondition** | Employee logged in |
| **Steps** | 1. Submit travel request with blank `destination_city` and blank `business_justification` |
| **Expected Result** | ❌ HTTP 422; RFC 7807 ProblemDetails with two field-level errors: `"destination_city: required"`, `"business_justification: minimum 50 characters"`; form stays open; no DB record created |
| **KPI Validated** | F2, F3 |

---

### TC-TR-003 · Travel start date in the past
| Field | Value |
|-------|-------|
| **Type** | Functional – Negative |
| **Priority** | P0 |
| **Steps** | Submit request with `start_date = today - 1 day` |
| **Expected Result** | ❌ FluentValidation blocks: `"start_date must be at least T+2 business days from today"` |
| **KPI Validated** | F2 |

---

### TC-TR-004 · International travel without visa document
| Field | Value |
|-------|-------|
| **Type** | Functional – Negative |
| **Priority** | P0 |
| **Steps** | 1. Set destination = `Singapore` (international flag auto-detected) 2. Submit without uploading visa document |
| **Expected Result** | ❌ HTTP 422; error = `"visa_document: required for international travel"`; additional L2 approval flag set |
| **KPI Validated** | F1, F2 |

---

### TC-TR-005 · Estimated cost exceeds department budget
| Field | Value |
|-------|-------|
| **Type** | Functional – Edge Case 🧩 |
| **Priority** | P1 |
| **Precondition** | Department remaining budget = ₹10,000 |
| **Steps** | Submit request with `estimated_cost = ₹15,000` |
| **Expected Result** | ❌ HTTP 422; error = `"estimated_cost exceeds remaining department budget of ₹10,000"`; real-time budget gauge shows 0 remaining |
| **KPI Validated** | F1, F2 |

---

### TC-TR-006 · Blackout date enforcement
| Field | Value |
|-------|-------|
| **Type** | Functional – Edge Case 🧩 |
| **Priority** | P1 |
| **Precondition** | HR Admin has configured March 28–31 as blackout (year-end close) |
| **Steps** | Submit request with `start_date = March 29` |
| **Expected Result** | ❌ HTTP 422; error = `"travel dates conflict with blackout period: March 28–31 (Year-End Close)"`; CFO approval route shown as alternative |
| **KPI Validated** | F1 |

---

### TC-TR-007 · L1 approval — happy path
| Field | Value |
|-------|-------|
| **Type** | Functional – Happy Path |
| **Priority** | P0 |
| **Precondition** | Request in `PENDING_L1`; estimated cost = ₹30,000 (domestic, below L2 threshold) |
| **Steps** | 1. Manager opens approval queue 2. Reviews request detail 3. Clicks Approve with comment |
| **Expected Result** | Status → `APPROVED`; TAN generated (`TAN-2025-000042`); employee notified via email + SignalR push; advance disbursal initiated |
| **KPI Validated** | F1, NF10 |

---

### TC-TR-008 · Auto-escalation to L2 — cost > ₹50,000
| Field | Value |
|-------|-------|
| **Type** | Functional – Business Rule |
| **Priority** | P0 |
| **Precondition** | Request estimated cost = ₹75,000 |
| **Steps** | L1 manager approves |
| **Expected Result** | Status → `PENDING_L2`; Dept Head notified within 60 s; L1 action recorded in approval history |
| **KPI Validated** | F1 |

---

### TC-TR-009 · Approver on leave — auto-delegate
| Field | Value |
|-------|-------|
| **Type** | Functional – Edge Case 🧩 |
| **Priority** | P1 |
| **Precondition** | L1 approver has active leave in HRMS; delegate configured |
| **Steps** | Submit new travel request → system checks HRMS leave API |
| **Expected Result** | Request routed to delegate approver; original approver's name shown as "On Leave (Delegated to [Name])"; notification sent to delegate |
| **KPI Validated** | F1, NF9 |

---

### TC-TR-010 · SLA breach — 24-hour approver inaction
| Field | Value |
|-------|-------|
| **Type** | Functional – Edge Case 🧩 |
| **Priority** | P1 |
| **Precondition** | Request in `PENDING_L1` for 20 hours |
| **Steps** | Hangfire `SlaMonitorJob` fires at T+20h (warning), T+24h (escalation) |
| **Expected Result** | At T+20h: warning email to L1 approver. At T+24h: auto-escalate to skip-level; SLA breach logged with `sla_breached = true` in approvals table |
| **KPI Validated** | F1, K6 |

---

### TC-TR-011 · Duplicate overlapping travel request
| Field | Value |
|-------|-------|
| **Type** | Functional – Edge Case 🧩 |
| **Priority** | P1 |
| **Steps** | Employee has active request for March 15–17; submits another for March 16–19 |
| **Expected Result** | ❌ HTTP 409; error = `"Overlapping travel request exists for March 15–17 (TAN pending)"`; no duplicate created |
| **KPI Validated** | F1 |

---

### TC-TR-012 · Cancellation before travel
| Field | Value |
|-------|-------|
| **Type** | Functional |
| **Priority** | P1 |
| **Precondition** | Request in `APPROVED` status; travel start > 12 hours away |
| **Steps** | Employee clicks Cancel → Manager confirms |
| **Expected Result** | Status → `CANCELLED`; advance recovery workflow triggered if advance was disbursed; budget restored; cancellation audit log entry created |
| **KPI Validated** | F1, K19 |

---

## Module 2 — Expense Claim Management
> **KPI Ref:** F1, F2, F3, F4, F6

---

### TC-EC-001 · Submit valid expense claim linked to TAN
| Field | Value |
|-------|-------|
| **Type** | Functional – Happy Path |
| **Priority** | P0 |
| **Precondition** | TAN = `TAN-2025-000042` in `COMPLETED` status; advance paid = ₹25,000 |
| **Steps** | 1. Open Expense → New Claim → Link TAN 2. Add line item: category=`HOTEL`, date=travel date, vendor=`Taj Palace`, amount=₹4,800, upload receipt 3. Add line item: category=`CAB`, amount=₹1,200 4. Run Policy Check 5. Submit |
| **Expected Result** | HTTP 201; claim number `ECL-2025-000101`; `total_claimed = ₹6,000`; `advance_adjusted = ₹6,000`; `net_reimbursable = ₹0`; status = `PENDING_L1` |
| **KPI Validated** | F1, F2 |

---

### TC-EC-002 · OCR receipt extraction — high confidence
| Field | Value |
|-------|-------|
| **Type** | Functional – Integration 🔄 |
| **Priority** | P0 |
| **Steps** | 1. Upload clear JPG receipt (₹4,800 hotel bill with GSTIN) 2. Wait for OCR callback |
| **Expected Result** | Azure AI Document Intelligence returns: `amount=4800`, `vendor="Taj Palace"`, `gstin="27AABCT3518Q1ZV"`, `date="2025-03-15"`, `confidence ≥ 0.85`; fields auto-populated in form; no manual correction prompt |
| **KPI Validated** | F4 |

---

### TC-EC-003 · OCR receipt — low confidence fallback
| Field | Value |
|-------|-------|
| **Type** | Functional – Edge Case 🧩 |
| **Priority** | P1 |
| **Steps** | Upload blurred/low-resolution receipt image |
| **Expected Result** | OCR returns `confidence = 0.60`; fields shown as pre-filled but highlighted in amber with label "Please verify — low confidence extraction"; employee must manually confirm each flagged field before saving |
| **KPI Validated** | F4 |

---

### TC-EC-004 · Duplicate receipt detection
| Field | Value |
|-------|-------|
| **Type** | Functional – Security 🔒 |
| **Priority** | P0 |
| **Steps** | 1. Employee uploads receipt-A.jpg for claim ECL-001 2. Same employee uploads identical receipt-A.jpg in new claim ECL-002 |
| **Expected Result** | ❌ HTTP 422; error = `"Duplicate receipt detected — this image was submitted in claim ECL-001 (2025-03-10)"`; upload blocked; pHash comparison logged |
| **KPI Validated** | F6, K9 |

---

### TC-EC-005 · Cross-employee duplicate receipt
| Field | Value |
|-------|-------|
| **Type** | Security 🔒 – Edge Case 🧩 |
| **Priority** | P0 |
| **Steps** | Employee B uploads same receipt image previously submitted by Employee A |
| **Expected Result** | ❌ Flagged with fraud score increment; Finance alert raised; claim held in `FRAUD_REVIEW` status |
| **KPI Validated** | F6, K9 |

---

### TC-EC-006 · Policy check — hotel exceeds grade-band limit
| Field | Value |
|-------|-------|
| **Type** | Functional – Business Rule |
| **Priority** | P0 |
| **Precondition** | Employee grade = B; hotel limit = ₹5,000/night |
| **Steps** | Add hotel line item: amount = ₹7,500 |
| **Expected Result** | Policy check P2 fails: `"Hotel ₹7,500 exceeds Grade-B Metro limit of ₹5,000/night"`; line item highlighted red; employee can reduce or request Finance exception |
| **KPI Validated** | F1, F2 |

---

### TC-EC-007 · Policy check — expense date outside travel window
| Field | Value |
|-------|-------|
| **Type** | Functional – Business Rule |
| **Priority** | P0 |
| **Steps** | Add line item with `expense_date` 5 days after `travel_end_date` |
| **Expected Result** | Policy check P1 fails: `"Expense date 2025-03-22 is outside travel window March 15–17 (buffer: ±1 day)"`; item blocked |
| **KPI Validated** | F1 |

---

### TC-EC-008 · GST validation — valid GSTIN
| Field | Value |
|-------|-------|
| **Type** | Integration 🔄 |
| **Priority** | P1 |
| **Steps** | Enter GSTIN = `27AABCT3518Q1ZV` on hotel line item |
| **Expected Result** | GST Portal API returns valid; vendor name confirmed = `Indian Hotels Company Ltd`; ITC eligible flag set; response cached 24 hours |
| **KPI Validated** | F1, K18 |

---

### TC-EC-009 · GST validation — invalid GSTIN
| Field | Value |
|-------|-------|
| **Type** | Functional – Negative |
| **Priority** | P1 |
| **Steps** | Enter GSTIN = `27XXXXXX0000X0XX` |
| **Expected Result** | ❌ GST API returns invalid; inline error = `"GSTIN not found in government registry"`; ITC not eligible; employee warned before proceeding |
| **KPI Validated** | F1 |

---

### TC-EC-010 · Mileage claim — GPS distance validation
| Field | Value |
|-------|-------|
| **Type** | Functional – Business Rule |
| **Priority** | P1 |
| **Steps** | Employee claims personal vehicle mileage: entered distance = 150 km; GPS-calculated distance (Mumbai → Pune) = 148 km |
| **Expected Result** | Within 10 % tolerance → accepted. If entered = 200 km → ❌ P10 violation: `"Entered distance 200 km exceeds GPS-calculated 148 km by > 10%"` |
| **KPI Validated** | F1 |

---

### TC-EC-011 · Fraud scoring — high risk claim
| Field | Value |
|-------|-------|
| **Type** | Functional – Security 🔒 |
| **Priority** | P0 |
| **Precondition** | ML fraud service running; claim submitted on weekend with 18 line items |
| **Steps** | Submit claim with anomalous pattern: 18 line items, all round numbers, weekend submission, 3× employee's monthly average |
| **Expected Result** | ML service returns `fraud_score = 78`; claim status set to `FRAUD_REVIEW`; Finance team alerted via email + Azure Service Bus event; claim cannot be auto-approved |
| **KPI Validated** | F1, K9, K17 |

---

### TC-EC-012 · Net reimbursable calculation — advance > claimed
| Field | Value |
|-------|-------|
| **Type** | Functional – Business Rule |
| **Priority** | P0 |
| **Steps** | `total_claimed = ₹15,000`; `advance_paid = ₹20,000` |
| **Expected Result** | `net_reimbursable = ₹-5,000`; UI shows "Recovery Required: ₹5,000 will be deducted from next salary"; recovery record created in `AdvanceRecovery` table |
| **KPI Validated** | F1 |

---

### TC-EC-013 · Draft claim — 30-day auto-purge
| Field | Value |
|-------|-------|
| **Type** | Functional – Edge Case 🧩 |
| **Priority** | P1 |
| **Precondition** | Draft claim created 30 days ago with no activity |
| **Steps** | Hangfire `PurgeStaleDraftsJob` fires at 2:00 AM |
| **Expected Result** | Draft soft-deleted; employee receives email: `"Draft claim ECL-DRAFT-099 has been purged after 30 days of inactivity"`; record moved to archive with `status = PURGED` |
| **KPI Validated** | F3, K20 |

---

### TC-EC-014 · Alcohol expense auto-rejection
| Field | Value |
|-------|-------|
| **Type** | Functional – Business Rule |
| **Priority** | P0 |
| **Steps** | Add meal line item; OCR extracts description containing keyword "Beer", "Whisky" |
| **Expected Result** | ❌ Policy check P9 auto-rejects line item: `"Alcohol expenses are not reimbursable per company policy"`; item cannot be submitted |
| **KPI Validated** | F1 |

---

## Module 3 — Reimbursement Processing
> **KPI Ref:** K5 (Processing Time), K8 (Payroll Integration), K15 (Success Metrics)

---

### TC-RM-001 · Nightly batch creation
| Field | Value |
|-------|-------|
| **Type** | Functional – Integration 🔄 |
| **Priority** | P0 |
| **Precondition** | 15 claims in `FINANCE_APPROVED` status; batch cutoff 11:59 PM IST |
| **Steps** | Hangfire `ReimbursementBatchJob` fires at midnight |
| **Expected Result** | Batch record created; all 15 claims grouped under `BATCH-2025-031`; batch status = `PENDING_CONTROLLER_REVIEW`; Finance Controller dashboard shows batch summary |
| **KPI Validated** | F1, K8 |

---

### TC-RM-002 · Finance Controller approves batch → payroll push
| Field | Value |
|-------|-------|
| **Type** | Functional – Happy Path |
| **Priority** | P0 |
| **Steps** | 1. Finance Controller reviews batch summary 2. Confirms totals 3. Clicks "Push to Payroll" |
| **Expected Result** | Azure Service Bus message published to `reimbursement.events` topic; Payroll API receives payment file; batch status = `PROCESSING`; each employee's claim status = `PAYMENT_PENDING` |
| **KPI Validated** | F1, K8 |

---

### TC-RM-003 · Payment confirmation webhook
| Field | Value |
|-------|-------|
| **Type** | Integration 🔄 |
| **Priority** | P0 |
| **Steps** | Payroll system sends callback: `{ "batchId": "BATCH-2025-031", "status": "PAID", "processedAt": "2025-03-12T14:30:00Z" }` |
| **Expected Result** | All claims in batch → `PAID`; employees notified via SignalR (in-app), email, SMS within 30 s; `paid_at` timestamp recorded |
| **KPI Validated** | F1, NF10, K8 |

---

### TC-RM-004 · Payment failure — auto-retry
| Field | Value |
|-------|-------|
| **Type** | Functional – Edge Case 🧩 |
| **Priority** | P0 |
| **Steps** | Payroll API returns `{ "status": "FAILED", "reason": "INVALID_IFSC" }` for 1 employee in batch |
| **Expected Result** | Failed record: `retry_count = 1`; retry queued after 1 hour; after 3 failures: `status = PAYMENT_FAILED`; Finance Controller alerted; other employees in batch unaffected |
| **KPI Validated** | F1, F3 |

---

### TC-RM-005 · Employee with unreconciled advance blocked from new advance
| Field | Value |
|-------|-------|
| **Type** | Functional – Business Rule |
| **Priority** | P1 |
| **Precondition** | Employee has 2 unreconciled advances > 48 hours post travel end |
| **Steps** | Employee submits new travel request with `advance_required = ₹10,000` |
| **Expected Result** | ❌ HTTP 422: `"New advance blocked — 2 prior advances unreconciled. Please submit expense claims for TAN-2025-000035 and TAN-2025-000038 first."` |
| **KPI Validated** | F5 |

---

## Module 4 — Reporting & Analytics
> **KPI Ref:** K6 (Finance Journey), K15, K18

---

### TC-RP-001 · Travel spend report — department filter
| Field | Value |
|-------|-------|
| **Type** | Functional |
| **Priority** | P1 |
| **Steps** | Finance opens Reports → Travel Spend → Filter by `dept = Engineering`, `period = Q4 2025` |
| **Expected Result** | Report renders < 10 s; data pulled from Azure SQL read replica; shows: total spend, budget utilisation %, top 5 employees by spend, category breakdown chart |
| **KPI Validated** | K6, K10 |

---

### TC-RP-002 · Policy violation report
| Field | Value |
|-------|-------|
| **Type** | Functional |
| **Priority** | P1 |
| **Steps** | Finance opens Policy Violation Report for last 30 days |
| **Expected Result** | Lists all 14-point policy check failures with: employee name, claim number, violation type, amount over limit, resolution (approved exception / reduced / rejected) |
| **KPI Validated** | K6, K18 |

---

### TC-RP-003 · XLSX export — async job
| Field | Value |
|-------|-------|
| **Type** | Functional – Integration 🔄 |
| **Priority** | P1 |
| **Steps** | 1. Click Export → XLSX on a 10,000-row report 2. System starts async job 3. User navigates away |
| **Expected Result** | Job ID returned immediately; SignalR push when complete; signed Azure Blob URL (15-min TTL) provided for download; file virus-scanned before link issued |
| **KPI Validated** | F1, K6 |

---

### TC-RP-004 · Advance outstanding report
| Field | Value |
|-------|-------|
| **Type** | Functional |
| **Priority** | P1 |
| **Steps** | Finance opens Advance Outstanding Report |
| **Expected Result** | Lists employees with advance unpaid > 30 days; columns: employee, TAN, advance amount, disbursement date, days outstanding; sort by days outstanding descending |
| **KPI Validated** | K6 |

---

## Module 5 — Budget Management
> **KPI Ref:** F1, K15

---

### TC-BG-001 · Budget alert at 90 % consumption
| Field | Value |
|-------|-------|
| **Type** | Functional – Integration 🔄 |
| **Priority** | P1 |
| **Precondition** | Department budget = ₹5,00,000; utilised = ₹4,40,000 (88 %) |
| **Steps** | New travel request approved for ₹12,000 → utilisation hits 90.4 % |
| **Expected Result** | Azure Service Bus `budget.alert.90pct` event published; Dept Head + Finance Controller emailed: `"Engineering dept budget 90.4% consumed (₹4,52,000 / ₹5,00,000)"`; real-time gauge updates on Finance dashboard |
| **KPI Validated** | F1, K11 |

---

### TC-BG-002 · Budget freeze by Finance Controller
| Field | Value |
|-------|-------|
| **Type** | Functional |
| **Priority** | P1 |
| **Steps** | Finance Controller freezes Engineering dept budget |
| **Expected Result** | All new travel requests for Engineering return ❌ HTTP 403: `"Department budget currently frozen by Finance. Contact finance@company.com"`; existing approved requests unaffected |
| **KPI Validated** | F1 |

---

## Module 6 — Administration
> **KPI Ref:** K7 (HR Admin Journey), K18

---

### TC-AD-001 · HR Admin updates per-diem rates
| Field | Value |
|-------|-------|
| **Type** | Functional |
| **Priority** | P1 |
| **Steps** | HR Admin navigates to Policy Config → Per Diem → Grade B Metro → Update from ₹1,800 to ₹2,000 → Save |
| **Expected Result** | Change saved; Redis cache for `policy.perdiem` invalidated immediately; new requests use updated rate; change logged in audit_logs with `actor_id`, `old_value`, `new_value`, `timestamp` |
| **KPI Validated** | K7, K19 |

---

### TC-AD-002 · Blackout calendar configuration
| Field | Value |
|-------|-------|
| **Type** | Functional |
| **Priority** | P1 |
| **Steps** | HR Admin adds blackout: March 28–31, 2026, label = "Year-End Close" |
| **Expected Result** | Blackout saved; immediately effective for all new requests; calendar UI shows blocked dates in red; existing approved travel in that window not affected |
| **KPI Validated** | K7 |

---

## Security Test Cases
> **KPI Ref:** K9 (Security), NF3, NF4, NF6, NF7, NF8

---

### TC-SEC-001 · Unauthenticated API access
| Field | Value |
|-------|-------|
| **Type** | Security 🔒 |
| **Priority** | P0 |
| **Steps** | Call `GET /api/v1/travel/requests` with no Authorization header |
| **Expected Result** | ❌ HTTP 401; `WWW-Authenticate: Bearer` header present; no data returned |
| **KPI Validated** | K9 |

---

### TC-SEC-002 · Employee accessing another employee's claim
| Field | Value |
|-------|-------|
| **Type** | Security 🔒 |
| **Priority** | P0 |
| **Steps** | Employee A calls `GET /api/v1/expenses/claims/{claimId_of_Employee_B}` with valid JWT |
| **Expected Result** | ❌ HTTP 403; Row-Level Security (SQL predicate `fn_securitypredicate`) blocks query at DB level; returns `"Access denied"` — no data leakage |
| **KPI Validated** | K9, NF6 |

---

### TC-SEC-003 · Self-approval prevention
| Field | Value |
|-------|-------|
| **Type** | Security 🔒 |
| **Priority** | P0 |
| **Steps** | Manager who submitted their own claim attempts to call `POST /api/v1/expenses/claims/{id}/approve` |
| **Expected Result** | ❌ HTTP 403; policy check P12: `"Self-approval is not permitted"`; attempt logged in audit_log with `action = SELF_APPROVAL_ATTEMPT` |
| **KPI Validated** | K9, K19 |

---

### TC-SEC-004 · JWT token tampering
| Field | Value |
|-------|-------|
| **Type** | Security 🔒 |
| **Priority** | P0 |
| **Steps** | Modify JWT payload claim `role = "FinanceController"` and re-sign with incorrect key |
| **Expected Result** | ❌ HTTP 401; Azure AD MSAL token validation rejects invalid signature; access denied |
| **KPI Validated** | K9 |

---

### TC-SEC-005 · TLS version enforcement
| Field | Value |
|-------|-------|
| **Type** | Security 🔒 |
| **Priority** | P0 |
| **Steps** | Attempt HTTPS connection using TLS 1.2 to Azure APIM endpoint |
| **Expected Result** | ❌ Connection rejected; only TLS 1.3 accepted; HSTS header present in all responses |
| **KPI Validated** | NF4 |

---

### TC-SEC-006 · Rate limiting enforcement
| Field | Value |
|-------|-------|
| **Type** | Security 🔒 |
| **Priority** | P1 |
| **Steps** | Send 1,001 API requests within 1 minute from same authenticated user |
| **Expected Result** | Request 1,001 returns HTTP 429; `Retry-After` header present; APIM throttle policy applied; no service degradation for other users |
| **KPI Validated** | K9, NF1 |

---

### TC-SEC-007 · GDPR data erasure
| Field | Value |
|-------|-------|
| **Type** | Compliance 🔒 |
| **Priority** | P0 |
| **Steps** | HR Admin submits erasure request for offboarded employee |
| **Expected Result** | Within 72 hours: `full_name → "EMP_ANON_[hash]"`, `email → null`, `mobile → null`, `bank_account_no → null`, `pan_number → null`; financial amounts and dates retained; anonymisation logged; audit_log shows erasure event |
| **KPI Validated** | NF8, K18 |

---

### TC-SEC-008 · SQL injection attempt
| Field | Value |
|-------|-------|
| **Type** | Security 🔒 |
| **Priority** | P0 |
| **Steps** | Submit `business_justification = "'; DROP TABLE TravelRequests; --"` |
| **Expected Result** | ❌ FluentValidation rejects on field length/pattern; EF Core parameterised queries prevent execution; input sanitised; attempt logged |
| **KPI Validated** | K9 |

---

## Performance Test Cases
> **KPI Ref:** NF1, NF2, K10, K12

---

### TC-PF-001 · API response time — P95 under normal load
| Field | Value |
|-------|-------|
| **Type** | Performance ⚡ |
| **Priority** | P0 |
| **Tool** | k6 |
| **Steps** | Run k6 script: 2,000 virtual users, 10-minute sustained load, mix of GET + POST endpoints |
| **Expected Result** | ✅ P95 response time < 2,000 ms; P99 < 3,000 ms; error rate < 0.1 %; Redis cache hit rate > 80 % |
| **KPI Validated** | NF1, K10 |

---

### TC-PF-002 · Concurrency — 20,000 simultaneous users
| Field | Value |
|-------|-------|
| **Type** | Performance ⚡ |
| **Priority** | P0 |
| **Tool** | Azure Load Testing (k6 hosted) |
| **Steps** | Ramp to 20,000 VUs over 15 minutes; sustain for 30 minutes; mix: 60 % reads, 30 % claim submissions, 10 % approvals |
| **Expected Result** | ✅ No HTTP 5xx errors; P95 < 2 s throughout; AKS HPA scales pods within 90 s of CPU ≥ 70 %; SQL connection pool never exhausted |
| **KPI Validated** | NF2, K10 |

---

### TC-PF-003 · Month-end peak — 3× normal load
| Field | Value |
|-------|-------|
| **Type** | Performance ⚡ |
| **Priority** | P1 |
| **Steps** | Simulate 6,000 concurrent expense claim submissions (month-end rush) |
| **Expected Result** | ✅ Azure Service Bus queues messages without loss; Hangfire batch job completes within 30-min window; no data loss; all claims persisted |
| **KPI Validated** | NF2, K10 |

---

### TC-PF-004 · Dashboard load time
| Field | Value |
|-------|-------|
| **Type** | Performance ⚡ |
| **Priority** | P1 |
| **Steps** | Finance Controller opens main dashboard (1,000 concurrent Finance users) |
| **Expected Result** | ✅ Dashboard renders < 1,500 ms; widgets populated from Redis cache; no full SQL scan; Lighthouse performance score ≥ 85 |
| **KPI Validated** | NF1, K10 |

---

### TC-PF-005 · Redis cache — reference data TTL
| Field | Value |
|-------|-------|
| **Type** | Performance ⚡ |
| **Priority** | P1 |
| **Steps** | 1. Make 1,000 requests for `GET /api/v1/policy/entitlements` 2. Check App Insights cache hit ratio |
| **Expected Result** | ✅ Cache hit rate ≥ 95 % after warm-up; SQL read replica query count near zero for reference data endpoints |
| **KPI Validated** | K10 |

---

## Integration Test Cases
> **KPI Ref:** K8 (Integrations), NF9, NF10

---

### TC-INT-001 · HRMS employee sync — org change
| Field | Value |
|-------|-------|
| **Type** | Integration 🔄 |
| **Priority** | P0 |
| **Steps** | Trigger `hrms.employee.changed` event on Azure Service Bus (employee transfers department) |
| **Expected Result** | Consumer processes event within 30 minutes; employee's `department_id`, `reporting_manager`, `cost_centre_id` updated in ETEP DB; old approvals unaffected; new requests use updated hierarchy |
| **KPI Validated** | K8, NF9 |

---

### TC-INT-002 · SSO login via Azure AD
| Field | Value |
|-------|-------|
| **Type** | Integration 🔄 |
| **Priority** | P0 |
| **Steps** | Navigate to ETEP → redirected to Azure AD login → enter valid corporate credentials + MFA |
| **Expected Result** | ✅ JWT issued; roles synced from Azure AD groups (e.g., `ETEP-FinanceReviewer` group → Finance Reviewer role in app); session created in Redis; login event in audit_log |
| **KPI Validated** | K8 |

---

### TC-INT-003 · Email notification delivery
| Field | Value |
|-------|-------|
| **Type** | Integration 🔄 |
| **Priority** | P0 |
| **Steps** | Travel request submitted; approval notification should fire |
| **Expected Result** | ✅ SendGrid delivers email to L1 approver within 60 seconds; email contains: employee name, request summary, deep link to approval queue; delivery receipt logged |
| **KPI Validated** | K8, NF10 |

---

### TC-INT-004 · Notification DLQ — SendGrid failure
| Field | Value |
|-------|-------|
| **Type** | Integration 🔄 – Edge Case 🧩 |
| **Priority** | P1 |
| **Steps** | Simulate SendGrid API returning 500 for 5 consecutive messages |
| **Expected Result** | Messages moved to Azure Service Bus Dead Letter Queue; Application Insights alert fires: `"DLQ > 5 messages"`; PagerDuty incident created; failed messages retried after manual resolution |
| **KPI Validated** | K8, K11 |

---

### TC-INT-005 · Payroll integration — batch file format
| Field | Value |
|-------|-------|
| **Type** | Integration 🔄 |
| **Priority** | P0 |
| **Steps** | Finance Controller triggers payroll push for batch with 50 employees |
| **Expected Result** | ✅ Batch file generated in SAP-compatible format; all 50 records have valid IFSC, account number (from Always Encrypted columns), amount; file delivered via Azure Service Bus; no plaintext sensitive data in logs |
| **KPI Validated** | K8, K9 |

---

### TC-INT-006 · Azure AI Document Intelligence — availability fallback
| Field | Value |
|-------|-------|
| **Type** | Integration 🔄 – Edge Case 🧩 |
| **Priority** | P1 |
| **Steps** | Simulate Azure AI Document Intelligence service returning 503 during receipt upload |
| **Expected Result** | Circuit breaker opens after 5 failures; user sees: `"Auto-extraction temporarily unavailable. Please enter receipt details manually."`; claim can still be submitted with manual entry; App Insights alert fires |
| **KPI Validated** | K8, K11 |

---

## UI / UX Test Cases
> **KPI Ref:** K7 (UI/UX Guidelines)

---

### TC-UI-001 · WCAG 2.1 AA compliance — travel request form
| Field | Value |
|-------|-------|
| **Type** | Accessibility |
| **Priority** | P0 |
| **Tool** | axe-core via Playwright |
| **Steps** | Run axe-core scan on `/travel/new-request` page |
| **Expected Result** | ✅ 0 WCAG 2.1 AA violations; all form fields have associated `<label>`; color contrast ratio ≥ 4.5:1; all buttons have ARIA labels |
| **KPI Validated** | K7 |

---

### TC-UI-002 · Responsive layout — mobile viewport
| Field | Value |
|-------|-------|
| **Type** | UI |
| **Priority** | P1 |
| **Tool** | Playwright (viewport 375×812) |
| **Steps** | Open claim submission form on 375 px viewport |
| **Expected Result** | ✅ Single-column layout; no horizontal scroll; receipt camera button prominent; all touch targets ≥ 44×44 px; Angular Material components adapt to mobile breakpoint |
| **KPI Validated** | K7 |

---

### TC-UI-003 · Inline validation — real-time feedback
| Field | Value |
|-------|-------|
| **Type** | UI |
| **Priority** | P0 |
| **Steps** | On travel request form, enter `estimated_cost = -500` then tab away |
| **Expected Result** | ✅ Red inline error appears immediately below field: `"Amount must be greater than 0"`; submit button remains disabled until all errors resolved |
| **KPI Validated** | K7, F2 |

---

### TC-UI-004 · Session timeout warning
| Field | Value |
|-------|-------|
| **Type** | UI – Security 🔒 |
| **Priority** | P1 |
| **Steps** | Leave app idle for 25 minutes |
| **Expected Result** | At 25 min: non-intrusive toast: `"Your session expires in 5 minutes"`; at 30 min: modal with Extend / Logout options; if no action, session ends; form data saved as draft |
| **KPI Validated** | K7, K9 |

---

### TC-UI-005 · SignalR real-time approval notification
| Field | Value |
|-------|-------|
| **Type** | UI – Integration 🔄 |
| **Priority** | P0 |
| **Steps** | Manager has approval queue open; another employee submits a new request |
| **Expected Result** | ✅ Without page refresh: notification badge increments; toast appears: `"New travel request from Priya Sharma requires your approval"`; queue list updates live via SignalR |
| **KPI Validated** | K7, K8 |

---

## Deployment & DevOps Test Cases
> **KPI Ref:** K13 (Deployment Strategy)

---

### TC-DP-001 · Azure DevOps pipeline — SonarQube gate
| Field | Value |
|-------|-------|
| **Type** | DevOps |
| **Priority** | P0 |
| **Steps** | Submit PR with 72 % unit test coverage (below 80 % gate) |
| **Expected Result** | ❌ SonarQube Quality Gate fails; Azure DevOps pipeline blocks merge; PR comment added: `"Coverage 72% is below required threshold of 80%"` |
| **KPI Validated** | K13, NF12 |

---

### TC-DP-002 · Trivy container image scan — critical CVE
| Field | Value |
|-------|-------|
| **Type** | DevOps – Security 🔒 |
| **Priority** | P0 |
| **Steps** | Introduce known vulnerable NuGet package with Critical CVE into Dockerfile |
| **Expected Result** | ❌ Trivy scan fails; Docker push blocked; pipeline fails at "Container Security Scan" stage; CVE details in pipeline log |
| **KPI Validated** | K13, K9 |

---

### TC-DP-003 · Blue-green deployment rollback
| Field | Value |
|-------|-------|
| **Type** | DevOps |
| **Priority** | P0 |
| **Steps** | Deploy v2.0 to green slot; post-deployment smoke test detects error rate > 2 % |
| **Expected Result** | ✅ Azure DevOps release gate triggers `helm rollback`; traffic shifted back to blue slot within 15 minutes; rollback notification sent to DevOps team; no data loss |
| **KPI Validated** | K13 |

---

### TC-DP-004 · EF Core migration — staging validation
| Field | Value |
|-------|-------|
| **Type** | DevOps – Integration 🔄 |
| **Priority** | P0 |
| **Steps** | Apply new EF Core migration to Staging Azure SQL MI |
| **Expected Result** | ✅ Migration completes without errors; all FK constraints, indexes, RLS predicates and Always Encrypted columns preserved; rollback script (`Down()`) tested on separate schema |
| **KPI Validated** | K13 |

---

## Audit & Compliance Test Cases
> **KPI Ref:** K18, K19, K20

---

### TC-AU-001 · Immutable audit log — tamper attempt
| Field | Value |
|-------|-------|
| **Type** | Compliance 🔒 |
| **Priority** | P0 |
| **Steps** | Attempt `UPDATE audit_logs SET action = 'APPROVED' WHERE log_id = '...'` via DBA credentials |
| **Expected Result** | ❌ SQL permission denied; `audit_logs` table has no UPDATE/DELETE grants for any application role; only `INSERT` granted to `app_audit_writer` role |
| **KPI Validated** | K19, NF6 |

---

### TC-AU-002 · Temporal table — point-in-time query
| Field | Value |
|-------|-------|
| **Type** | Compliance |
| **Priority** | P0 |
| **Steps** | Query `TravelRequests FOR SYSTEM_TIME AS OF '2025-03-10T12:00:00'` for a request that was later cancelled |
| **Expected Result** | ✅ Returns row state as it was on March 10 at 12:00 (status = `PENDING_L1`); full history queryable; temporal table retention matches 7-year policy |
| **KPI Validated** | K19 |

---

### TC-AU-003 · Archival job — records > 3 years moved to cold storage
| Field | Value |
|-------|-------|
| **Type** | Compliance – Integration 🔄 |
| **Priority** | P1 |
| **Steps** | Hangfire `ArchivalJob` runs; targets expense claims with `created_at < today - 3 years` |
| **Expected Result** | ✅ Records moved to Azure Blob Storage (cold tier); primary DB entries replaced with archival pointer; retrieval via Audit Portal completes < 5 minutes; purge of records > 7 years requires 2-person authorisation |
| **KPI Validated** | K20 |

---

### TC-AU-004 · Sensitive data masking in UI
| Field | Value |
|-------|-------|
| **Type** | Security 🔒 – Compliance |
| **Priority** | P0 |
| **Steps** | Finance Reviewer opens employee reimbursement detail page |
| **Expected Result** | ✅ Bank account shown as `****4321`; PAN shown as `*****678K`; unmasking requires explicit button click + reason entry; unmask event logged in audit_log with `actor_id`, `timestamp`, `field_unmasked` |
| **KPI Validated** | K9, K19, NF8 |

---

## UAT Acceptance Test Cases
> **KPI Ref:** K1, K6, K15 — Validated by Business Pilot Group

---

### TC-UAT-001 · End-to-end employee journey
| Field | Value |
|-------|-------|
| **Type** | UAT |
| **Priority** | P0 |
| **Steps** | 1. Employee submits travel request 2. L1 approves 3. Employee completes travel 4. Employee submits expense claim with receipts 5. Manager approves 6. Finance approves 7. Reimbursement paid |
| **Expected Result** | ✅ Complete journey executed without manual intervention; total elapsed time (steps 1–7): < 3 business days; employee receives payment confirmation |
| **KPI Validated** | K1, K6, K15 |

---

### TC-UAT-002 · Finance month-end close workflow
| Field | Value |
|-------|-------|
| **Type** | UAT |
| **Priority** | P0 |
| **Steps** | Finance team runs month-end: reviews all pending claims, approves batch, triggers payroll push, downloads GL export |
| **Expected Result** | ✅ All actions completed within Finance dashboard without email or Excel; batch payment file exported in < 2 minutes; Finance FTE effort ≤ 2.5 hours for entire month-end cycle |
| **KPI Validated** | K6, K15 |

---

### TC-UAT-003 · HR Admin policy update — effective immediately
| Field | Value |
|-------|-------|
| **Type** | UAT |
| **Priority** | P1 |
| **Steps** | HR Admin updates hotel limit for Grade-C Tier-1 city from ₹3,000 to ₹3,500 |
| **Expected Result** | ✅ Change effective within 60 seconds (Redis cache invalidation); next expense submission uses new limit; no code deployment required |
| **KPI Validated** | K7 |

---

## Test Summary Matrix

| Module | Total TCs | P0 | P1 | Functional | Security | Performance | Integration | UAT |
|--------|-----------|----|----|-----------|----------|-------------|-------------|-----|
| Travel Requests | 12 | 8 | 4 | 10 | 1 | — | 1 | — |
| Expense Claims | 14 | 9 | 5 | 10 | 3 | — | 1 | — |
| Reimbursements | 5 | 4 | 1 | 4 | — | — | 1 | — |
| Reporting | 4 | 1 | 3 | 3 | — | — | 1 | — |
| Budget Mgmt | 2 | 1 | 1 | 2 | — | — | — | — |
| Administration | 2 | — | 2 | 2 | — | — | — | — |
| Security | 8 | 7 | 1 | — | 8 | — | — | — |
| Performance | 5 | 2 | 3 | — | — | 5 | — | — |
| Integration | 6 | 4 | 2 | — | 1 | — | 5 | — |
| UI / UX | 5 | 3 | 2 | 3 | 1 | — | 1 | — |
| Deployment | 4 | 4 | — | — | 1 | — | 1 | — |
| Audit & Compliance | 4 | 3 | 1 | — | 2 | — | 1 | — |
| UAT | 3 | 2 | 1 | 3 | — | — | — | 3 |
| **TOTAL** | **74** | **48** | **26** | **37** | **17** | **5** | **12** | **3** |

---

## KPI-to-Test-Case Traceability

| KPI Ref | KPI Description | Test Cases |
|---------|----------------|-----------|
| F1 | Workflow Accuracy | TC-TR-001, TC-TR-007, TC-TR-008, TC-EC-001, TC-RM-001, TC-RM-002 |
| F2 | Mandatory Field Enforcement | TC-TR-002, TC-TR-003, TC-TR-004, TC-EC-001, TC-EC-006 |
| F3 | Exception Handling | TC-TR-002, TC-EC-013, TC-RM-004 |
| F4 | OCR Accuracy | TC-EC-002, TC-EC-003 |
| F5 | Advance Reconciliation | TC-RM-005 |
| F6 | Duplicate Receipt Detection | TC-EC-004, TC-EC-005 |
| NF1 | Response Time P95 < 2 s | TC-PF-001, TC-PF-004 |
| NF2 | 20,000 Concurrent Users | TC-PF-002, TC-PF-003 |
| NF3 | Encryption at Rest | TC-AU-004, TC-INT-005 |
| NF4 | TLS 1.3 | TC-SEC-005 |
| NF8 | GDPR Erasure | TC-SEC-007 |
| NF9 | HRMS Sync < 30 min | TC-INT-001 |
| NF10 | Notification < 60 s | TC-TR-001, TC-INT-003 |
| K6 | User Journey Completeness | TC-UAT-001, TC-UAT-002, TC-RP-001 |
| K7 | WCAG + UX Standards | TC-UI-001, TC-UI-002, TC-UI-003, TC-UI-004, TC-UI-005 |
| K8 | Integration KPIs | TC-INT-001 to TC-INT-006 |
| K9 | Security Standards | TC-SEC-001 to TC-SEC-008 |
| K10 | Redis + Auto-scale | TC-PF-001, TC-PF-005, TC-BG-001 |
| K11 | Observability Alerts | TC-INT-004, TC-INT-006 |
| K13 | Deployment Pipeline | TC-DP-001 to TC-DP-004 |
| K15 | Success Metrics | TC-UAT-001, TC-UAT-002 |
| K18 | Compliance | TC-SEC-007, TC-EC-008, TC-RP-002 |
| K19 | Audit Framework | TC-AU-001, TC-AU-002, TC-AU-004 |
| K20 | Data Retention | TC-EC-013, TC-AU-003 |
