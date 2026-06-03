# Execution Plan — Sandbox Migration Shield

## Summary

| Property | Value |
|----------|-------|
| Product | Sandbox Migration Shield |
| Scope ID | x_snc_sms |
| Release Target | Zurich → Australia |
| Timeline | 8 weeks (MVP v1.0) |
| Team Size | 1 developer + manual QA |
| Methodology | Agile / Kanban |

## Phase 1: Foundation & Scaffolding (Week 1)

| Task ID | Task | Owner | Effort | Status | Deliverable |
|---------|------|-------|--------|--------|-------------|
| T01 | Create scoped application in PDI (x_snc_sms) | Dev | 2h | ⬜ Pending | sys_app record + scope |
| T02 | Define custom tables: x_snc_sms_scan_result, x_snc_sms_scan_run, x_snc_sms_exemption, x_snc_sms_audit_log | Dev | 4h | ⬜ Pending | Table definitions + fields + indexes |
| T03 | Configure cross-scope privilege grants (read on sys_script, sys_script_include, sys_script_client, sys_ui_policy, sys_ui_action, sys_dictionary) | Dev | 2h | ⬜ Pending | ACL grants in scope record |
| T04 | Create user roles: x_snc_sms.admin, x_snc_sms.user, x_snc_sms.approver, x_snc_sms.viewer | Dev | 2h | ⬜ Pending | Role definitions + ACL rules |
| T05 | Set up ACL rules for all custom tables | Dev | 3h | ⬜ Pending | ACL entries per table per operation |
| T06 | Create .gitignore with __pycache__/, *.pyc, reports/ entries | Dev | 0.5h | ⬜ Pending | .gitignore file |
| **Phase 1 Total** | | | **13.5h** | | |

## Phase 2: Core Scanner (Week 2)

| Task ID | Task | Owner | Effort | Status | Deliverable |
|---------|------|-------|--------|--------|-------------|
| T07 | Implement KB2944435 Pattern Registry (PATTERNS[] array with code, regex, severity, message) | Dev | 4h | ⬜ Pending | SandboxScanner.js — PATTERNS[] constant |
| T08 | Implement SandboxScanner.scanAll() — multi-table GlideRecord paginated scanning | Dev | 8h | ⬜ Pending | SandboxScanner.js — scanAll method |
| T09 | Implement severity classification: BLOCKING / WARNING / COMPATIBLE | Dev | 2h | ⬜ Pending | SandboxScanner.js — classify method |
| T10 | Implement scan_run lifecycle: create → RUNNING → populate → completed_at + counts | Dev | 3h | ⬜ Pending | SandboxScanner.js — scanRun lifecycle |
| T11 | Implement singleton lock: detect existing RUNNING scan, prevent concurrent execution | Dev | 2h | ⬜ Pending | SandboxScanner.js — isScanRunning check |
| T12 | Implement incremental scan mode: only scripts modified since last scan | Dev | 3h | ⬜ Pending | SandboxScanner.js — incremental mode |
| T13 | Write unit tests for SandboxScanner (≥10 test cases covering all tables, severities, edge cases) | Dev | 4h | ⬜ Pending | tests/test_sandbox_scanner.js |
| T14 | Run unit tests → fix failures → achieve 100% PASS | Dev | 2h | ⬜ Pending | Test execution log |
| **Phase 2 Total** | | | **28h** | | |

## Phase 3: Migration Engine (Week 3)

| Task ID | Task | Owner | Effort | Status | Deliverable |
|---------|------|-------|--------|--------|-------------|
| T15 | Implement MigrationEngine.preview() — extract script, classify type, generate Script Include code + replacement call | Dev | 6h | ⬜ Pending | MigrationEngine.js — preview method |
| T16 | Implement MigrationEngine.execute() — create Script Include, update source field, mark scan_result MIGRATED | Dev | 5h | ⬜ Pending | MigrationEngine.js — execute method |
| T17 | Implement MigrationEngine.rollback() — restore original script, delete generated Script Include | Dev | 4h | ⬜ Pending | MigrationEngine.js — rollback method |
| T18 | Implement inline expression detection vs multi-line script detection | Dev | 2h | ⬜ Pending | MigrationEngine.js — classifyScript |
| T19 | Implement Script Include name sanitization (remove special chars, append "_Migrated") | Dev | 1h | ⬜ Pending | MigrationEngine.js — sanitizeName |
| T20 | Write unit tests for MigrationEngine (≥8 test cases: preview, execute, rollback, edge cases) | Dev | 4h | ⬜ Pending | tests/test_migration_engine.js |
| T21 | Run unit tests → fix failures → achieve 100% PASS | Dev | 2h | ⬜ Pending | Test execution log |
| **Phase 3 Total** | | | **24h** | | |

## Phase 4: Exemption Manager (Week 4)

| Task ID | Task | Owner | Effort | Status | Deliverable |
|---------|------|-------|--------|--------|-------------|
| T22 | Implement ExemptionManager.createExemption() — with validation (justification min 50 chars, approver required) | Dev | 3h | ⬜ Pending | ExemptionManager.js — createExemption |
| T23 | Implement ExemptionManager.approveExemption() / rejectExemption() — status transitions | Dev | 2h | ⬜ Pending | ExemptionManager.js — approve/reject |
| T24 | Implement ExemptionManager.revokeExemption() — return to NEW status | Dev | 1h | ⬜ Pending | ExemptionManager.js — revokeExemption |
| T25 | Implement ExemptionManager.getExpiringExemptions(daysThreshold) — for scheduled job | Dev | 2h | ⬜ Pending | ExemptionManager.js — getExpiringExemptions |
| T26 | Implement ExemptionManager.calculateReadinessScore() — exclude exempted scripts from BLOCKING count | Dev | 2h | ⬜ Pending | ExemptionManager.js — calculateReadinessScore |
| T27 | Write unit tests for ExemptionManager (≥8 test cases: create, approve, reject, revoke, expire, score) | Dev | 3h | ⬜ Pending | tests/test_exemption_manager.js |
| T28 | Run unit tests → fix failures → achieve 100% PASS | Dev | 1h | ⬜ Pending | Test execution log |
| **Phase 4 Total** | | | **14h** | | |

## Phase 5: REST API & Scheduled Jobs (Week 5)

| Task ID | Task | Owner | Effort | Status | Deliverable |
|---------|------|-------|--------|--------|-------------|
| T29 | Implement Scripted REST endpoint: POST /api/x_snc_sms/v1/scan | Dev | 2h | ⬜ Pending | scan_api.js |
| T30 | Implement Scripted REST endpoint: GET /api/x_snc_sms/v1/scan/{id} | Dev | 2h | ⬜ Pending | scan_api.js — get handler |
| T31 | Implement Scripted REST endpoint: POST /api/x_snc_sms/v1/migrate | Dev | 2h | ⬜ Pending | migration_api.js |
| T32 | Implement Scripted REST endpoint: POST /api/x_snc_sms/v1/preview-migration | Dev | 1h | ⬜ Pending | migration_api.js — preview |
| T33 | Implement Scripted REST endpoints: POST + GET/PATCH /api/x_snc_sms/v1/exemption/{id} | Dev | 3h | ⬜ Pending | exemption_api.js |
| T34 | Implement Scripted REST endpoint: GET /api/x_snc_sms/v1/dashboard | Dev | 2h | ⬜ Pending | dashboard_api.js |
| T35 | Implement Scheduled Job: Weekly Auto-Scan (Sunday 02:00) | Dev | 2h | ⬜ Pending | weekly_scan.js |
| T36 | Implement Scheduled Job: Exemption Expiry Check (daily 08:00) | Dev | 2h | ⬜ Pending | exemption_check.js |
| T37 | Write REST API integration tests (≥6 scenarios: scan, migrate, preview, exemption CRUD, dashboard) | Dev | 4h | ⬜ Pending | tests/test_rest_api.js |
| T38 | Run integration tests → fix failures → achieve 100% PASS | Dev | 2h | ⬜ Pending | Test execution log |
| **Phase 5 Total** | | | **22h** | | |

## Phase 6: UI & Portal Widgets (Week 6)

| Task ID | Task | Owner | Effort | Status | Deliverable |
|---------|------|-------|--------|--------|-------------|
| T39 | Implement Dashboard Widget: readiness score gauge + severity pie chart + trend line | Dev | 8h | ⬜ Pending | dashboard.js |
| T40 | Implement Scan Widget: trigger button + progress indicator + completion summary | Dev | 4h | ⬜ Pending | scan_widget.js |
| T41 | Implement Script List Widget: filterable/sortable table with severity badges, status indicators, search | Dev | 6h | ⬜ Pending | script_list.js |
| T42 | Implement Exemption Widget: creation form with justification textarea, approver picker, expiry date picker | Dev | 4h | ⬜ Pending | exemption_widget.js |
| T43 | Manual UI smoke test: login → trigger scan → verify results appear → create exemption → approve → check dashboard | QA | 3h | ⬜ Pending | PDI smoke test log |
| **Phase 6 Total** | | | **25h** | | |

## Phase 7: QA & Validation (Week 7)

| Task ID | Task | Owner | Effort | Status | Deliverable |
|---------|------|-------|--------|--------|-------------|
| T44 | Run full test suite: all unit tests + integration tests + UI smoke tests | QA | 4h | ⬜ Pending | Test run log (100% PASS) |
| T45 | Regression testing: verify Phase 3 blocking patterns against known-sandbox-breaking scripts | QA | 3h | ⬜ Pending | Regression test log |
| T46 | Performance testing: scan 1,000 mock scripts, verify <2 minute duration | QA | 3h | ⬜ Pending | Performance report |
| T47 | Cross-scope access audit: verify all platform table reads work with granted ACLs | QA | 2h | ⬜ Pending | ACL verification log |
| T48 | Rollback testing: migrate 10 scripts → rollback all 10 → verify original content restored | QA | 2h | ⬜ Pending | Rollback test log |
| T49 | Security review: verify no credential leaks, ACL enforcement, audit log immutability | QA | 3h | ⬜ Pending | Security review report |
| T50 | Final README review: word count ≥2000, Mermaid diagram, ROI analysis, troubleshooting | Dev | 2h | ⬜ Pending | README.md final |
| **Phase 7 Total** | | | **19h** | | |

## Phase 8: Deployment & Marketing (Week 8)

| Task ID | Task | Owner | Effort | Status | Deliverable |
|---------|------|-------|--------|--------|-------------|
| T51 | Package scoped app: export sys_app.xml + all artifacts | Dev | 2h | ⬜ Pending | Sys app XML export |
| T52 | Create Update Set for manual installation path | Dev | 1h | ⬜ Pending | Update Set XML |
| T53 | Push to GitHub: commit all source, docs, tests → push to origin/main | Dev | 1h | ⬜ Pending | GitHub push verification |
| T54 | Create MARKETING.md — enterprise one-pager | Dev | 2h | ⬜ Pending | MARKETING.md |
| T55 | Create PITCH.md — 5-minute investor pitch | Dev | 1h | ⬜ Pending | PITCH.md |
| T56 | Create DEMO_SCRIPT.md — live demo walkthrough | Dev | 2h | ⬜ Pending | DEMO_SCRIPT.md |
| T57 | Write 3-post LinkedIn thread | Dev | 1h | ⬜ Pending | LINKEDIN_POST.md |
| T58 | PDI smoke test: deploy to dev362840 → run scan → verify results | QA | 2h | ⬜ Pending | PDI smoke test log |
| T59 | Create DONE.marker | Dev | 0.1h | ⬜ Pending | DONE.marker |
| **Phase 8 Total** | | | **12.1h** | | |

## Resource Summary

| Phase | Week | Hours | Key Milestone |
|-------|------|-------|---------------|
| Phase 1: Foundation | 1 | 13.5h | Scope + tables + ACLs ready |
| Phase 2: Core Scanner | 2 | 28h | Scanner 100% tests passing |
| Phase 3: Migration Engine | 3 | 24h | MigrationEngine 100% tests passing |
| Phase 4: Exemption Manager | 4 | 14h | ExemptionManager 100% tests passing |
| Phase 5: REST API + Scheduled Jobs | 5 | 22h | All endpoints + jobs tested |
| Phase 6: UI + Portal | 6 | 25h | Portal widgets + UI smoke test |
| Phase 7: QA & Validation | 7 | 19h | Full regression + perf + security |
| Phase 8: Deploy + Marketing | 8 | 12.1h | GitHub push + marketing collateral |
| **Total** | **8 weeks** | **157.6h** | |

## Critical Path

```
T01 → T07 → T08 → T15 → T16 → T29 → T44 → T51 → T53 (DONE.marker)
  └─ T03 (cross-scope grants — parallel, but blocking for T08 scanning)
```

**Critical path length:** 8 weeks. No parallelization beyond cross-scope grants (T03).

## Dependencies Between Phases

```
Phase 1 ──► Phase 2 ──► Phase 3 ──► Phase 4 ──► Phase 5
                                                     │
                              Phase 6 ◄───────────────┘
                                                     │
                              Phase 7 ◄───────────────┘
                                                     │
                              Phase 8 ◄───────────────┘
```

## Risk Mitigation Triggers

| If | Then |
|----|------|
| Cross-scope grants not approved by Week 1 | Escalate to platform admin; scanner blocked |
| Pattern registry missing known pattern | Add pattern via hotfix; increment registry version |
| Migration rollback fails | Block execute() until rollback passes; escalate P1 |
| PDI hibernates during QA | Fall back to Python mock CI; mark PDI tests deferred |
| Transaction timeout on >5K scripts | Increase setLimit to 1000; add progress logging per 100 scripts |

## Definition of Done

Each task is complete when:
1. Code committed to GitHub with proper copyright header
2. Unit tests written and passing (100%)
3. No lint errors (ESLint where applicable)
4. Documentation updated (inline comments + design docs)
5. Peer reviewed (for Phase 2+ core components)
6. Integration test passed (for Phase 5+ components)
7. No hardcoded credentials in source code (G5 gate)
