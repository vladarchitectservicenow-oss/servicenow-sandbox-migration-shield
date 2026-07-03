# Sandbox Migration Shield — Dependency Report

**Product:** Sandbox Migration Shield  
**Scope:** `x_snc_sms`  
**Author:** Vladimir Kapustin  
**Last Updated:** 2026-07-03

---

## 1. ServiceNow Platform Dependencies

| Dependency | Version | Type | Criticality | Notes |
|------------|---------|------|-------------|-------|
| ServiceNow Instance | Zurich+ | Runtime | P0 — BLOCKING | Requires Zurich or later for KB2944435 Phase 2+3 awareness |
| GlideRecord API | Built-in | Runtime | P0 — BLOCKING | Core data access for all scan/migrate/exempt operations |
| GlideDateTime API | Built-in | Runtime | P0 — BLOCKING | Timestamp handling for scan runs, exemptions, audit logs |
| GlideAggregate API | Built-in | Runtime | P1 — HIGH | Readiness score calculation, status counting |
| GlideSystem (gs) API | Built-in | Runtime | P1 — HIGH | Logging (gs.info, gs.error), user context (gs.getUserID), event queue (gs.eventQueue) |
| Event Queue | Built-in | Runtime | P2 — MEDIUM | Approver notifications for exemption workflow |
| sys_script table | Platform | Read | P0 — BLOCKING | Primary scan target — Business Rules |
| sys_script_include table | Platform | Read/Write | P0 — BLOCKING | Scan target + migration destination |
| sys_script_client table | Platform | Read | P0 — BLOCKING | Client Script scan target |
| sys_ui_policy table | Platform | Read | P0 — BLOCKING | UI Policy scan target |
| sys_ui_action table | Platform | Read | P0 — BLOCKING | UI Action scan target |
| sys_dictionary table | Platform | Read | P0 — BLOCKING | Dictionary default_value/calculation scan target |
| Scripted REST API Framework | Built-in | Runtime | P2 — MEDIUM | External integration endpoint |

---

## 2. External Dependencies

| Dependency | Type | Criticality | Notes |
|------------|------|-------------|-------|
| None | — | — | Fully self-contained scoped application. No external services, libraries, or APIs required. |

---

## 3. Cross-Scope Access Requirements

| Target Scope | Access Type | Tables | Justification |
|-------------|-------------|--------|---------------|
| Global | READ | sys_script, sys_script_include, sys_script_client, sys_ui_policy, sys_ui_action, sys_dictionary | Scan engine must read script content from all platform tables |
| Global | WRITE | sys_script_include | Migration engine creates new Script Includes in target scope |
| Global | WRITE | sys_script, sys_script_client, sys_ui_policy, sys_ui_action, sys_dictionary | Migration engine updates source records with replacement calls; rollback restores originals |

**Risk:** Cross-scope write access to platform tables requires explicit privilege grants. Without them, migration operations will silently fail (GlideRecord returns empty on cross-scope queries). See Risk Report § P1-03.

---

## 4. Plugin Dependencies

| Plugin | ID | Required | Notes |
|--------|----|----------|-------|
| None | — | No | No plugin dependencies. Works on vanilla Zurich+ instances. |

---

## 5. Scheduled Job Dependencies

| Job | Schedule | Depends On | Notes |
|-----|----------|------------|-------|
| Weekly Scan | Every Sunday 02:00 | SandboxScanner Script Include | Automated full scan of all script tables |
| Weekly Report | Every Monday 08:00 | scan_run + scan_result tables | Email summary to platform owners |
| Exemption Expiry Check | Daily 06:00 | ExemptionManager Script Include | Flags exemptions expiring within 30 days |

---

## 6. Dependency Risk Matrix

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Instance below Zurich | Low | BLOCKING | Document minimum version requirement; add version check on install |
| Cross-scope access denied | Medium | BLOCKING | Document required privilege grants; add pre-flight check |
| Plugin-dependent table missing | None | N/A | No plugin dependencies |
| GlideAggregate unavailable | Very Low | HIGH | Fallback to GlideRecord counting (slower but functional) |
| Event Queue disabled | Low | LOW | Exemption workflow degrades gracefully — manual approval still works |

---

## 7. Version Compatibility Matrix

| ServiceNow Release | KB2944435 Phase | Compatibility | Notes |
|--------------------|-----------------|---------------|-------|
| Xanadu | Phase 1 (Notification) | ✅ Compatible | Scanner works; migration optional |
| Yokohama | Phase 2 (Review) | ✅ Compatible | Full functionality |
| Zurich | Phase 2→3 (Transition) | ✅ Primary Target | All features active |
| Australia | Phase 3 (Enforcement) | ✅ Compatible | Critical — blocking enforcement active |
