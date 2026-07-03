# Sandbox Migration Shield — Validation Checklist

**Product:** Sandbox Migration Shield  
**Scope:** `x_snc_sms`  
**Author:** Vladimir Kapustin  
**Last Updated:** 2026-07-03

---

## Pre-Deployment Validation

| # | Check | Status | Notes |
|---|-------|--------|-------|
| V-001 | Instance is Zurich or later | ☐ | Verify via `gs.getProperty('glide.war')` |
| V-002 | Cross-scope read access to sys_script* tables | ☐ | Test: `new GlideRecord('sys_script').getRowCount() > 0` |
| V-003 | Cross-scope write access to sys_script_include | ☐ | Test: create temp SI, verify, delete |
| V-004 | Scoped app imports without errors | ☐ | Studio → Import Application |
| V-005 | All custom tables created (x_snc_sms_*) | ☐ | Verify 4 tables: scan_run, scan_result, exemption, audit_log |
| V-006 | All Script Includes compile without errors | ☐ | Open each in Studio → check for syntax errors |
| V-007 | Scheduled jobs registered | ☐ | Verify weekly_scan, weekly_report in sys_trigger |
| V-008 | REST API endpoints registered | ☐ | Verify in sys_ws_definition |
| V-009 | No hardcoded credentials in source | ☐ | `grep -rPn 'password=' src/` — only property access allowed |
| V-010 | Copyright headers on all .js files | ☐ | `grep -r 'Vladimir Kapustin' src/` — all files must match |

---

## Functional Validation

| # | Check | Status | Notes |
|---|-------|--------|-------|
| V-011 | Full scan completes on all 6 tables | ☐ | SMS-TC-001 |
| V-012 | BLOCKING pattern: Packages.java detected | ☐ | SMS-TC-002 |
| V-013 | BLOCKING pattern: eval() detected | ☐ | SMS-TC-003 |
| V-014 | WARNING pattern: gs.sleep() detected | ☐ | SMS-TC-004 |
| V-015 | COMPATIBLE scripts correctly classified | ☐ | SMS-TC-005 |
| V-016 | Migration preview generates valid code | ☐ | SMS-TC-006 |
| V-017 | Migration execute creates SI + updates source | ☐ | SMS-TC-007 |
| V-018 | Migration rollback restores original state | ☐ | SMS-TC-008 |
| V-019 | Exemption create → approve cycle works | ☐ | SMS-TC-009 |
| V-020 | Exemption revoke resets scan_result | ☐ | SMS-TC-010 |
| V-021 | Readiness score calculation is accurate | ☐ | SMS-TC-011 |
| V-022 | Exemption expiry detection works | ☐ | SMS-TC-012 |
| V-023 | Scan deduplication prevents duplicates | ☐ | SMS-TC-013 |
| V-024 | Scope filter limits scan correctly | ☐ | SMS-TC-014 |
| V-025 | Table filter limits scan correctly | ☐ | SMS-TC-015 |

---

## Edge Case Validation

| # | Check | Status | Notes |
|---|-------|--------|-------|
| V-026 | Empty script field skipped | ☐ | EC-001 |
| V-027 | Whitespace-only script skipped | ☐ | EC-002 |
| V-028 | >3900 char script: snippet truncated, pattern still detected | ☐ | EC-003 |
| V-029 | Multiple patterns: all recorded, worst severity wins | ☐ | EC-004 |
| V-030 | BLOCKING + WARNING: BLOCKING wins | ☐ | EC-005 |
| V-031 | Double-migration prevented | ☐ | EC-006 |
| V-032 | Rollback of non-migrated script returns error | ☐ | EC-007 |
| V-033 | Short justification rejected | ☐ | EC-008 |
| V-034 | Past expiry date rejected | ☐ | EC-009 |
| V-035 | Missing approver rejected | ☐ | EC-010 |
| V-036 | Zero-script instance: scan completes, score=100 | ☐ | EC-011 |
| V-037 | Concurrent scans: both complete independently | ☐ | EC-012 |
| V-038 | Name collision handled | ☐ | EC-013 |
| V-039 | Deleted source script: migration returns error | ☐ | EC-014 |
| V-040 | Long script name: truncated correctly | ☐ | EC-016 |
| V-041 | Non-ASCII characters: patterns still match | ☐ | EC-017 |
| V-042 | Invalid scanResultId: returns error | ☐ | EC-018 |
| V-043 | All-exempt: readiness score = 100 | ☐ | EC-019 |

---

## Regression Validation

| # | Check | Status | Notes |
|---|-------|--------|-------|
| V-044 | Pattern registry integrity after changes | ☐ | REG-001 |
| V-045 | Migration preview/execute/rollback cycle intact | ☐ | REG-002 |
| V-046 | Exemption state machine intact | ☐ | REG-003 |
| V-047 | Scan deduplication intact | ☐ | REG-004 |
| V-048 | Readiness score accuracy intact | ☐ | REG-005 |
| V-049 | Cross-scope read access intact | ☐ | REG-006 |
| V-050 | Scheduled jobs execute without errors | ☐ | REG-007 |
| V-051 | REST API endpoints functional | ☐ | REG-008 |

---

## Documentation Validation

| # | Check | Status | Notes |
|---|-------|--------|-------|
| V-052 | README ≥ 2000 words | ☑ | 2555 words — PASS |
| V-053 | README has Mermaid diagram | ☑ | Architecture diagram present — PASS |
| V-054 | README has ROI section | ☑ | ROI Analysis section present — PASS |
| V-055 | README has Troubleshooting section | ☑ | Troubleshooting section present — PASS |
| V-056 | README has no duplicate sections | ☑ | Verified — PASS |
| V-057 | LICENSE contains "Vladimir Kapustin" | ☑ | Copyright line present — PASS |
| V-058 | LICENSE is AGPL-3.0 | ☑ | Verified — PASS |
| V-059 | README license badge matches LICENSE | ☑ | AGPL-3.0 badge — PASS |
| V-060 | Phase 1 docs: architecture_summary.md | ☑ | 12,240 bytes — PASS |
| V-061 | Phase 1 docs: dependency_report.md | ☑ | 4,601 bytes — PASS |
| V-062 | Phase 1 docs: risk_report.md | ☑ | 10,202 bytes — PASS |
| V-063 | Phase 1 docs: execution_plan.md | ☑ | 6,861 bytes — PASS |
| V-064 | Phase 2 docs: test_suite_SOP.md (10+ scenarios) | ☑ | 15 scenarios — PASS |
| V-065 | Phase 2 docs: regression_cases.md | ☑ | 8 regression cases — PASS |
| V-066 | Phase 2 docs: edge_cases.md | ☑ | 20 edge cases — PASS |
| V-067 | Phase 2 docs: validation_checklist.md | ☑ | This document — PASS |
| V-068 | .gitignore exists | ☑ | Present — PASS |
| V-069 | No __pycache__ in repo | ☑ | Verified — PASS |
| V-070 | No hardcoded credentials in source | ☑ | Verified — PASS |

---

## Security Validation

| # | Check | Status | Notes |
|---|-------|--------|-------|
| V-071 | No PDI credentials in source code | ☑ | Verified — PASS |
| V-072 | No API tokens in source code | ☑ | Verified — PASS |
| V-073 | Cross-scope access documented | ☑ | In dependency_report.md — PASS |
| V-074 | Audit trail covers all operations | ☑ | scan, migrate, exempt, approve, revoke — PASS |
| V-075 | Rollback safety: pre-state stored before writes | ☑ | MigrationEngine stores pre-state first — PASS |

---

## Summary

| Category | Total | Pass | Fail | Pending |
|----------|-------|------|------|---------|
| Pre-Deployment | 10 | 0 | 0 | 10 |
| Functional | 15 | 0 | 0 | 15 |
| Edge Cases | 18 | 0 | 0 | 18 |
| Regression | 8 | 0 | 0 | 8 |
| Documentation | 19 | 19 | 0 | 0 |
| Security | 5 | 5 | 0 | 0 |
| **Total** | **75** | **24** | **0** | **51** |

**Note:** 51 checks marked PENDING require PDI access for live testing. All documentation and security checks pass. Functional, edge case, and regression tests are documented and ready for execution when PDI is available.
