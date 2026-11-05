# Capture who referred a new client, with an honest "not known" escape


| | |
|---|---|
| **Ticket** | DDT-862, DDT-1022, DDT-1083, DDT-1086 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-09-22 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-08-28 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex / LWC / Flow / custom metadata / picklists and record types) |
| **Area** | Intake: referral capture when a lead is converted or a client is hired |
| **Related** | DDT-992, DDT-570, DDT-815, DDT-810, DDT-1084, DDT-1178, DDT-1154, DDT-1285, DDT-1358 |
| **Code** | PR #397, PR #450, PR #451, PR #459 and PR #462 in the ddt-salesforce repo |


---


## 1. Summary

When a client said "an attorney referred me", staff could pick a referral type but could not record who, unless that person was already in Salesforce, and nothing made them try. DDT-862 let staff create the referrer inline and added an explicit "asked, they could not name who referred them" answer with a required one-line note. DDT-1022 measured why referral rows were still almost never created on the main intake path. DDT-1083 and DDT-1086 then added two missing referral types, moved the gate into the shared picker used by all three conversion wizards, put the same gate on the DDT Intake Flow, and switched it on for leads whose how-heard answer is a referral. In the first week, referral rows created on the Court Case path went from 3 (in the three weeks before) to 9.

## 2. What was reported

- 2026-08-28 (DDT-862): when a new lead or client was referred, staff should create a record for the referrer and not be allowed to move on without doing so.
- 2026-09-16 (DDT-1022): 47 Leads since 2026-07-28 said they were referred, 25 of them converted with a Court Case, and only 1 referral row on that path named a referrer. Find out why.
- 2026-09-22 (DDT-1083): implement DDT-1022's decision: add the missing referral types, port the DDT-862 gate into the shared picker, then turn it on.
- 2026-09-22 (DDT-1086): DDT-862 leftovers: the escape was missing from the three conversion wizards, two wizards still said "Referral Source (optional), Leave blank otherwise", and two display defects.

## 3. Scope

**In scope**

- `Inbound_Contact__c` fields `Referrer_Not_Known__c` (checkbox) and `Referrer_Described_As__c` (text 255), with field access in the `How_Heard_Attribution` permission set.
- `ReferralSourceController` (inline create of a referrer: Contact for an outside attorney, Account for a provider or business partner, Person Account for a previous client; duplicates refused) and `ReferralSourceService.validateCapture` (the one server-side rule).
- LWC `referralPicker` (shared child), `inboundContactReferrerPicker` (record-page panel), the three conversion wizards `convertClientModal`, `convertToHiredMatter`, `newPotentialClient`, and the new flow screen component `referralPickerFlow`.
- Controllers `ClientConvertController`, `HiredMatterConvertController`, `NewPotentialClientController` and `SendContractService` (contract send also checks the rule).
- Picklist values `Friend / Family` and `Internal` on `Inbound_Contact__c.Referrer_Type__c` and the four `Referral_Source_Type__c` mirrors (Lead, Opportunity, Matter, Account); `Referral - Friend / Family` on `Attributed_Origin__c` (Lead, Opportunity) and in `AttributionCreditService.TYPE_TO_ORIGIN`; eight record types that carry those picklists.
- `Attribution_Config__mdt` fields `Referrer_Gate_Enabled__c` and `Referrer_Gate_Scope__c`.
- Flow `Intake_Court_Case_v2` (the Court Case screen of the DDT Intake Flow) and the invocable `ReferralCaptureFlowAction`.

**Out of scope**

- A hard block with no escape: rejected by the DDT-992 decision (see section 6).
- The Business Accounts subflow of the intake flow: it asks no how-heard question (0 of 653 Court Cases in 30 days used it).
- Changing which leads are gated beyond referral how-heard answers: left as a Setup switch.
- The client-type stamp on newly created referrer Accounts: DDT-1084.

## 4. How it works (background)

A referral is stored as an `Inbound_Contact__c` row carrying `Referrer_Type__c` and a lookup to the referring party (`Referring_Account__c` or `Referring_Attorney__c`). On conversion, `ReferralSourceService.linkOnConversion` links that row to the client and the Court Case or Matter. The attribution engine (`AttributionCreditService.resolve`, step 2a) credits a record to the referral tier based on the referral type, not the name, so a referral with no named party still counts as a referral. `Referrer__c` is a formula and cannot be written.

Clients were already telling intake they were referred, but that answer went into the `How_did_you_hear_about_us__c` picklist, which says nothing about who.

## 5. Root cause

DDT-862 audit (2026-09-10, production): all referral components were search-only, so a referrer not already in Salesforce could not be recorded. In total only 11 referral rows had ever existed; 4 named a referring account, 7 had a type and no name, and `Referring_Attorney__c` had never been filled once. No validation required referral capture.

DDT-1022 (2026-09-16, production, read-only) found four reasons the Court Case path stayed empty:

1. The picker in the conversion wizards was optional and ungated. Its header read "Referral Source (optional)" and "Leave blank otherwise".
2. The wizard's save checked the validity of its own inputs with a query over its own template. The picker is a separate component with its own shadow root, so that check could never reach the picker's fields.
3. The escape from DDT-862 existed only on the record-page panel, which runs after a referral row exists. On conversion no row exists yet.
4. `Referrer_Type__c` had four values (Medical Provider, Business Partner, Previous Client, Outside Attorney). "Referred by Friend / Family" (12 Leads) and "Referred by Employee" (9 Leads) fitted none of them: 21 of 47 referred Leads could not be typed at all.

On the first day live (2026-09-23) a referred Court Case still got through without a referrer. It came from a fourth path: the DDT Intake Flow, a home-page screen flow that converts with standard Lead convert. It handled 184 of 556 conversions in the previous 30 days, all of the Hired ones.

## 6. Solution

1. **Decision (DDT-992, 2026-09-11): require a referrer, with an escape.** Staff either name a referrer or tick "I asked, they could not name who referred them" and type one line about what the caller did say. They can always save. Why: a hard block either loses the lead or trains staff to type junk, and junk in a referrer field looks like data. The escape is not a `Referrer_Type__c` value, because the engine gates the referral tier on that field.
2. **Inline create and the escape on the panel (PR #397, 2026-09-15).** The create button sits in the shared child, so all surfaces got it. The escape and its note are enforced server-side: not-known with a blank note is rejected, and naming a referrer clears both.
3. **Vocabulary first (PR #450).** `Friend / Family` became its own type with origin `Referral - Friend / Family`; `Internal` was a straight add because the engine already mapped it to `Referral - Internal`. Why first: a gate without the right values forces staff to pick a type they know is wrong. Restricted picklist values are enforced per record type when saving, so eight record types were retrieved fresh from production and patched with only the new values.
4. **Gate in the shared picker (PR #450).** The tickbox and note moved into `referralPicker`, which now exposes `reportValidity()`; the three wizards call it from their save and pass `referrerNotKnown` and `referrerDescribedAs` to Apex. The "(optional), Leave blank otherwise" text was removed.
5. **A switch, scoped to referral answers.** `Referrer_Gate_Enabled__c` is on, and `Referrer_Gate_Scope__c` defaults to "Referral how-heard only". Why: the engine credits any typed row, not-known included, to the referral tier, so gating every conversion would push ad leads into "type + not known" and credit them as referrals. "Every conversion" is one Setup change, no deploy.
6. **Display fixes (DDT-1086).** "simplyheard" without a space was the LWC compiler trimming whitespace at a line-broken element boundary; fixed with non-breaking spaces and pinned by a jest assertion. The note field's double error message became one.
7. **The DDT Intake Flow (PR #459, 2026-09-23).** Built from the active production version of `Intake_Court_Case_v2`, not git's: the picker sits on the Court Case screen for every branch, and `ReferralCaptureFlowAction` checks the same rule and writes the row through `linkOnConversion`, updating an existing row on a re-hire rather than duplicating it.

## 7. Testing

- DDT-862: 30 Apex tests passing; the screen was opened in production and both paths were exercised by hand, which found the two display defects.
- `ReferralVocabularyTest` inserts one row per new value; it proved the record-type enforcement before the record types were added.
- Server-side proof in the partial-copy sandbox through the real controllers: a referred Lead with nothing captured was refused before any save; Friend / Family with a named Person Account produced a referral row and a Court Case resolving to `Referral - Friend / Family`, basis Structured Referral; Internal resolved to `Referral - Internal`; a billboard lead converted with nothing and created no referral row; switching the gate off let the refused lead through.
- Production deploys: fields and record types 17/17; classes and LWC with 175 tests, 0 failures; intake-flow classes with 113 tests, 0 failures (new class 11/11 in the sandbox). Jest: 419 tests green.
- Click test in the sandbox (2026-09-24): Lead Triage refuses a referred lead without a referrer, the escape needs a type and a note, the intake flow's Court Case screen blocks Next the same way, and a billboard lead converts with nothing.
- Week-later count (2026-09-25): 9 referral rows on the Court Case path since 2026-09-22, against 3 for the three weeks before.
- Not covered: nothing in CI renders a component, so screen behaviour rests on the manual checks.

## 8. Step by step

1. Count the gap: Leads whose how-heard answer is a referral, against referral rows linked to their Court Case.
2. Check every conversion path for the picker, including screen flows, not only the wizards you know.
3. Make sure every answer staff actually give has a type value, and that the engine maps each value to an origin.
4. Add new restricted picklist values to every record type that uses the field, retrieved fresh from production.
5. Put the rule in one server-side method and call it from every save path.
6. Expose a validity method on the child component and call it from the parent's save.
7. Deploy with the gate behind a custom metadata switch, then click-test each path.
8. A week later, re-count referral rows per week on the path.

## 9. Rollback

- Behaviour off, no deploy: untick `Referrer_Gate_Enabled__c` on the Default config record; the picker becomes optional again everywhere.
- Code: redeploy the previous versions of the classes and LWC bundles. Leave the new picklist values in place; deleting a value that rows already carry is the one thing not to do.
- Intake flow: set the flow definition back to the previous active version.
- Record types: pre-change production copies are kept in `docs/deploys/ddt-1083/rollback-recordtypes/`.

## 10. Known limits

- The Business Accounts intake subflow is not gated.
- When the type is missing, the screen shows both the field's own "Complete this field" and the fuller sentence below it. Harmless.
- Git's copy of the DDT Intake Flow was older than production's at the time; anyone editing it must start from production.
- `Attributed_Origin__c` gained a value that an external partner system reads; the notice to that partner was sent separately.
- Whether firm leadership confirmed the "Referral how-heard only" scope is not recorded in the ticket.
- Later refinements (asking who only when the answer leaves a gap, returning clients recorded as referrals, one combined panel) are DDT-1178, DDT-1154 and DDT-1285.

## 11. Reuse for similar issues

- Signs of the same problem: an answer is captured in a picklist but the related record it should create is almost never created. Count both before building anything.
- A required field inside a child LWC is not checked by the parent's input sweep; expose `reportValidity()` on the child.
- Fix the vocabulary before you add a gate, or the gate produces wrong data.
- Prefer "require, with a documented escape" over a hard block when the honest answer is sometimes "unknown".
- Put behaviour switches in custom metadata so they can be changed in Setup.
- Look for every conversion path, including screen flows on the home page.
