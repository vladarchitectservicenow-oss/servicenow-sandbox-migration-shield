# Sandbox Migration Shield — Regression Cases

**Product:** Sandbox Migration Shield  
**Scope:** `x_snc_sms`  
**Author:** Vladimir Kapustin  
**Last Updated:** 2026-07-03

---

## Purpose

This document defines regression test cases to ensure that fixes and enhancements do not break existing functionality. Run after every code change, before every release, and as part of CI/CD pipeline.

---

## REG-001: Pattern Registry Integrity

**Risk:** Adding/modifying KB2944435 patterns could break existing detection or introduce false positives.  
**Trigger:** Any change to `SandboxScanner.PATTERNS` array.

**Test Steps:**
1. Run full scan on a known test instance with pre-seeded scripts
2. Compare blocking_count, warning_count, compatible_count against baseline
3. Verify all 7 default patterns still produce correct classifications

**Baseline:** Counts from last successful full scan on test instance.  
**Pass:** All counts match baseline ±0.  
**Fail:** Any count differs from baseline.

---

## REG-002: Migration Engine — Preview/Execute/Rollback Cycle

**Risk:** Changes to MigrationEngine could break the core migration workflow.  
**Trigger:** Any change to `MigrationEngine.js`.

**Test Steps:**
1. Create a test Business Rule with `Packages.java.lang.String` (BLOCKING)
2. Run scan → get scan_result
3. Preview migration → verify generated code structure
4. Execute migration → verify Script Include created, source updated
5. Rollback → verify Script Include deleted, source restored
6. Verify audit log has MIGRATE + ROLLBACK entries

**Pass:** All 6 steps succeed.  
**Fail:** Any step fails.

---

## REG-003: Exemption Lifecycle

**Risk:** Changes to ExemptionManager could corrupt exemption state machine.  
**Trigger:** Any change to `ExemptionManager.js`.

**Test Steps:**
1. Create exemption (PENDING) → verify scan_result = EXEMPT
2. Approve exemption → verify status = APPROVED, approved_at set
3. Revoke exemption → verify status = REVOKED, scan_result = NEW
4. Attempt to approve already-approved exemption → verify error
5. Attempt to revoke already-revoked exemption → verify error
6. Verify audit log has CREATE + APPROVE + REVOKE entries

**Pass:** All 6 steps produce correct state transitions.  
**Fail:** Any state transition is incorrect.

---

## REG-004: Scan Deduplication

**Risk:** Changes to `_saveResult()` could create duplicate scan_result records.  
**Trigger:** Any change to `SandboxScanner._saveResult()`.

**Test Steps:**
1. Run full scan (Scan #1) → record total_scripts
2. Run full scan again (Scan #2) without modifying scripts
3. Count scan_results with status=NEW for Scan #2
4. Verify no duplicate (table, sys_id, field) tuples with status=NEW

**Pass:** Scan #2 total_scripts = Scan #1 total_scripts; no duplicates.  
**Fail:** Duplicate records found.

---

## REG-005: Readiness Score Accuracy

**Risk:** Changes to `calculateReadinessScore()` could produce incorrect compliance metrics.  
**Trigger:** Any change to `ExemptionManager.calculateReadinessScore()`.

**Test Steps:**
1. Seed instance with known counts: 10 COMPATIBLE, 5 MIGRATED, 3 BLOCKING, 2 EXEMPT
2. Calculate readiness score
3. Expected: round((10 + 5) / (10 + 5 + 3)) × 100 = round(15/18 × 100) = 83

**Pass:** Score = 83.  
**Fail:** Score ≠ 83.

---

## REG-006: Cross-Scope Read Access

**Risk:** Platform upgrades or ACL changes could revoke cross-scope read access.  
**Trigger:** After any ServiceNow upgrade or ACL modification.

**Test Steps:**
1. Execute `new GlideRecord('sys_script').getRowCount()`
2. Execute `new GlideRecord('sys_script_include').getRowCount()`
3. Execute `new GlideRecord('sys_script_client').getRowCount()`
4. Execute `new GlideRecord('sys_ui_policy').getRowCount()`
5. Execute `new GlideRecord('sys_ui_action').getRowCount()`
6. Execute `new GlideRecord('sys_dictionary').getRowCount()`

**Pass:** All six return >0.  
**Fail:** Any returns 0 — cross-scope access revoked.

---

## REG-007: Scheduled Job Execution

**Risk:** Changes to scheduled job scripts could break automated scanning.  
**Trigger:** Any change to `weekly_scan.js` or `weekly_report.js`.

**Test Steps:**
1. Execute weekly_scan scheduled job manually
2. Verify scan_run record created with status=COMPLETED
3. Execute weekly_report scheduled job manually
4. Verify report generated without errors

**Pass:** Both jobs complete without errors.  
**Fail:** Either job fails or produces incorrect output.

---

## REG-008: REST API Endpoints

**Risk:** Changes to REST API could break external integrations.  
**Trigger:** Any change to `scanner_api.js`.

**Test Steps:**
1. `GET /api/x_snc_sms/v1/scan/status` → verify returns JSON with last scan summary
2. `POST /api/x_snc_sms/v1/scan/trigger` → verify triggers new scan
3. `GET /api/x_snc_sms/v1/results?severity=BLOCKING` → verify returns filtered results

**Pass:** All three endpoints return valid JSON with expected structure.  
**Fail:** Any endpoint returns error or incorrect data.

---

## Regression Run History

| Date | Trigger | Scenarios Run | Pass | Fail | Notes |
|------|---------|---------------|------|------|-------|
| 2026-07-03 | Initial baseline | REG-001 through REG-008 | — | — | Pending PDI availability |
