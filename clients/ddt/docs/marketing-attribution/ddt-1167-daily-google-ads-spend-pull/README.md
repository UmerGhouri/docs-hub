# Pull Google Ads and LSA spend daily into draft spend rows


| | |
|---|---|
| **Ticket** | DDT-1167 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-10-08 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-10-01 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, custom metadata), Google Ads API |
| **Area** | Marketing cost reporting |
| **Related** | DDT-1166, DDT-1168, DDT-1170, DDT-1226, DDT-1272, DDT-959, DDT-962, DDT-964 |
| **Code** | PR #608, PR #614, PR #876 in the ddt-salesforce repo |


---


## 1. Summary

A daily job now reads cost per campaign from Google Ads (paid search) and the Local Services Ads accounts and writes it as draft rows in the Marketing Spend object from DDT-1166. It re-reads the current and previous month each day, so late Google revisions land, and it never touches an approved month: a changed figure goes on an adjustment row instead. It rides the existing minute dispatcher, so it adds no scheduled job. September's two channel totals matched Google to the cent and were approved on 2026-10-07. A later outage on 2026-11-02, caused by a change elsewhere, was fixed the same day.

## 2. What was reported

- 2026-10-01: pull yesterday's cost per campaign from Google Ads and the LSA accounts daily, as drafts under the right channel-month (Paid Search for PPC, LSA for Local Services). Google is the source of truth, never the bookkeeping system. Approved months get an adjustment row. No new scheduled job (production was at 92 of 100). Log every run so a silent stop is visible. Done when September's rows equal the Google Ads screen to the cent, the campaign rows add up, and a broken credential shows as a failed run.

## 3. Scope

**In scope**

- New `MarketingSpendGoogleAdsPull` (Queueable) and its test class.
- `GlobalMinuteScheduler`: one wrapped call at 06:27 Chicago.
- New custom metadata `Marketing_Spend_Source__mdt` with one record per Google account (one PPC, three LSA) and an `Active__c` switch.
- New field `Syncing_Log__c.Process__c` so run summaries can be filtered from other Google logs; read access for the spend permission sets.
- PR #614: rounding fix. PR #876: read Google's answer from the call, not the log.

**Out of scope**

- Meta spend (DDT-1170) and lead vendor cost (DDT-1169).
- Moving the Google Ads developer token out of code (DDT-1226), which the pull then used.

## 4. How it works (background)

All Google Ads calls in Salesforce go through one shared callout method, which also serves the lead ingest and the conversation sync. The shared minute dispatcher hosts recurring jobs; its rules are: wrap each call in try/catch, put new calls first, and pin the timezone (Chicago month boundaries here).

## 5. Root cause

Not applicable for the feature. Two defects were found and fixed along the way:

- **One cent over.** September Paid Search read one cent above Google's account total, because the pull rounded each campaign before adding them up. Google adds the unrounded amounts and rounds once.
- **2026-11-02 outage.** A change under DDT-1490 made the shared callout write a log row only when a call fails. The pull read each account's response from that log row, so on a successful call there was no row and every account failed with "List index out of bounds: 0". That change's test run had not included the pull's tests.

## 6. Solution

1. **Drafts per campaign** under the right channel-month; a campaign that drops out is kept at 0 so rows always add up.
2. **Approved months are never touched.** A different re-read goes on one open adjustment row, updated later, and removed if Google returns to the approved figure.
3. **All or nothing per channel.** A channel is rewritten only when every account feeding it answered, so one LSA timeout cannot overwrite a month or raise a false adjustment.
4. **Every run is logged** (200 with totals, 204 for an empty run, 500 with errors), so a broken credential shows as a failed run.
5. **Round once** (PR #614). The channel total is the summed raw amounts rounded once, equal to Google's total. Campaign rows are floored to the cent and the remaining cents go to the largest dropped fractions, so they still add up exactly; one campaign can show a cent less than Google's campaign view. Firm leadership was told before approving.
6. **Return the response directly** (PR #876). A new raw variant of the shared callout returns the response body; the pull uses it, and the other callers behave as before.

## 7. Testing

- PR #608: 11 tests (drafts, re-read, approved month to adjustment, adjustment updated and removed, broken credential to a 500 naming the accounts, unknown channel refused, window on the 1st, Chicago minute, dispatcher, no source). Full local test run in production because the dispatcher runs every minute.
- First production run (2026-10-05): status 200, 4 monthly and 12 campaign rows; LSA matched Google exactly; Paid Search one cent over.
- PR #614: a new test replays September's real amounts; against the old class it fails (expected the Google figure, got one cent more). After the next run, Paid Search matched Google and the campaign rows added up.
- 2026-10-07: September Paid Search and LSA approved on the 5th-of-month screen, both matching Google to the cent.
- PR #876: 55 of 55 in a full-copy sandbox; on the old code the same tests gave 15 failures with the production error. Only one day's spend was missing, and the next run filled it because each run re-reads both months.
- Not covered: breaking the credential in the usual sandbox, which cannot hold the spend objects; the failed-run path is covered by the unit test.

## 8. Step by step

1. List the source accounts in custom metadata, each mapped to a default channel.
2. Pull cost per campaign for the previous and current month, in the reporting timezone.
3. Write drafts only; route changes to approved months onto an adjustment row.
4. Rewrite a channel only when every account feeding it answered.
5. Add amounts unrounded and round the total once; allocate cents to campaign rows so they add up.
6. Log every run with a status and totals.
7. When changing a shared callout, run the tests of every caller.

## 9. Rollback

Redeploy `MarketingSpendGoogleAdsPull` (and, for PR #876, the shared callout class) from before the change. Turning a source off is a custom metadata switch (`Active__c`). Approved rows are never changed by the pull, so rollback cannot alter them.

## 10. Known limits

- The pull cannot be exercised in the usual sandbox, which is at its custom-object limit.
- A campaign row can be one cent below that campaign's own figure in Google, by design.
- A change to the shared callout can break the pull if its tests are not included in that change's run, as happened on 2026-11-02.

## 11. Reuse for similar issues

- Signs of the same problem: a total off by one cent from the source system. Check whether parts are rounded before adding.
- Never read a call's result from a side-effect such as a log row; take it from the call's return value.
- Re-read recent periods every day so late revisions land, and protect approved periods with adjustment rows.
- Host recurring work on the existing dispatcher instead of adding scheduled jobs when the org is near its limit.
