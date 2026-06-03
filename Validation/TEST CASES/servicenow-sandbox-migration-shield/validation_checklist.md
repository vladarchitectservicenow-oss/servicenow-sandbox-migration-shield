# Validation Checklist — Sandbox Migration Shield

## Legend

| Code | Category |
|------|----------|
| D | Documentation |
| T | Testing |
| R | Reliability |
| L | Licensing & Legal |
| S | Security |
| G | Gate (quality gate) |
| F | Functional |

---

## Phase 1 — Documentation (D)

- [ ] D01: architecture_summary.md ≥ 50 lines with component table, data flow, performance benchmarks
- [ ] D02: dependency_report.md ≥ 50 lines with table names, plugin IDs, role lists, version matrix
- [ ] D03: risk_report.md ≥ 10 risk entries with severity tags (P0-P3)
- [ ] D04: execution_plan.md ≥ 6 phases with task/owner/status tables
- [ ] D05: ARCHITECTURE.md — high-level architecture diagram present
- [ ] D06: DESIGN.md — UI/UX design documented
- [ ] D07: PRD.md — user stories (≥12), functional requirements (≥5), non-functional requirements
- [ ] D08: SPEC.md — implementation detail per user story, algorithm pseudocode
- [ ] D09: README.md — ≥2000 words, Mermaid diagram, ROI analysis, troubleshooting table
- [ ] D10: MARKETING.md — enterprise one-pager with CTA
- [ ] D11: PITCH.md — 5-minute investor pitch
- [ ] D12: DEMO_SCRIPT.md — live demo walkthrough
- [ ] D13: No duplicate README sections (`grep '^## ' README.md | sort | uniq -d` returns empty)
- [ ] D14: README section count between 12–21 (`grep -c '^## ' README.md`)

## Phase 2 — Testing (T)

- [ ] T01: test_suite_SOP.md ≥ 10 scenarios (T01–T12 format with precondition/steps/expected/pass criteria)
- [ ] T02: regression_cases.md ≥ 8 cases (R01–R10 format)
- [ ] T03: edge_cases.md ≥ 5 cases (E01–E08 format)
- [ ] T04: validation_checklist.md ≥ 60 items with D/T/R/L/S/G/F codes
- [ ] T05: SandboxScanner unit tests — 100% pass rate
- [ ] T06: MigrationEngine unit tests — 100% pass rate
- [ ] T07: ExemptionManager unit tests — 100% pass rate
- [ ] T08: REST API integration tests — 100% pass rate
- [ ] T09: Regression test suite — 100% pass rate (no existing functionality broken)
- [ ] T10: Performance test — full scan completes in <2 minutes for 5,000 scripts
- [ ] T11: Execution history log exists at tests/execution_history/run_*.log
- [ ] T12: PDI smoke test passed (or deferred with PDI_STATUS.md documenting hibernation)

## Phase 3 — Reliability (R)

- [ ] R01: Singleton lock prevents concurrent scans (T04 verified)
- [ ] R02: Paginated scanning does not skip records on instances with >500 scripts (R03 verified)
- [ ] R03: Migration rollback restores byte-exact original script (R04 verified)
- [ ] R04: Transaction timeout recovery — scan resumes from last paginated page on retry
- [ ] R05: Emails fail gracefully — scan still completes if SMTP down
- [ ] R06: PDI hibernation does not corrupt custom tables (R09 verified)
- [ ] R07: Cross-scope grants detection: "not configured" error displayed before scan
- [ ] R08: Exemption expiry auto-transition does not affect other exemptions (R06 verified)
- [ ] R09: All mutations logged to x_snc_sms_audit_log with timestamp, user, action, target

## Phase 4 — Licensing & Legal (L)

- [ ] L01: LICENSE file present at repo root
- [ ] L02: LICENSE contains "Copyright (C) 2026 Vladimir Kapustin" (check with `grep`)
- [ ] L03: LICENSE is full AGPL-3.0 text (675+ lines), not short SPDX reference
- [ ] L04: Every .js source file has copyright header: `// Product Name — description // Copyright (C) 2026 Vladimir Kapustin // SPDX-License-Identifier: AGPL-3.0`
- [ ] L05: README license section matches LICENSE file (both AGPL-3.0, not mixed MIT/AGPL)
- [ ] L06: .gitattributes or .reuse/dep5 present for license compliance (optional but recommended)
- [ ] L07: No third-party code without attribution in src/ directory

## Phase 5 — Security (S)

- [ ] S01: No hardcoded credentials in source code (G5 gate — grep for literal passwords/tokens)
- [ ] S02: All credential reads use process.env or config files — no inline fallback values
- [ ] S03: ACL rules exist for all 4 custom tables (scan_result, scan_run, exemption, audit_log)
- [ ] S04: Cross-scope grants explicitly configured — no "trust all scopes" rule
- [ ] S05: REST API endpoints require authentication (snc_internal role minimum)
- [ ] S06: Audit log is append-only — no write/delete ACLs for any role on audit_log table
- [ ] S07: .gitignore present and excludes __pycache__/, *.pyc, reports/
- [ ] S08: No API keys, tokens, or secrets in marketing materials (MARKETING.md, PITCH.md)
- [ ] S09: No instance URLs in source code comments (PDI URL only in PDI_STATUS.md)

## Phase 6 — Gates (G)

- [ ] G01: test_suite_SOP.md ≥ 10 scenarios including negative cases
- [ ] G02: tests/execution_history/*.log confirms ALL PASS
- [ ] G03: README.md ≥ 2000 words with Mermaid + ROI (verified via GitHub raw)
- [ ] G04: Every src/ file has AGPL-3.0 copyright header
- [ ] G05: Git push verified via GitHub API (200 on branch query)
- [ ] G06: No hardcoded credentials in source code (exclude process.env patterns)
- [ ] G07: README license header matches LICENSE file (both AGPL-3.0)
- [ ] G08: No duplicate README sections (section count 12–21, no `sort | uniq -d` output)
- [ ] G09: DONE.marker file present at repo root with timestamp of completion
- [ ] G10: Phase 1+2 docs verified on remote via GitHub Contents API (200 for each file)

## Phase 7 — Functional (F)

- [ ] F01: Scan covers all 6 target tables (sys_script, sys_script_include, sys_script_client, sys_ui_policy, sys_ui_action, sys_dictionary)
- [ ] F02: Severity classification: BLOCKING / WARNING / COMPATIBLE mapped correctly to KB2944435 patterns
- [ ] F03: Full scan and incremental scan both produce correct results
- [ ] F04: Migration preview shows exact generated code without modifying source
- [ ] F05: Migration execute creates Script Include + updates source + marks MIGRATED
- [ ] F06: Rollback restores original script + removes generated Script Include
- [ ] F07: Exemption create validates justification length (≥50 chars)
- [ ] F08: Exemption approval updates status and recalculates readiness score
- [ ] F09: Exemption expiry auto-transitions to EXPIRED after expires_at date
- [ ] F10: Dashboard returns correct readiness_score formula
- [ ] F11: REST endpoints return correct HTTP status codes (200/400/404/409)
- [ ] F12: Scheduled jobs fire correctly (weekly scan, daily exemption check)

---

## Validation Summary

| Phase | Items | Passed | Failed | Skipped | % Complete |
|-------|-------|--------|--------|---------|------------|
| Phase 1 — Documentation | 14 | | | | |
| Phase 2 — Testing | 12 | | | | |
| Phase 3 — Reliability | 9 | | | | |
| Phase 4 — Licensing | 7 | | | | |
| Phase 5 — Security | 9 | | | | |
| Phase 6 — Gates | 10 | | | | |
| Phase 7 — Functional | 12 | | | | |
| **Total** | **73** | | | | |

---

## Validation Notes

- PDI smoke tests may be deferred if dev362840 is hibernating. Document in PDI_STATUS.md.
- Performance tests (T10, F03) require instance with 5000+ scripts — simulate with Python mock for CI.
- Cross-scope grants MUST be configured before any scan test; pre-flight check detects misconfiguration.
