# Returning clients were recorded as referrals


| | |
|---|---|
| **Ticket** | DDT-1154 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-09-30 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-09-29 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC) |
| **Area** | Intake: referral capture and attribution credit |
| **Related** | DDT-824, DDT-826, DDT-862, DDT-902, DDT-1083, DDT-1086, DDT-1178, DDT-1285, DDT-1360, DDT-1459 |
| **Code** | PR #547 in the ddt-salesforce repo |


---


## 1. Summary

When staff opened a new case for a client who had hired the firm before, the Referral Source section still demanded a referrer. The only way through was to choose "Previous Client", leave the referrer blank and tick "they cannot name anyone", and the credit engine then recorded the case as a client referral instead of a returning client. The shared referral rule now asks the credit engine whether the client is returning; if so, no referrer is demanded and a one-line note shows the last hire date. The client's own Account can no longer be picked as their referrer, and in the credit rule a proven return beats a referral that names nobody.

## 2. What was reported

- 2026-09-29: firm leadership opened a new Court Case for a returning client through the DDT Intake Flow, was forced through the referral section, and the saved case read `Attributed_Origin__c` = Referral - Client and `Attribution_Basis__c` = Structured Referral instead of Returning. Of 28 cases ever credited as Structured Referral, 9 belonged to clients with an earlier hired case: 3 clear mis-captures and 6 suspects with a named referrer. This also weakened the evidence used to clear the sales-comp cutover in DDT-902.

## 3. Scope

**In scope**

- `ReferralSourceService.validateCapture` and `gateApplies`, `ReferralSourceController.getCaptureRules`: take the client record and skip the referrer gate for returning clients.
- `AttributionCreditRunner.returningView`: exposes the credit engine's own returning rule.
- `AttributionCreditService`: a referral that names nobody no longer outranks a proven return.
- The `referralPicker` LWC and its five hosts (`referralPickerFlow`, `convertClientModal`, `convertToHiredMatter`, `newPotentialClient`, `inboundContactReferrerPicker`), plus the controllers and services that pass the client record through.

**Out of scope**

- Asking "how did you hear" only once and linking named referrers (DDT-1178).
- Holding the history re-score in DDT-826: at most 9 records, fixable afterwards.

## 4. How it works (background)

The referral picker is one LWC used on five screens. A server rule decides whether a referrer must be named, based on the how-heard answer. The credit engine separately decides the case's origin, and already has a "Returning Match" rule based on identity keys, a minimum gap (`Attribution_Config__mdt.Returning_Min_Gap_Days__c`) and a cold-client window (five years in production).

## 5. Root cause

The picker's gate looked only at the how-heard answer. Nothing asked whether the client had hired the firm before, although the credit engine already answers exactly that. The staff member's only honest-looking way out was the referral box, and the credit engine then believed the structured referral over the return.

## 6. Solution

1. **Ask the credit engine, don't redefine "returning".** The gate calls the engine's own returning rule. Why: two definitions of "returning" would drift apart.
2. **Show one line instead of the form**: "Returning client, last hired" with the date, and a small link to record a referrer anyway for the rare genuine case.
3. **No self-referral.** Picking the client's own Account as Referring Party is refused with a message saying what to do instead.
4. **Credit rule.** With a proven return, a referral that names nobody is passed over. A named referrer, and an Internal referral, still win.
5. **Fix in the shared server rule**, so all five screens change at once.
6. **Records.** Firm leadership's calls, recorded on the ticket: a returning client who arrives through a channel keeps the channel credit and is flagged returning (DDT-824 rule 4).

## 7. Testing

- Sandbox: 397 of 397 tests across 23 test classes. Two attribution test classes run separately showed 76 of 86, with the 10 failures matching the sandbox baseline exactly (sandbox gaps in phone keys and picklist values).
- Break run: with the old `AttributionCreditService`, the 6 new credit-rule tests fail.
- Jest: 46 tests pass across the five picker hosts.
- Production (2026-09-30): two of the three clear records now read Returning. Of the six suspects, two read Returning and four credit their channel (paid search, Google Business Profile, lead vendors) flagged returning.
- The third clear record's last hire was past the five-year window, so it is not Returning by rule. Leadership first left it for intake to re-capture, then on 2026-10-16 decided to leave it as it is.

## 8. Step by step

1. Find the rule that forces a question and every screen that uses it.
2. Reuse the existing engine rule that already answers the question; do not write a second one.
3. Pass the client record into the shared rule so it can check history.
4. Refuse obviously wrong answers (the client as their own referrer).
5. Adjust the credit rule so an empty answer cannot outrank proven history, with a test that fails when reverted.
6. List the affected records and check each after deploy.

## 9. Rollback

Redeploy the 14 classes and 6 LWC bundles from before PR #547. No schema or data changes are involved. Records re-credited while it was live keep their new values until re-scored.

## 10. Known limits

- A client past the cold-client window is not treated as returning and is still asked for a referrer.
- One historic record remains with the client shown as their own referrer, by leadership decision.

## 11. Reuse for similar issues

- Signs of the same problem: staff tick an escape box with notes like "stop" or "n/a". The form is asking a question that does not apply.
- When a screen asks something the system already knows, have the screen ask the system.
- Fix shared rules on the server so every screen that uses them is fixed together.
- In ranking rules, an answer that names nobody should not beat evidence from the data.
