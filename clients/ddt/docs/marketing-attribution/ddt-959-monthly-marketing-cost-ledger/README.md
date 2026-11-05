# Monthly marketing cost ledger with confidence labels


| | |
|---|---|
| **Ticket** | DDT-959 |
| **Project** | Marketing Attribution |
| **Status** | Done 2026-09-12 |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-09-09 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (attribution data, reports), QuickBooks, Google Ads and LSA platform data |
| **Area** | Marketing budget: cost per signed client |
| **Related** | DDT-826, DDT-824, DDT-574, DDT-961, DDT-960, DDT-962, DDT-964, DDT-1166, DDT-1167 |
| **Code** | No code change |


---


## 1. Summary

Attribution said which channel won each client but not what each channel cost. This ticket built the smallest economic layer on top: a hand-assembled monthly ledger of marketing spend by channel and vendor, joined to signed clients from Salesforce, with a confidence label on every row. August 2026 was assembled and signed off on 2026-09-11 as the baseline month. The work also found that the old campaign cost feed in Salesforce had been frozen since spring, and that many "Unknown" signings were hiding real answers behind a record-provenance flag.

## 2. What was reported

- 2026-09-09: build a monthly cost ledger (month, channel, vendor, media spend, agency fee, credits, signed clients, collected fees, cost per signed client, confidence, decision, evidence). Manual first; no automation or dashboard. Done when one full month covers every major paid line, media-only and all-in return are readable for paid search, LSA is costed from charged leads or account cost, lead vendors are split by vendor or marked "Not usable yet", and at least one budget decision is recorded with its number.

## 3. Scope

**In scope**

- One month (August 2026) across every paid line above the ticket's threshold: LSA, Google paid search, third-party lead providers, paid social, referral fees and agency fees.
- Signed-client counts by attributed channel from Salesforce, and the definition of the "collected fees" column.
- Confidence labels per row: Defensible, Directional, Not usable yet.

**Out of scope**

- Automating cost ingestion (DDT-964, later DDT-1166 and DDT-1167).
- Changing the credit rule. The provenance-flag finding went to DDT-824 as scope input.
- June and July were not levelled; August is the baseline and the comparable run starts there, by firm leadership's decision.

## 4. How it works (background)

Each Court Case carries an attributed channel from the credit engine. Hires per channel per month come from `Hire_Date__c` and `Attributed_Channel__c`. Spend lives outside Salesforce: in the accounting system and in the ad platforms' own billing.

## 5. Root cause

Not a defect: new reporting work. Findings recorded along the way:

- **The Salesforce cost series was frozen.** Campaign snapshot rows were still written on the 1st of each month, but with identical cumulative values since about May, and no Google Ads account was connected. Monthly spend could not be reconstructed from it.
- **Two counting mistakes were caught and corrected.** A first August table used the wrong revenue field (a historical snapshot field instead of the live paid amount) and a UTC month boundary. The UTC boundary dropped a genuine August signing made late on 31 August, Chicago time. The corrected query uses the date field `Hire_Date__c`, which has no timezone.
- **Unknown signings were shadowed.** 37 of 50 August "Unknown" signings carried a "Manually Created" provenance value that outranked the client's how-heard answer, so the real answer was never read. Across June to August, 112 of 140 Unknowns had this shape.

## 6. Solution

1. **Platform beats the books for anything a platform bills.** LSA and paid search are costed from the platforms' own figures; the accounting system lags them.
2. **Charged and counted are separate columns.** Every line carries both, and the reason travels with any difference. Examples: a non-marketing payment booked to the lead-provider account was counted as zero, and a vendor billed twice in one window was levelled to one month. The ledger totals on counted.
3. **Cost per signed client only for month one.** No return figure was published, because August spend bought August signings while August cash came mostly from earlier clients.
4. **"Collected fees" must say which number it is:** the cohort's paid-to-date or cash received in the period. They answer different questions.
5. **A sign-off page instead of a spreadsheet:** one tick per spend line, checked against the books, and a month-level sign-off that stays locked until every line is ticked.
6. **Rule-version column.** Each month records which attribution rule version its counts came from, so the later re-score (DDT-826) shows as a labelled change, not as the business moving.

## 7. Testing

- The eight accounting lines footed exactly to the marketing total in each of June, July and August.
- The corrected August signed-client count and paid total matched the independent figure exactly.
- August cash received cross-checked against accounting income within 1.6%.
- Final August labels: LSA Defensible, paid search Defensible, paid social Directional, lead vendors Not usable yet.
- Budget decision recorded: the lead-vendor line is effectively one vendor, so any lead-vendor spend question is a question about that vendor. Lead vendors cost roughly five times per signed client what LSA does over June to August, after correcting the shadowed Unknowns (an upper bound; down from about 7.2 times before the correction).

## 8. Step by step

1. Pull the month's signed clients by attributed channel using `Hire_Date__c`, not a UTC date-time window.
2. Pull spend from each ad platform for platform-billed lines, and from the books for everything else, at vendor grain where it exists.
3. Record charged and counted separately, with the reason for any difference.
4. Label each row Defensible, Directional or Not usable yet using the ticket's rules.
5. Check that the lines foot to the accounting total.
6. Record the rule version the counts came from.
7. Sign off each line, then the month.

## 9. Rollback

Not applicable: no system change.

## 10. Known limits

- Month one is hand-built; automation is separate work.
- The lead-vendor row stays Not usable yet while many signings sit in Unknown.
- June and July are raw account totals and are not comparable with August.
- Signed counts will move when DDT-826 re-scores history; the rule-version column marks that.

## 11. Reuse for similar issues

- Signs: someone divides spend by signings and gets a surprising ratio. Check the date boundary, the revenue field and the Unknown bucket before believing it.
- Use the platform as the source for platform-billed spend, and keep charged and counted apart.
- Never compare a period's spend with the same period's cash when the cash comes from earlier cohorts.
- Show the query with every number posted, so a reviewer can re-run it.
