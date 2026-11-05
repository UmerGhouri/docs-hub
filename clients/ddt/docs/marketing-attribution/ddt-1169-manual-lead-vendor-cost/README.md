# Enter lead vendor cost by hand each month


| | |
|---|---|
| **Ticket** | DDT-1169 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-10-15 (reached production) |
| **Priority** | Medium |
| **Reported by** | Firm leadership, 2026-10-01 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, picklist, permission set) |
| **Area** | Marketing cost reporting |
| **Related** | DDT-1166, DDT-1168, DDT-1344, DDT-810, DDT-964 |
| **Code** | PR #700 in the ddt-salesforce repo |


---


## 1. Summary

Lead vendors with no ad API (Martindale-Avvo, Nolo, LegalMatch) were meant to be drafted monthly from QuickBooks into the Marketing Spend rows. There was no QuickBooks connection, and all three vendors post to one QuickBooks account, so firm leadership re-scoped the ticket on 2026-10-12 to manual entry. A `Manual` source value was added, the Google pull now skips manual rows entirely, and the Approver permission set can create the vendor rows from the record page. September was entered, approved and matched QuickBooks to the cent.

## 2. What was reported

- 2026-10-01: draft lead vendor cost monthly from QuickBooks as one row per vendor under the Lead Vendors / Directories channel, mapping vendor to QuickBooks account in custom metadata. Done when the month's vendor rows equal the QuickBooks bills and approving them shows Lead Vendors' cost per signed client on the 5th-of-month screen.
- 2026-10-12 (re-scope): no QuickBooks connection; make manual entry work. Add a Manual source, make sure nothing overwrites a manual row or deletes its vendor rows, and allow vendor rows to be created from the record page.

## 3. Scope

**In scope**

- `Marketing_Spend__c.Source__c`: new value Manual (restricted picklist: API, QuickBooks, Manual).
- `MarketingSpendGoogleAdsPull`: skip any row whose Source is Manual, whatever the channel, and log the skips.
- `Marketing_Spend_Approver`: create and edit on vendor (campaign) rows, edit on the parent's Charged and Source and on the vendor rows' Charged and name.
- Tests in `MarketingSpendGoogleAdsPullTest` and `MarketingSpendScreenStampTest`.

**Out of scope**

- A QuickBooks integration.
- A simpler one-step entry screen, moved to DDT-1344 after leadership found the manual steps cumbersome.

## 4. How it works (background)

Each month has one `Marketing_Spend__c` row per channel, with campaign rows under it that must add up to Charged before approval (DDT-1166). The Google pull (DDT-1167) writes Paid Search and LSA drafts daily, and a nightly job stamps signed-client counts.

## 5. Root cause

Not applicable: the original data source was unavailable. The re-scope needed a guarantee that automated jobs would not overwrite hand-typed numbers.

## 6. Solution

1. **Manual source value.** Marks a row as typed by a person.
2. **Pull skips Manual rows.** No draft, no vendor rows written or zeroed, and no adjustment, even on an approved month. The pull already only touched rows carrying its own key, so a hand-typed vendor row is never matched or zeroed even under an API draft. Skips appear in the run log.
3. **Nightly stamp unchanged.** It never wrote Charged.
4. **Record-page entry.** The Campaign Rows related list with New was already on the layout; the Approver set was read-only, so it gained create and edit rights. Why: the Approver set is the documented route, rather than relying on admin access.
5. **Entry steps** given to leadership: set Source to Manual and type the month total in Charged, add one row per vendor, tick Approved (refused unless the rows add up). Add vendor rows only on Manual months, or that month's total breaks.

## 7. Testing

- Production check-only (the usual sandbox cannot hold the spend objects): 42 of 42 across four test classes.
- An end-to-end test, as an Approver user, creates three vendor rows, marks the month Manual, approves it, runs the nightly stamp, and checks Charged, approval and the vendor rows are unchanged. It also checks the Approver's create and edit permissions directly, because Apex in a test ignores permissions.
- Pull tests: a manual month is left exactly as typed while API rows beside it are written; an approved manual month gets no adjustment; a hand-typed row under an API draft keeps its amount. With both guards removed, all three new pull tests fail.
- Live (2026-10-15): leadership entered September with three vendor rows, approved it, and the total matched QuickBooks to the cent; the screen then showed Lead Vendors' cost per signed client.

## 8. Step by step

1. Add a source value that marks hand-entered rows.
2. Make every automated writer skip those rows, and log the skips.
3. Make sure automated writers only touch rows carrying their own key.
4. Grant create and edit to the approving permission set, not by profile.
5. Write a test that enters, approves and then runs every nightly job, asserting nothing changed.

## 9. Rollback

Redeploy `MarketingSpendGoogleAdsPull` and the Approver permission set from before PR #700. Leave the Manual picklist value: existing manual rows use it.

## 10. Known limits

- QuickBooks gives one combined line for all three vendors, so the split between vendors relies on the vendor bills or on assuming the previous month's pattern; the total is what matches.
- Delete stays off for vendor rows, because validation rules do not run on delete; a mistyped row is set to 0 before approval or removed by an admin.
- The Approver can also edit Charged on an API draft; approval still locks the row.
- Manual entry is several steps per month (follow-up DDT-1344).

## 11. Reuse for similar issues

- When a planned integration is unavailable, a manual-entry path can ship first if every automated writer is taught to leave hand-entered data alone.
- Key automated rows so a job can never match or delete rows it did not create.
- Test permissions explicitly when the test runs as code that would otherwise ignore them.
