# Matter "Next Court Date" falls behind the real next hearing


| | |
|---|---|
| **Ticket** | DDT-1301 |
| **Project** | Red Flag Panel |
| **Status** | Live since 2026-10-15 (reached production) |
| **Priority** | Medium |
| **Reported by** | Support team (follow-up from DDT-1291), 2026-10-12 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex service, triggers, scheduled job) |
| **Area** | Court appearances: next court date |
| **Related** | DDT-1291 |
| **Code** | PR #715 in the ddt-salesforce repo |


---


## 1. Summary

Every open matter shows a "Next Court Date" that pages, list views and a messaging copy field read. On some matters it showed an old or wrong hearing. Flows wrote it with "the date of the appearance just saved", so an older hearing created later pulled it backwards, and a deleted or rescheduled hearing left its date behind. A new Apex service now keeps it on the nearest upcoming hearing with no disposition, or blank when there is none, on every appearance change and in a nightly sweep. Open matters were corrected after a backup.

## 2. What was reported

- 2026-10-12: on 651 open hired matters with an open hearing, `Next_Court_Date_Time__c` disagreed with the latest open hearing on 13 (10 behind, 3 ahead). Find which writer leaves it stale, who reads it, and fix it; correct the 13 with a backup first.

## 3. Scope

**In scope**

- New `NextCourtDateService` and its test class.
- The existing Court Appearance trigger, extended to after insert, update and undelete (not a second trigger).
- One line in the existing Opportunity before-update trigger.
- A new nightly entry in the org's existing minute scheduler, at 00:17 Chicago time.
- One-time correction of open matters.

**Out of scope**

- No flow was edited.
- Closed matters: they keep their last hearing date as history. Applying the rule to 20,402 closed matters would have blanked most of them.

## 4. How it works (background)

The matter has a plain date-time field for the next court date and a Date copy. About ten flows and an Apex controller write fields with this name; the same API name also exists on two other objects. Separately, a roll-up holds the latest hearing with no disposition, but it cannot show the nearest one, and adding a nearer hearing does not change it.

## 5. Root cause

All 13 matters explained (production, 2026-10-12):

1. An older hearing created later pulls the field backwards (7 of 13). The Court Appearance before-save flow writes each new appearance's date onto the matter, even when a later hearing already exists.
2. The field points at a hearing that no longer has that date (6 of 13). Nothing writes the field when an appearance is deleted, and the after-save flow only writes on a date change for an appearance marked "Most Recent".

## 6. Solution

Rules decided by firm leadership on the ticket:

- Show the nearest hearing with no disposition, today or later, worked out from the appearances themselves (because a nearer hearing does not move the roll-up).
- Blank when there is none: "Next Court Date should only ever show a hearing that's actually scheduled."
- Open matters only.

Implementation:

- `NextCourtDateService` writes both the date-time and Date fields from the nearest qualifying hearing (counted from midnight Chicago today), or blanks both.
- The Court Appearance trigger calls it on create, delete, restore, or a change to date, disposition or matter.
- The Opportunity trigger puts the nearest hearing back in the same save if a flow overwrites the date or a matter is reopened.
- A nightly sweep at 00:17 Chicago moves on matters whose shown hearing has passed; a missed night catches up.
- Off switch without a deploy: a trigger-settings record for this service.

Before changing data, all 367 active flows were scanned: none sends a client text or email when this date changes. The red flags read the roll-up, not this field, so cases waiting on the client keep flagging.

## 7. Testing

- Tests run through the real trigger and both flows: nearer hearing added to a case with a later one; later hearing created after a nearer one; disposition moves then blanks; delete moves then blanks; past-only hearing gives blank; closed matter left alone; reopen gives nearest; nightly sweep; the switch; the scheduler entry. Without the trigger changes, 5 of the 10 fail.
- Check-only validation against production: 69 of 69; deploy 72 of 72.
- Data correction after a backup of all 796 open matters: 39 updated (28 blanked, 6 moved to a nearer hearing, 2 moved on from a passed or stale one, 3 Date-copy fixes), 0 failed. Read-back: every open matter showed its correct next hearing.
- Firm leadership spot-checked production on 2026-10-14: 0 open hired matters with a next court date in the past, 0 blank while a future hearing exists.

## 8. Step by step

1. Separate which writers touch the field on the right object.
2. Explain every bad record by a writer path.
3. Agree the rule with the business: nearest or latest, blank or last date, which records.
4. Compute the value from the child records, not from a roll-up that misses the case.
5. Hook it into the existing triggers and correct any flow write in the same save.
6. Add a nightly sweep for time passing.
7. Check that nothing sends a message when the field changes.
8. Back up, correct the data, read back.

## 9. Rollback

Switch off through the trigger-settings record (no deploy). Code: redeploy the four existing files from the previous commit and delete the two new classes. Data: restore from the backup CSV.

## 10. Known limits

- A matter closed while a hearing is still upcoming keeps that future date; firm leadership confirmed this is fine, because closed matters are not worked.
- Two matters had an appearance with no court date entered; that is a separate data issue.

## 11. Reuse for similar issues

- Signs of the same problem: a summary date on a parent that disagrees with its child records after deletes or out-of-order inserts.
- Derive the value from the children on every change, and add a nightly pass for values that expire with time.
- Extend existing triggers rather than adding new ones, and give the logic an off switch.
