# Adding a user to intake analytics and performance reports

| | |
|---|---|
| **Ticket** | DDT-227 |
| **Project** | Support tickets |
| **Status** | Done 2026-06-24 |
| **Priority** | Medium |
| **Reported by** | Intake team, 2026-06-24 |
| **Resolved by** | Support team |
| **Systems** | RingCentral (Analytics, Performance Reports) |
| **Area** | Phones: per-user call reporting for the Intake team |
| **Related** | DDT-228 (same user's missed-call rate, found from this report), DDT-226 (the Salesforce equivalent for sales reports) |
| **Code** | Not applicable: the change was made in RingCentral's Analytics settings |

---

## 1. Summary

Intake leadership tracks each Intake user's call volume, inbound and outbound mix, missed-call percentage and average handle time in RingCentral's Performance Reports. An intake specialist was not in the set of users that view covered. The support team added him, and on the requester's confirmation also removed a user who should no longer appear.

## 2. What was reported

- **DDT-227 (2026-06-24).** The Performance Reports **Users** tab for the Intake view showed 9 of 9 users, and the intake specialist was not among them. The requester asked for him to be added so his activity could be reported like the rest of the team. His RingCentral user was enabled.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Intake Performance Reports user filter | RingCentral Analytics setting | Intake specialist added |
| Same filter | RingCentral Analytics setting | One user who should no longer appear removed, at the requester's request |

**Out of scope**

- **Call routing.** How calls reach the user is a separate setting; it came up next and is DDT-228.
- **Salesforce.** Nothing changed in Salesforce.

## 4. How it works (background)

```
RingCentral Analytics -> Performance Reports -> Users tab
  -> shows only the users in the view's user filter (or group)
  -> per user: total calls, inbound / outbound, % missed, average handle time
```

A user who is enabled in RingCentral but not in that filter makes and takes calls normally; they are simply not reported in that view.

## 5. Root cause

The view's user list is chosen explicitly. When a person joins Intake, nothing adds them to it automatically, and this specialist had never been added.

## 6. Solution

The support team edited the Intake view's user selection in RingCentral Analytics to include the specialist, and asked whether a second user listed there should be removed. The requester said yes, and that user was removed. The exact menu path used is not recorded in the ticket.

## 7. Testing

- **Live check:** the requester thanked the team after the change; a specific check of the new user count is not recorded.
- **Not covered:** nothing alerts when a new Intake user is missing from the view.

## 8. Step by step

1. **Confirm the user is enabled** in RingCentral admin.
2. **Open RingCentral Analytics, then Performance Reports, then the Users tab**, with the Intake view or saved filter the requester uses.
3. **Edit the user filter (or the group it is based on)** and add the user.
4. **Ask whether anyone listed should be removed**, such as people who have left Intake, and remove them if so.
5. **Save the view** and confirm the user count changes and the user appears with call data for a day they worked.

## 9. Rollback

Remove the user from the filter again. Safe: it only changes what the report shows, not call handling or call history.

## 10. Known limits

- **Every new Intake hire needs this step.** The list does not follow team membership on its own.
- **A user's "missed" percentage in this report can be misleading** when calls ring the whole team at once: a call a teammate answers counts as missed for everyone else. DDT-228 found exactly that for this user.

## 11. Reuse for similar issues

**How to recognise one:** "X isn't on the call report" while X can make and take calls normally.

- **Check the report's user selection first**, not the user's account.
- **Do the reporting update as part of onboarding**, alongside the Salesforce sales reports (DDT-226), since both use fixed lists.
- **Clean up leavers at the same time**, so team totals and counts stay right.
