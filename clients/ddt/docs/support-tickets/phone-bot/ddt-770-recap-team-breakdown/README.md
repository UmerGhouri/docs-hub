# Hourly and daily recap team breakdown

| | |
|---|---|
| **Ticket** | DDT-770, DDT-808 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-24 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-08-19 |
| **Resolved by** | Support team |
| **Systems** | Phone bot (hourly and daily recaps), RingCentral (queues, user departments) |
| **Area** | Call reporting: answer rate by team |
| **Related** | None recorded |
| **Code** | Phone bot code, outside the `ddt-salesforce` repo. No PR number is recorded in the tickets. |

---

## 1. Summary

The phone bot's hourly report and daily recap grouped answer rates as Intake, Case Management, PI and a catch-all **Other**. Staff asked for named teams instead. The breakdown now lists Intake, Case Management, SOS, PI, Collections and Attorneys on both reports. Calls that rang a person's direct extension, rather than a queue, are now assigned to a team by that person's RingCentral department, which cut "Other" from 25% of calls to 7%.

## 2. What was reported

- **DDT-770, 2026-08-19.** The hourly report's "By team" section had an "Other" row that hid which team actually took the call. The first request was to replace it with PI and Attorneys. In the same thread, staff added SOS and Collections, and asked for the change on the daily recap as well.
- **DDT-808, 2026-08-22.** A follow-up after the daily recap still showed the old grouping. It set the full list of six teams, asked for both reports, and asked how Collections should be shown, since the person handling collections has no RingCentral queue of their own and receives calls by internal transfer.

DDT-770 was closed as a duplicate of DDT-808, which holds the delivered solution.

## 3. Scope

**In scope**

- The team mapping used by the hourly report and the daily recap.
- Assigning calls to a direct extension to a team by the answering user's department.

**Out of scope**

- **Calls routed through a queue.** Their team assignment is unchanged.
- **RingCentral user data.** Blank or wrong departments are fixed in RingCentral, not in the phone bot (section 10).
- **Salesforce.** Nothing in the `ddt-salesforce` repo changed.

## 4. How it works (background)

```
Calls during the hour (or day)
  -> each call is assigned to a team
       queue-routed call          -> the team that owns the queue
       direct-extension call      -> before: no queue, so "Other"
  -> per team: answered / total and pickup %
  -> posted as "Answer rate by team" on the hourly report and the daily recap
```

## 5. Root cause

Not a fault: a reporting change. The reason "Other" was large is that team assignment depended on the queue a call rang. Calls to a person's direct extension, including internal transfers to the collections role, had no queue and so fell into "Other".

## 6. Solution

- The team list on both reports became **Intake, Case Management, SOS, PI, Collections, Attorneys**.
- **Direct-extension calls** are now assigned by the answering user's **department** in RingCentral. Queue-routed calls keep their existing mapping.

How Collections in particular is represented is not spelled out in the ticket beyond the department-based handling above.

## 7. Testing

From the resolution comment on DDT-808 (2026-08-24):

| Check | Result |
|---|---|
| Calls accounted for in the verification sample | 250 of 250 |
| Share of calls in "Other" | 25% before, 7% after |
| Phone bot test suite | 822 passing, 15 of them new |

The tickets do not name the individual tests or record the requester's confirmation of the new layout.

## 8. Step by step

1. Agree the final team list with the requester in writing. Here it grew from four teams to six inside one thread.
2. Map each RingCentral queue to a team.
3. For people who take calls on their direct extension without a queue, decide the team from their RingCentral department.
4. Apply the same mapping to both the hourly and the daily report.
5. Check a full day's calls: every call should land in exactly one team, and the "Other" share should drop.
6. List any users whose RingCentral department is blank or wrong, and have them fixed in RingCentral.

## 9. Rollback

Not recorded in the ticket. The change is in the phone bot, so rolling back means redeploying its previous version.

## 10. Known limits

- **7% of calls still land in "Other".**
- **Department data must be right.** At the time of the fix, 4 RingCentral users had no department and one user's department and job title needed correcting. Their direct-extension calls are grouped wrongly until that is fixed.
- **A person moving team** must have their RingCentral department updated, or their direct calls stay with the old team.

## 11. Reuse for similar issues

**How to recognise one:** a report has a large catch-all bucket, or a team that does not own a queue does not appear.

- **Find what the bucket is made of** before renaming it. Here it was mostly direct-extension calls.
- **Use a second signal when the first is missing.** Queue first, then the user's department.
- **Change every report that shows the breakdown**, not just the one in the screenshot.
- **Fix source data where it lives.** Wrong departments are a RingCentral admin fix.
