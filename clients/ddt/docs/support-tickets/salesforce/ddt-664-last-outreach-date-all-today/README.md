# All leads show today's date for Last Outreach

| | |
|---|---|
| **Ticket** | DDT-664 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-24 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-08-06 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (record-triggered Flow), RingCentral call logging |
| **Area** | Lead and client outreach tracking: Last Outreach Date |
| **Related** | DDT2-656 (a Days Since Last Outreach report anomaly) |
| **Code** | PR #235 in the `ddt-salesforce` repo |

---

## 1. Summary

A staff member found every one of his leads showing today's date in **Last Outreach Date**, although he had not contacted them all that day. The flow that stamps the field, `RC_Last_Outreach`, had two defects: its outbound-only test matched inbound calls too, and it stamped the date the flow ran instead of the date of the call. When call logs synced late, every record they touched got today's date. The flow now counts outbound calls only and stamps the call's own date, clamped so it can never be in the future.

## 2. What was reported

- **2026-08-06:** a staff member reported that all his leads showed today's date for Last Outreach Date. He confirmed it applied to all his leads as far as he knew. He also mentioned that potential leads did not refresh when he did outreach, which had been happening for some time.

Why it matters: Last Outreach Date drives outreach cadence reporting and which leads reps call next. If it is wrong on every lead, genuinely overdue leads are hidden.

**Scale, from the investigation:** over the 45 days before the fix, 16,474 tasks qualified for this flow. Replaying the old and new rules over them, **5,773 tasks stamped under the old rule and not under the new, and all 5,773 were inbound calls.**

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `RC_Last_Outreach` | Record-triggered Flow on Task | Outbound-only direction test on the Client, Lead and Contact voice rules; new `OutreachDate` formula replacing `ContactStartDate`; all twelve references repointed |

**Out of scope**

- **SMS rules.** They have no direction test either, but `CallType` is not reliably populated on SMS tasks, so adding one risked stopping SMS stamping entirely. Noted for a separate ticket.
- **Backfilling records that already had a wrong or blank date.** The PR says this was handled separately; the ticket does not record it.
- **Potential leads not refreshing** after outreach. Mentioned by the reporter; not addressed in this change.

## 4. How it works (background)

```
A call is logged in Salesforce as a Task, in one of two shapes:
  RC app log      call object "s-..."      CallType populated (Inbound / Outbound)
  RingSense log   call object "RS~s-..."   CallType BLANK; direction only in the
                                           subject, e.g. "[RingSense] Outbound call with ..."
  -> RC_Last_Outreach runs on the Task
  -> if the rule matches and the task is not already counted (Last_Outreach_Done__c)
  -> stamps Last_Outreach_Date__c on the related Lead / client / Contact
```

RingSense logs are the majority: 13,914 of the 16,474 qualifying tasks in the 45-day sample.

## 5. Root cause

**1. The direction test did nothing.** The Client and Lead voice rules used:

```
(1 AND (2 OR 5) AND 3) OR 4
```

where condition 2 was `CallType = 'Outbound'` and condition 5 was `CallType = 'Inbound'`. `(2 OR 5)` is true for every call, so inbound calls counted as outreach. All three rules also ended in `OR 4`, a bare match on a `[RingSense]` subject. That matched either direction and also bypassed the `Last_Outreach_Done__c` guard, so re-saving a RingSense task stamped the record again.

**2. It stamped the wrong day.** The date written was:

```
DATEVALUE({!$Flow.CurrentDateTime})
```

That is the moment the flow ran, not the moment of the call. When RingCentral synced call logs late, each task arrived after the call and stamped today's date. The ticket does not record which particular sync produced the all-today pattern on 2026-08-06.

## 6. Solution

### 6.1 Outbound only, read from either task shape

All three voice rules now use:

```
(1 OR 2) AND 3
```

- 1: `CallType = 'Outbound'` (RC app logs)
- 2: subject contains `[RingSense] Outbound` (RingSense logs)
- 3: not already counted

Why both: RingSense tasks have a blank `CallType`, so a test on `CallType` alone would have silently stopped most outreach stamping. Reading direction from the subject is already the idiom in this org (other flows do it). It is safe because every one of the 13,914 RingSense subjects in the sample contained a direction word with uniform casing (`[RingSense] Outbound` 8,796, `[RingSense] Inbound` 5,118). The separate RC logging-type condition was dropped as redundant, and the Contact rule, which already required outbound, was rewritten to read the same as the other two.

### 6.2 Stamp the call's date

A new formula, `OutreachDate`, replaces `ContactStartDate`:

```
IF(ISBLANK({!$Record.ActivityDate}), TODAY(),
   IF({!$Record.ActivityDate} > TODAY(), TODAY(), {!$Record.ActivityDate}))
```

- **Falls back to today** when `ActivityDate` is blank, so the change can never blank a real value.
- **Clamps to today**, because `ActivityDate` is set from UTC: a late-evening Central call can carry tomorrow's date (64 of 2,179 recent tasks, 2.9%). Unclamped, that would put Last Outreach in the future and make Days Since Last Outreach negative. The clamp still keeps the true date for tasks that synced a day late (57, 2.5%).
- **`TODAY()`, not `DATEVALUE($Flow.CurrentDateTime)`.** `DATEVALUE` on a datetime resolves in GMT, so between 7pm and midnight Central the clamp would compare against tomorrow and fail to fire. `TODAY()` resolves in the running user's time zone. Every creator of a qualifying task in the 45-day sample was on America/Chicago, so that was checked, not assumed.

### 6.3 Repoint every reference

All twelve references to the date across the six record updates were changed, including the backstop filters of the form `Last_Outreach_Date__c <= date`. Those were nearly useless while the date was always today; against the call date they stop an older, late-arriving call from moving a newer date backwards.

The change was built on the **active production version** of the flow, which carried backstop filters never committed to git; they came along in this diff.

## 7. Testing

**Sandbox, with savepoint-wrapped anonymous Apex, all rolled back, 10 of 10:**

| Case | Result |
|---|---|
| RingSense outbound | stamps the call date, not today |
| RingSense inbound | rejected |
| RC app outbound | stamps the call date |
| RC app inbound | rejected |
| Blank `ActivityDate` | falls back to today, does not blank |
| Future `ActivityDate` | clamped to today |
| Done flag | set |
| Re-saved task | does not re-stamp |
| Older late call | does not move a newer date backwards |
| Cross-midnight clamp | stamps the local date (sandbox user moved to a time zone where local and GMT dates differed, then restored) |

**Replay:** both rules run over all 16,474 production tasks from 45 days; the only difference was 5,773 inbound tasks no longer stamping. No outbound call lost its stamp.

**Production dry run:** succeeded, 32 of 32 tests. The flow's definition in production was last modified on 2026-08-24, matching the deploy.

**Not covered:** SMS stamping, and correcting records that were already wrong.

## 8. Step by step

1. **Retrieve the active production version** of `RC_Last_Outreach` and work on that, not the repo copy.
2. **Read each rule's condition logic** and check that the direction test actually excludes inbound calls.
3. **List the task shapes** that reach the flow and where each carries direction (`CallType` or subject).
4. **Replay old and new rules** over recent production tasks, and confirm only inbound tasks change.
5. **Replace the run-time date** with the `OutreachDate` formula and repoint every reference, filters included.
6. **Test in a sandbox** with rolled-back anonymous Apex for each case in section 7.
7. **Dry-run against production**, get approval, deploy, and confirm the new version is the active one.
8. **Check the next day's Last Outreach dates** against actual outbound calls.

## 9. Rollback

Reactivate the previous flow version in Setup → Flows → `RC_Last_Outreach` → View Details and Versions. That brings back inbound calls counting as outreach and run-date stamping.

## 10. Known limits

- **`TODAY()` depends on the running user's time zone.** RingSense tasks are created by the owner of the scheduled sync job. If that job is ever rescheduled by a user outside Central time, the clamp shifts with it.
- **SMS outreach still has no direction test.**
- **Records already stamped wrongly stay wrong** until backfilled.

## 11. Reuse for similar issues

**How to recognise one:** a "last activity" date that jumps to today across many records at once.

- **Check whether the date is the event's or the automation's.** `$Flow.CurrentDateTime` is when the flow ran, not when the thing happened.
- **Read condition logic literally.** `(Outbound OR Inbound)` is always true.
- **Find every shape of record** that triggers the automation. Integration logs often carry the same fact in different fields.
- **Replay a rule change over real data** before deploying, and confirm only the records you meant to change do.
- **`DATEVALUE` on a datetime is GMT; `TODAY()` is the user's time zone.** In a Central org that is a five-or-six-hour window every evening.
