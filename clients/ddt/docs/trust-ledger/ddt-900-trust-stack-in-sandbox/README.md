# Put the trust feature in the development sandbox


| | |
|---|---|
| **Ticket** | DDT-900 |
| **Project** | Trust Ledger |
| **Status** | Done 2026-09-25 (sandbox setup; nothing deployed to production) |
| **Priority** | Medium |
| **Reported by** | Firm leadership, 2026-09-01 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (development sandbox: objects, Apex, LWC, permission sets) |
| **Area** | Development process / rehearsal environment for trust changes |
| **Related** | DDT-894 |
| **Code** | No code change (sandbox deploy of components retrieved from production) |


---


## 1. Summary

The whole trust feature (trust ledger, tie-out screen, their objects, classes and permission sets) had only ever existed in production, so every trust change went live without anyone being able to click it first. The trust stack was deployed into the development sandbox. The real blocker turned out to be that the sandbox had reached its custom object limit; two dead objects were removed from the sandbox only to make room. Both trust screens now load in the sandbox.

## 2. What was reported

- 2026-09-01: trying to stage a large tie-out change somewhere safe, a deploy to the development sandbox failed because the tie-out controller did not exist there. A check showed both trust objects missing from the sandbox and no trust Apex classes at all. That day the change shipped straight to production by decision; this ticket was filed so the choice would be a decision rather than a default.

## 3. Scope

**In scope (sandbox only), 67 components retrieved from production**

- The two trust objects, `Trust_Reconciliation__c` and `Trust_Ledger_Entry__c`.
- Seven trust fields on `Matter__c` and `Expense__c.Net_Expense_Amount__c`.
- Nine Apex classes (including `SettlementRecoveryTriggerHandler`) and two triggers.
- The `trustLedgerPanel` and `trustTieOut` LWCs.
- Four permission sets and the `Trust_Ledger_Amend` custom permission.
- The `Trust_Reconciliation` tab, with admin visibility as in production.
- Removal from the sandbox of `Report_Snapshot__c`, `OLD_Payment__c` and three sandbox-only references to the latter.

**Out of scope**

- Production: nothing changed there.
- The 21 sandbox-only objects still in use were not touched.

## 4. How it works (background)

Salesforce limits how many custom objects an org can hold. A sandbox can drift from production over time; this one carried more custom objects than production because 39 existed only in the sandbox.

## 5. Root cause

The ticket expected dependency wrangling. The real blocker: both trust objects failed with "reached maximum number of custom objects", and roughly 500 further errors cascaded from that. The sandbox had 223 non-namespaced custom objects against production's 199.

## 6. Solution

1. Free two object slots in the sandbox by deleting objects that production had already removed:
   - `Report_Snapshot__c`: 0 records, no references; already gone from production and from git.
   - `OLD_Payment__c`: 0 records, from 2018; already gone from production. Its three sandbox-only references were removed first, each matching what production had already done: the `Custom_Case__c.Amount_Due__c` formula replaced with production's version, the `Custom_Case__c.Total_of_Payments_Made__c` roll-up (always 0) deleted, and an unused lookup on the S-Docs relationship object (0 of 9,954 rows populated) deleted.
   - Deleted with purge so the slots freed at once, after a metadata backup.
2. Deploy the 67 trust components retrieved from production.
3. Add the trust tab, assign the trust permission sets to the developer's sandbox user, and place `trustLedgerPanel` on the sandbox Matter page's Financials tab by hand in App Builder, in the same position as production.

Why delete only those two: they were empty, unreferenced in production, and already removed there, so the sandbox moved closer to production rather than further away.

## 7. Testing

- After the deploy, `TrustLedgerEntryTriggerTest`, `TrustLedgerPanelControllerTest` and `TrustReconciliationControllerTest` passed 69 of 69 in the sandbox.
- The deploy itself ran with no tests (sandbox only). With named tests it was blocked by the 75% per-class coverage bar, because other tests covering those classes fail in the sandbox for unrelated drift.
- Live check: the Trust Reconciliation screen and the Trust Ledger panel both load in the sandbox.

## 8. Step by step

1. Try deploying the feature's components from production to the sandbox and read the first error, not the cascade.
2. If it is the custom object limit, list sandbox-only objects and pick ones with 0 records that production already removed.
3. Remove their references first, matching production's versions, then delete with purge after a metadata backup.
4. Deploy the feature's objects, fields, classes, triggers, components, permission sets and tab.
5. Assign permission sets, place components on pages, run the feature's tests and open each screen.

## 9. Rollback

Not applicable for production. In the sandbox, the trust components can be deleted and the two removed objects restored from the metadata backup.

## 10. Known limits

- The sandbox Matter page was edited by hand and must never be deployed to production.
- Other tests in the sandbox still fail for unrelated drift, so a full test run there is not meaningful.
- The sandbox stays in step with production only if later trust changes are deployed there too.

## 11. Reuse for similar issues

- Signs of the same problem: a sandbox deploy fails with hundreds of errors; look for one root error such as an org limit.
- Clean sandbox-only leftovers that production already removed before raising a limit.
- Any feature that has only ever lived in production should get a rehearsal copy before its next change.
