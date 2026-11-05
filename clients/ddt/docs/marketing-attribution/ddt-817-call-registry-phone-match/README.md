# Call phone numbers did not match the tracking-number registry


| | |
|---|---|
| **Ticket** | DDT-817 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-09-10 |
| **Priority** | Urgent |
| **Reported by** | Firm leadership, 2026-08-24 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (formula field, lookup field, permission set), RingCentral |
| **Area** | Attribution: matching a call's dialled number to its tracking-number row |
| **Related** | DDT-816, DDT-818, DDT-825, DDT-868, DDT-965, DDT-1082 |
| **Code** | PR #272 and #344 in the ddt-salesforce repo |


---


## 1. Summary

The attribution design joins each inbound call to the tracking-number registry by phone number. As written, that join matched zero rows and would not error: calls store numbers in E.164 (with the leading +1) while the registry stores 10-digit local numbers in three different formats. A new formula field `Phone_E164__c` on the registry produces the E.164 form, a lookup `Phone_Tracking_Number__c` was added to Call Session, and both were granted in the call-logging permission set. After the change 13,781 inbound calls in 90 days matched a registry number, and the result was re-checked as a non-admin user.

## 2. What was reported

- 2026-08-24 (DDT-817): add `Phone_E164__c` to `Phone_Tracking_Number__c` and a lookup to it on `Call_Session__c`, ship the field-level security with them, and flag a decision on calls that carry a bare extension instead of a number. Done when a join of the last 90 days of inbound calls to the registry returns a non-zero count, run as the user the flow runs as, not as a System Administrator.

## 3. Scope

**In scope**

- `Phone_Tracking_Number__c.Phone_E164__c` (formula text).
- `Call_Session__c.Phone_Tracking_Number__c` (lookup).
- `DDT_Call_Logging` permission set: read on both, edit on the lookup only.
- A deploy note with the verification queries.

**Out of scope**

- Populating the new lookup on each call. Nothing wrote it at go-live; that became DDT-1082.
- A marketing channel field on Call Session. Deriving the channel belongs in the credit engine only, to avoid a second dictionary.
- Changing the extension-versus-number logic in `RC_SessionBuilder` (decided against, see section 6).

## 4. How it works (background)

`Phone_Number_ONLY__c` on the registry strips brackets, spaces, dashes, dots and plus signs but keeps no country code, giving 10 digits. `Call_Session__c.Internal_Party_Number__c` (the firm number that was dialled) stores E.164. Every other call object in the org already had a `Phone_Tracking_Number__c` lookup; Call Session was the only one missing it. `Lead.Phone_E164__c` already existed as a formula, so the new field reuses that name.

## 5. Root cause

The two sides stored the same number in different formats (10 digits versus +1 plus 10 digits), so an equality join found nothing and raised no error. Measured before the change: the join matched zero rows.

## 6. Solution

- `Phone_E164__c` = `'+1'` plus `Phone_Number_ONLY__c`, guarded on `LEN(Phone_Number_ONLY__c) = 10` so a malformed row yields blank, not a wrong number.
- The Call Session lookup, matching the other call objects.
- Field-level security in the same deploy, because a field the user cannot read fails the same way as a field that does not exist.

**The extension decision.** 2,112 of 16,952 inbound calls over 90 days (12.5%) carried a bare RingCentral extension. Research on 2026-08-29 over all 17,355 inbound calls found: 2,163 carried an extension; 1,321 of those had a sibling call from the same caller within 24 hours that did carry a firm number (most are the child leg of a blind transfer, and 1,522 were already flagged as transferred); 842 had no sibling; 161 of those were answered; 71 lasted 60 seconds or more. So the real unattributable share is about 0.4%, not 12.5%. Decision (firm leadership): accept and record it. Changing `RC_SessionBuilder` to put a number on transfer legs would double-count 1,321 calls.

**The verification step.** The first replacement query used a semi-join on `Phone_E164__c`, which SOQL rejects (`MALFORMED_QUERY: semi join sub selects can only query id fields`), because sub-selects take Id fields only. PR #344 changed it to a two-step read: select `Phone_E164__c` from the registry, then use those values in an `IN` list. That keeps the check honest: if the field were missing or unreadable, there would be nothing to paste.

## 7. Testing

- Production after deploy (2026-09-10, 3/3 components): 48 of 48 registry rows had `Phone_E164__c`; 13,781 inbound calls in 90 days matched a registry number (the 14,185 measured on 2026-08-29 was the same population in an earlier rolling window, so the doc states a band of 13,000 to 15,000); the lookup was readable and empty, as expected.
- An earlier version of the verification query filtered only on fields that already existed, so it returned the same count before and after the deploy and could not fail. A reviewer caught it and it was replaced.
- Non-admin check, 2026-09-30 (check-only in production, nothing written): an intake user holding `DDT_Call_Logging` read 49 registry numbers and matched 13,668 inbound calls in 90 days. As a control, a user without the set was refused outright, so the check can fail.

## 8. Step by step

1. Compare the stored format of the join key on both objects with a sample query; do not assume they match.
2. Add a formula that converts one side to the other's format, guarded so bad input yields blank.
3. Add any missing lookup so the match can be stored.
4. Grant read on the new fields in the permission set the running and reporting users hold, in the same deploy.
5. Verify the join with a query that reads the new field, run once as an admin and once as a non-admin, plus a control user who lacks the set.

## 9. Rollback

Remove the permission set grants and delete the two fields with a destructive deploy. Not recorded in the ticket; the fields only add data and nothing depended on them at go-live.

## 10. Known limits

- The formula covers 10-digit North American numbers only.
- The lookup on Call Session stayed empty until DDT-1082.
- About 0.4% of inbound calls (orphaned extension legs) cannot be attributed by number.
- This adds a fourth formula `Phone_E164__c`. A formula cannot back an index or External Id, which is why DDT-825 introduced a stored key alongside it.
- Six other Call Session linkage fields were found to be readable by almost nobody; that is DDT-868.

## 11. Reuse for similar issues

- A join that matches nothing usually fails silently; measure the match count before and after any change.
- A verification query must read the thing that was changed, or it cannot fail. Execute it before publishing it.
- SOQL semi-joins accept only Id fields; use a two-step read for text keys.
- Before "fixing" a large unmatched share, find out why it exists; here it was transfer legs already attributed on a sibling row.
- Prove field access as a non-admin, with a control user who should be refused.
