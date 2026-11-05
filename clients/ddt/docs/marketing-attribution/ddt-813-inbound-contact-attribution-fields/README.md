# New attribution fields on Inbound Contact, visible on the page and in the permission set


| | |
|---|---|
| **Ticket** | DDT-813 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-10-01 |
| **Priority** | Medium |
| **Reported by** | Firm leadership, 2026-08-24 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (custom fields, page layout, permission sets, Apex) |
| **Area** | Attribution: the Inbound Contact touch log schema |
| **Related** | DDT-811, DDT-814, DDT-817, DDT-818, DDT-824, DDT-825 |
| **Code** | PR #361, #372 and #580 in the ddt-salesforce repo |


---


## 1. Summary

The touch log (`Inbound_Contact__c`) needed fourteen new fields before calls, web visits and Leads could be logged as touches: source, click ids, call session key, credit flags, the linked-party discriminator and a Contact lookup. They were deployed together with the record page layout (pulled into git for the first time) and the `How_Heard_Attribution` permission set, so the fields were both present and visible. `Occurred_On__c` was made required, which needed two Apex fixes to avoid silently losing referrals. A follow-up granted read on Call Session so the permission set works on its own. Final state verified on 2026-10-01.

## 2. What was reported

- 2026-08-24 (DDT-813): add the new fields to Inbound Contact, get them on the page, and get them into the permission set. The original plan named six fields; the ticket corrected that to fourteen and said not to create `Touch_Type__c` because `Channel_Type__c` already exists. Done when a user holding only `How_Heard_Attribution` can see the fields on a record and read them by query, and an insert without `Occurred_On__c` is refused.
- 2026-09-30 (reopened by firm leadership): a user with `How_Heard_Attribution` but not the call-logging permission set could not see the Session lookup. Add read-only access to Call Session so the set works alone.

## 3. Scope

**In scope**

- Fourteen fields on `Inbound_Contact__c`: `Automated_Source__c`, `GCLID__c`, `Click_Id_Other__c`, `Click_Id_Type__c`, `Session__c`, `Call_Session_Key__c` (unique External Id), `Raw_Payload__c`, `Attributed_Channel__c`, `Weight__c`, `Is_First__c`, `Is_Last__c`, `Is_Assist__c`, `Linked_Party_Type__c`, `Contact__c`.
- `Occurred_On__c` made required; `Channel_Type__c` value list extended (9 values, none removed).
- The layout `Inbound_Contact__c-Inbound Contact Layout`, retrieved into git and updated.
- `How_Heard_Attribution`: field grants, with `Raw_Payload__c`, `Attributed_Channel__c`, `Is_First__c`, `Is_Last__c` and `Is_Assist__c` read-only (only the credit engine writes them), later read on `Call_Session__c`.
- `ReferralSourceService`, `ReferralSourceController`, `InboundContactSchemaTest`, and the object description (PR #372).

**Out of scope**

- Stamping `Is_First__c` / `Is_Last__c` / `Is_Assist__c`. That happens at conversion and belongs to DDT-824.
- Call-detail fields on Call Session (outcome, recording, AI read). They stay on the call-logging permission set by design.
- Backfilling `Occurred_On__c` on the 11 rows that existed before the change (see section 6).

## 4. How it works (background)

The record page renders the Inbound Contact layout through a single detail panel with no per-field components. The layout existed only in production (last changed 2026-07-28) and not in git. So a fields-only deploy would succeed while the fields stayed invisible. Field-level security is the other half: a field the user cannot read behaves exactly like a field that does not exist ("No such column").

`Occurred_On__c` is the event time. Every first-versus-last ordering and recency calculation reads it; without it the only time is the created date, which counts edits rather than events.

## 5. Root cause

Not applicable as a fault: this was new schema. Three problems surfaced during the build, each caught by a failed validation, not by the ticket:

- Two Apex paths create Inbound Contact rows without `Occurred_On__c` (`ReferralSourceService` at conversion and `ReferralSourceController` for the referral panel). Making the field required would have thrown `REQUIRED_FIELD_MISSING`, and the conversion path's try/catch only writes a debug line, so referrals would have stopped saving with no visible error.
- Neither query selected `Occurred_On__c`, so a null check on it would itself have thrown.
- Salesforce rejects field permissions on a required field, so the grant for `Occurred_On__c` had to be removed from the permission set.

A fourth issue was git drift on the object definition: git held 29 lines, production 171, missing `actionOverrides` and six other properties. Deploying the git copy would have stripped them from a live object.

## 6. Solution

- **PR #361 (2026-09-11):** the fourteen fields, the layout, the permission set grants, a schema test, and both Apex creation paths now set `Occurred_On__c = System.now()` only when it is blank. Restricted picklists (`Channel_Type__c`, `Click_Id_Type__c`, `Linked_Party_Type__c`) shipped with their value lists in the same deploy.
- **PR #372 (2026-09-14):** the object description was updated to match what the object now does. Rather than deploy git's stale copy, the change took production's own object definition and changed one element; a re-retrieve after deploy confirmed all 30 `actionOverrides` survived and `main` matched production.
- **The 11 legacy rows were left blank on purpose.** An early review suggested backfilling them with the created date. That was withdrawn: it would put a fabricated event time on every row. Instead the Apex paths stamp a true time the first time a row is touched, and untouched rows keep null, which the readers' `ORDER BY ... NULLS LAST` deliberately sorts last.
- **PR #580 (2026-10-01):** `How_Heard_Attribution` gained read-only access on `Call_Session__c`. Salesforce hides a lookup when the user cannot read the target object, so without it a holder of only this set could not see the Session lookup. Additive only.

## 7. Testing

- Verified in production on 2026-09-11 and 2026-09-12: 14 of 14 fields, 14 of 14 on the layout, the read-only split exactly as specified, `Occurred_On__c` not nillable, the done-when query returning a row.
- Full local test run after the deploy: 2,599 passing, 0 failing. `PhoneKeyServiceTest` (DDT-825) also inserts this object and had to be included in the test set.
- The done-when as a non-admin, 2026-09-30 (check-only in production, nothing written): a user holding the set ran the query and got a row with all 15 fields, the 5 engine fields were read-only, a save without Occurred On was refused, and a user without the set was denied. All three original assignees were System Administrators, who bypass field-level security, so an admin check could not prove the grants.
- After PR #580: a user holding `How_Heard_Attribution` but not the call-logging set could open the Session lookup from an Inbound Contact record.
- Gap found and closed: on 2026-09-12 the sandbox had 0 of 14 fields, which blocked DDT-814 from being built there; the same metadata was then deployed to the sandbox.

## 8. Step by step

1. Retrieve the record page layout from production into `force-app/main/default/layouts/` before changing anything.
2. Create the fields (and picklist value lists) in source; do not create a field that duplicates an existing one.
3. Find every Apex and flow path that creates the object; stamp any newly required field there when blank, and add it to their queries.
4. Add the fields to the layout and to the permission set, with engine-written fields read-only; remove any grant on a required field.
5. Deploy fields, layout, permission set and Apex together in one PR.
6. For object descriptions, start from production's object definition, not git's.
7. Verify as a non-admin user who holds only the permission set: open a record, run the query, try an insert without the required field.
8. Check that every lookup on the object points at an object the permission set can read.

## 9. Rollback

Not recorded in the ticket. Removing the fields would need a destructive deploy and would lose any data written to them by DDT-814, DDT-818, DDT-819 and DDT-821. Making `Occurred_On__c` optional again and reverting the permission set changes are ordinary deploys.

## 10. Known limits

- The 11 pre-existing rows still hold a null `Occurred_On__c`. A direct inline edit on one of them in the UI hits the required-field error until something sets the time. Their `Weight__c` is also blank.
- `Raw_Payload__c` is a long text area on a searchable object: it cannot be filtered, sorted, reported on or used as a key, and anything in it becomes searchable text. It must not hold caller details.
- The ticket listed four other permission sets with object read and View All on this object but no field grants. Whether they were given read on the new fields or left blind on purpose is not recorded in the ticket.
- Holders of only `How_Heard_Attribution` see the Call Session record but not its call details.

## 11. Reuse for similar issues

- A green fields-only deploy proves nothing about visibility. Check the layout (and whether it is in git) and the permission set, then test as a non-admin.
- Before making a field required, search every Apex class and flow that inserts the object; a swallowed exception turns a required field into silent data loss.
- Do not backfill a timestamp with a made-up value; a null that sorts last is more honest than a fabricated time.
- Before deploying an object definition from git, diff it against production; drift runs both ways.
- When a lookup disappears for a user, check read access on the target object, not just the field.
