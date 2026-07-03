# Sandbox Migration Shield — Execution Plan

**Product:** Sandbox Migration Shield  
**Scope:** `x_snc_sms`  
**Author:** Vladimir Kapustin  
**Last Updated:** 2026-07-03

---

## 1. Deployment Phases

### Phase 1: Pre-Deployment (Day 0)
| Step | Action | Owner | Duration | Verification |
|------|--------|-------|----------|-------------|
| 1.1 | Verify instance is Zurich+ | Admin | 5 min | `gs.getProperty('glide.war')` → check version |
| 1.2 | Grant cross-scope read access to sys_script* tables | Admin | 10 min | Test query: `new GlideRecord('sys_script').getRowCount() > 0` |
| 1.3 | Grant cross-scope write access to sys_script_include | Admin | 5 min | Test insert of temp record, then delete |
| 1.4 | Import scoped app via Studio or update set | Admin | 15 min | Verify `x_snc_sms` scope appears in application picker |
| 1.5 | Run pre-flight health check | System | 2 min | All tables created, Script Includes compiled without errors |

### Phase 2: Initial Scan (Day 1)
| Step | Action | Owner | Duration | Verification |
|------|--------|-------|----------|-------------|
| 2.1 | Execute full scan via Workspace UI or scheduled job | Admin | 2-5 min | scan_run record created with status COMPLETED |
| 2.2 | Review BLOCKING findings | Admin | 30-60 min | Filter scan_results by severity=BLOCKING |
| 2.3 | Classify BLOCKING scripts: migrate vs exempt | Admin | 30 min | Tag each as MIGRATE or EXEMPT in review notes |
| 2.4 | Create exemptions for justified BLOCKING scripts | Admin | 15 min/script | Exemption record created, approver notified |

### Phase 3: Migration (Day 2-5)
| Step | Action | Owner | Duration | Verification |
|------|--------|-------|----------|-------------|
| 3.1 | Preview migration for first script | Admin | 2 min | Review generated Script Include code in preview |
| 3.2 | Execute migration | Admin | 1 min | scan_result status → MIGRATED; Script Include created |
| 3.3 | Test migrated script in sub-production | Admin | 15 min | Verify Business Rule / Client Script still functions |
| 3.4 | Repeat 3.1-3.3 for all BLOCKING scripts | Admin | 1-3 days | All BLOCKING scripts either MIGRATED or EXEMPT |
| 3.5 | Run follow-up scan to verify zero new BLOCKING | System | 2 min | blocking_count = 0 in scan_run summary |

### Phase 4: Compliance (Day 5-7)
| Step | Action | Owner | Duration | Verification |
|------|--------|-------|----------|-------------|
| 4.1 | Approve pending exemptions | Approver | 15 min | All exemptions APPROVED or REVOKED |
| 4.2 | Generate readiness report | Admin | 5 min | Readiness score ≥ 95% |
| 4.3 | Export audit trail for compliance review | Admin | 10 min | Audit log covers all scan/migrate/exempt actions |
| 4.4 | Schedule weekly automated scans | Admin | 5 min | Scheduled job active, next run within 7 days |

### Phase 5: Ongoing Operations (Week 2+)
| Step | Action | Owner | Duration | Verification |
|------|--------|-------|----------|-------------|
| 5.1 | Weekly automated scan runs | System | Auto | scan_run created every Sunday |
| 5.2 | Review weekly report (Monday) | Admin | 15 min | No new BLOCKING findings |
| 5.3 | Handle exemption expiries (daily check) | System | Auto | Notifications sent 30 days before expiry |
| 5.4 | Pre-upgrade scan (before each release upgrade) | Admin | 5 min | Compare against baseline, identify new risks |

---

## 2. Rollback Plan

### Emergency Rollback (Migration Caused Production Issue)
1. Identify affected scan_result by script name or sys_id
2. Execute `ExemptionManager.rollback(scanResultId)` — restores original script, deletes auto-created Script Include
3. Verify original script content restored via direct table query
4. Re-classify script as EXEMPT if migration was premature
5. Document incident in audit log

### Full Application Rollback
1. Deactivate scheduled jobs (weekly_scan, weekly_report, exemption_expiry_check)
2. Export exemption registry for compliance records
3. Uninstall scoped application via Studio
4. Custom tables (`x_snc_sms_*`) are removed — data is lost unless exported
5. Auto-created Script Includes remain (they're in global scope) — manual cleanup required

---

## 3. Success Criteria

| Criterion | Target | Measurement |
|-----------|--------|-------------|
| Scan coverage | 100% of script tables | scan_run.total_scripts matches expected count |
| Scan duration | <2 min for <5,000 scripts | scan_run.completed_at - scan_run.started_at |
| Migration success rate | >95% of BLOCKING scripts migrated | MIGRATED count / total BLOCKING |
| Readiness score | ≥95% after migration | ExemptionManager.calculateReadinessScore() |
| False positive rate | <5% | Manual review of BLOCKING classifications |
| Rollback success rate | 100% | Tested on 3 sample scripts pre-deployment |
| Audit trail completeness | 100% of actions logged | audit_log count matches scan + migrate + exempt actions |

---

## 4. Resource Requirements

| Resource | Quantity | Notes |
|----------|----------|-------|
| ServiceNow Admin | 1 FTE, 5 days | Initial scan + migration execution |
| Platform Owner | 0.2 FTE, 2 days | Review + approval of exemptions |
| Compliance Architect | 0.1 FTE, 1 day | Audit trail review |
| Sub-production instance | 1 | Migration testing before production |
| Production instance | Zurich+ | Target deployment |

---

## 5. Timeline

```
Week 1:
  Day 0: Pre-deployment (install, grants, health check)
  Day 1: Initial scan + BLOCKING review
  Day 2-4: Migration execution (batch by severity)
  Day 5: Compliance review + approvals
  
Week 2:
  Day 7: First automated weekly scan
  Day 8: Weekly report review
  Ongoing: Exemption expiry monitoring
```

**Total elapsed time:** 5 business days to full readiness.  
**Ongoing maintenance:** 15 min/week for report review.

---

## 6. Communication Plan

| Audience | Message | Channel | Timing |
|----------|---------|---------|--------|
| Platform Team | "KB2944435 migration starting — expect scan results by EOD" | Slack/Teams | Day 0 |
| Application Owners | "Your scoped apps will be scanned for sandbox compatibility" | Email | Day 0 |
| CISO / Compliance | "Readiness report available after migration completes" | Email | Day 5 |
| All Admins | "Weekly automated scans active — review Monday reports" | Slack/Teams | Day 7 |

---

## 7. Dependencies & Prerequisites

| Dependency | Status | Owner | Due Date |
|------------|--------|-------|----------|
| Instance on Zurich+ | Must be confirmed | Platform Owner | Day 0 |
| Cross-scope read grants | Must be configured | Admin | Day 0 |
| Cross-scope write grants | Must be configured | Admin | Day 0 |
| Sub-production instance available | Required for migration testing | Platform Owner | Day 1 |
| Approver identified for exemptions | Required | Compliance Architect | Day 1 |
| Email notifications configured | Nice to have | Admin | Day 3 |
