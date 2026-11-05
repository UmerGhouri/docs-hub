# Outbound calls missing from the RingCentral call log

| | |
|---|---|
| **Ticket** | DDT-1340 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-16 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-10-14 |
| **Resolved by** | Support team |
| **Systems** | RingCentral, Salesforce (user record, call session sync) |
| **Area** | Call activity: linking RingCentral calls to the Salesforce user who made them |
| **Related** | Not recorded in the ticket |
| **Code** | Not applicable: fixed by correcting a user record, no code change |

---

## 1. Summary

A staff member made about 6 outbound calls and received 1 inbound call in a day, but only the 1 call showed against her name. The ticket assumed RingCentral had never captured the outbound calls. In fact RingCentral recorded all of them. Salesforce could not link them to her because her RingCentral extension was not connected to her Salesforce user. That was corrected, and her calls from 6 to 14 October now show under her name.

## 2. What was reported

On 2026-10-14 a staff member reported that her activity showed 1 call for the day when she had made about 6 outbound calls and taken 1 inbound. A first check of the RingCentral call log filtered to her extension also showed only the 1 inbound call, so the ticket concluded the outbound calls "were never captured by RingCentral".

Why it matters: her call activity feeds performance tracking, so it looked as though she handled 1 call a day.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| The staff member's Salesforce user | Data | Her RingCentral extension connected to her user record |
| Her calls from 6 to 14 October | Data | Now linked to her name in Salesforce |

**Out of scope**

- **RingCentral device registration, caller ID and provisioning.** The ticket listed these as possible causes. They were not the problem, because RingCentral had recorded every call.
- **Code changes.** None were needed.

## 4. How it works (background)

```
Staff member places or takes a call in RingCentral
  -> RingCentral records the call and its legs
  -> Salesforce receives the call and builds a Call Session (RC_SessionBuilder)
  -> each leg's extension is matched to a Salesforce user:
       active users with User.RingCentral_Extension__c set
  -> matched: the call shows under that user's name
     unmatched: the call exists, but under nobody
```

`RC_SessionBuilder` loads every **active** user whose custom field `RingCentral_Extension__c` is filled and maps extension → user. A user with that field blank is invisible to the matching, however many calls she makes.

## 5. Root cause

The staff member's RingCentral extension was **not connected to her Salesforce user**. Salesforce received her calls but had no user to attach them to, so they did not appear under her name.

**The ticket's first guess was wrong.** It said the outbound calls were missing from RingCentral itself and listed device registration, a shared device, caller ID and provisioning as suspects. On investigation, RingCentral had recorded all of the calls. The gap was entirely on the Salesforce side.

Why the first check of the RingCentral log showed only the 1 inbound call is not recorded in the ticket.

## 6. Solution

1. **Connect the extension to the user.** Her Salesforce user record was updated with her RingCentral extension, so new calls match her.
2. **Re-link the earlier calls.** Her calls from 6 to 14 October were linked to her name. How this was done (a re-sync or a backfill) is not recorded in the ticket.

**Use the custom field.** The matching reads `User.RingCentral_Extension__c` ("RingCentral Extension"), not the standard `Extension` field on the user record. Filling the standard field has no effect on call matching.

## 7. Testing

The staff member was asked to make her next call as usual and check that it showed under her name in Salesforce. She replied with thanks; an explicit confirmation of a new call is not recorded in the ticket.

**Not covered:** no automated test; this was a data fix.

## 8. Step by step

1. **Check RingCentral first, across the whole account,** not only the user's extension. Confirm the calls exist and note the extension they were made from.
2. **Check the user record in Salesforce.** Query the user's `RingCentral_Extension__c`:

    ```sql
    SELECT Id, Name, IsActive, RingCentral_Extension__c
    FROM User
    WHERE RingCentral_Extension__c != null AND IsActive = true
    ```

   If the user isn't in the result, the extension isn't connected.
3. **Check no other active user holds the same extension.** A reused extension still sitting on another user will attach the calls to that user instead.
4. **Set `RingCentral_Extension__c`** on the user to her RingCentral extension number.
5. **Re-link the calls already received** for the affected dates, so the past days show correctly.
6. **Ask the user to make a call** and confirm it shows under her name.

## 9. Rollback

Not applicable: clearing the field would only break the call matching again.

## 10. Known limits

- **New users need the field filled.** A new starter's calls will not show under their name until `RingCentral_Extension__c` is set on their user.
- **Only active users are matched.** If an extension is reused, an inactive user who still holds it does not take the calls, but the new user must be given it.
- **The standard `Extension` field is a trap.** It looks like the right place and does nothing for call matching.

## 11. Reuse for similar issues

**How to recognise one:** a user says their calls are missing, but RingCentral shows the calls were made.

- **Confirm in RingCentral whether the calls exist before blaming the phone.** Here the first look said they didn't, and they did.
- **Then check `User.RingCentral_Extension__c`** on the Salesforce user. A blank or wrong value is the most common reason calls exist but show under nobody.
- **Look for the extension on another user,** active or inactive, before assigning it.
- **Fix the past as well as the future:** after connecting the extension, re-link the calls from the days it was missing.
