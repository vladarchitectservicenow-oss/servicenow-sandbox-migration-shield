# Architecture Summary — Sandbox Migration Shield

## Product Overview

Sandbox Migration Shield (`x_snc_sms`) is a ServiceNow scoped application that automates migration readiness for ServiceNow KB2944435 — the phased replacement of the server-side JavaScript sandbox. It scans all script-bearing tables, classifies incompatibilities, migrates inline logic to secure Script Includes, and provides an audit-ready exemption registry.

## System Context

```
┌──────────────────────────────────────────────────────────────────┐
│                        SERVICE NOW INSTANCE                       │
│                                                                   │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────────────┐  │
│  │ Service      │   │ Workspace UI │   │ Scripted REST APIs   │  │
│  │ Portal       │   │ (UXF Client) │   │ POST /api/x_snc_sms/ │  │
│  │ Widgets      │   │              │   │ v1/scan, /migrate,   │  │
│  │              │   │              │   │ /exemption, /dashboard│  │
│  └──────┬───────┘   └──────┬───────┘   └──────────┬───────────┘  │
│         │                  │                       │              │
│         └──────────────────┼───────────────────────┘              │
│                            │                                      │
│  ┌─────────────────────────┴─────────────────────────────────┐   │
│  │                    SCRIPT INCLUDES LAYER                    │   │
│  │                                                             │   │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌────────────┐  │   │
│  │  │ SandboxScanner  │  │ MigrationEngine │  │ Exemption  │  │   │
│  │  │                 │  │                 │  │ Manager    │  │   │
│  │  │ - scanAll()     │  │ - migrate()     │  │ - create() │  │   │
│  │  │ - classify()    │  │ - preview()     │  │ - approve()│  │   │
│  │  │ - score()       │  │ - rollback()    │  │ - expire() │  │   │
│  │  └────────┬────────┘  └────────┬────────┘  └──────┬─────┘  │   │
│  │           └────────────────────┼──────────────────┘         │   │
│  └────────────────────────────────┼───────────────────────────┘   │
│                                   │                                │
│  ┌────────────────────────────────┴───────────────────────────┐   │
│  │                      DATA LAYER                              │   │
│  │                                                              │   │
│  │  Custom Tables:                                              │   │
│  │  ┌──────────────────┐ ┌───────────────────┐ ┌────────────┐  │   │
│  │  │x_snc_sms_scan_   │ │x_snc_sms_scan_    │ │x_snc_sms_  │  │   │
│  │  │result            │ │run                │ │exemption   │  │   │
│  │  └──────────────────┘ └───────────────────┘ └────────────┘  │   │
│  │  ┌──────────────────┐                                        │   │
│  │  │x_snc_sms_audit_  │                                        │   │
│  │  │log               │                                        │   │
│  │  └──────────────────┘                                        │   │
│  │                                                              │   │
│  │  Platform Tables (READ-ONLY):                                │   │
│  │  sys_script, sys_script_include, sys_script_client,         │   │
│  │  sys_ui_policy, sys_ui_action, sys_dictionary               │   │
│  └──────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

## Component Architecture

### Core Script Includes

| Component | API Name | Responsibility | Critical Algorithms |
|-----------|----------|----------------|---------------------|
| SandboxScanner | `x_snc_sms.SandboxScanner` | Multi-table script scan + severity classification | Regex-based KB2944435 pattern matching, pagination-safe GlideRecord scanning |
| MigrationEngine | `x_snc_sms.MigrationEngine` | Inline logic extraction → Script Include generation | AST-like pattern extraction, function wrapping, rollback restore |
| ExemptionManager | `x_snc_sms.ExemptionManager` | Exemption lifecycle + audit trail | Status state machine (PENDING → APPROVED/REJECTED → EXPIRED), readiness score recalculation |

### Data Flow

```
User/API Trigger
       │
       ▼
SandboxScanner.scanAll(scope, fullScan)
       │
       ├──► Create scan_run record (status: RUNNING)
       │
       ├──► For each target table in [sys_script, sys_script_include,
       │    sys_script_client, sys_ui_policy, sys_ui_action, sys_dictionary]:
       │      │
       │      ├──► GlideRecord query (active records, paginated setLimit(500))
       │      │
       │      ├──► Extract script fields (script, condition, default_value,
       │      │    calculation, execute_function, ui_script)
       │      │
       │      ├──► Run against KB2944435 Pattern Registry:
       │      │    - Packages.java / Packages.javax → BLOCKING
       │      │    - eval() calls → BLOCKING
       │      │    - Deprecated gs.* methods → WARNING
       │      │    - new Packages → BLOCKING
       │      │    - GlideStringUtil → WARNING
       │      │    - Cross-scope access → WARNING
       │      │
       │      └──► Write scan_result record
       │             (severity, issue_code, source_code_snippet, scan_run ref)
       │
       └──► Update scan_run: completed_at, total_scripts, blocking_count,
            warning_count, compatible_count
```

### Migration Flow

```
User selects scan_result.sys_id
       │
       ▼
MigrationEngine.preview(sourceSysId)
       │
       ├──► Load scan_result → get script_table, script_sys_id, script_field
       ├──► GlideRecord.get source script record
       ├──► Extract script content from identified field
       ├──► Classify: inline expression vs multi-line script
       ├──► Generate Script Include:
       │      - name: sanitize(source_name) + "_Migrated"
       │      - api_name: "x_snc_sms." + camelCase(name)
       │      - script: wrapped function(current, previous, g_scratchpad)
       ├──► Generate replacement call for source field
       └──► Return PREVIEW: {generated_script_include, replacement_call}
              │
              ▼ (User confirms)
       MigrationEngine.execute(previewToken)
              │
              ├──► Create sys_script_include record
              ├──► Update source script field with replacement call
              ├──► Update scan_result: status = MIGRATED
              ├──► Write audit_log: action = MIGRATE
              └──► Recalculate readiness score
```

### REST API Contract

| Endpoint | Method | Input | Output | Auth |
|----------|--------|-------|--------|------|
| `/api/x_snc_sms/v1/scan` | POST | `{scope, full_scan}` | `{scan_run_id, summary}` | snc_internal |
| `/api/x_snc_sms/v1/scan/{id}` | GET | path `id` | `{scan_run, results[]}` | snc_internal |
| `/api/x_snc_sms/v1/migrate` | POST | `{scan_result_id}` | `{migration_result}` | snc_internal |
| `/api/x_snc_sms/v1/preview-migration` | POST | `{scan_result_id}` | `{preview: {si, replacement}}` | snc_internal |
| `/api/x_snc_sms/v1/exemption` | POST | `{scan_result_id, justification, approver_id, expiry}` | `{exemption_id}` | snc_internal |
| `/api/x_snc_sms/v1/exemption/{id}` | GET/PATCH | path `id` | `{exemption}` | snc_internal |
| `/api/x_snc_sms/v1/dashboard` | GET | — | `{readiness_score, severity_breakdown, top10}` | snc_internal |

## Data Model

### Custom Tables

**x_snc_sms_scan_result** — Individual finding per script per issue

| Field | Type | Indexed | Description |
|-------|------|---------|-------------|
| script_table | String (100) | Yes | Source table name |
| script_sys_id | GUID | Yes | Reference to source record |
| script_name | String (255) | No | Display name |
| script_field | String (100) | No | Field containing the script |
| severity | Choice | Yes | BLOCKING / WARNING / COMPATIBLE |
| issue_code | String (50) | Yes | KB2944435 pattern identifier |
| issue_description | String (4000) | No | Human-readable description |
| source_code_snippet | String (4000) | No | Excerpt of problematic code |
| status | Choice | Yes | NEW / MIGRATED / EXEMPT / IGNORED |
| scan_run | Reference | Yes | FK → x_snc_sms_scan_run |

**x_snc_sms_scan_run** — Container for a scan execution session

| Field | Type | Indexed | Description |
|-------|------|---------|-------------|
| scan_type | Choice | No | FULL / INCREMENTAL / MANUAL |
| scope | String (100) | No | App scope filter |
| started_at | DateTime | No | Scan start time |
| completed_at | DateTime | No | Scan end time |
| total_scripts | Integer | No | Total scanned |
| blocking_count | Integer | No | BLOCKING severity count |
| warning_count | Integer | No | WARNING severity count |
| compatible_count | Integer | No | COMPATIBLE count |

**x_snc_sms_exemption** — Documented exception for incompatible scripts

| Field | Type | Indexed | Description |
|-------|------|---------|-------------|
| scan_result | Reference | Yes | FK → x_snc_sms_scan_result |
| business_justification | String (4000) | No | Why not migrated (min 50 chars) |
| approver | Reference | No | FK → sys_user |
| approved_at | DateTime | No | Approval timestamp |
| expires_at | Date | Yes | Required review date |
| status | Choice | Yes | PENDING / APPROVED / REJECTED / EXPIRED |

**x_snc_sms_audit_log** — Immutable audit trail

| Field | Type | Indexed | Description |
|-------|------|---------|-------------|
| action | Choice | No | SCAN / MIGRATE / EXEMPTION_CREATE / EXEMPTION_APPROVE / ROLLBACK |
| target_table | String (100) | Yes | Affected table |
| target_sys_id | GUID | Yes | Affected record |
| details | String (4000) | No | JSON serialized operation details |
| performed_by | Reference | No | FK → sys_user |
| timestamp | DateTime | Yes | Auto-set on insert |

### Platform Tables (Read-only access)

| Table | Purpose | Fields Scanned |
|-------|---------|----------------|
| sys_script | Business Rules | script, condition |
| sys_script_include | Script Includes | script |
| sys_script_client | Client Scripts | script |
| sys_ui_policy | UI Policies | script_true, script_false, condition |
| sys_ui_action | UI Actions | script, condition |
| sys_dictionary | Dictionary Entries | default_value, calculation, dependent |

## Performance Characteristics

| Operation | Target | Scaling Factor |
|-----------|--------|----------------|
| Full scan (5K scripts) | <2 minutes | O(n) per script, O(1) per pattern |
| Single script classification | <10ms | Constant per pattern registry lookup |
| Migration preview | <500ms | Single GlideRecord.get + string ops |
| Migration execution | <2s | Two GlideRecord writes + audit log |
| Dashboard load | <3s | Single GlideAggregate query |
| Exemption create/approve | <1s | Single insert + update |

## Performance Optimizations

1. **Paginated scanning:** `gr.setLimit(500)` with `gs.sleep(0)` to avoid transaction timeouts
2. **Pattern regex caching:** Compiled regexes stored in `this._patterns[]` at Script Include load time
3. **Batch audit logging:** Audit entries written in a single `GlideRecord.insert()` call per operation
4. **GlideAggregate for dashboard:** COUNT, GROUP BY on severity/status fields — avoids loading individual records
5. **Indexed queries:** All WHERE clauses target indexed fields (script_table, severity, status, scan_run)

## Security Model

| Resource | Create | Read | Write | Delete | Notes |
|----------|--------|------|-------|--------|-------|
| x_snc_sms_scan_run | snc_internal | snc_internal | snc_internal | admin | Only via Scanner |
| x_snc_sms_scan_result | snc_internal | snc_internal | snc_internal | admin | Only via Scanner/MigrationEngine |
| x_snc_sms_exemption | snc_internal | snc_internal | snc_internal | admin | Approval workflow controls writes |
| x_snc_sms_audit_log | snc_internal | snc_internal | — | — | Insert-only, never updated |
| Platform script tables | — | snc_internal | — | — | Read-only access only |

## Compatibility Matrix

| Release | Status | Notes |
|---------|--------|-------|
| Utah | Untested | May lack some APIs, test before deploying |
| Vancouver | Untested | — |
| Washington DC | Untested | — |
| Xanadu | Compatible | All APIs available |
| Yokohama | Compatible | Full support |
| Zurich | **Primary target** | KB2944435 Phase 2 active |
| Australia | Compatible | KB2944435 Phase 3 enforcement |

## Integration Points

| Integration | Type | Direction | Protocol |
|-------------|------|-----------|----------|
| Schedule Jobs | Inbound trigger | Push | sys_trigger → SandboxScanner |
| Scripted REST API | Outbound API | Pull | HTTP/REST → JSON |
| Service Portal | UI | Push | Widget client → Script Includes |
| Workspace | UI | Push | UXF client → Script Includes |
| Email notification | Outbound | Push | sys_email → SMTP |

## Architectural Decisions

| Decision | Rationale | Alternative Considered |
|----------|-----------|----------------------|
| Scoped app (x_snc_sms) | Isolation + clean namespace + easy packaging | Global scope — rejected: harder to package/distribute |
| Script Includes (not Flow Designer actions) | Direct GlideRecord access needed for multi-table scanning | Flow Designer — rejected: cannot iterate arbitrary tables efficiently |
| Regex-based pattern matching | KB2944435 patterns are deterministic regex matches | AST-based JS parser — rejected: over-engineered, GlideRhino AST unavailable in scoped apps |
| Paginated scanning (setLimit(500)) | Avoids transaction timeout on instances with 5000+ scripts | Single query — rejected: fails on large instances |
| Separate scan_run container | Enables historical comparison and trend analysis | Single flat scan_result table — rejected: no temporal data |
| Immutable audit log (insert-only) | Compliance requirement — records must never be altered | Updatable log — rejected: SOX/HIPAA audit trail requirement |

## Known Limitations

1. **GlideRhino AST unavailable:** Cannot truly parse JS in scoped apps — relies on regex which may miss obfuscated patterns
2. **Cross-scope table access:** Platform tables (sys_script, etc.) must have read ACLs granted to the scoped app
3. **Migration covers only inline logic:** External library calls, complex class hierarchies, and eval-wrapped dynamic code cannot be automatically migrated
4. **Single-instance scope:** No built-in multi-instance federation — each instance requires separate installation
