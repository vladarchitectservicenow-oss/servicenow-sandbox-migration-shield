# Test Suite SOP — Sandbox Migration Shield

## Overview

This document defines the Standard Operating Procedure for testing Sandbox Migration Shield. All test scenarios reference the three core components: **SandboxScanner** (scan + classify), **MigrationEngine** (migrate + rollback), and **ExemptionManager** (create + approve + expire).

## Test Environment

| Variable | Value |
|----------|-------|
| PDI Instance | dev362840.service-now.com |
| Scope | x_snc_sms |
| Auth | admin credentials |
| Pre-requisite | Cross-scope read ACLs granted on sys_script, sys_script_include, sys_script_client, sys_ui_policy, sys_ui_action, sys_dictionary |
| Test Data | Minimum 5 Business Rules, 3 Script Includes, 2 Client Scripts, 2 UI Policies present in PDI |

## Test Scenarios

---

### T01: Full Scan — All Tables

| Field | Value |
|-------|-------|
| **ID** | T01 |
| **Component** | SandboxScanner |
| **Priority** | P0 |
| **Precondition** | Cross-scope grants configured; scan_run table empty |
| **Steps** | 1. Call `SandboxScanner.scanAll(null, true)` (full scan, all tables)<br>2. Verify scan_run record created with status RUNNING<br>3. Wait for scan completion (status: SUCCESS)<br>4. Query x_snc_sms_scan_result for the scan_run<br>5. Verify results exist for all 6 target tables |
| **Expected Result** | scan_run.total_scripts > 0; scan_result records found for each table (sys_script, sys_script_include, sys_script_client, sys_ui_policy, sys_ui_action, sys_dictionary); all scan_result.severity set to one of BLOCKING/WARNING/COMPATIBLE |
| **Negative Case** | Call scanAll() without cross-scope grants → should return zero results AND display error banner "Cross-scope access not configured" |
| **Pass Criteria** | 5+ tables produce results; no table returns empty without error; severity classification present on all rows |

---

### T02: Severity Classification — BLOCKING Patterns

| Field | Value |
|-------|-------|
| **ID** | T02 |
| **Component** | SandboxScanner.classify() |
| **Priority** | P0 |
| **Precondition** | Script contains `Packages.java.lang.System.getProperty("user.dir")` |
| **Steps** | 1. Create a test Business Rule with script containing `Packages.java.lang.String`<br>2. Run SandboxScanner against this record<br>3. Check the scan_result.severity for this record |
| **Expected Result** | severity = BLOCKING; issue_code starts with "KB2944435-001" |
| **Negative Case** | Script without any Package references → classified as COMPATIBLE (not BLOCKING) |
| **Pass Criteria** | BLOCKING correctly assigned for KB2944435-001 pattern; COMPATIBLE correctly assigned for clean script |

---

### T03: Severity Classification — WARNING Patterns

| Field | Value |
|-------|-------|
| **ID** | T03 |
| **Component** | SandboxScanner.classify() |
| **Priority** | P1 |
| **Precondition** | Script contains `gs.print("debug")` |
| **Steps** | 1. Create a test Script Include with `gs.print("test")` call<br>2. Run SandboxScanner against this record<br>3. Check the scan_result.severity |
| **Expected Result** | severity = WARNING; issue_code = "KB2944435-003" |
| **Negative Case** | Script uses `gs.info("test")` instead → classified as COMPATIBLE (gs.info is safe) |
| **Pass Criteria** | WARNING correctly assigned; gs.info does NOT trigger false positive |

---

### T04: Concurrent Scan Prevention (Singleton Lock)

| Field | Value |
|-------|-------|
| **ID** | T04 |
| **Component** | SandboxScanner |
| **Priority** | P1 |
| **Precondition** | An existing scan_run with status RUNNING exists in the database |
| **Steps** | 1. Create a scan_run record with status=RUNNING manually<br>2. Call SandboxScanner.scanAll(null, true)<br>3. Observe return value |
| **Expected Result** | Scanner returns error object: `{success: false, message: "Scan already in progress"}`. No new scan_run created. |
| **Negative Case** | No RUNNING scan exists → scanAll() proceeds normally, creates new scan_run |
| **Pass Criteria** | Singleton lock prevents duplicate scan; error message includes who started the running scan |

---

### T05: Migration Preview — Inline Expression

| Field | Value |
|-------|-------|
| **ID** | T05 |
| **Component** | MigrationEngine.preview() |
| **Priority** | P0 |
| **Precondition** | scan_result exists for a BLOCKING sys_script record with inline expression: `current.priority + ' - ' + current.assigned_to` |
| **Steps** | 1. Call MigrationEngine.preview(scan_result_sys_id)<br>2. Examine returned preview object |
| **Expected Result** | Preview contains: `{generated_script_include: {name, api_name, script}, replacement_call: "new x_snc_sms.FunctionName().getValue(current)"}`. No records created or modified in database. |
| **Negative Case** | Call preview with invalid scan_result sys_id → returns `{success: false, message: "Scan result not found"}` |
| **Pass Criteria** | Preview shows exact generated code; source record NOT modified (verify GlideRecord.get shows original content) |

---

### T06: Migration Execute — Script Include Creation

| Field | Value |
|-------|-------|
| **ID** | T06 |
| **Component** | MigrationEngine.execute() |
| **Priority** | P0 |
| **Precondition** | Preview confirmed by user (T05 passed); migration_token valid |
| **Steps** | 1. Call MigrationEngine.execute(migration_token)<br>2. Verify new sys_script_include created with correct name and api_name<br>3. Verify source sys_script.script field updated to replacement call<br>4. Verify scan_result.status = MIGRATED<br>5. Verify audit_log entry created with action=MIGRATE |
| **Expected Result** | Script Include exists; source script updated; scan_result.status = MIGRATED; audit_log entry present |
| **Negative Case** | Call execute() twice with same token → returns `{success: false, message: "Already migrated"}` |
| **Pass Criteria** | All 4 verification steps pass; source script functional (no syntax errors in replacement call) |

---

### T07: Migration Rollback — Full Restore

| Field | Value |
|-------|-------|
| **ID** | T07 |
| **Component** | MigrationEngine.rollback() |
| **Priority** | P0 |
| **Precondition** | Script was migrated (T06); original script content stored in scan_result.source_code_snippet |
| **Steps** | 1. Call MigrationEngine.rollback(scan_result_sys_id)<br>2. Verify source sys_script.script restored to original content<br>3. Verify generated sys_script_include deleted (or marked inactive)<br>4. Verify scan_result.status = ROLLED_BACK<br>5. Verify audit_log entry with action=ROLLBACK |
| **Expected Result** | Script restored; generated SI removed; status ROLLED_BACK; audit log complete |
| **Negative Case** | Call rollback on a scan_result with status=NEW (never migrated) → returns error |
| **Pass Criteria** | g_form/current/previous context preserved in restored script (byte-exact match with stored original) |

---

### T08: Exemption Creation with Validation

| Field | Value |
|-------|-------|
| **ID** | T08 |
| **Component** | ExemptionManager.createExemption() |
| **Priority** | P0 |
| **Precondition** | scan_result exists with severity=BLOCKING, status=NEW |
| **Steps** | 1. Call ExemptionManager.createExemption(scan_result_sys_id, "Business justification for keeping this script — it interfaces with a legacy system that cannot be rewritten", approver_sys_id, "2027-06-03")<br>2. Verify exemption record created with status=PENDING<br>3. Verify scan_result.status = EXEMPT (after approval) |
| **Expected Result** | Exemption created; justification stored; scan_result linked |
| **Negative Case** | Justification < 50 chars → validation error returned; exemption NOT created |
| **Pass Criteria** | Exemption created with all fields populated; validation rejects short justifications |

---

### T09: Exemption Approval Workflow

| Field | Value |
|-------|-------|
| **ID** | T09 |
| **Component** | ExemptionManager.approveExemption() |
| **Priority** | P1 |
| **Precondition** | Exemption exists with status=PENDING (from T08) |
| **Steps** | 1. Call ExemptionManager.approveExemption(exemption_sys_id)<br>2. Verify exemption.status = APPROVED<br>3. Verify exemption.approved_at timestamp set<br>4. Verify scan_result.status = EXEMPT<br>5. Verify readiness score recalculated (exempted script excluded from BLOCKING count) |
| **Expected Result** | Exemption approved; timestamps correct; readiness score reflects exclusion |
| **Negative Case** | Call approveExemption() on already APPROVED exemption → returns error "Already approved" |
| **Pass Criteria** | State transitions valid; readiness score formula correct: (total_scripts - blocked_exempted) / total_scripts * 100 |

---

### T10: Exemption Expiry — Scheduled Job Trigger

| Field | Value |
|-------|-------|
| **ID** | T10 |
| **Component** | ExemptionManager.getExpiringExemptions() |
| **Priority** | P1 |
| **Precondition** | Exemptions exist; one expires in 25 days, one in 35 days, one already expired |
| **Steps** | 1. Call ExemptionManager.getExpiringExemptions(30) (30-day threshold)<br>2. Check returned list |
| **Expected Result** | List contains exemption expiring in 25 days. Does NOT contain exemption expiring in 35 days. Already-expired exemption has status=EXPIRED (auto-transition). |
| **Negative Case** | No exemptions expiring within threshold → returns empty array (not error) |
| **Pass Criteria** | Correct threshold logic; already-expired auto-transitions to EXPIRED status |

---

### T11: Dashboard Readiness Score Calculation

| Field | Value |
|-------|-------|
| **ID** | T11 |
| **Component** | ExemptionManager.calculateReadinessScore() |
| **Priority** | P1 |
| **Precondition** | scan_result table populated: 50 BLOCKING, 100 WARNING, 500 COMPATIBLE; 20 of 50 BLOCKING are exempted |
| **Steps** | 1. Call ExemptionManager.calculateReadinessScore()<br>2. Check returned score and breakdown |
| **Expected Result** | readiness_score = (500 + 100 + 20) / 650 * 100 = 95.38%. Breakdown: blocking=50, blocking_exempted=20, warning=100, compatible=500, total=650 |
| **Negative Case** | Zero scripts in database → score = 100% (nothing to migrate) |
| **Pass Criteria** | Score formula verified; breakdown counts match database state; exempted scripts excluded from "remaining risk" count |

---

### T12: REST API — Full Scan Lifecycle

| Field | Value |
|-------|-------|
| **ID** | T12 |
| **Component** | Scripted REST API |
| **Priority** | P1 |
| **Precondition** | REST API endpoints deployed; auth token available |
| **Steps** | 1. POST /api/x_snc_sms/v1/scan with body `{full_scan: true}` → get scan_run_id<br>2. GET /api/x_snc_sms/v1/scan/{scan_run_id} → verify results array populated<br>3. GET /api/x_snc_sms/v1/dashboard → verify readiness_score present<br>4. POST /api/x_snc_sms/v1/preview-migration with `{scan_result_id}` → verify preview returned<br>5. POST /api/x_snc_sms/v1/exemption with valid body → verify exemption created |
| **Expected Result** | All 5 API calls return HTTP 200 with valid JSON. Dashboard returns score. Preview returns generated code. |
| **Negative Case** | GET /api/x_snc_sms/v1/scan/{invalid_id} → HTTP 404. POST without required fields → HTTP 400 with error message array. |
| **Pass Criteria** | All CRUD operations functional via REST; proper HTTP status codes; JSON response schema valid |

---

## Test Execution Order

```
T01 (Full Scan) → T02/T03 (Classification) → T04 (Concurrency)
                                              ↓
T05 (Preview) → T06 (Execute) → T07 (Rollback)
                                     ↓
T08 (Create Exemption) → T09 (Approve) → T10 (Expiry)
                                               ↓
                                          T11 (Dashboard)
                                               ↓
                                          T12 (REST API)
```

## Pass/Fail Criteria

- **CRITICAL (T01, T02, T05, T06, T07, T08):** Must PASS 100%. Any failure = release blocked.
- **HIGH (T03, T04, T09, T10, T11):** Must PASS 100%. Failure = release requires sign-off.
- **MEDIUM (T12):** Must PASS 100%. Minor schema deviations acceptable if REST contract preserved.

## Test Data Requirements

| Data | Min Required | Source |
|------|-------------|--------|
| Business Rules with Java Packages | 3 | PDI sys_script table |
| Script Includes with deprecated APIs | 2 | PDI sys_script_include table |
| Clean scripts (COMPATIBLE) | 5 | Any script table |
| Exemption approver user | 1 | PDI sys_user table |
| Valid REST API credential | 1 | PDI user with snc_internal role |

## Test Execution Log Template

```markdown
| Scenario | Status | Duration | Evidence | Notes |
|----------|--------|----------|----------|-------|
| T01      | PASS   | 45s      | Screenshot t01_scan_results.png | |
| T02      | PASS   | 10s      | Log t02_classification.log | |
| ...      |        |          |          | |
```
