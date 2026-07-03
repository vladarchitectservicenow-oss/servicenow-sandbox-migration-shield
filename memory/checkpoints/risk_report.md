# Sandbox Migration Shield — Risk Report

**Product:** Sandbox Migration Shield  
**Scope:** `x_snc_sms`  
**Author:** Vladimir Kapustin  
**Last Updated:** 2026-07-03

---

## P0 — Critical (Blocking / Showstopper)

### P0-01: Cross-Scope Write Access Denied
**Description:** MigrationEngine writes to `sys_script_include` (create) and source tables (update). If cross-scope privilege grants are missing, these operations silently return empty — no error, no record created. Users believe migration succeeded when it didn't.
**Impact:** Data loss — source scripts overwritten with replacement calls but Script Include never created. Rollback may also fail if pre-state wasn't stored.
**Likelihood:** Medium (common on locked-down production instances)
**Mitigation:**
1. Pre-flight check on install: attempt a test write to a temp record, verify success
2. Document required cross-scope grants explicitly in installation guide
3. Add try/catch with explicit error logging on every cross-scope write
4. Store pre-migration state BEFORE attempting Script Include creation
**Status:** MITIGATED — pre-state stored first, audit log captures failures

### P0-02: Source Script Overwritten Without Backup
**Description:** If MigrationEngine.execute() crashes between updating the source script and storing pre-migration state, the original script is lost with no recovery path.
**Impact:** Permanent data loss — original Business Rule / Client Script / UI Policy code destroyed.
**Likelihood:** Low (transactional — both operations in same script execution)
**Mitigation:**
1. Pre-migration state stored as JSON BEFORE any write operation
2. Rollback function reads from stored pre-state, not from current record
3. Audit log entry written before migration execution
**Status:** MITIGATED — pre-state stored first in execute() flow

### P0-03: KB2944435 Pattern False Positives
**Description:** Regex patterns may match legitimate code (e.g., `evaluate` containing `eval`, `Packages` in comments). False BLOCKING classification causes unnecessary migration work.
**Impact:** Wasted admin time migrating compatible scripts; loss of trust in tool accuracy.
**Likelihood:** Medium (regex-based detection is inherently imprecise)
**Mitigation:**
1. Pattern registry is extensible — admins can add/remove patterns
2. Severity classification is conservative: WARNING for ambiguous patterns
3. Preview mode allows human review before migration execution
4. Future: AST-based parsing for higher precision
**Status:** PARTIALLY MITIGATED — regex is the current approach; AST upgrade planned for v1.1

### P0-04: Scan Timeout on Large Instances
**Description:** Instances with >10,000 scripts may exceed GlideRecord transaction timeout during full scan. `setLimit(500)` per table helps but doesn't solve the total volume problem.
**Impact:** Incomplete scan results; administrators unaware of unscanned scripts.
**Likelihood:** Low (500 limit per table × 6 tables = 3,000 max per scan; pagination available)
**Mitigation:**
1. `setLimit(500)` per table prevents single-table timeout
2. Pagination support: scan by scope, by table, or incremental
3. Scheduled weekly scans distribute load over time
4. Scan progress tracked in scan_run record — resume capability
**Status:** MITIGATED — pagination + scheduling handles large instances

---

## P1 — High (Significant Impact)

### P1-01: No CI/CD Pipeline
**Description:** No automated testing, linting, or deployment pipeline. Manual testing only.
**Impact:** Regressions undetected; deployment errors in production.
**Likelihood:** High (manual processes are error-prone)
**Mitigation:**
1. Add GitHub Actions workflow for JS linting + unit tests
2. Add pre-commit hooks for copyright header verification
3. Add PDI smoke test automation
**Status:** OPEN — planned for v1.1

### P1-02: Incomplete Test Coverage
**Description:** No automated unit tests for SandboxScanner pattern matching, MigrationEngine extraction logic, or ExemptionManager state transitions.
**Impact:** Bugs in classification or migration logic undetected until production use.
**Likelihood:** Medium
**Mitigation:**
1. Node.js mock runtime tests for all three Script Includes
2. Test each KB2944435 pattern against known-positive and known-negative samples
3. Test migration preview/execute/rollback cycle end-to-end
**Status:** OPEN — test suite documented in Phase 2 validation

### P1-03: Cross-Scope Read Access Denied
**Description:** SandboxScanner reads from platform tables (sys_script, sys_script_include, etc.). Without cross-scope read grants, GlideRecord queries return zero results — scanner reports "0 scripts found" with no error.
**Impact:** False sense of security — administrators believe instance is clean when scanner simply couldn't read scripts.
**Likelihood:** Medium
**Mitigation:**
1. Pre-flight check: query sys_script with getRowCount(), verify >0
2. If count is 0, warn administrator about possible cross-scope access issue
3. Document required read grants in installation guide
**Status:** PARTIALLY MITIGATED — pre-flight check planned for v1.1

### P1-04: Migration Rollback Failure
**Description:** If the auto-created Script Include was modified after migration (e.g., admin added custom logic), rollback deletes it — losing the admin's changes.
**Impact:** Data loss of post-migration modifications.
**Likelihood:** Low (admins unlikely to modify auto-generated SIs before rollback)
**Mitigation:**
1. Add warning on rollback: "This will delete Script Include X. Any modifications made after migration will be lost."
2. Option to keep Script Include and only restore source script
**Status:** OPEN — planned for v1.1

### P1-05: Exemption Expiry Without Renewal
**Description:** Exemptions expire but no enforcement mechanism prevents the script from running. The exemption record shows EXPIRED but the script continues executing.
**Impact:** Compliance gap — auditors see expired exemptions with no follow-up action.
**Likelihood:** High (no automated enforcement)
**Mitigation:**
1. Scheduled job flags expiring exemptions 30 days before expiry
2. Add enforcement option: re-scan exempted scripts on expiry, re-classify as BLOCKING
3. Email notifications to approver and exemption creator
**Status:** PARTIALLY MITIGATED — expiry detection exists; enforcement planned for v1.1

---

## P2 — Medium (Moderate Impact)

### P2-01: Performance on Very Large Instances
**Description:** Instances with >50,000 script records across all tables may experience slow scan performance even with pagination.
**Impact:** Scan duration >5 minutes; UI timeout possible.
**Likelihood:** Low (most instances have <5,000 scripts)
**Mitigation:**
1. Incremental scan mode (scan only scripts modified since last scan)
2. Background scheduled job instead of synchronous UI trigger
3. Scope filtering to reduce scan surface
**Status:** MITIGATED — incremental + scheduled modes available

### P2-02: Hardcoded Table List
**Description:** `SCRIPT_TABLES` array is hardcoded in SandboxScanner. If ServiceNow adds new script-bearing tables in future releases, they won't be scanned.
**Impact:** Incomplete coverage for future ServiceNow releases.
**Likelihood:** Low (table list stable across releases)
**Mitigation:**
1. Make SCRIPT_TABLES configurable via system property
2. Add dynamic discovery: query sys_dictionary for tables with script-type fields
**Status:** OPEN — planned for v1.2

### P2-03: No PDF Export for Audit Reports
**Description:** Scan results and exemption registry are viewable in-platform but not exportable as PDF for executive review.
**Impact:** Extra manual work for compliance reporting.
**Likelihood:** Medium (common audit requirement)
**Mitigation:**
1. Add PDF export via Scripted REST API + server-side PDF generation
2. Email scheduled reports as PDF attachments
**Status:** OPEN — planned for v1.2

### P2-04: Single-User Migration
**Description:** MigrationEngine assumes single-user operation. Concurrent migrations on the same script could create race conditions.
**Impact:** Duplicate Script Includes; inconsistent scan_result state.
**Likelihood:** Low (migration is typically a planned, single-admin activity)
**Mitigation:**
1. Add lock mechanism: check scan_result status before migration, reject if already MIGRATED or IN_PROGRESS
2. Use GlideRecord with `setWorkflow(false)` for atomic updates
**Status:** PARTIALLY MITIGATED — status check exists; explicit lock planned for v1.1

---

## P3 — Low (Minor / Cosmetic)

### P3-01: No Dark Mode UI
**Description:** Workspace UI and Service Portal widgets use default ServiceNow styling without dark mode support.
**Impact:** Minor UX inconvenience for dark mode users.
**Mitigation:** Use ServiceNow's built-in theming (Now Experience UI Framework inherits instance theme).
**Status:** ACCEPTED — relies on platform theming

### P3-02: Error Messages in English Only
**Description:** All gs.info/gs.error messages and UI text are in English. No i18n support.
**Impact:** Non-English-speaking admins may struggle with error diagnosis.
**Mitigation:** Add sys_ui_message entries for all user-facing strings; use gs.getMessage().
**Status:** OPEN — planned for v1.2

### P3-03: No Upgrade Impact Predictor UI
**Description:** The "Upgrade Impact Predictor" feature (US-06) is documented in PRD but not yet implemented.
**Impact:** Administrators planning Zurich→Australia upgrades can't preview new scripts that would be blocked.
**Mitigation:** Implement as v1.1 feature — compare current scan against baseline from previous release.
**Status:** OPEN — planned for v1.1

---

## Risk Summary

| Severity | Count | Mitigated | Open |
|----------|-------|-----------|------|
| P0 — Critical | 4 | 3 | 1 (false positives — inherent to regex) |
| P1 — High | 5 | 0 | 5 |
| P2 — Medium | 4 | 1 | 3 |
| P3 — Low | 3 | 1 | 2 |
| **Total** | **16** | **5** | **11** |

**Overall Risk Posture:** MODERATE. Core scanning and migration functionality is solid. Primary gaps are in automated testing (P1-02), CI/CD (P1-01), and cross-scope access validation (P1-03). These are addressable in v1.1 without architectural changes.
