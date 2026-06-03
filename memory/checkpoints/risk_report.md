# Risk Report — Sandbox Migration Shield

## Risk Register

### Critical (P0) — Showstoppers

| ID | Risk | Probability | Impact | Mitigation | Owner | Status |
|----|------|------------|--------|------------|-------|--------|
| R01 | **Cross-scope privilege grants produce zero results:** Scanner's `GlideRecord` queries return zero rows silently if read ACLs for platform tables (sys_script, etc.) are not granted to the scoped app | HIGH (70%) | **CRITICAL** — Scanner passes with zero findings, user believes instance is clean, Phase 3 blocks scripts in production | **Pre-flight check:** On first run, query sys_script table and validate `gr.getRowCount() > 0`. If zero, display "Cross-scope access not configured — please grant read ACLs" in red banner | Admin | Mitigated |
| R02 | **KB2944435 pattern registry incomplete:** ServiceNow may introduce additional blocking patterns not covered by the initial regex set | MEDIUM (50%) | **HIGH** — Scripts flagged as COMPATIBLE become BLOCKING at runtime after future KB update | **Design:** Pattern registry in a modular `PATTERNS[]` array with version-tagged entries. Document update procedure in ADMIN_GUIDE.md. Run pattern coverage audit quarterly against latest KB | Dev | Mitigated |
| R03 | **MigrationEngine overwrites script field without backup:** If MigrationEngine.update fails mid-operation (transaction timeout), the original script content is lost | LOW (10%) | **CRITICAL** — Corrupted Business Rule or UI Policy in production | **Defense-in-depth:** (1) Store original script in `scan_result.source_code_snippet` before any mutation; (2) Transaction wraps both Script Include create + source update in a single batch; (3) Rollback reads from audit log's stored original | Dev | Mitigated |
| R04 | **Exemption expires but script is still incompatible:** Exemption has expires_at date; if not renewed, the script may be blocked in production after Phase 3 | MEDIUM (40%) | **HIGH** — Production incident from blocked scripts with expired exemptions | **Scheduled Job:** Daily check at 08:00 scans for exemptions expiring in 30 days, emails owner. Expired exemptions auto-change scan_result status back to NEW | Dev | Mitigated |

### High (P1) — Requires Monitoring

| ID | Risk | Probability | Impact | Mitigation | Owner | Status |
|----|------|------------|--------|------------|-------|--------|
| R05 | **Regex false negatives (obfuscated patterns):** Minified/obfuscated scripts may contain blocked APIs in unrecognizable forms (e.g., `window['ev'+'al']('code')`) | MEDIUM (30%) | MEDIUM — Missed BLOCKING → runtime failure | Document limitation clearly in README. Recommend code review for obfuscated scripts. Add heuristic detection: flag any script with `eval` in non-standard contexts (computed property access, concatenation) as MANUAL_REVIEW | Dev | Acknowledged |
| R06 | **Migration introduces functional regression:** Extracted logic in Script Include may behave differently from inline code due to scoping differences (`current`, `previous`, `g_scratchpad` availability) | MEDIUM (25%) | HIGH — Production bug from migrated script | **Preview mode mandatory:** User must preview generated Script Include code BEFORE executing. Test suite covers scoping edge cases. Rollback mechanism tested for immediate reversal | Dev | Mitigated |
| R07 | **Scanner transaction timeout on 10,000+ scripts:** Despite pagination (setLimit(500)), instances with very large script inventories may still timeout | LOW (15%) | MEDIUM — Scan incomplete, false sense of completeness | Incremental scan mode: scan only scripts modified since last run. Full scan as off-hours scheduled job with 2-hour window. Progress logging per 100 scripts | Dev | Mitigated |
| R08 | **Concurrent scan runs corrupt state:** Two admins trigger scans simultaneously → duplicate scan_run records, conflicting scan_result writes | LOW (10%) | MEDIUM — Data inconsistency | **Singleton lock:** `SandboxScanner` checks for existing scan_run with status RUNNING. If found, returns "Scan already in progress (started by {user} at {time})" | Dev | Mitigated |
| R09 | **PDI hibernation blocks testing:** dev362840 hibernates after 10 days, making PDI smoke tests unreliable | HIGH (80%) | LOW — Testing delayed, not blocked | Deferred to manual PDI-wake. Core tests run in Python mock CI (no PDI needed). Status documented in PDI_STATUS.md | QA | Acknowledged |

### Medium (P2) — Acceptable

| ID | Risk | Probability | Impact | Mitigation | Owner | Status |
|----|------|------------|--------|------------|-------|--------|
| R10 | **GlideStringUtil deprecation misclassification:** Some GlideStringUtil methods remain compatible — flagging all as WARNING creates noise | MEDIUM (40%) | LOW — Admin fatigue from false warnings | Pattern registry distinguishes safe methods (e.g., `GlideStringUtil.base64Encode`) from deprecated ones. Granularity: per-method patterns, not catch-all class match | Dev | Accepted |
| R11 | **REST API rate limiting on dashboard refresh:** Dashboard widget polled every 10s by multiple users → GlideAggregate spikes | LOW (10%) | LOW — Brief slowdown | Client-side caching: widget polls max every 30s. Server-side: cache dashboard response for 60s in a system property | Dev | Accepted |
| R12 | **Email report fails if SMTP misconfigured:** Weekly scheduled scan report depends on sys_email functioning | MEDIUM (30%) | LOW — Report not sent; scan results still available in UI | Graceful degradation: if email fails, log warning to system log; do not block scan execution | Dev | Accepted |

### Low (P3) — Monitor Only

| ID | Risk | Probability | Impact | Mitigation | Owner | Status |
|----|------|------------|--------|------------|-------|--------|
| R13 | **Class.create() incompatible with future ES12 migration:** ServiceNow is planning ES12 JavaScript engine — `Class.create()` will be deprecated | LOW (5%) | MEDIUM — Future maintenance cost | Documented in dependency_report.md. Migration path to ES6 `class` syntax prepared in DESIGN.md. Not blocking for Zurich/Australia | Dev | Tracked |
| R14 | **Portal widget rendering breaks on mobile:** Dashboard designed for desktop viewport; mobile may clip UI | LOW (15%) | LOW — Mobile admin usage is rare | Responsive fallback: widgets use flex-wrap. Documented limitation in README | Dev | Accepted |
| R15 | **Non-English instance character encoding:** Script names/descriptions in CJK or Cyrillic may cause display issues in scan results | LOW (10%) | LOW — String fields use UTF-8, GlideRecord handles encoding | Tested with Russian characters in script names (passed). No known issues | QA | Verified |

## Risk Summary Heat Map

```
Impact
  ↑
  │  R02   R04   │  R01   R03
  │  R06         │
  │──────────────│──────────────
  │  R05   R07   │
  │  R08   R10   │
  │  R09   R12   │  R11   R13
  │──────────────│──────────────
  │              │
  └──────────────────────────→ Probability

  P3 (Low)       P2 (Med)   P1 (High)   P0 (Critical)
```

### Risk Count by Severity

| Severity | Count | Mitigated | Acknowledged | Accepted |
|----------|-------|-----------|-------------|----------|
| P0 (Critical) | 4 | 3 | 1 | 0 |
| P1 (High) | 5 | 4 | 1 | 0 |
| P2 (Medium) | 3 | 0 | 0 | 3 |
| P3 (Low) | 3 | 0 | 1 | 2 |
| **Total** | **15** | **7** | **3** | **5** |

## Top 5 Risks to Watch

1. **R01 (P0):** Cross-scope grants — pre-flight check mitigates but requires admin action
2. **R02 (P0):** Pattern registry completeness — ongoing maintenance burden; quarterly audit required
3. **R04 (P0):** Exemption expiry → production blocking — daily scheduled job mitigates
4. **R03 (P1):** Migration data loss — defense-in-depth (backup + transaction + rollback)
5. **R06 (P1):** Migration functional regression — preview mode mandatory; test suite covers edge cases

## Risk Register Versioning

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-06-03 | Initial risk assessment for v1.0 MVP |

## Review Cadence

- **After each production scan cycle:** Review R01, R02, R07
- **After each migration execution:** Review R03, R06
- **Quarterly:** Pattern registry coverage audit (R02, R10)
- **Before Australia upgrade:** Full risk register review
