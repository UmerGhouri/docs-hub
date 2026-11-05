# Emergency contacts of clients routed to Intake

| | |
|---|---|
| **Ticket** | DDT-741, DDT-883, DDT-945, DDT-946, DDT-1257 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-28 (DDT-945); last ticket in the group closed 2026-10-07 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-08-14 |
| **Resolved by** | Support team |
| **Systems** | RingCentral, the external call router, Salesforce (formula fields, lookup field, flows, permission set) |
| **Area** | Inbound call routing for people calling about an existing client |
| **Related** | DDT-849 (the Lost / closed-client intake queue rule) |
| **Code** | PR #336, #369, #373 (DDT-945); PR #347, #534, #576 (DDT-946) in the ddt-salesforce repo |

---

## 1. Summary

When an existing client's emergency contact (a parent, spouse and so on) called the main line, the call rang Intake instead of the client's case manager. The router picks the extension from a formula on the caller's Contact record. For emergency contacts that formula fell through to Intake, because every emergency contact sits under one shell "Emergency Contacts" account rather than under the client.

The fix added a `Related_Client__c` lookup from each emergency contact to its client, pointed both routing formulas at it, backfilled it on existing records, and added a flow that fills it for new ones. An emergency contact now rings exactly where its client rings. Two side problems were fixed along the way: a duplicate-creation bug in the intake flows (DDT-946), and a few records that were not emergency contacts at all.

## 2. What was reported

- **DDT-741 (2026-08-14).** A caller listed in Salesforce as an existing client's emergency contact rang through the Intake path for 94 seconds and was lost. She should have reached Case Management.
- **DDT-883 (2026-08-31).** Another emergency contact went Intake, then Intake + Case Management, then voicemail. The same day a different emergency contact was routed correctly to the client's case manager, so routing looked inconsistent. Comments on the ticket added existing clients (not only emergency contacts) also landing on Intake first.
- **DDT-945 (2026-09-07).** The underlying fix: emergency contacts should ring their client's extension. Measured at filing: 1,246 emergency contacts, most computing to the Intake extension.
- **DDT-946 (2026-09-07).** Found during the same investigation: 7 records under the emergency-contact record type were not emergency contacts (a courthouse clerk, three attorneys, a medical provider), and 24 records were duplicates, every pair created within 60 seconds of each other.
- **DDT-1257 (2026-10-06).** An emergency contact rang the Intake queue and was not answered. Investigation showed this was the agreed routing working as designed (see 5.4).

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Contact.Related_Client__c` | Lookup to Account | New (DDT-945) |
| `Contact.Contact_Owner_Extension__c` | Formula | New first branch reads the related client's extension |
| `Contact.Contact_Owner_Fallback_Extension__c` | Formula | New first branch reads the related client's fallback extension |
| `Related_Client_Field_Access` | Permission set | Read-only access to the new field |
| `Emergency_Contact_Set_Client_Lookup` | Record-triggered flow | Fills the lookup for new emergency contacts |
| `ddt-945-phase2-backfill.apex` | Anonymous Apex | One-off backfill of the lookup on existing records |
| `Intake_Emergency_Contact`, `SF_Client_Create_Emergency_Contact` | Screen flows | Duplicate search made exact (DDT-946) |
| `Participant_Role__c` | Picklist | Two attorney participant roles added (DDT-946) |
| Individual Contact records | Data | DDT-741 extension filled on one contact; DDT-946 re-filed, merged or deleted records |

**Out of scope**

- **The call router.** It lives outside Salesforce and was not changed. It reads values Salesforce pushes to it.
- **Re-parenting emergency contacts onto the client account.** Not possible (see 5.2).
- **Fallback extensions.** About 70% of client accounts have no fallback extension, so their emergency contacts have none either. That needs its own ticket.
- **Emergency contacts with no client link** (about 20 real records). Left ringing Intake: when nobody knows whose contact someone is, Intake is the honest answer.
- **The PI phone line.** It uses fixed-order routing, so an emergency contact calling that line does not reach the case manager. Noted by the requester after testing; not changed.
- **Routing differently by authorization to speak.** Decided against: routing a call discloses nothing, and whoever answers still decides what to say.

## 4. How it works (background)

```
Caller dials the main line
  -> RingCentral -> the external router looks up the caller's number
  -> it reads the routing values Salesforce pushed for that Contact:
       Contact_Owner_Extension__c           (who rings first)
       Contact_Owner_Fallback_Extension__c  (who rings next)
  -> Salesforce pushes those values when the Contact is saved
     (BatchFlowyteContactRouting / FlowyteRoutingService)
```

`Contact_Owner_Extension__c` before the fix, in words:

1. Referring Attorney record type: Intake.
2. Otherwise a hand-typed `Owner_Extension__c`, if present.
3. Otherwise the parent account's `Client_Owner_Extension__c`.
4. Otherwise a related account's extension.
5. Otherwise Intake.

`Account.Client_Owner_Extension__c` decides where a client rings, by `Primary_Client_Type__c`: an active client rings its case manager; a potential client rings the intake owner; a past client closed within 180 days rings the Defense or PI case-management line; a lost client, or a past client closed more than 180 days ago, rings the Lost / closed-client intake queue (DDT-849).

## 5. Root cause

### 5.1 Emergency contacts were parented to a shell account

All emergency contacts had the shell account "Emergency Contacts" as their direct parent (`AccountId`). The real client was linked only through an indirect `AccountContactRelation` (`IsDirect = false`, with the relationship such as "Father" in `Roles`). The shell account has no extension, so branch 3 of the formula returned blank and the formula fell through to Intake.

### 5.2 The ticket's proposed fix could not work

DDT-945 proposed re-parenting each emergency contact so its client became the direct parent. Testing one record first showed why not: **every client account behind these contacts is a Person Account** (all 1,210 checked), and Salesforce does not allow a Contact under a Person Account. The fix had to be a lookup instead.

### 5.3 Why routing looked inconsistent (DDT-883)

Both callers in DDT-883 were checked in Salesforce and the RingCentral logs. The router behaved consistently; the data differed. The caller routed correctly belonged to an active client with a case manager. The caller routed to Intake matched a client whose records were split across two accounts: on the hired account the calling number was only a secondary number (routing reads the primary only), while on a second account with only a lost case that same number was the primary, and the hired account had no case manager assigned. The final data clean-up for that client is not recorded in the ticket.

For DDT-741, the one reported contact had no extension set. It was filled in so she routed to Case Management.

### 5.4 DDT-1257 was the agreed rule, not a fault

The caller's client was a "Past Defense Client" whose last case closed 222 days before the call. Past clients closed more than 180 days ago ring the Lost / closed-client intake queue, and the emergency contact rings wherever the client rings. That was ratified on DDT-945 (see 6.1), so the routing was correct.

### 5.5 Duplicate emergency contacts (DDT-946)

The first diagnosis was a fault-retry loop: the create commits, a later step faults, the user is sent back and creates the record again. Review showed that premise was wrong: the contact create and the relationship insert run in one transaction, so a fault rolls both back and nothing is left to duplicate. A forced-fault sandbox test confirmed it.

The real cause: both intake flows search for an existing contact before creating one, but the search matched `Name Contains {FullName}`, where `FullName` is first name, a space, and last name. First name is optional. With it blank, `FullName` starts with a space that the stored `Name` does not contain, so the search missed and a resubmit created a second record.

## 6. Solution

### 6.1 A client lookup on the emergency contact (DDT-945)

New field `Related_Client__c` (Lookup to Account). Both formulas gained a new branch, placed right after the Referring Attorney branch:

```
Contact_Owner_Extension__c:
  IF(NOT(ISBLANK(Related_Client__r.Client_Owner_Extension__c)),
     Related_Client__r.Client_Owner_Extension__c,
     ...existing branches...)

Contact_Owner_Fallback_Extension__c:
  IF(NOT(ISBLANK(Related_Client__r.Client_Owner_Extension__c)),
     Related_Client__r.Client_Owner_Fallback_Extension__c,
     ...existing branches...)
```

Why this shape:

- **A lookup, not a code change in the router**, so the field on screen, the audit count and the phone all agree. The firm's measure of the problem ("how many ring Intake") is read off this formula.
- **The new branch comes before the hand-typed `Owner_Extension__c`**, so the client wins over stale hand-typed values without clearing them one by one.
- **The fallback branch tests the client's primary extension but returns the client's fallback.** This is deliberate, so both formulas take the same branch. A client with a primary but no fallback still takes it and correctly returns a blank fallback.
- **Clients who ring the Lost / closed-client queue keep it.** About 30% of linked emergency contacts belong to such clients. Excluding them would make an emergency contact ring somewhere different from its client, which breaks the rule. This was ratified on the ticket.
- **Contacts linked to two clients** took the link that carries a relationship role; in each case both clients rang the same place.

### 6.2 Backfill existing records

`scripts/apex/ddt-945-phase2-backfill.apex` filled `Related_Client__c` from the indirect relationship, filtered on:

```
Contact.RecordType.DeveloperName = 'Emergency_Contact'
  AND the relationship's IsDirect = false
  AND Related_Client__c = null
```

The record-type filter matters. `IsDirect = false` on its own also matched 19 contacts who are not emergency contacts (mostly clients who are also someone else's emergency contact); stamping those would have misrouted them on day one. The record type matched the shell account's population exactly.

Result reported on the ticket: 1,226 contacts stamped in 7 chunks with zero failures. Contacts ringing Intake went from 892 to 24, and contacts ringing their own client from 0 to 1,202.

### 6.3 Keep new records correct

`Emergency_Contact_Set_Client_Lookup` is record-triggered on `AccountContactRelation` create where `IsDirect = false`. A decision checks the contact's record type by developer name, and the update only runs when `Related_Client__c` is blank, so it never overwrites an existing link. It deliberately does **not** filter on `Roles` containing "Emergency Contact": intake writes the real relationship (Mother, Spouse and so on), and only 82 of 1,238 such relationships carried that literal role.

The flow was deployed as Draft and activated on 2026-09-14, after the backfill. The older flow `Emergency_Contact_Set_Owner_Extension` stays active until it is retired; the two do not conflict because the lookup branch wins.

### 6.4 Stop the duplicates (DDT-946)

In both `Intake_Emergency_Contact` and `SF_Client_Create_Emergency_Contact`, the pre-create search `Try_Get_Contact_Record` now matches exactly on:

- `AccountId` = the Emergency Contacts shell account (scopes the search to emergency contacts)
- `LastName` = the entered last name (required)
- `Phone` = the entered phone (required)

A resubmit now finds the existing contact and shows the existing "Possible duplicate contact found" screen, and the user adopts it. Adopting a contact who is already the emergency contact for another client (a parent of two siblings, say) and adding a new relationship is the intended outcome, not a duplicate.

Switching the org's duplicate rule from Alert to Block was considered and rejected: it would block genuine same-name contacts across every contact-creation path, and the flow needed fixing anyway.

Production does not deploy flows as active, so each deploy created a Draft version; version 5 of both flows was then activated through the FlowDefinition (recorded in PR #576).

### 6.5 Data clean-up (DDT-946)

- 20 duplicate groups (24 redundant records) were **merged**, not deleted, so activities and relationships stayed on the surviving record. A backup was taken first.
- The medical provider was re-created as an Account on the Medical Providers record type, and both contact copies removed.
- The three attorneys and the courthouse clerk were recorded as **Participants** on the relevant court case, not moved to the Referring Attorney record type (none were referring attorneys). Two attorney roles were added to `Participant_Role__c`. The clerk kept a Contact record so the number still matches on caller ID.

## 7. Testing

**Why a green deploy proves nothing here.** The router lives outside Salesforce, so CI never dials a phone, and CI executes none of a flow's runtime. Every check below was a live one.

| Check | Result |
|---|---|
| Spot check after backfill | Stamped contacts computed to their client's exact extension |
| New emergency contacts after the backfill (DDT-945, 2026-09-25) | 55 created, none landed on Intake |
| Live calls, Defense and PI, from a test emergency contact created through the normal intake screen (2026-09-28) | Initially rang Intake; after corrections, every call routed to the case manager on the file |
| Duplicate search, sandbox, both flows, including a blank first name | A resubmit reached the duplicate screen; one record |
| Duplicate search, production (DDT-946) | One record, no duplicate, blank first name included |

Re-measured on 2026-09-25: 1,301 emergency contacts, 1,277 ringing their own client, 24 still on Intake (the unlinked ones).

**Not covered:** the PI line (fixed-order routing), and fallback extensions for clients that have none.

## 8. Step by step

1. **Find what the router reads.** For RingCentral routing it is `Contact_Owner_Extension__c` and `Contact_Owner_Fallback_Extension__c` on the caller's Contact.
2. **Check the caller's record before blaming the router.** Look at the record type, the direct parent account, any hand-typed `Owner_Extension__c`, and whether the number is the primary or a secondary number. Check whether the client is split across two accounts.
3. **Check the client's own routing.** Read `Primary_Client_Type__c`, `Last_Case_Closed_Date__c` and the case manager. A past client closed more than 180 days ago rings the Lost / closed-client queue by design.
4. **For a contact missing its client link,** confirm the indirect `AccountContactRelation` exists and that `Related_Client__c` is filled. If not, fill it from the relationship.
5. **For a new deploy of this design:** deploy the field, both formulas, the permission set and the flow (Draft). Assign the permission set with anonymous Apex.
6. **Run the backfill** with the record-type filter in 6.2. Confirm the routing push fired for the changed contacts, or run `BatchFlowyteContactRouting`.
7. **Activate** `Emergency_Contact_Set_Client_Lookup`.
8. **Live test:** create a test emergency contact through the normal intake screen on one Defense and one PI client, check the formula shows the client's extension, then call the main line from that number. Delete the test contacts afterwards.
9. **Duplicates:** if two identical emergency contacts appear within a minute of each other, check the pre-create search in both intake flows still matches on shell account, last name and phone.

## 9. Rollback

- **Routing:** reverting the two formulas is a complete rollback on its own. Deactivate the lookup flow, then run `BatchFlowyteContactRouting` so the router receives the old values. Do not delete `Related_Client__c` as part of a rollback; it is harmless left in place, and a deleted field counts against the object's field limit until purged.
- **Duplicate search:** re-activate version 4 of each intake flow in Setup → Flows. No data change.
- **Merged duplicates:** a merge cannot be undone in Salesforce; restore from the backup taken first.

## 10. Known limits

- **Fallback extensions are blank for about 70% of clients**, so their emergency contacts have no second ring.
- **The PI phone line** uses fixed-order routing and does not route emergency contacts to the case manager.
- **About 20 emergency contacts have no client link** and still ring Intake.
- **Emergency contacts of lost or long-closed clients ring the Lost / closed-client queue**, by design. Staff may read this as Intake (DDT-1257).
- **A client split across two accounts**, or with no case manager, routes wrongly however the emergency contact is set up.
- **Formula changes do not refresh the router on their own.** The push fires on DML against the Contact.
- **A hand-typed `Owner_Extension__c` still wins** for contacts without a related client, and returns no fallback.

## 11. Reuse for similar issues

**How to recognise one:** "this caller should have reached the case team but rang Intake."

- Read the formula the router reads, branch by branch, against the caller's actual record. Most routing tickets are data, not the router.
- Two calls routed differently under the same rule usually means two different records, not an inconsistent router.
- Test the ticket's proposed fix on one record first. Re-parenting failed on the first record because the clients were Person Accounts.
- Scope a backfill by record type, not by a relationship flag that other records can share.
- A screen flow's "does this already exist?" search should match exactly on required fields. A `Contains` on a name built from an optional field misses.
- Merge duplicates rather than deleting them, and take a backup first.
- Prove routing with a real phone call from a record created through the real intake screen.
