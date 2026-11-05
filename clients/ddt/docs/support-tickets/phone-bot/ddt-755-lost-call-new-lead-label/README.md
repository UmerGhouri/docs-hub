# Lost-call alert calls an existing client a new lead

| | |
|---|---|
| **Ticket** | DDT-755, DDT-840, DDT-1054 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-19 |
| **Priority** | Medium (DDT-755, DDT-840), High (DDT-1054) |
| **Reported by** | Firm staff, 2026-08-18 |
| **Resolved by** | Support team |
| **Systems** | Phone bot (lost-call alerts), RingCentral call routing, Salesforce (Contact formula field) |
| **Area** | Lost-call alerts: how a caller is labelled and which team the call is routed to |
| **Related** | None recorded |
| **Code** | PR #199 in the `ddt-salesforce` repo |

---

## 1. Summary

Lost-call alerts in the phone-bot channel sometimes said **"new lead"** or **"unknown caller (Lead)"** for people the firm already knew. The tickets turned out to have different causes.

- **DDT-755:** emergency contacts of existing clients had no routing extension of their own, so the Contact formula that picks a lost call's extension fell through to the intake default. The call went to Intake, and the alert treated the caller as a new lead. PR #199 added one branch to the formula so these contacts use their related account's owner extension.
- **DDT-1054:** not a bug. The client's phone number had been replaced on her record, so Salesforce no longer held it anywhere and the only match was a placeholder Lead.
- **DDT-840:** closed the same day it was opened. What was done is not recorded in the ticket.

## 2. What was reported

- **DDT-755, 2026-08-18.** Two lost-call alerts from the evening of 2026-08-17. Each named an existing Salesforce Contact in its `Client:` line, yet the header said "new lead". In the second call the caller was an emergency contact of an existing client, and the call rang Intake first instead of Case Management. Both calls went to voicemail; at the time, only one person in the queues that rang was available.
- **DDT-840, 2026-08-26.** A lost-call alert titled "LOST CALL — new lead" whose `Client:` line named an existing Contact, on a call routed through Case Management. The header contradicted the body of the same alert.
- **DDT-1054, 2026-09-18.** A lost call from a hired client was labelled "unknown caller (Lead)". The ticket also noted that every member of the queue that rang was on Do Not Disturb or already on a call, so nobody could answer.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Contact.Contact_Owner_Extension__c` | Formula field (Number) | One branch added before the final intake default (DDT-755) |

**Out of scope**

- **`Contact.Contact_Owner_Fallback_Extension__c`.** It mirrors the same branch order and was deliberately left unchanged (see section 10).
- **The routing itself.** Which queues ring, and in what order, is configured in RingCentral and the routing service. The phone bot only reports what happened.
- **Queue staffing (DDT-1054).** Everyone on Do Not Disturb at the time of the call is a staffing question, not a system fault. No decision on it is recorded in the ticket.
- **Contacts with no link to any client.** See section 10.

## 4. How it works (background)

```
Call comes in
  -> RingCentral creates a call session; the router looks up the caller in Salesforce
  -> the matched Contact's Contact_Owner_Extension__c says which extension owns the caller
  -> if nobody answers, the phone bot posts a LOST CALL alert
       header: classification of the caller (new lead / existing client)
       body:   "Client:" line with the Salesforce match
```

Before the change, `Contact_Owner_Extension__c` resolved like this:

```
Referring Attorney                        -> intake extension
Owner_Extension__c set                    -> Owner_Extension__c
Account.Client_Owner_Extension__c set     -> Account.Client_Owner_Extension__c
otherwise                                 -> intake extension
```

**Caller matching (from DDT-1054):** the matcher runs one search over Contact and Lead phone fields and prefers a Contact. If no Contact holds the number, the only hit can be the placeholder "RingCentral Caller" Lead that RingCentral creates a second into the call.

## 5. Root cause

### DDT-755: emergency contacts fell through to intake

Emergency contacts have no `Owner_Extension__c` and no owning Account of their own. They sit under a generic emergency-contacts account with no case manager assigned. So every one of them reached the last branch of the formula and resolved to the intake extension. The call was routed to Intake, and the alert announced a caller the firm already knew as a new lead.

`Contact.Related_Account_Owner_Extension__c` already existed and was populated on **256** Contacts, but nothing read it.

How the Contacts split when the fix was written:

| Bucket | Count | After the fix |
|---|---|---|
| `Related_Account_Owner_Extension__c` set | 256 | fixed |
| `Owner_Extension__c` set | 62 | already correct (branch 2) |
| Neither set | 849 | still route to intake |

### DDT-1054: the number was no longer in Salesforce

The ticket asked whether the number was stored in the wrong format. It was not a format or lookup problem.

- Field history showed the number had been the client's `Account.Phone` until 2026-08-07, when a staff member **replaced** it with a new number. The same save overwrote `Client_Phone__c`. `MobilePhone` and `OtherPhone` were both empty.
- So no Contact held the number. The only match was the placeholder "RingCentral Caller" Lead, which is why the alert said "unknown caller (Lead)".
- Call sessions from that number resolved to the client's Contact right up to 2026-08-06, and none did after.

### DDT-840

Not recorded in the ticket. The ticket has no comments and no linked code change.

## 6. Solution

### DDT-755: read the related account's extension

One branch inserted before the final default, so no branch above it moves and nothing that routed correctly before can route differently now:

```
IF(NOT(ISBLANK(Related_Account_Owner_Extension__c)),
   Related_Account_Owner_Extension__c,
   <intake extension>)
```

Why this way:

- **Additive, at the bottom.** Putting the branch last means it only ever catches Contacts that used to fall through to intake.
- **Types checked first.** The formula returns Number, so every branch must be numeric. `Contact_Owner_Extension__c` is Number (18,0) and `Related_Account_Owner_Extension__c` is Number (15,0).
- **No drift.** Production and `main` were byte-identical on this field before the change, so it was not applied over an unrecorded production edit.

After the deploy, the Support team also backfilled the related extension on the emergency contacts of current (active) clients, and reported that the rest were being worked through.

### DDT-1054: no code change

The answer was to explain the cause: the old number had been replaced on the client record. Nothing was deployed. Whether the old number was added back to the record is not recorded in the ticket.

### DDT-840

Not recorded in the ticket.

## 7. Testing

**Sandbox check (DDT-755).** The field was deployed to a sandbox and queried on all four paths:

| Path | Expected | Result |
|---|---|---|
| Only `Related_Account_Owner_Extension__c` set | that extension | matched on all 5 records checked |
| Only `Owner_Extension__c` set | that extension | matched on all 3 |
| Only `Account.Client_Owner_Extension__c` set | that extension | matched on all 3 |
| Nothing set | intake extension | matched on all 3 |

The first row is the fix. The other three are the regression check: they prove the new branch did not displace anything above it.

**Live check:** the deploy doc lists "confirm a known emergency contact now routes to the case manager, not intake" as the requester's step. Its result is not recorded in the ticket.

**Not covered:** there is no automated test. A formula field has no Apex to test, and the alert classification lives in the phone bot, outside this repo.

## 8. Step by step

**When an alert calls a known caller a new lead:**

1. Find the caller's Contact and read `Contact_Owner_Extension__c`. If it is the intake extension, the call went to Intake by design of the formula.
2. Check which branch it took: `Owner_Extension__c`, `Account.Client_Owner_Extension__c` and `Related_Account_Owner_Extension__c`. If all are blank, the Contact has no link to a client's owner and needs one.
3. If the alert says "unknown caller (Lead)", search Salesforce for the number on Contact phone fields. If nothing holds it, check the client's field history for a replaced phone number (DDT-1054).

**To repeat the DDT-755 change:**

1. Retrieve `Contact_Owner_Extension__c` from production and compare it to `main`.
2. Add the branch from section 6 before the final default.
3. Deploy it to a sandbox and query one Contact per path, as in section 7.
4. Get the pull request approved, then deploy the single field to production. A formula has nothing to recalculate: records show the new value the next time they are read.
5. Backfill `Related_Account_Owner_Extension__c` on the Contacts that need it.

## 9. Rollback

Redeploy the previous version of `Contact_Owner_Extension__c`. The added branch has no dependents, so removing it is safe, but emergency contacts go back to routing to Intake.

## 10. Known limits

- **849 Contacts still route to intake.** They have neither extension field set. About 15% could be matched to a client by phone; about 85% have no link to any client at all. 19 have no phone, so they cannot be routed at any extension. No formula can reach them: they need an intake-side step that sets the extension when the contact is created, or a one-time assisted backfill.
- **The fallback field is out of step.** `Contact_Owner_Fallback_Extension__c` still resolves by the old order. For the 256 Contacts this fixed, the primary extension now goes to the related account's owner while the fallback does not. If the two should agree, that is follow-up work.
- **Replacing a phone number loses the match.** When staff overwrite a client's number instead of adding the new one, calls from the old number stop matching the client (DDT-1054).
- **Unanswerable queues.** A correct label does not help if everyone in the queue is on Do Not Disturb.

## 11. Reuse for similar issues

**How to recognise one:** the alert's header and its `Client:` line disagree, or a caller the team knows is announced as new.

- **Read the routing formula before the alert code.** The label followed the extension the call was routed to, so the fix was in Salesforce, not in the phone bot.
- **Look for a field that exists but nothing reads.** `Related_Account_Owner_Extension__c` was populated on 256 Contacts and unused.
- **Add a formula branch at the bottom**, just before the default, so it can only change records that used to hit the default. Then test every existing path, not just the new one.
- **For "unknown caller", check field history before the matcher.** A number that was replaced rather than added looks exactly like a lookup bug.
