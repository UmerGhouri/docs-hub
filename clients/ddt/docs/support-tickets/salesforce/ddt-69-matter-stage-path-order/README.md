# Matter stage path order wrong: Treatment before Case Opening

| | |
|---|---|
| **Ticket** | DDT-69 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-02 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-06-01 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Case Plan Builder, Matter path, BlueBack tasks) |
| **Area** | PI Matter stage path |
| **Related** | DDT-67 and DDT-143 (stage rename orphaning records) |
| **Code** | None. Configuration change in the Case Plan Builder |

---

## 1. Summary

The PI Matter stage path listed **Treatment** before **Case Opening & Evaluation**. Because each stage has required BlueBack tasks, a user could not move a Matter to the stage it actually belonged in.

The two stages were swapped back in the Case Plan Builder. No code changed.

## 2. What was reported

A staff member asked for the stage order to be fixed: it should be Intake → Case Opening → Treatment. They could not move a Matter to the correct stage because of the tasks under BlueBack.

Order at the time of the report:

`Intake & Onboarding → Treatment → Case Opening & Evaluation → Med Retrieval → Demand Phase → Negotiation → Litigation → Settlement → Resolved`

Order wanted:

`Intake & Onboarding → Case Opening & Evaluation → Treatment → Med Retrieval → Demand Phase → Negotiation → Litigation → Settlement → Resolved`

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| PI stages "Case Opening & Evaluation" and "Treatment" | `SLA_Tracking__c` stage records | Order swapped in the Case Plan Builder (`PI_Stage_Priority__c`) |

**Out of scope**

- **Re-sequencing the BlueBack tasks inside each stage.** The ticket asked whether this was needed; only the stage order was changed.
- **An audit of Matters left on the wrong stage by the old order.** Asked for in the ticket; not recorded as done.

## 4. How it works (background)

The PI path is built from `SLA_Tracking__c` rows maintained in the Case Plan Builder:

```
SLA_Tracking__c (Main_Stage__c = true, Case_Type__c blank)   one row per PI stage
   PI_Stage_Priority__c  = position on the path (1, 2, 3, ...)
SLA_Tracking__c (Main_Stage__c = false)                      the stage's tasks (shown in BlueBack)
```

Reordering stages in the Plan Builder calls `MatterPlanBuilderController.reorderPIStages`, which writes `PI_Stage_Priority__c = 1, 2, 3 ...` in the order the stages are listed. The Matter path and BlueBack read that order. Required BlueBack tasks on a stage gate the move out of it, so a path in the wrong order can block a user from reaching the right stage.

## 5. Root cause

The `PI_Stage_Priority__c` values put Treatment in position 2 and Case Opening & Evaluation in position 3. How they came to be in that order is **not recorded in the ticket**.

The user was blocked because the path made them pass through Treatment, with its required tasks, before reaching Case Opening & Evaluation.

## 6. Solution

In the Case Plan Builder's PI tab, Case Opening & Evaluation was moved above Treatment, and the new order saved. Per the ticket, with the stages in the right order the BlueBack tasks line up with the right stage and no longer block the move.

This is a reorder, not a rename: the stage names don't change, so the Matters on them are not affected (contrast DDT-67, where a rename orphaned records).

## 7. Testing

Not recorded in the ticket whether the reporting user confirmed they could move the affected Matter. The resolution comment asked for that confirmation.

**Live check (2026-11-05):** the PI stages in production, in path order:

```sql
SELECT Name, PI_Stage_Priority__c
FROM SLA_Tracking__c
WHERE Main_Stage__c = true AND Case_Type__c = null AND PI_Stage_Priority__c != null
ORDER BY PI_Stage_Priority__c
```

| # | Stage |
|---|---|
| 1 | Intake & Onboarding |
| 2 | Case Opening & Evaluation |
| 3 | Treatment |
| 4 | Records Retrieval |
| 5 | Demand Phase |
| 6 | Negotiation |
| 7 | Litigation |
| 8 | Settlement |
| 9 | Resolved |

Case Opening & Evaluation comes before Treatment. Stage 4 is now "Records Retrieval", from the later rename in DDT-67.

## 8. Step by step

1. **Run the query in section 7** to see the current order.
2. **Open the Case Plan Builder**, PI tab.
3. **Move the stage** to its correct position in the stage list and save the order.
4. **Re-run the query** and confirm the positions.
5. **Open a Matter on one of the moved stages** and check the path and BlueBack show the stages in the new order.
6. **Ask the reporting user to retry** the stage move that was blocked.

## 9. Rollback

Reorder the stages back in the Case Plan Builder. Safe: only the position numbers change.

## 10. Known limits

- **Matters that progressed under the old order were not audited.** Any Matter moved to Treatment before Case Opening may have skipped that stage's tasks.
- **Task order within a stage was not reviewed.**
- **Nothing stops the order being changed again by mistake.** Anyone with access to the Plan Builder can reorder stages.

## 11. Reuse for similar issues

**How to recognise one:** a user says they "can't move" a record to a stage, and the stage they want is later on the path than one they have not finished.

- **Read the order from the data** (`PI_Stage_Priority__c`, or the Court Case equivalent) rather than from a screenshot.
- **Reorder, don't rename or delete.** A reorder keeps stage names, so live records stay where they are. A rename or delete can orphan them (DDT-67).
- **Required tasks gate stage moves.** When a path is in the wrong order, the gate shows up as a user being blocked, not as an error in the path itself.
