# Hired credit not applied to the intake rep

| | |
|---|---|
| **Ticket** | DDT-987 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-11 |
| **Priority** | High |
| **Reported by** | Intake team, 2026-09-09 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Court Case / Opportunity, Sales Comp dashboard) |
| **Area** | Intake: hired credit (points) for the rep who signed the client |
| **Related** | None |
| **Code** | None. Answered from the existing data; nothing was changed. |

---

## 1. Summary

An intake rep reported that a Court Case they had signed on 2026-09-08 was not giving them hired credit. It was. Points are counted by the **Intake Owner** field, which named the rep, and the hire was already counting for them on the Sales Comp dashboard. The field that named someone else, **Hired By (User)**, only records who moved the case to Hired and has no effect on points. Nothing needed to change.

## 2. What was reported

The rep saw a different user's name on the Court Case and concluded the credit for the hire had gone to someone else. The ticket asked to find out why the credit was missing, correct it, and spot-check that the attribution rule worked for other recent hires.

One Court Case was involved.

## 3. Scope

**In scope:** checking which field drives hired credit, and confirming the credit on the one Court Case.

**Out of scope:** any change to fields, automation or dashboards. The credit was already correct, so there was nothing to fix.

## 4. How it works (background)

Two user fields on the Court Case (the `Opportunity` object, relabelled "Court Case") look alike but mean different things:

| Field | API name | What it records | Affects points? |
|---|---|---|---|
| Intake Owner | `Intake_Owner__c` | The intake rep the case is credited to | **Yes**: hired credit on the Sales Comp dashboard goes by this field |
| Hired By (User) | `Hired_by_User__c` | Whoever clicked the case over to Hired | No |

```
Rep signs the client -> the case is marked Hired (by whoever clicks it)
  -> Hired By (User) = the person who clicked
  -> Intake Owner stays the rep
  -> Sales Comp dashboard counts the hire against Intake Owner
```

## 5. Root cause

Not a defect. The **Intake Owner** on the Court Case was the reporting rep, and the hire was already counting as 2 points for them on the Sales Comp dashboard. **Hired By (User)** showed a different staff member because that person had moved the case to Hired. The rep read that field as the credit field.

The ticket's first guess, that the credit-attribution automation had stopped working, was wrong. Nothing was missing.

## 6. Solution

No change was made. The support team replied on the ticket explaining that points follow Intake Owner, that the hire was already counted, and that Hired By (User) does not affect points.

## 7. Testing

Not applicable: no change was made. The check was reading the Court Case's Intake Owner and confirming the hire on the Sales Comp dashboard.

A spot-check of other recent hires, asked for in the ticket, is not recorded in the ticket.

## 8. Step by step

When a rep reports missing hired credit:

1. Open the Court Case and read **Intake Owner** (`Intake_Owner__c`). If it names the rep, the credit is theirs.
2. Open the Sales Comp dashboard and confirm the hire appears against that rep.
3. If **Hired By (User)** names someone else, explain that it only records who moved the case to Hired and does not affect points.
4. Only if Intake Owner is wrong or blank is there something to correct. That is a different problem from this one.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- The two field labels are easy to confuse, so this question may come up again. Nothing on the page explains the difference.

## 11. Reuse for similar issues

**How to recognise one:** "my credit/points/commission went to someone else" where the user is pointing at a field that names another person.

- **Find which field the report or dashboard actually groups by** before assuming automation failed. Open the dashboard component's source report and read its grouping.
- **Similar-sounding user fields often mean different things** (who owns it, who created it, who clicked a button). Confirm which one feeds the number in question.
- **Answer with the evidence:** the field value and the dashboard line that counts it.
