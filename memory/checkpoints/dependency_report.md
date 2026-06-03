# Dependency Report — Sandbox Migration Shield

## Scope Information

| Property | Value |
|----------|-------|
| **Scoped App Name** | Sandbox Migration Shield |
| **Scope ID** | x_snc_sms |
| **Target Release** | Zurich → Australia |
| **App Version** | 1.0.0 |

## Table Dependencies

### Custom Tables

| Table | Rows Expected | Growth Rate | Backup Priority |
|-------|--------------|-------------|-----------------|
| x_snc_sms_scan_result | 500–10,000 per full scan | Linear with script count | MEDIUM |
| x_snc_sms_scan_run | 1 per scan (1–2 per week in steady state, ~100/year) | Constant | LOW |
| x_snc_sms_exemption | 5–10% of BLOCKING results (50–500 typical) | Slow growth | HIGH (compliance) |
| x_snc_sms_audit_log | 1 per mutation (~500–5,000/year) | Linear with usage | HIGH (compliance) |

### Platform Tables (Read Access Required)

| Table | ACL Requirement | Cross-Scope Grant Needed | Notes |
|-------|----------------|-------------------------|-------|
| sys_script | Read | Yes (x_snc_sms) | Business Rules — scan all scopes |
| sys_script_include | Read | Yes (x_snc_sms) | Script Includes — all scopes |
| sys_script_client | Read | Yes (x_snc_sms) | Client Scripts |
| sys_ui_policy | Read | Yes (x_snc_sms) | UI Policies |
| sys_ui_action | Read | Yes (x_snc_sms) | UI Actions |
| sys_dictionary | Read | Yes (x_snc_sms) | Dictionary entries with defaults/calculations |
| sys_scope | Read | No (platform) | Scope identification for filtering |
| sys_user | Read | No (platform) | Approver reference for exemptions |
| sys_email | Execute | No (platform) | Email notification for weekly reports |
| sys_trigger | Read | No (platform) | Scheduled job definitions |

**Cross-Scope Privilege Requirement:** Grant `read` access on `sys_script`, `sys_script_include`, `sys_script_client`, `sys_ui_policy`, `sys_ui_action`, `sys_dictionary` to scope `x_snc_sms`. Without these grants, `GlideRecord` queries return zero results silently.

## ServiceNow Platform Dependencies

### Required Plugins

| Plugin ID | Plugin Name | Required By | Notes |
|-----------|-------------|-------------|-------|
| com.glide.scripted_rest_api | Scripted REST APIs | REST API endpoints | Standard on all instances |
| com.glide.ui.service-portal | Service Portal | Portal widgets | Standard on Zurich+ |
| com.snc.sla.engine | SLA Engine | Scheduled jobs | Standard — used by sys_trigger |

### Required Platform APIs

| API | Used In | Compatibility | Risk |
|-----|---------|---------------|------|
| GlideRecord (query, get, insert, update) | All components | All releases | None — core API |
| GlideAggregate | Dashboard score calculation | Xanadu+ | Low — well-established |
| GlideDateTime | Audit timestamps | All releases | None |
| GlideSystem (gs.info, gs.getUser, gs.now) | Logging, user context | All releases | None |
| sn_ws.RESTMessageV2 | REST API response construction | All releases | None |
| Class.create() | Script Include classes | All releases | None — but deprecated for ES12 migration in future |
| GlideRecord.addQuery with 3-arg (field, op, value) | Filtered scanning | All releases | None |
| GlideRecord.setLimit | Paginated scanning | All releases | None |

### Deprecated APIs Used (Pre-migration Check)

| API | Status | Action |
|-----|--------|--------|
| gs.sleep() | Deprecated in Zurich | Use GlideSystem.sleep() or remove — scanning uses setLimit() pagination, not sleep |
| gs.print() | Deprecated | Use gs.info() instead |
| Class.create() | Functional but ES12-incompatible | No action needed for current releases; noted for future ES12 migration |

## Role Dependencies

### Required User Roles

| Role Name | Granted To | Purpose |
|-----------|------------|---------|
| x_snc_sms.admin | Platform admins | Full access: scan, migrate, manage exemptions, view audit log |
| x_snc_sms.user | Instance admins | Run scans, view results, create exemptions |
| x_snc_sms.approver | Designated approvers | Approve/reject exemption requests |
| x_snc_sms.viewer | Auditors, compliance | Read-only: view dashboards, scan results, audit log |
| snc_internal | Internal REST callers | REST API access (system user) |

### ACL Rules

| Table | Role Required — Create | Role Required — Read | Role Required — Write | Role Required — Delete |
|-------|----------------------|---------------------|----------------------|----------------------|
| x_snc_sms_scan_result | x_snc_sms.admin (system) | x_snc_sms.user | x_snc_sms.admin (system) | x_snc_sms.admin |
| x_snc_sms_scan_run | x_snc_sms.admin (system) | x_snc_sms.user | x_snc_sms.admin (system) | x_snc_sms.admin |
| x_snc_sms_exemption | x_snc_sms.user | x_snc_sms.user | x_snc_sms.admin + x_snc_sms.approver | x_snc_sms.admin |
| x_snc_sms_audit_log | System only | x_snc_sms.viewer | Never (immutable) | Never (immutable) |

## External System Dependencies

| System | Purpose | Protocol | Failure Mode |
|--------|---------|----------|-------------|
| SMTP Server | Email reports | SMTP (sys_email) | Emails not sent — non-blocking |
| ServiceNow Instance | Core platform | Internal API | N/A — same instance |

**No external service dependencies.** The application is fully self-contained within the ServiceNow instance.

## Version Compatibility Matrix

| Component | Minimum Version | Recommended | Maximum Tested |
|-----------|----------------|-------------|----------------|
| ServiceNow Instance | Xanadu | Zurich | Australia |
| Scoped App Runtime | Utah Patch 5 | Zurich Patch 7 | Australia |
| Scripted REST API Plugin | v1.0 (any) | Current | Current |
| Service Portal | v1.0 (any) | Current | Current |

## Source Code Inventory

| File | Type | Lines (est.) | Module Dependencies |
|------|------|-------------|-------------------|
| src/sys_app.xml | App definition | ~200 | None |
| src/script_includes/SandboxScanner.js | Script Include | ~250 | GlideRecord, GlideSystem |
| src/script_includes/MigrationEngine.js | Script Include | ~200 | GlideRecord, GlideSystem |
| src/script_includes/ExemptionManager.js | Script Include | ~150 | GlideRecord, GlideSystem |
| src/rest_endpoints/scan_api.js | Scripted REST | ~80 | SandboxScanner |
| src/rest_endpoints/migration_api.js | Scripted REST | ~60 | MigrationEngine |
| src/rest_endpoints/exemption_api.js | Scripted REST | ~60 | ExemptionManager |
| src/scheduled_jobs/weekly_scan.js | Scheduled Job | ~20 | SandboxScanner |
| src/scheduled_jobs/exemption_check.js | Scheduled Job | ~30 | ExemptionManager |
| src/acl/scan_result.acl | ACL rules | ~40 | None |
| src/acl/exemption.acl | ACL rules | ~40 | None |
| src/acl/audit_log.acl | ACL rules | ~20 | None |
| src/service_portal/dashboard.js | Portal Widget | ~100 | SandboxScanner |

## npm/Python Dependencies

**No npm or Python dependencies.** The application is pure ServiceNow server-side JavaScript running inside the Scoped App runtime. CLI tools (if present) use Python standard library only.

## Risk Assessment Summary

| Risk Area | Severity | Details |
|-----------|----------|---------|
| Cross-scope privilege grants missing | **P0** | Scanner returns zero results silently if grants not configured — see R01 |
| KB2944435 pattern registry completeness | **P1** | Missing pattern → incompatible script passes as COMPATIBLE — see R02 |
| Transaction timeout on large instances | **P2** | 5000+ scripts may exceed 2-minute timeout despite pagination — see R03 |
| Migration rollback data loss | **P1** | Rollback must restore exact original script — see R04 |
| Exemption expiry gap | **P2** | Exemption expires but script still incompatible — production blocking risk — see R05 |
