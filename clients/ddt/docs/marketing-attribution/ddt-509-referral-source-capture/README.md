# Record who referred a client, once, and carry it through conversion


| | |
|---|---|
| **Ticket** | DDT-509 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-07-28 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-07-24 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex / LWC / custom object / permission set) |
| **Area** | Intake; lead conversion; referral attribution |
| **Related** | DDT-555, DDT-556, DDT-557, DDT-570, DDT-810, DDT-811, DDT-824 |
| **Code** | PR #80 in the ddt-salesforce repo |


---


## 1. Summary

The firm had no structured way to record who referred a client (an outside attorney, a medical provider, a business partner, a past client). This feature added a "Referral Source" control to the Lead, Client, Court Case and Matter pages and inside all three conversion wizards. Staff pick a referral type and then the referring party. Behind the scenes one hidden record holds the referral, it carries forward on conversion with no re-entry, and a derived `Referral_Source__c` and `Referral_Source_Type__c` are stamped on the parent records for reporting. The existing "How did you hear about us?" field was not changed. It went live on 2026-07-28.

## 2. What was reported

- 2026-07-24: the feature was built and validated in the partial-copy sandbox and handed over for a production deploy as one package (record-page panel plus inline capture in the three wizards), with instructions to deploy the new components as-is and apply small edits to the current production versions of the existing controllers.

## 3. Scope

**In scope**

- New object `Inbound_Contact__c` with its fields (`Referrer_Type__c`, `Referring_Account__c`, `Referring_Attorney__c`, `Referrer__c` formula, `Channel_Type__c`, `How_did_you_hear__c`, `Occurred_On__c`, captured contact fields, parent lookups to Lead, Opportunity, Matter and Account), tab, record page and view override. Not user-facing.
- New fields `Referral_Source__c` (text) and `Referral_Source_Type__c` (picklist) on Opportunity, `Matter__c`, Account and Lead.
- Apex: `ReferralSourceService`, `ReferralSourceController`, `InboundContactTriggerHandler`, trigger `InboundContactTrigger`, `ReferralSourceServiceTest`.
- LWC: `referralPicker` (shared child) and `inboundContactReferrerPicker` (record-page panel).
- Edits to `ClientConvertController`, `NewPotentialClientController`, `HiredMatterConvertController` and the wizards `convertClientModal`, `newPotentialClient`, `convertToHiredMatter`: each carries the referral into `ReferralSourceService.linkOnConversion(...)`.
- Permission set `How_Heard_Attribution` (additive, read access to the new fields).

**Out of scope**

- "How did you hear about us?": untouched, by design.
- Flow-based conversion paths (the DDT Intake Flow, matter flows, the website lead API), the channel-fallback and returning-client tiers, and a historical backfill. These were listed as a later phase. The channel and returning-client tiers were built later in DDT-555.
- Placing the panel on pages: done in App Builder on the current production pages, not from the branch copies.

## 4. How it works (background)

Referring attorneys are Contacts with the title "Attorney" under one business account set up for referring attorneys; the attorney picker is scoped to that account. Other referral types search the matching kind of business account (for example Business Partner). On conversion, the referral moves from the Lead to the new Client, Court Case and Matter. The Client keeps the original referral (write-once); later engagements capture their own.

## 5. Root cause

Not applicable: new feature. Before it, referral source existed only as free text and picklist answers, which could not name the referring party.

## 6. Solution

Deployed to production on 2026-07-28 after merging current `main` into the feature branch (no conflicts) and reconciling the controller edits against production's versions.

Changes made during the deploy, and why:

- **`HiredMatterConvertController` kept a production-only fix.** The branch predated a signer-name fix already live in production. Deploying the branch copy as-is would have reverted that fix silently, and it compiled clean, so nothing would have flagged it. The deployed version carries both changes.
- **The referrer id is converted inside the service's try/catch.** In one controller it sat outside, so a bad value could break the conversion.
- **Server-side check on `Referrer_Type__c`.** It is a restricted picklist. A bad value used to throw inside a try/catch that swallowed errors, so the referral vanished silently. It is now checked against the field's allowed values, with a check that the referrer matches the type.
- **Hardcoded values centralised**, including the referring-attorney account name, now a named constant.
- **Opportunity field access added to the permission set.** A review before deploy found the permission set granted access to the new fields on Matter, Account and Lead but not on Opportunity. New fields are invisible even to system administrators until access is granted, so the Court Case fields would have looked empty with no error.
- **Conversion never fails because of a referral.** All four conversion paths call `linkOnConversion` inside a try/catch.

## 7. Testing

- Partial-copy sandbox: `ReferralSourceServiceTest` 9 of 9; full run of the related classes 29 of 29. The two Opportunity fields could not be tested there at first because the sandbox had reached its custom-field limit; they were later deployed and validated there during DDT-555.
- A test reported as a "pre-existing failure" in `HiredMatterConvertControllerTest` was in fact missing from the branch. With production's version of the controller it passed; nothing was excluded.
- Production check: a referral on a Lead, converted inside a savepoint and rolled back, stamped both the Client and the Court Case.
- Post-deploy check: the object, classes, and the derived fields on all four parents were present; the permission set carried all 8 field permissions; the permission set was assigned to 3 users.
- Not covered: real staff use. Adoption was later found to be very low (see DDT-570 and DDT-811).

## 8. Step by step

1. Deploy the new object, fields, classes, trigger and LWC components as built.
2. Apply the controller and wizard edits to production's current versions, not to the branch copies.
3. Grant read access to every new field on every object, including Opportunity, in the permission set; deploy it with or after the fields.
4. Run the named test classes for the changed controllers.
5. Assign the permission set (here by anonymous Apex) and confirm the fields can be queried.
6. Add the panel to the current production record pages in App Builder.
7. Convert a test Lead inside a savepoint and confirm the referral lands on the Client and Court Case.

## 9. Rollback

Remove the panel from the record pages, then redeploy the earlier versions of the three controllers and three wizards. The new object, fields and permission set are additive and can stay. Referral rows already captured stay on the object.

## 10. Known limits

- The attorney picker depends on the name of one business account. Renaming that account would silently empty the picker.
- A referrer must already exist as the right kind of business account to be picked (see DDT-570).
- Flow-based conversion paths did not carry the referral at launch; this left most new Court Cases with no referral row (raised again in DDT-811 and later tickets).
- The permission set was on 3 users at go-live.
- The deploy found earlier UTM mapping in two controllers that was live in production but missing from git. It was flagged to be committed separately.

## 11. Reuse for similar issues

- When a branch is older than production, deploy the new components as-is and re-apply edits to the existing files on top of production's versions.
- Every new field needs read access in a permission set before anyone, including admins, can see it. Check each object.
- Never let an optional step (like recording a referral) roll back a conversion: wrap it, but validate inputs first so the wrapper does not hide data loss.
