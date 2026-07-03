# Sandbox Migration Shield — Test Suite SOP

**Product:** Sandbox Migration Shield  
**Scope:** `x_snc_sms`  
**Author:** Vladimir Kapustin  
**Last Updated:** 2026-07-03

---

## Test Environment

| Parameter | Value |
|-----------|-------|
| Instance | `dev362840.service-now.com` |
| Scope | `x_snc_sms` |
| Test User | `admin` |
| Pre-requisite | App installed, cross-scope grants configured |

---

## Scenario 1: Full Scan — All Tables

**ID:** SMS-TC-001  
**Priority:** P0  
**Objective:** Verify SandboxScanner.fullScan() scans all six script-bearing tables and produces correct aggregate counts.

**Steps:**
1. Navigate to Workspace UI → Sandbox Migration Shield → Scan
2. Click "Run Full Scan"
3. Wait for scan completion (status → COMPLETED)
4. Open scan_run record

**Expected Results:**
- scan_run.status = "COMPLETED"
- scan_run.total_scripts > 0
- scan_run.blocking_count + scan_run.warning_count + scan_run.compatible_count = total_scripts
- scan_run.started_at and completed_at are populated
- scan_result records exist for each script field found

**Pass Criteria:** All expected results met.

---

## Scenario 2: BLOCKING Pattern Detection — `Packages.java`

**ID:** SMS-TC-002  
**Priority:** P0  
**Objective:** Verify KB2944435-001 pattern correctly flags `Packages.java.*` usage.

**Steps:**
1. Create a test Business Rule with script: `var x = Packages.java.lang.String;`
2. Run full scan
3. Filter scan_results by script_name matching test BR

**Expected Results:**
- scan_result.severity = "BLOCKING"
- scan_result.issue_code = "KB2944435-001"
- scan_result.issue_description contains "Java package access blocked"

**Pass Criteria:** Test script correctly classified as BLOCKING.

---

## Scenario 3: BLOCKING Pattern Detection — `eval()`

**ID:** SMS-TC-003  
**Priority:** P0  
**Objective:** Verify KB2944435-002 pattern correctly flags `eval()` calls.

**Steps:**
1. Create a test Client Script with: `eval("gs.info('test')");`
2. Run full scan
3. Filter scan_results for the test script

**Expected Results:**
- scan_result.severity = "BLOCKING"
- scan_result.issue_code = "KB2944435-002"

**Pass Criteria:** eval() call correctly classified as BLOCKING.

---

## Scenario 4: WARNING Pattern Detection — `gs.sleep()`

**ID:** SMS-TC-004  
**Priority:** P1  
**Objective:** Verify KB2944435-003 pattern correctly flags deprecated `gs.*` methods as WARNING.

**Steps:**
1. Create a test Business Rule with: `gs.sleep(1000);`
2. Run full scan
3. Filter scan_results

**Expected Results:**
- scan_result.severity = "WARNING"
- scan_result.issue_code = "KB2944435-003"

**Pass Criteria:** Deprecated gs method correctly classified as WARNING.

---

## Scenario 5: COMPATIBLE Script — No Issues

**ID:** SMS-TC-005  
**Priority:** P1  
**Objective:** Verify clean scripts are classified as COMPATIBLE.

**Steps:**
1. Create a test Business Rule with safe code: `var gr = new GlideRecord('incident'); gr.query();`
2. Run full scan
3. Filter scan_results

**Expected Results:**
- scan_result.severity = "COMPATIBLE"
- scan_result.issue_code is empty
- scan_result.issue_description = "[]"

**Pass Criteria:** Clean script correctly classified as COMPATIBLE.

---

## Scenario 6: Migration Preview

**ID:** SMS-TC-006  
**Priority:** P0  
**Objective:** Verify MigrationEngine.preview() generates correct Script Include code without executing.

**Steps:**
1. Identify a BLOCKING scan_result from TC-002
2. Execute `MigrationEngine.preview(scanResultId)`
3. Inspect returned preview object

**Expected Results:**
- preview.script_include_name is a valid sanitized name
- preview.api_name starts with "x_snc_sms."
- preview.generated_script contains valid Class.create() structure
- preview.replacement_call contains `new x_snc_sms.`
- preview.original_code matches source script
- No Script Include created on instance (preview only)

**Pass Criteria:** Preview returns valid code without side effects.

---

## Scenario 7: Migration Execute

**ID:** SMS-TC-007  
**Priority:** P0  
**Objective:** Verify MigrationEngine.execute() creates Script Include and updates source.

**Steps:**
1. Identify a BLOCKING scan_result from TC-002
2. Execute `MigrationEngine.execute(scanResultId)`
3. Verify Script Include created in sys_script_include
4. Verify source script updated with replacement call
5. Verify scan_result.status = "MIGRATED"

**Expected Results:**
- New Script Include exists with api_name matching preview
- Source script field contains replacement call (not original code)
- scan_result.migration_pre_state contains JSON with original content
- scan_result.migrated_to_si references the new Script Include sys_id
- Audit log entry created with action = "MIGRATE"

**Pass Criteria:** All five verifications pass.

---

## Scenario 8: Migration Rollback

**ID:** SMS-TC-008  
**Priority:** P0  
**Objective:** Verify MigrationEngine.rollback() restores original script and cleans up.

**Steps:**
1. Use the migrated scan_result from TC-007
2. Execute `MigrationEngine.rollback(scanResultId)`
3. Verify source script restored to original content
4. Verify Script Include deleted
5. Verify scan_result.status = "NEW"

**Expected Results:**
- Source script field contains original code (from pre-state)
- Script Include no longer exists in sys_script_include
- scan_result.migration_pre_state is empty
- scan_result.migrated_to_si is empty
- Audit log entry created with action = "ROLLBACK"

**Pass Criteria:** Full restoration confirmed.

---

## Scenario 9: Exemption Create + Approve

**ID:** SMS-TC-009  
**Priority:** P0  
**Objective:** Verify full exemption lifecycle: create → approve.

**Steps:**
1. Identify a BLOCKING scan_result
2. Execute `ExemptionManager.createExemption(scanResultId, "Business-critical legacy integration — migration planned for Q4 2026. Justification: this script interfaces with SAP ECC via a custom JDBC connector that cannot be refactored within the current release cycle.", "admin", "2026-12-31")`
3. Verify exemption record created
4. Execute `ExemptionManager.approveExemption(exemptionId)`

**Expected Results:**
- Exemption record created with status = "PENDING"
- scan_result.status = "EXEMPT"
- After approval: exemption.status = "APPROVED", approved_at populated
- Audit log entries for both CREATE and APPROVE actions

**Pass Criteria:** Full create→approve cycle works.

---

## Scenario 10: Exemption Revoke

**ID:** SMS-TC-010  
**Priority:** P1  
**Objective:** Verify exemption revocation resets scan_result.

**Steps:**
1. Use the approved exemption from TC-009
2. Execute `ExemptionManager.revokeExemption(exemptionId)`
3. Verify exemption status
4. Verify scan_result status

**Expected Results:**
- exemption.status = "REVOKED"
- scan_result.status = "NEW" (no longer EXEMPT)
- Audit log entry for REVOKE action

**Pass Criteria:** Revocation correctly resets both records.

---

## Scenario 11: Readiness Score Calculation

**ID:** SMS-TC-011  
**Priority:** P1  
**Objective:** Verify readiness score formula: (compatible + migrated) / total × 100.

**Steps:**
1. Ensure instance has a mix of COMPATIBLE, MIGRATED, BLOCKING, and EXEMPT scan_results
2. Execute `ExemptionManager.calculateReadinessScore()`
3. Manually calculate expected score

**Expected Results:**
- Score is an integer 0-100
- Score = round(((compatible_count + migrated_count) / (total - exempt_count)) × 100)
- EXEMPT scripts excluded from denominator

**Pass Criteria:** Calculated score matches manual computation.

---

## Scenario 12: Exemption Expiry Detection

**ID:** SMS-TC-012  
**Priority:** P1  
**Objective:** Verify getExpiringExemptions() returns exemptions expiring within threshold.

**Steps:**
1. Create an exemption expiring in 15 days
2. Create an exemption expiring in 60 days
3. Execute `ExemptionManager.getExpiringExemptions(30)`

**Expected Results:**
- Returns the 15-day exemption
- Does NOT return the 60-day exemption
- Each entry has exemption_id, scan_result, approver, expires_at, justification

**Pass Criteria:** Correct filtering by expiry threshold.

---

## Scenario 13: Scan Deduplication

**ID:** SMS-TC-013  
**Priority:** P1  
**Objective:** Verify re-scanning doesn't create duplicate scan_result records.

**Steps:**
1. Run full scan (Scan #1) — note total_scripts count
2. Run full scan again (Scan #2) without modifying any scripts
3. Compare scan_result counts

**Expected Results:**
- Scan #2 total_scripts = Scan #1 total_scripts
- No duplicate scan_results for same table+sys_id+field with status=NEW
- Existing scan_results updated with new scan_run reference

**Pass Criteria:** No duplicate records created.

---

## Scenario 14: Scope-Filtered Scan

**ID:** SMS-TC-014  
**Priority:** P2  
**Objective:** Verify scope parameter limits scan to specific application.

**Steps:**
1. Run `SandboxScanner.fullScan("global")` — scan only global scope
2. Run `SandboxScanner.fullScan()` — scan all scopes
3. Compare total_scripts

**Expected Results:**
- Scope-filtered scan returns fewer or equal scripts
- All returned scripts have sys_scope = "global"

**Pass Criteria:** Scope filtering works correctly.

---

## Scenario 15: Single-Table Scan

**ID:** SMS-TC-015  
**Priority:** P2  
**Objective:** Verify targetTable parameter limits scan to one table.

**Steps:**
1. Run `SandboxScanner.fullScan(null, "sys_script")`
2. Verify scan_results only from sys_script table

**Expected Results:**
- All scan_results have script_table = "sys_script"
- No results from sys_script_include, sys_script_client, etc.

**Pass Criteria:** Table filtering works correctly.

---

## Test Execution Summary

| Scenario | Priority | Status | Notes |
|----------|----------|--------|-------|
| SMS-TC-001: Full Scan | P0 | PENDING | Requires PDI |
| SMS-TC-002: Packages.java Detection | P0 | PENDING | Requires PDI |
| SMS-TC-003: eval() Detection | P0 | PENDING | Requires PDI |
| SMS-TC-004: gs.sleep() Warning | P1 | PENDING | Requires PDI |
| SMS-TC-005: COMPATIBLE Script | P1 | PENDING | Requires PDI |
| SMS-TC-006: Migration Preview | P0 | PENDING | Requires PDI |
| SMS-TC-007: Migration Execute | P0 | PENDING | Requires PDI |
| SMS-TC-008: Migration Rollback | P0 | PENDING | Requires PDI |
| SMS-TC-009: Exemption Create+Approve | P0 | PENDING | Requires PDI |
| SMS-TC-010: Exemption Revoke | P1 | PENDING | Requires PDI |
| SMS-TC-011: Readiness Score | P1 | PENDING | Requires PDI |
| SMS-TC-012: Expiry Detection | P1 | PENDING | Requires PDI |
| SMS-TC-013: Deduplication | P1 | PENDING | Requires PDI |
| SMS-TC-014: Scope Filter | P2 | PENDING | Requires PDI |
| SMS-TC-015: Table Filter | P2 | PENDING | Requires PDI |
