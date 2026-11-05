# Decision: require a referrer, with a "could not name them" escape


| | |
|---|---|
| **Ticket** | DDT-992 |
| **Project** | Marketing Attribution |
| **Status** | Done 2026-09-12 |
| **Priority** | Medium |
| **Reported by** | Firm operations, 2026-09-10 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (referral capture components, attribution resolver) |
| **Area** | Referral capture at intake |
| **Related** | DDT-862, DDT-1083, DDT-1086, DDT-1298 |
| **Code** | No code change |


---


## 1. Summary

A request (DDT-862) asked that staff not be allowed to save a referred Lead or client without naming who referred them. That clashed with the attribution resolver, which deliberately accepts a referral type without a name. This ticket put the choice to firm leadership. The decision was option 2: a hard requirement with a documented escape. Staff either name a referrer or tick "I asked, they could not name who referred them" and type a one-line description of what the caller did say. They can always save, and an unknown referrer becomes a deliberate answer instead of a blank.

## 2. What was reported

- 2026-09-10: decide between soft require, hard block with an escape, and hard block with no escape, before anyone builds the enforcement half of DDT-862.

## 3. Scope

**In scope**

- The decision, the name and shape of the escape, and the constraint on where it may be stored.

**Out of scope**

- Building it. The enforcement and escape were built under DDT-862 and its follow-ups (DDT-1083, DDT-1086, DDT-1298).
- The other half of DDT-862, letting staff create a referrer record inline, which this decision did not block.

## 4. How it works (background)

Referral captures are stored on the touch log (`Inbound_Contact__c`) with a referral type and optional referrer lookups. `AttributionCreditService.resolve()` gates the referral tier on the type, not the name, so a staff member who knows "a medical provider referred this" but not which practice still lands the case in the referral tier.

## 5. Root cause

Not a defect: a conflict between a requested rule and an existing design. The data at the time: of the 11 touch-log rows ever created, 7 had a referral type but no named referrer, and the attorney referrer lookup had never been filled once. Only 3 rows came from real staff use. A hard block with no escape would have blocked most real referral captures, and pushed staff either to abandon the record or to type something to get past the screen. Junk in a referrer field is worse than a blank, because it looks like data and flows into attribution.

## 6. Solution

Decided 2026-09-11 by firm leadership:

1. **Hard block with an escape.** A tickbox, "I asked, they could not name who referred them", and when ticked, a required one-line description ("a doctor's office in a nearby suburb", "a friend who was a client last year").
2. **Two new fields hold the escape**, suggested as `Referrer_Not_Known__c` (checkbox) and `Referrer_Described_As__c` (text). The escape cannot be a value in an existing field: `Referrer__c` is a formula and the referrer fields are lookups, which cannot hold "not known".
3. **Do not add "not known" to `Referrer_Type__c`.** The resolver gates the referral tier on that picklist, so an escape value there would corrupt the one field it depends on.

Why the description is required: an unnamed referral is otherwise a dead end, and one typed sentence lets someone follow it up later.

## 7. Testing

Not applicable: a decision ticket with no build. Testing belongs to the DDT-862 build.

## 8. Step by step

1. Before building a hard validation on a capture field, count how often real records leave it blank and why.
2. Check whether downstream logic already treats the blank state as valid.
3. Offer the decision owner the options with that data, including an explicit escape.
4. Store the escape in its own fields, never as a fake value in a field other logic reads.

## 9. Rollback

Not applicable: no system change.

## 10. Known limits

- The decision's last checklist item, updating DDT-862 for the builder, was still open when the ticket closed.

## 11. Reuse for similar issues

- Signs: a request to make a field mandatory where people often genuinely do not know the answer.
- A required field with no escape produces made-up data. Give staff a recorded "do not know" plus a short description instead.
- Keep escape values out of fields that drive logic; give them their own fields.
