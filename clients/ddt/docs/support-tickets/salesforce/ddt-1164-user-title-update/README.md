# Updating a user's title

| | |
|---|---|
| **Ticket** | DDT-1164 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-01 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-09-30 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (User record, Reports) |
| **Area** | Case manager reporting: the Case Managers CC Feed Posts report |
| **Related** | None |
| **Code** | Not applicable: one field changed on a User record |

---

## 1. Summary

A staff member who had moved into a case manager role did not appear on the **Case Managers CC Feed Posts** report. The report filters on the user's **Title** being "Case Manager", and his Title still said "Intake Specialist". Updating the Title on his User record put him on the report.

## 2. What was reported

On 2026-09-30 a staff member reported that a colleague now working as a case manager was missing from the Case Managers CC Feed Posts report, which is used to monitor case manager activity on Court Case feeds. She also asked whether she or the colleague could change it themselves.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| The staff member's User record | `User.Title` | "Intake Specialist" → "Case Manager" |

**Out of scope**

- **The report.** Its filter was correct; no change.
- **Profile, role or permissions.** Not part of the request.

## 4. How it works (background)

```
Case Managers CC Feed Posts report
  filter: Title = "Case Manager"
  → anyone whose Title is anything else is left out, whatever work they do
```

## 5. Root cause

The report keys on a free-text field, `User.Title`, that is not updated automatically when someone changes role. The colleague's Title still held his previous job title.

## 6. Solution

Set `User.Title` to exactly "Case Manager", matching the report filter's spelling.

Whether the requester could do this herself is not answered in the ticket; the support team made the change.

## 7. Testing

After the change, the colleague appeared on the report with 195 posts from the last 30 days. The requester confirmed on 2026-10-01 that he was visible.

## 8. Step by step

1. Open the report and read its filter on Title (here, "Case Manager").
2. Setup → Users → open the user → Edit.
3. Set **Title** to the exact value the filter uses. Save.
4. Refresh the report and confirm the user appears.

## 9. Rollback

Set the Title back to its previous value. Safe; it changes only which reports include the user.

## 10. Known limits

- Every future role change needs the same manual edit, or the person drops out of (or stays on) the report.
- A typo or extra space in Title silently excludes the user.
- Other reports or automation that read `User.Title` change their behaviour for this user too.

## 11. Reuse for similar issues

**How to recognise one:** "person X is missing from report Y" when the person clearly does that work.

- **Read the report's filters first** and compare each one to the person's record.
- **Title, department and similar text fields are often used as team filters** and are rarely updated when people change roles.
- **Match the filter's spelling exactly.**
