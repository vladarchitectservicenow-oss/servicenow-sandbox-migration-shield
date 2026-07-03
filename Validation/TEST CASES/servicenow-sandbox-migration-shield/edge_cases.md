# Sandbox Migration Shield — Edge Cases

**Product:** Sandbox Migration Shield  
**Scope:** `x_snc_sms`  
**Author:** Vladimir Kapustin  
**Last Updated:** 2026-07-03

---

## Purpose

This document catalogs edge cases, boundary conditions, and failure modes that must be handled correctly. Each edge case includes expected behavior and verification method.

---

## EC-001: Empty Script Field

**Description:** A Business Rule exists but its `script` field is empty or null.  
**Expected Behavior:** Scanner skips the field (content.length < 3 check). No scan_result created.  
**Risk if mishandled:** NullPointerException or empty scan_result with no useful data.  
**Verification:** Create a BR with empty script field, run scan, verify no scan_result for that field.

---

## EC-002: Script with Only Whitespace

**Description:** Script field contains only spaces, tabs, or newlines.  
**Expected Behavior:** Scanner skips (content.length < 3 after trim).  
**Risk if mishandled:** False COMPATIBLE classification for empty content.  
**Verification:** Create BR with `"   \n  "` as script, run scan, verify no result.

---

## EC-003: Script Exceeding 3900 Characters

**Description:** A script field contains >3900 characters.  
**Expected Behavior:** `source_code_snippet` truncated to 3900 chars + "...". Full content NOT stored in scan_result.  
**Risk if mishandled:** Truncation in middle of a pattern keyword → false negative.  
**Verification:** Create BR with 5000-char script containing `eval()` at position 4000. Verify BLOCKING still detected (regex runs on full content, only snippet is truncated).

---

## EC-004: Multiple Patterns in One Script

**Description:** A single script matches multiple KB2944435 patterns (e.g., both `Packages.java` and `eval()`).  
**Expected Behavior:** Severity = BLOCKING (worst-case wins). `issue_description` JSON array contains all matched patterns.  
**Risk if mishandled:** Only first pattern recorded; severity downgraded to WARNING if BLOCKING pattern checked second.  
**Verification:** Create BR with both `Packages.java.lang.String` and `eval()`. Verify severity=BLOCKING, issue_description has 2 entries.

---

## EC-005: Script with BLOCKING + WARNING Patterns

**Description:** Script matches one BLOCKING and one WARNING pattern.  
**Expected Behavior:** Severity = BLOCKING (BLOCKING always wins over WARNING).  
**Risk if mishandled:** If WARNING checked after BLOCKING and overwrites severity.  
**Verification:** Create BR with `Packages.java` (BLOCKING) and `gs.sleep()` (WARNING). Verify severity=BLOCKING.

---

## EC-006: Migration of Already-Migrated Script

**Description:** User attempts to migrate a scan_result with status=MIGRATED.  
**Expected Behavior:** MigrationEngine should detect status and return error or skip.  
**Risk if mishandled:** Duplicate Script Include created; source script overwritten with nested replacement call.  
**Verification:** Migrate script → attempt second migration → verify error returned.

---

## EC-007: Rollback of Non-Migrated Script

**Description:** User attempts to rollback a scan_result with status=NEW or EXEMPT.  
**Expected Behavior:** `rollback()` returns `{ error: "Script not in MIGRATED state" }`.  
**Risk if mishandled:** Attempts to parse null pre-state → JSON parse error.  
**Verification:** Call rollback on NEW scan_result → verify error message.

---

## EC-008: Exemption with Short Justification

**Description:** User provides justification <50 characters.  
**Expected Behavior:** `createExemption()` returns `{ error: "Justification must be at least 50 characters" }`.  
**Risk if mishandled:** Exemption created with inadequate audit trail → compliance failure.  
**Verification:** Call createExemption with "too short" → verify error.

---

## EC-009: Exemption with Past Expiry Date

**Description:** User sets expiry date in the past.  
**Expected Behavior:** `createExemption()` returns `{ error: "Expiration date must be in the future" }`.  
**Risk if mishandled:** Immediately-expired exemption → compliance gap.  
**Verification:** Call createExemption with `expiryDate = "2020-01-01"` → verify error.

---

## EC-010: Exemption Without Approver

**Description:** User omits approverId.  
**Expected Behavior:** `createExemption()` returns `{ error: "Approver is required" }`.  
**Risk if mishandled:** Exemption created with no approval chain → un-auditable.  
**Verification:** Call createExemption with `approverId = ""` → verify error.

---

## EC-011: Scan on Instance with Zero Scripts

**Description:** Fresh instance with no custom scripts.  
**Expected Behavior:** scan_run created with total_scripts=0, status=COMPLETED. No scan_result records.  
**Risk if mishandled:** Division by zero in readiness score; null pointer on empty result set.  
**Verification:** Run scan on clean instance → verify total_scripts=0, readiness score returns 100.

---

## EC-012: Concurrent Scan Runs

**Description:** Two scan runs triggered simultaneously.  
**Expected Behavior:** Both complete independently. Each scan_run has its own set of scan_results.  
**Risk if mishandled:** Race condition on scan_result deduplication — one scan overwrites the other's results.  
**Verification:** Trigger two scans in rapid succession → verify both scan_runs have correct counts.

---

## EC-013: Script Include Name Collision

**Description:** Migration generates a Script Include name that already exists.  
**Expected Behavior:** MigrationEngine should detect collision and append suffix or return error.  
**Risk if mishandled:** `insert()` fails silently or overwrites existing Script Include.  
**Verification:** Create a Script Include with the auto-generated name → attempt migration → verify collision handled.

---

## EC-014: Source Script Deleted Before Migration

**Description:** scan_result references a script that was deleted between scan and migration.  
**Expected Behavior:** `_getSourceScript()` returns null → `preview()` returns `{ error: "Source script not found or already migrated" }`.  
**Risk if mishandled:** Null reference → script crash.  
**Verification:** Create BR → scan → delete BR → attempt migration → verify error.

---

## EC-015: Exemption Approval by Non-Approver

**Description:** User other than the designated approver attempts to approve.  
**Expected Behavior:** Currently not enforced (any user with write access can approve). This is a known gap.  
**Risk if mishandled:** Unauthorized approval → compliance gap.  
**Verification:** Document as known limitation. Add approver validation in v1.1.

---

## EC-016: Very Long Script Name

**Description:** Script name exceeds 100 characters.  
**Expected Behavior:** `_generateName()` sanitizes and truncates to valid Script Include name length.  
**Risk if mishandled:** `insert()` fails due to field length constraint.  
**Verification:** Create BR with 200-char name → migrate → verify Script Include created with truncated name.

---

## EC-017: Script with Non-ASCII Characters

**Description:** Script contains Unicode characters (Cyrillic comments, emoji in strings).  
**Expected Behavior:** Regex patterns still match correctly. `source_code_snippet` stores UTF-8 content.  
**Risk if mishandled:** Encoding error; regex fails on multi-byte characters.  
**Verification:** Create BR with `// проверка` comment + `eval()` → verify BLOCKING detected.

---

## EC-018: MigrationEngine with No Scan Results

**Description:** MigrationEngine.preview() called with non-existent scanResultId.  
**Expected Behavior:** Returns `{ error: "Scan result not found" }`.  
**Risk if mishandled:** Null reference → script crash.  
**Verification:** Call preview with invalid ID → verify error.

---

## EC-019: Readiness Score with All Scripts Exempt

**Description:** All BLOCKING scripts are exempted; no COMPATIBLE or MIGRATED scripts.  
**Expected Behavior:** `calculateReadinessScore()` returns 100 (total = 0 after excluding exempt → return 100).  
**Risk if mishandled:** Division by zero.  
**Verification:** Exempt all scripts → calculate score → verify 100.

---

## EC-020: GlideAggregate Unavailable (Fallback)

**Description:** GlideAggregate API is restricted or unavailable.  
**Expected Behavior:** Fallback to GlideRecord counting (slower but functional).  
**Risk if mishandled:** Readiness score calculation crashes.  
**Verification:** Currently not implemented — GlideAggregate is assumed available. Add fallback in v1.1.
