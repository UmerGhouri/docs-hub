# Ask "how did you hear about us" once, and "who?" only when needed


| | |
|---|---|
| **Ticket** | DDT-1178 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-10-15 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-10-01 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC, custom metadata) |
| **Area** | Intake: referral capture |
| **Related** | DDT-1154, DDT-1180, DDT-862, DDT-902, DDT-1068, DDT-1083, DDT-1086, DDT-1285, DDT-1294, DDT-1330, DDT-1358, DDT-1362 |
| **Code** | PR #606, PR #706, PR #710, PR #712 in the ddt-salesforce repo |


---


## 1. Summary

After staff answered "how did you hear about us" with a named referral partner, the Referral Source section asked for a Referral Type and a Referring Party: the same answer twice. Worse, the partner had no Salesforce record, so the search could never find it and staff had to tick "could not name anyone" with a required note. The shared referral rule now reads the existing channel dictionary and returns one of four outcomes: nothing more (a channel), link the named referrer automatically, ask only "Who referred them?" (a general referral), or returning client (from DDT-1154). The Referral Type dropdown is gone where the answer already says the type, but the field is still written. All five screens that use the picker, and the Lead page panel, follow the rule.

## 2. What was reported

- 2026-10-01: converting a PI lead, firm leadership picked the firm's largest referral partner (a medical provider network with 31 PI hires in 12 months) as how-heard, and the screen then demanded a type and a party it could not find. Rule decided the same day: ask how-heard once; ask a follow-up only when the answer leaves a gap. Create the missing referrer records, link dictionary rows to them by a stable key (not a record Id), and clean up the affected row.
- 2026-10-12: reopened for three gaps: the Lead page panel still showed the dropdown, one dictionary row had a wrong type, and no click-through had been done on all five screens.

## 3. Scope

**In scope**

- `ReferralSourceService.captureRule` and `effectiveCapture`, `ReferralSourceController.getCaptureRules`, reading `Channel_Mapping__mdt` (`Origin_Type__c`, `Referrer_Name__c`).
- `Referrer_Type__c` derived through the existing `CompCaseSourceService.referrerTypeFromToken`.
- Callers: `ClientConvertController`, `HiredMatterConvertController`, `NewPotentialClientController`, `ReferralCaptureFlowAction`, `SendContractService`, `DochlyContractService`.
- LWCs `referralPicker`, `convertClientModal`, `convertToHiredMatter`, `newPotentialClient`, `referralPickerFlow` (PR #606) and `inboundContactReferrerPicker` on the Lead page (PR #706, PR #712).
- Data: 8 missing referrer Business Accounts created, one test row corrected.
- PR #710: one dictionary row deleted.

**Out of scope**

- Re-labelling outside law offices (DDT-1294).
- A slow first page load briefly showing the full picker (DDT-1330).

## 4. How it works (background)

`Channel_Mapping__mdt` is the dictionary that turns a how-heard answer into a channel and origin type. Of 23 answers marked as referrals, 14 name a specific referrer (9 distinct referrers) and 9 are general (friend or family, past client, outside attorney, employee). The picker LWC is shared by five screens; the record-page panel on Leads is a separate wrapper.

## 5. Root cause

The dictionary already held what the screen needed (whether an answer is a referral, what kind, and who), but the screen never read it. It asked the type and party regardless, and named partners had no Account to link to.

## 6. Solution

1. **One server rule** (PR #606) returns none, named, ask, or returning, and is applied on the server before every caller validates or writes, so a named referrer is linked even if a screen sends nothing.
2. **Keep writing `Referrer_Type__c`**, derived from the dictionary through the existing comp-source mapping. Why: it feeds structured-referral credit and sales comp (DDT-902); reusing the mapping avoids a second one.
3. **Match named referrers on the exact Account name**, never a record Id, because Ids differ between sandbox and production and custom metadata deploys the same value everywhere. Zero or two matches fall back to asking.
4. **Legacy brands link to new firm Accounts**, not to the employees who brought them; look-alike Person Accounts turned out to be past clients.
5. **Lead page panel** (PR #706) now reads the Lead's how-heard answer and passes it to the picker; it waits for both the answer and the saved referral before drawing, so the old dropdown does not flash first, and writes nothing on page load.
6. **"Change referrer" fix** (PR #712): review found it saved the rule's type rather than the type shown on screen; it now saves what is on screen.
7. **Bad dictionary row removed** (PR #710): one named referrer was a name typed once as a lead source, not a firm or client. Its row and the empty Account created from it were deleted after a backup; the other named rows were checked against record counts and kept.

## 7. Testing

- PR #606: 227 of 227 in the sandbox; Jest 555 of 555; a break run shows the named-answer tests fail without the named lookup. Production probe: 14 named, 9 ask, channels none.
- PR #706: 49 panel tests (42 new) and 84 across the picker family; many negative controls each failing their own tests; rendered in the sandbox for named, ask, channel and blank.
- PR #712: 53 panel tests and 88 across the family; negative controls for each fix.
- PR #710: production check-only 100 of 100 across three classes that read the live dictionary.
- Live (2026-10-14 and 2026-10-15): checked on real production Leads and the Convert Client page; sandbox click-through of the intake Court Case flow, Convert to Hired Matter and New Potential Client passed 12 of 12.
- The original example (the medical network linking as a medical-provider referral) was overtaken by DDT-1285, which made that partner a paid vendor, so in production it is now a channel and asks nothing; the named-referrer behaviour was proven with it in the sandbox.

## 8. Step by step

1. Classify every how-heard answer in the dictionary as channel, named referrer, or general referral.
2. Return that outcome from one server rule and apply it before every write.
3. Create Accounts for named referrers and match them by exact name.
4. Remove the redundant question from every screen, but keep writing any field other processes depend on.
5. Click through each screen with one example of every outcome.
6. Review dictionary rows that came from a single typed lead before creating records for them.

## 9. Rollback

Redeploy the classes and LWC bundles from before PR #606 (and the panel from before PR #706). Rows saved while it was live keep their links and are still read correctly by the old code. The created Accounts can be deleted, but that blanks the links on rows that use them; count them first. The deleted dictionary row can be redeployed from git.

## 10. Known limits

- A slow first load of the Lead page can briefly show the full picker (DDT-1330); nothing is saved on load.
- A named referrer whose Account name changes, or is duplicated, falls back to asking.
- The record-page panel off Leads still shows the full picker as a repair tool.

## 11. Reuse for similar issues

- Signs of the same problem: staff tick "could not name anyone" for a well-known partner. Check whether the answer already names them and whether a record exists.
- When configuration already classifies an answer, have the screen read it instead of asking again.
- Link configuration to records by a stable key such as an exact name, never by record Id.
- Data-entry tokens can leak into a dictionary; check counts before turning one into a record.
