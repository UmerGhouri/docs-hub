# Write the sales-comp case source from attribution


| | |
|---|---|
| **Ticket** | DDT-902 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-10-14 (reached production) |
| **Priority** | Medium |
| **Reported by** | Firm leadership, 2026-09-01 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex batch, custom metadata, list view) |
| **Area** | PI sales compensation: case source |
| **Related** | DDT-574, DDT-814, DDT-824, DDT-825, DDT-935, DDT-1083, DDT-1178, DDT-1285 |
| **Code** | PR #467, PR #490, PR #492, PR #515, PR #693, PR #717 and PR #719 in the ddt-salesforce repo |


---


## 1. Summary

PI cases are paid out of a shared sales pool by points, and the points depend on where the case came from. A sales manager typed that source by hand on each Matter. A nightly job now derives it from attribution facts, with a fixed precedence so a referral always beats a paid click and a case found on a defense consult always scores as organic. Hand-set values are flagged Manual and never overwritten. The job shipped switched off, was switched on 2026-09-28 once the referral tier had fired on real records, and was refined on 2026-10-12 and 2026-10-14 so employee referrals and the referral type are handled correctly.

## 2. What was reported

- 2026-09-01: stop a human typing `Matter__c.Sales_Case_Source__c`, the field that decides what a salesperson is paid, and let attribution write it. Do not overwrite a value a manager set. Never write "Arrived Pre-Signed". A case uncovered on a defense consult must score "Found on a Defense Consult" (5 points), not "Paid Ads" (2 points), even when attribution correctly says the traffic was paid.
- 2026-10-08 (reopened): three Matters scored wrongly because intake picked the wrong referral type. Fix them, add a guard so an employee referrer never earns a referral label, and make the referral type follow the referrer record that was picked.

## 3. Scope

**In scope**

- `CompCaseSourceService` (the rule and fact gathering), `CompCaseSourceBatch` (nightly pass over hired PI Matters in a rolling window), a dispatcher branch at 01:37 Chicago on `GlobalMinuteScheduler`, and a two-line hook in `MatterTriggerHandlerNew`.
- New field `Matter__c.Sales_Case_Source_Set_By__c` (`Manual` / `Attribution`) and two `Attribution_Config__mdt` fields: the on/off switch and the lookback window.
- List view "PI Hired: Case Source missing" for Matters the job deliberately leaves blank.
- Referral capture: the referral type now follows the referrer that was picked (`ReferralSourceService`, `ReferralCaptureFlowAction` and the referral panel's controller).

**Out of scope**

- `Sales_Tier__c` and `Sales_Board_Points__c` are formulas over the source and were not touched.
- "Arrived Pre-Signed" stays manual: it means no consult happened, which is not a marketing fact.
- The batch does not skip months already paid out; flagged as its own follow-up.

## 4. How it works (background)

Under the firm's PI sales compensation SOP each PI case has a fixed board value and a point value by source: organic sources (word of mouth, past client, client referral, found on a defense consult) score 5, paid ads 2, medical provider and attorney or partner referrals 1. Attribution measures what the firm bought; compensation measures what the rep created. For a PI case found during a paid defense consult, those two answers deliberately differ.

## 5. Root cause

Not a defect: new feature. The ticket recorded why a straight read of the attributed channel would be wrong: under the old resolver, where a client named a referrer and a paid click also existed, the paid click won 18 of 18 times. After go-live, three Matters scored wrongly because the referral type was typed separately from the referrer and nothing checked that they agreed (a staff member typed as a previous client, a medical provider typed as a business partner).

## 6. Solution

Precedence, highest wins:

| # | Value | Fact that produces it |
|---|---|---|
| 1 | Found on a Defense Consult | The same Account has a defense Court Case dated on or before the Matter's hire date |
| 2 | Past or Current Client | The same Account has an earlier hired Matter |
| 3 | A referral label | The referral type on the touch log, or the curated how-heard dictionary row when no type was recorded |
| 4 | Paid Ads (LSA, PPC, LGE) | The attributed channel is a bucket marked Paid in `Channel_Mapping__mdt` |
| 5 | Word of Mouth or Direct | Nothing above matched |

Key choices and why:

- **Never over a human.** Manual flag means skip. A filled value with no flag is stamped Manual on the first run and left alone, protecting months of hand-filled history. Any human edit to the source stamps Manual.
- **Dictionary fallback for referrals** (approved by firm leadership): only 14 typed referrers existed in a year against 262 PI hires, so without it most real referrals would have scored as Word of Mouth at 5 points.
- **Friend / Family** scores Word of Mouth or Direct, so Client Referral keeps meaning a past client.
- **Employee referrals.** The firm pays a bounty on them, so an employee referrer never earns a referral label; if only the residual remains, the job writes nothing and the Matter appears in the list view for the sales manager (PR #490, PR #693). Narrowed on 2026-10-13: when the referring employee is also the Matter's Intake Owner, it scores Word of Mouth or Direct, 5 points (PR #717).
- **Root fix** (PR #719): the referral type follows the referrer record that was picked (a staff user is Internal, a medical-provider Account is Medical Provider, a client with an earlier hire is Previous Client, a referring attorney is Outside Attorney). Intake picks a type only when creating a new referrer.
- **Shipped dark.** A metadata checkbox gated every entry point until the cutover gate was met.

## 7. Testing

- 27 service tests and 6 batch tests at build, covering every precedence branch, every flag state, the switch and a 200-Matter bulk pass; a test walks all 56 fact combinations to prove "Arrived Pre-Signed" can never be returned. Production check-only validation: 52 tests, 0 failed.
- The seven done-when checks, closed 2026-10-14:
  1. Runs: 3 Matters on 2026-09-28 and 5 on 2026-10-01 hired in the last 7 days carried an Attribution-set source.
  2. Defense-consult override proven in the sandbox (5 points); 11 live Matters carried it at close.
  3. Referral beats a paid click: proven in the sandbox and on a live Matter.
  4. A Manual value survived a run; flipped to Attribution, it was rewritten.
  5. "Arrived Pre-Signed" written by the job: 0.
  6. Pool before and after the first run explained Matter by Matter (test Matters were removed).
  7. Cutover gate: the structured referral tier had fired 16 times on real records by 2026-09-25.
- Root fix proven in the sandbox: picking a staff member gave Internal and a medical provider gave Medical Provider without anyone choosing.

## 8. Step by step

1. Add a "set by" companion field so the job can tell its own writes from a person's.
2. Write the rule as a pure function of gathered facts, with precedence in one place.
3. Gate every entry point on a metadata switch, deployed off.
4. Host the nightly pass on the existing dispatcher, not a new scheduled job.
5. On the first run, flag existing hand-filled values Manual without changing them.
6. Switch on only after the input signal has fired on real records, then run the done-when checks on live data.
7. Give anything the job deliberately leaves blank a list view for a human.

## 9. Rollback

Untick the switch in `Attribution_Config__mdt`; the job then writes nothing. Values already written stay, flagged Attribution, and can be corrected by hand (which flags them Manual). Code changes can be reverted by PR.

## 10. Known limits

- Re-runs inside the lookback window can rewrite a month already paid out; not yet guarded.
- The dictionary fallback depends on the curated how-heard rows being correct.
- One Matter was relabelled by hand to match the narrowed employee rule and pinned Manual.

## 11. Reuse for similar issues

- Signs: a pay or billing field is typed by hand from facts the system already holds.
- When automating a field people already fill in, add a "set by" flag and never overwrite Manual.
- Write the business override (here, defense consult beats paid) as an explicit rule above the generic answer, and test it with a record built to tempt the wrong answer.
- Derive a category from the record picked, not from a second, separate pick that can disagree with it.
- Ship dark behind a switch and turn it on against a measured gate, not a date.
