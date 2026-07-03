# Sandbox Migration Shield — Architecture Summary

**Product:** Sandbox Migration Shield  
**Scope:** `x_snc_sms`  
**Author:** Vladimir Kapustin  
**Release:** Zurich → Australia  
**Version:** 1.0.0  
**Last Updated:** 2026-07-03

---

## 1. Executive Summary

Sandbox Migration Shield is a ServiceNow scoped application that automates the KB2944435 migration lifecycle. ServiceNow's phased sandbox replacement introduces blocking enforcement for incompatible server-side JavaScript — affecting Business Rules, Script Includes, Client Scripts, UI Policies, UI Actions, and Dictionary entries across every application scope. Enterprise instances routinely contain 500–5,000+ scripts with no built-in migration tooling.

The product provides: automated multi-table scanning, severity classification (BLOCKING/WARNING/COMPATIBLE), one-click inline-to-Script-Include migration with preview and rollback, and an exemption registry with full audit trail for SOX/HIPAA compliance.

---

## 2. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                     SERVICE NOW INSTANCE                             │
│                                                                      │
│  ┌──────────────────┐  ┌──────────────────┐  ┌───────────────────┐  │
│  │  Service Portal   │  │   Workspace UI   │  │   REST API Layer  │  │
│  │  (Widgets)        │  │   (UXF Client)   │  │   (Scripted REST) │  │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬──────────┘  │
│           │                     │                      │             │
│           └─────────────────────┼──────────────────────┘             │
│                                 │                                    │
│  ┌──────────────────────────────┴──────────────────────────────┐    │
│  │                   SCRIPT INCLUDES LAYER                      │    │
│  │                                                              │    │
│  │  ┌────────────────┐ ┌────────────────┐ ┌─────────────────┐   │    │
│  │  │ SandboxScanner │ │MigrationEngine │ │ExemptionManager │   │    │
│  │  │ - scanAll()    │ │ - preview()    │ │ - create()      │   │    │
│  │  │ - classify()   │ │ - execute()    │ │ - approve()     │   │    │
│  │  │ - paginateScan │ │ - rollback()   │ │ - revoke()      │   │    │
│  │  └───────┬────────┘ └───────┬────────┘ └────────┬────────┘   │    │
│  │          └──────────────────┼───────────────────┘             │    │
│  └──────────────────────────────┼──────────────────────────────┘    │
│                                 │                                    │
│  ┌──────────────────────────────┴──────────────────────────────┐    │
│  │                     DATA LAYER                               │    │
│  │                                                              │    │
│  │  Custom Tables:                                              │    │
│  │  ┌──────────────────┐ ┌──────────────────┐ ┌──────────────┐  │    │
│  │  │x_snc_sms_scan_   │ │x_snc_sms_scan_   │ │x_snc_sms_    │  │    │
│  │  │result            │ │run               │ │exemption     │  │    │
│  │  └──────────────────┘ └──────────────────┘ └──────────────┘  │    │
│  │  ┌──────────────────┐                                        │    │
│  │  │x_snc_sms_audit_  │                                        │    │
│  │  │log               │                                        │    │
│  │  └──────────────────┘                                        │    │
│  │                                                              │    │
│  │  Platform Tables (READ):                                     │    │
│  │  sys_script, sys_script_include, sys_script_client,          │    │
│  │  sys_ui_policy, sys_ui_action, sys_dictionary                │    │
│  └──────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Component Details

### 3.1 SandboxScanner (`x_snc_sms.SandboxScanner`)

**Role:** Core scanning engine. Traverses all six script-bearing tables and identifies incompatible JavaScript patterns.

**Input:**
- `scope` (string, optional): Application scope filter
- `targetTable` (string, optional): Single table to scan
- `fullScan` flag: Full or incremental scan

**Output:**
- `scan_run` master record with aggregate counts
- Individual `scan_result` records per script field
- Audit log entries for every scan operation

**KB2944435 Pattern Registry (7 patterns):**

| Code | Pattern | Severity | Description |
|------|---------|----------|-------------|
| KB2944435-001 | `Packages.(java\|javax)` | BLOCKING | Java package access blocked |
| KB2944435-002 | `eval(` | BLOCKING | eval() calls blocked in Phase 3 |
| KB2944435-003 | `gs.(sleep\|print\|log)` | WARNING | Deprecated gs methods |
| KB2944435-004 | `new Packages` | BLOCKING | Package instantiation blocked |
| KB2944435-005 | `GlideStringUtil` | WARNING | Verify compatibility |
| KB2944435-006 | `gs.(setUser\|setPassword)` | BLOCKING | Credential-setting blocked |
| KB2944435-007 | `Class.(forName\|getClass\|getProtectionDomain)` | BLOCKING | Java reflection blocked |

**Key Design Decisions:**
- Deduplication: Before inserting a scan_result, checks for existing NEW-status record with same table+sys_id+field — updates instead of creating duplicates
- Pagination: `setLimit(500)` per table to avoid transaction timeouts on large instances
- Severity escalation: If any pattern matches BLOCKING, the entire script is BLOCKING (worst-case wins)

### 3.2 MigrationEngine (`x_snc_sms.MigrationEngine`)

**Role:** Extracts inline JavaScript logic and creates secure Script Includes. Supports preview mode and full rollback.

**Flow:**
1. `preview(scanResultId)` — reads source script, extracts logic, generates Script Include code and replacement call. Returns preview without executing.
2. `execute(scanResultId)` — stores pre-migration state, creates Script Include via `sys_script_include` table, updates source script with replacement call, marks scan_result as MIGRATED.
3. `rollback(scanResultId)` — restores original script content from stored pre-state, deletes the auto-created Script Include, resets scan_result to NEW.

**Key Design Decisions:**
- Pre-migration state stored as JSON in `migration_pre_state` field on scan_result — enables full rollback
- Script Include naming: `{scriptName}_{fieldName}` sanitized to alphanumeric + underscores
- Replacement call pattern: `new x_snc_sms.{CamelCaseName}().execute(current, previous, g_scratchpad);`
- Simple expressions (<200 chars, no semicolons) wrapped as `return (expr);` — multi-line scripts preserved as-is

### 3.3 ExemptionManager (`x_snc_sms.ExemptionManager`)

**Role:** Manages the exemption lifecycle with full audit trail for compliance.

**Functions:**
- `createExemption(scanResultId, justification, approverId, expiryDate)` — validates justification ≥50 chars, approver required, expiry in future. Creates exemption record, marks scan_result as EXEMPT, notifies approver via event queue.
- `approveExemption(exemptionId)` — transitions PENDING → APPROVED, records timestamp.
- `revokeExemption(exemptionId)` — transitions to REVOKED, resets scan_result to NEW.
- `getExpiringExemptions(daysThreshold)` — returns exemptions expiring within N days for renewal reminders.
- `calculateReadinessScore()` — computes percentage: (compatible + migrated) / total × 100, excluding exempted scripts.

---

## 4. Data Model

### Custom Tables

| Table | Purpose | Key Fields |
|-------|---------|------------|
| `x_snc_sms_scan_run` | Master scan record | scan_type, scope, started_at, completed_at, total_scripts, blocking_count, warning_count, compatible_count, status |
| `x_snc_sms_scan_result` | Per-script finding | script_table, script_sys_id, script_name, script_field, severity, issue_code, issue_description, source_code_snippet, status, scan_run, migration_pre_state, migrated_to_si |
| `x_snc_sms_exemption` | Exemption registry | scan_result, business_justification, approver, expires_at, status, approved_at |
| `x_snc_sms_audit_log` | Immutable audit trail | action, target_table, target_sys_id, details (JSON), user, timestamp |

### Platform Tables (Read-Only)

| Table | Fields Scanned |
|-------|---------------|
| `sys_script` | script, condition |
| `sys_script_include` | script |
| `sys_script_client` | script |
| `sys_ui_policy` | script_true, script_false, condition |
| `sys_ui_action` | script, condition |
| `sys_dictionary` | default_value, calculation |

---

## 5. Integration Points

| Integration | Type | Purpose |
|-------------|------|---------|
| Scripted REST API | `/api/x_snc_sms/v1/*` | External scan triggering, result retrieval |
| Service Portal Widgets | Client-side | Readiness dashboard for platform owners |
| Workspace UI (UXF) | Client-side | Admin interface for scan/migrate/exempt |
| Scheduled Jobs | Server-side | Weekly automated scans, exemption expiry reminders |
| Event Queue | `x_snc_sms.exemption.pending` | Approver notifications |

---

## 6. Security & Compliance

- **Least privilege:** Scanner reads platform tables only — no write access to `sys_script*` tables except during migration (explicit user action)
- **Audit trail:** Every scan, migration, exemption create/approve/revoke logged to `x_snc_sms_audit_log` with user, timestamp, and JSON details
- **Rollback safety:** Pre-migration state stored before any write — full rollback available
- **SOX/HIPAA ready:** Exemption registry with business justification, approval chain, and expiry tracking
- **No credential storage:** All operations use the logged-in user's session context

---

## 7. Performance Characteristics

| Metric | Target | Notes |
|--------|--------|-------|
| Scan duration | <2 min for <5,000 scripts | `setLimit(500)` per table, pagination available |
| Migration execution | <5 sec per script | Single Script Include creation + source update |
| Readiness score calc | <1 sec | GlideAggregate with COUNT, no row iteration |
| Audit log writes | <100ms | Simple insert, no triggers |
