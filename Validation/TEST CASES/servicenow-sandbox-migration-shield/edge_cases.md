# Edge Cases — Sandbox Migration Shield

## Test Matrix

| Case | Component | Scenario | Expected Behavior | Priority |
|------|-----------|----------|-------------------|----------|
| E01 | SandboxScanner | **Empty instance:** No scripts exist in any target table | Scanner returns scan_run with total_scripts=0, all counts=0, scan_type=FULL, status=SUCCESS. Dashboard shows readiness_score=100%. No scan_result records created. | P1 |
| E02 | SandboxScanner | **Script with zero-length script field:** sys_script record where `script` is empty string `""` | Scanner skips the record. Does NOT create a scan_result. Does NOT throw NullPointerException or TypeError. Total scanned count excludes this record. | P2 |
| E03 | SandboxScanner | **Script with >4000 characters and multiple matching patterns:** Single script contains 3 BLOCKING + 2 WARNING patterns in 8000-char body | Scanner creates 5 separate scan_result records (one per pattern match). Each source_code_snippet truncated to 4000 chars with "[...]" suffix. | P1 |
| E04 | MigrationEngine | **Migration target already exists:** Script Include with generated name already present in sys_script_include | MigrationEngine detects collision. Appends "_v2", "_v3" etc. to generated name until unique. Returns preview with unique name. Does NOT overwrite existing SI. | P2 |
| E05 | MigrationEngine | **Rollback after target SI manually deleted:** User migrates script, then manually deletes the generated SI from sys_script_include | Rollback detects SI missing. Restores source script field from stored original. Logs warning: "Generated SI not found — restored source script only." Scan_result.status = ROLLED_BACK. | P1 |
| E06 | ExemptionManager | **Exemption created for COMPATIBLE script:** User creates exemption for a scan_result with severity=COMPATIBLE | createExemption() accepts the request but logs a WARNING: "Exempting a COMPATIBLE script — this may indicate false positive in scanner." Creates exemption record normally. | P2 |
| E07 | ExemptionManager | **Exemption approved after expiry date passed:** Exemption expires June 1. User approves on June 5. | approveExemption() rejects with error: "Cannot approve expired exemption. Extend expiry date first." Status remains PENDING. | P1 |
| E08 | REST API | **Request body exceeds maximum JSON size:** POST /scan with 50MB JSON payload | API returns HTTP 413 "Payload too large" or 400 "Request body too large". Does NOT crash the REST endpoint. Does NOT create partial scan_run. | P2 |
