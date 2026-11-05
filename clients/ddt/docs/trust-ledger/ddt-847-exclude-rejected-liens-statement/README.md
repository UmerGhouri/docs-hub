# Keep rejected liens off the client settlement statement


| | |
|---|---|
| **Ticket** | DDT-847 |
| **Project** | Trust Ledger |
| **Status** | Live since 2026-08-27 (reached production) |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-08-26 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex / LWC / Visualforce PDF) |
| **Area** | Settlement statement: medical bills section |
| **Related** | None recorded |
| **Code** | PR #254 in the ddt-salesforce repo |


---


## 1. Summary

Liens whose Negotiation Status is "Rejected" printed on the client's settlement statement as medical bills of $0.00, which confused clients. A rejected lien means the provider is not claiming against the settlement, or the bill was covered by other insurance. The statement now leaves rejected liens out of the medical bills list and its totals, in both the on-screen statement and the PDF, while the lien records stay untouched in Salesforce.

## 2. What was reported

- 2026-08-26: a staff member asked (via Slack) that rejected liens stop appearing on the settlement statement. Requirements: exclude `Negotiation_Status__c = "Rejected"`; never delete or change the lien; keep the internal "Total Savings via negotiation" figure as it is; make sure medical bills, costs and net figures still add up; no change for statements with no rejected liens; a clean $0.00 with no empty rows when every lien is rejected.

## 3. Scope

**In scope**

- `SettlementStatementController`: `getStatementData` flags rejected liens and leaves them out of the liens total.
- `SettlementStatementPdfController`: skips flagged lines in the PDF.
- `lwc/settlementStatement` (JS, HTML, CSS): skips flagged lines in the statement preview and tags them in the internal panel.
- Test classes `SettlementStatementControllerTest`, `SettlementStatementPdfControllerTest` and the jest suite.

**Out of scope**

- `Lien__c` records: never written, never deleted.
- The Liens related list and "Total Savings via negotiation", which run off `LienManagerController` and keep counting rejected liens, as requested.
- The `Matter__c.Total_Liens__c` roll-up (see section 10).

## 4. How it works (background)

`getStatementData` is the single source for the on-screen statement, the statement preview, the finalize preview and the PDF. Net to client and the PDF's medical bills subtotal all read the one liens total it returns, so changing it there keeps every view in agreement.

## 5. Root cause

The statement listed every lien on the matter with no filter on negotiation status. A rejected lien carries a final negotiated amount of zero, so it printed as "reduced from" its original amount to $0.00.

## 6. Solution

- Each lien row gets an `excludedFromStatement` flag when its status is Rejected, and flagged rows add nothing to `totalLiens`. Deciding this once in `getStatementData` means no view can disagree about the number.
- The PDF and the LWC statement lines skip flagged rows.
- When every lien is rejected, the "has liens" flags go false, so medical bills prints $0.00 with no empty sub-list.
- The internal Lien Deductions panel still lists every lien, with rejected ones tagged "rejected — off statement" and a one-line note, so a case manager can see why the client statement is a line short.
- Rejected liens no longer accept a calculator discount, because modelling a reduction on a bill nobody is claiming would move the projected net for no reason. The table cells stay so column counts do not change.

Financial effect at release: none. Both rejected liens in production already had a zero final amount, so only the $0.00 rows disappeared.

## 7. Testing

- Apex: 93 of 93 passing in a production validate-only run, including 5 new tests (rejected lien held off totals but still returned; all rejected deducts nothing; no-rejected regression; PDF prints only the kept lien; PDF prints $0.00 with no rows when all are rejected).
- jest: 15 of 15, 4 new.
- No sandbox could compile this code (both were missing trust fields and objects), so production validate-only was the only test path. Production was diffed against `main` first: no drift in the settlement statement components.
- Live check 2026-08-27 on the example matter: the rejected lien no longer prints, it still shows on the Liens related list, and the medical bills total is unchanged.

## 8. Step by step

1. Find which code produces the statement (here one Apex method feeds the screen and the PDF).
2. Add an exclusion flag per lien row for the rejected status and leave flagged rows out of the total.
3. Skip flagged rows in every renderer (LWC and PDF).
4. Handle the "all excluded" case so no empty heading or list prints.
5. Keep internal views showing all records, with a visible tag.
6. Test totals with none, some and all liens rejected, then check a real matter.

## 9. Rollback

Revert PR #254 and redeploy the two controllers and the LWC bundle. No data was changed.

## 10. Known limits

- `Matter__c.Total_Liens__c` is a roll-up with no status filter, and the record page's net-to-client formula subtracts it. It agrees with the statement today only because every rejected lien has a zero final amount. A future rejected lien with a blank final amount would be excluded by the statement but still deducted on the record page. Filtering the roll-up was left as a separate decision because it changes the field's meaning in reports.

## 11. Reuse for similar issues

- Signs of a similar request: a client-facing document shows rows that are internally meaningful but zero or irrelevant to the client.
- Exclude in the one method that feeds every view, not in each renderer's template alone, so totals cannot drift apart.
- Keep the excluded rows visible internally with a reason tag.
