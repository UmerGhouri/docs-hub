# One Legal Growth Engine label across its ten lead-source tokens


| | |
|---|---|
| **Ticket** | DDT-1117 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-09-25 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-09-25 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex / custom metadata / reports / list views) |
| **Area** | Marketing attribution: lead vendor reporting |
| **Related** | DDT-961, DDT-1180, DDT-826, DDT-816 |
| **Code** | PR #494, PR #496 (deploy record) and PR #516 in the ddt-salesforce repo |


---


## 1. Summary

Legal Growth Engine (LGE) is not one source but a package sold to the firm on one invoice, bundling several lead products. Ten separate tokens were reported apart, so LGE signings could not be counted as one number against its one cost. A new vendor label on the channel dictionary now rolls the ten tokens up to "Legal Growth Engine" while each token still routes as before. Nolo and LegalMatch keep their own labels. A sweep then found nine reports and one list view reading only part of the token set, and fixed them.

## 2. What was reported

- 2026-09-25: roll the ten LGE tokens onto one label on history and on new records; keep Nolo and LegalMatch separate. Rule for later vendors: its own line on the invoice means its own label. A count of LGE signings for any window must return one number, with the window stated.
- 2026-09-25 (follow-up): a list view for PI cases from LGE showed zero. Re-point it, sweep for anything else filtering on the old sub-tokens, report how many were checked and found, and state the window on the view.

## 3. Scope

**In scope**

- New field `Channel_Mapping__mdt.Vendor_Label__c` and 13 dictionary rows: ten tokens to "Legal Growth Engine" (`Legal Growth Engine`, `Legal Growth Engine MVA`, `Abogado DUI`, `Abogado Directory`, `Abogado Traffic`, `FindLaw`, `FindLaw DUI`, `FindLaw Traffic`, `Super Lawyers`, `Ngage Lead`), the two Nolo tokens to "Nolo", and LegalMatch to "LegalMatch".
- `AttributionCreditService.resolve` (final channel branch) and `AttributionCreditServiceTest`.
- New report "LGE and Lead Vendors: Signings by Month".
- List view `PI_Cases_from_LegalGrowthEngine` on Matter, and nine LGE reports found by the sweep.

**Out of scope**

- Changing the tokens themselves or the shared channel bucket "Lead Vendors / Directories", which an outside partner system maps.
- A separate history pass (see section 6).
- Matter records: Matter is not credited by this engine.

## 4. How it works (background)

The engine resolves each record's winning token through `Channel_Mapping__mdt`, which gives a channel bucket. All 26 lead-vendor tokens share the bucket "Lead Vendors / Directories". `Attributed_Source__c` held the bucket, so vendors could not be told apart, and LGE's sub-products were not counted together.

## 5. Root cause

The dictionary had no level between the raw token and the shared bucket. Labelling the sub-products separately bought nothing because they cannot be costed apart, and it hid that the firm's single largest vendor invoice and ten tokens were the same thing.

The zero list view: it filtered on a field that never held the token on Matter (the tokens sit in `Matter__c.Automated_Source__c`, not the how-heard picklist), so it had always shown zero.

## 6. Solution

1. **Roll-up on a new dictionary field,** not on the token (a raw signal, never modified) and not on the bucket (shared by all 26 vendors and read by the partner system). The engine's final channel branch now sets the source to the row's `Vendor_Label__c` when present, else the bucket. `Attributed_Channel__c` is unchanged for every record.
2. **Report** "LGE and Lead Vendors: Signings by Month": hire-date month by attributed source, filtered to the lead-vendor channel, from 2025-10-01. Its description states that the window is the date filter and that LGE's cost figure is monthly.
3. **History left to the DDT-826 re-score.** A backfill script was written and dry-run in production but held; the scheduled DDT-826 re-score stamps the label on every record anyway under its flow-suppression window. Sizes differ by source: the ticket comment says 1,889 Court Cases and 3,110 Leads would move; the deploy doc's dry run counted 1,165 Opportunities for LGE (with 697 Nolo and 27 LegalMatch) out of 2,159, and 1,784 Leads for LGE (1,230 Nolo, 96 LegalMatch) out of 3,149.
4. **List view and sweep (PR #516).** The view now filters `Automated_Source__c` on all ten tokens, hired in the last 365 days. Sweep: 1,182 of 1,227 reports, 65 list views and 49 dashboards checked; nine reports and this one view read a subset of the tokens and were fixed; no dashboard was affected and nothing filtered on the LGE label alone.

Why Nolo stays separate: it is bought separately on its own invoice line and returns far better than LGE (7.2 times against 1.5 times), so folding it in would hide the best vendor inside the worst.

## 7. Testing

- Five new tests in `AttributionCreditServiceTest` (seeded rows plus a live-dictionary assertion over the 13 rows); production validation 63 of 63, 18 of 18 components. A readback guard confirmed the 13 metadata rows changed as intended with no collateral overwrite.
- Sandbox: deployed and proven by readback, report render and backfill rehearsal. Six unrelated test failures there pass in production.
- The re-pointed list view returned 6, the same as a direct query on the same filter and window.
- Not covered: the history move happens only when the DDT-826 re-score runs.

## 8. Step by step

1. Get the business definition: which tokens are one invoice line.
2. Add a label field to the dictionary rather than rewriting tokens or buckets.
3. Set the label on every row of the vendor and its look-alikes (here Nolo and LegalMatch), so each invoice line has one label.
4. Make the engine use the label when present, with a test over the live rows.
5. Build one report with the window in its filter and its description.
6. Sweep reports, list views and dashboards for filters on the old tokens; report checked and found counts.
7. Decide how history moves (a backfill or the next scheduled re-score) and say which.

## 9. Rollback

Not recorded in the ticket. The engine change is guarded by the field: clearing `Vendor_Label__c` on the 13 rows returns the source to the bucket for newly scored records, with no code deploy.

## 10. Known limits

- Until history is re-scored, older records sit under "Lead Vendors / Directories" in the new report and only records credited after the deploy sit under "Legal Growth Engine".
- The lead-vendor cost score stays "Directional", because a share of signings still carry no channel.
- New vendors need their own label row when added.

## 11. Reuse for similar issues

- Signs of the same problem: one vendor's numbers are split across several tokens, or a filtered list view returns exactly zero.
- A filter that matches nothing looks like a true zero; when a view shows zero, check the filter against a direct query.
- Put reporting roll-ups in a separate dictionary field so routing tokens stay untouched.
- Always state the date window next to a cost-per-client figure.
