# Removing Case Closed from criminal case status paths

| | |
|---|---|
| **Ticket** | DDT-74 |
| **Project** | Support tickets |
| **Status** | Done 2026-06-01 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-06-01 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Lightning page, Path components) |
| **Area** | Criminal case status tracking on the Hired Matter page |
| **Related** | DDT-77 (Hired Matter status workflow, same family of work) |
| **Code** | Not applicable: configured in Lightning App Builder, no PR is linked. |

---

## 1. Summary

"Case Closed" appeared as a stage in every criminal case sub-status path on the Hired Matter page. Closing a case belongs at the top level, not inside each phase. The fix removed Case Closed from all six sub-status paths, leaving it available only on the top-level Case Status.

## 2. What was reported

- **2026-06-01, DDT-74:** the Case Closed stage was on every criminal case status path. The request was to remove it so closure is tracked through one mechanism instead of as a stage on every path.

## 3. Scope

**In scope:** the Case Sub-Status path components on the Hired Matter Lightning page, for these six phases:

- Investigation
- First Appearance
- Pre-File
- Filed
- Trial
- Violation of Probation

Case Closed was removed as a stage from each.

**Out of scope**

- **The top-level Case Status.** Case Closed stays there. That is now the one place a case is closed.
- **Migrating records and auditing reports, list views and flows.** The ticket asked for these. The resolution note does not record whether any records sat on Case Closed in a sub-status, or whether any report or automation was changed.

## 4. How it works (background)

```
Hired Matter page
  Case Status path            (top level: the overall state, including Case Closed)
  Case Sub-Status paths       (one per phase: Investigation, First Appearance, Pre-File,
                               Filed, Trial, Violation of Probation)
```

Before the change each sub-status path also ended in Case Closed, so closure could be recorded in seven places.

## 5. Root cause

Not a defect: a design change. Having Case Closed in every phase's path duplicated the top-level status and gave users several places to close a case.

## 6. Solution

Case Closed was removed from each of the six Case Sub-Status paths on the Hired Matter Lightning page. Closure is recorded only on the top-level Case Status.

Why one place: a single closure signal is what reports and automations can rely on. Seven possible places means some closed cases are missed by anything that reads just one of them.

## 7. Testing

Not recorded in the ticket. The acceptance criteria asked that the path UI render correctly with the new sequence and that no record be stranded on the removed stage.

## 8. Step by step

1. Before changing anything, check whether any record is currently on Case Closed in a sub-status field, and move it to the right value. A path cannot show a stage you have removed.
2. Search reports, list views, flows and validation rules for references to the sub-status Case Closed value.
3. Open the Hired Matter Lightning page in App Builder (or Setup, Path Settings, depending on how the path is defined).
4. For each of the six sub-status paths, remove the Case Closed stage.
5. Save and activate. Open a Hired Matter in each phase and check the path renders.

## 9. Rollback

Add Case Closed back to the sub-status paths in the same place. Safe: no data was deleted.

## 10. Known limits

- If the Case Closed value still exists in the sub-status picklist, users can still pick it outside the path (for example in an edit form). The resolution note does not say whether the picklist value was removed.
- Reports or automations that read the sub-status for closure were not recorded as checked.

## 11. Reuse for similar issues

**How to recognise one:** the same state can be set in several places, and reports disagree about how many records are in it.

- Keep each state in one field. A closure stage belongs on the top-level status.
- Move records off a stage before removing it, and search reports and automations for the value.
