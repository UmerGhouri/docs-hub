# Stop intake from creating duplicate clients


| | |
|---|---|
| **Ticket** | DDT-1120 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-10-09 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-09-25 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC, Flow, matching and duplicate rules) |
| **Area** | Intake: lead conversion and new clients |
| **Related** | DDT-951, DDT-1081, DDT-1250 |
| **Code** | PR #525, PR #541, PR #631, PR #667, PR #673, PR #675 in the ddt-salesforce repo |


---


## 1. Summary

Intake was creating an exact duplicate client (Person Account) about once a week, even after a large duplicate clean-up (DDT-951). The only active duplicate rule on Person Account alerted and allowed, and the three lead-conversion wizards never reached it. The fix added one shared matcher (`ClientDuplicateGuard`) that the wizards use to find an existing client, then extended it so every new lead is checked against existing clients, shows a banner naming the match, and lands its case on that client when it converts or signs. The New Client button's matching rule now matches a phone however it was typed. Family members who share a phone number are never matched on phone alone.

## 2. What was reported

- 2026-09-25: the duplicate population from DDT-951 is regrowing. Two production examples: one client re-created by intake five days after the clean-up with the same phone and email as a 2022 client, and another pair with the same phone key created a day apart. Nothing blocked or flagged the second record. Asked: find which rules exist and why these passed, add matching on phone key and email, never block a legitimate family member at the same number, and merge the second pair.

## 3. Scope

**In scope**

- New `ClientDuplicateGuard` and its use in `NewPotentialClientController`, `HiredMatterConvertController` and `ClientConvertController` (PR #525), with the birthdate fix-forward (PR #541).
- Lead-time matching (PR #631): new field `Lead.Matched_Client__c` plus dismiss fields (`Match_Dismissed_At__c`, `Match_Dismissed_By__c`, `Match_Dismissed_Client__c`), new `LeadClientMatch` (before-save stamp from `LeadTriggerHandler`), new `ClientMatchInvocable` (first step of the `Intake_Client` subflow), new `leadClientMatch` banner LWC, permission set `Lead_Client_Match`.
- Matching rule `PersonAccount.Accelerator_Matching_Rule2` gains first name, last name and `Phone_Key__c` (PR #667), with `AccountPhoneKeyDuplicateRuleTest`.
- Banner spacing fix (PR #673) and the three Lead record pages recorded in git (PR #675).
- Data: three duplicate pairs merged after full backups.

**Out of scope**

- Matching on phone alone with a different name: by decision, families share numbers.
- A backfill of leads saved before the deploy: the banner matches them live.

## 4. How it works (background)

Clients are Person Accounts. Each carries `Phone_Key__c`, a normalised phone number stamped before save. Leads become clients through three wizards (New Potential Client, the PI hire wizard, Convert Client), the DDT Intake Flow (which creates the client directly), the standard New Client button, and the online contract signing path. Standard duplicate rules only run on some of these paths.

## 5. Root cause

- The only active Person Account duplicate rule alerts and allows, and the conversion wizards never hit it.
- `NewPotentialClientController` always inserted a new Account and had no "existing client" option. All three reported duplicates came from this path.
- The hire wizard inserted new clients with save allowed, and its signed-contract replay only created new clients. Convert Client converted "new" with no account.
- The New Client button's matching rule compared the phone as typed, so the same number in a different format saved with no warning (found in a sandbox test on 2026-10-02).
- A later duplicate (2026-10-03) slipped past the first guard because the lead's last name was a placeholder, so the name rule failed although phone and email matched.

## 6. Solution

1. **One matcher** (PR #525). Same phone key or email, and same first name, and a shared last-name word, and not two known different birthdates. Phone alone never matches. The potential-client wizard converts into the existing client; the hire wizard and Convert Client refuse "new" and name the client; a signed contract lands on the existing client (refusing would strand a signed contract).
2. **Birthdate fix-forward** (PR #541). Review found the Jr/Sr birthdate escape received a blank value on three of four call sites, so a father and son at one phone could match and the son's case be filed on the father silently. All sites now pass the lead's birthdate and the new client stores it.
3. **Same phone and same email = same client** (PR #631), even if the name differs. This catches the placeholder-name case.
4. **Check every lead** (PR #631). A before-save step stamps `Matched_Client__c`; the banner shows "This looks like ..., a client since ..." with Open client and Dismiss. Warn, never block, because walk-ins and leads texted in by a lawyer must still be creatable by hand. Dismiss is saved per lead and client with who and when, so repeat click-throughs can be seen.
5. **Downstream uses the match.** Firm leadership decided on 2026-10-08 that a confident match (same phone and email, no birthdate conflict) attaches the case automatically with a message naming the client, and anything doubtful stops and asks.
6. **New Client button** (PR #667). The matching rule also matches name plus `Phone_Key__c`. Phone as typed stays as an item for the few clients with no key. Deployed in five ordered steps (rule off, matching rule inactive, new items, active, rule on) because a matching rule cannot be edited while active.

## 7. Testing

- PR #525: 80 of 80 tests across four classes in the sandbox; both production cases reproduced in `ClientDuplicateGuardTest`.
- PR #541: with the blank birthdate put back, the two new tests failed ("The son must not be filed on the father"); then 84 of 84 passed. Audit after the first deploy: 0 cases attached to an existing client in the first 29 conversions, then 1 correct attach out of 70 conversions.
- PR #631: 122 of 122 in the sandbox across seven test classes; six deliberate breaks (rule 1, phone-alone, phone normalisation, birthdate escape, warn-never-block, lead stamp) each turned a validation red. Jest 5 of 5.
- PR #667: a production check-only of the new test against the old rule failed on "the same phone typed another way alerts", proving the gap; 5 of 5 after the change.
- PR #673: the old template fails the spacing test, the new one passes.
- Sandbox click-through on each path (2026-10-09): leads made by hand, by web form, as PI and through the Intake Flow all showed the existing test client; Convert Client and the PI hire put the case on that client; a dismissed pair converted into its own client; a family member at the same number got no match. In production the banner, the New Client phone check and the family case were checked on real records.
- Not covered: phone-only duplicates could not be tested in the sandbox, which does not stamp phone keys; that path was checked read-only in production.

## 8. Step by step

1. List every path that creates a client and test a deliberate duplicate on each.
2. Write one matcher with explicit rules (phone key or email plus name; phone and email alone; never phone alone; birthdate conflict means a different person).
3. Call it from every conversion path before any insert.
4. Stamp a match on lead save and show a warning banner; store dismissals with user and time.
5. Add the normalised phone key to the standard matching rule, in the ordered deactivate/activate sequence.
6. Merge known duplicates only after a full backup of both sides, then check every related record against the backup.

## 9. Rollback

Remove the banner from the Lead pages; redeploy `LeadTriggerHandler` and `ClientDuplicateGuard` from before PR #631; reactivate the previous `Intake_Client` version. For the matching rule, remove the test class first, then restore the old rule in the same ordered sequence. Cases a conversion already placed on an existing client must be re-parented by hand. Merges cannot be undone except from the backups.

## 10. Known limits

- A wrong match is worse than a missed duplicate, so doubtful matches only warn; some duplicates can still be created by a user who dismisses the banner.
- The family case (same phone, different first name) gets no alert on any path, by decision.
- Leads saved before the deploy are matched live by the banner rather than stamped.

## 11. Reuse for similar issues

- Signs of the same problem: duplicates keep appearing after a clean-up, and they all come from one creation path. Check whether the duplicate rule runs on that path at all.
- Standard duplicate rules do not protect custom Apex conversion paths; put the match in code that every path calls.
- Normalise phone numbers into a key before matching.
- Ask the business which error is worse (missed duplicate or wrong match) and let that decide warn versus block.
