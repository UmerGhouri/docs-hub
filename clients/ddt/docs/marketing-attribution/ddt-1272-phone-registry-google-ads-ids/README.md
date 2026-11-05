# Tie each tracking-number row to its Google Ads campaign by ID


| | |
|---|---|
| **Ticket** | DDT-1272 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-10-16 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-10-08 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex Queueable, custom fields, list view, permission set, layout), Google Ads API, CallRail, the firm's landing pages, RingCentral |
| **Area** | Phone attribution: the tracking-number registry |
| **Related** | DDT-816, DDT-1167, DDT-1286, DDT-1299, DDT-1326 |
| **Code** | PR #704, #711, #713, #720 and #809 in the ddt-salesforce repo |


---


## 1. Summary

The phone registry (`Phone_Tracking_Number__c`) maps each phone line to a marketing campaign, but only by a hand-typed name, so it could not tell when a campaign, its numbers or its landing pages changed. One live number had been marked inactive while its Google Ads campaign was still spending, and its calls landed Unclassified. Each registry row now carries the Google Ads customer ID and campaign ID. A daily read-only job reads Google Ads by those IDs, CallRail's tracker list and the live landing pages, and flags mismatches on the row. It never fixes anything. Two follow-ups made it follow landing-page redirects and ignore placeholder text inside form fields.

## 2. What was reported

- 2026-10-08: firm leadership asked for a system that ties the registry values in Salesforce to the actual Google Ads campaign, because campaign names and assets change. The trigger was one tracking number marked inactive on 2026-09-14 while it was still printed on the landing page of an enabled, spending campaign; its leads landed Unclassified and nobody could tell. The ticket asked to store stable IDs, refresh daily from the Google Ads API, read the landing pages for their phone numbers, and flag (not fix) mismatches, with no new scheduled job.

## 3. Scope

**In scope**

- 11 new fields on `Phone_Tracking_Number__c`: 3 entered by a person (`Google_Ads_Customer_Id__c`, `Google_Ads_Campaign_Id__c`, `Google_Ads_Call_Asset_Id__c`) and 8 written by the job (campaign name and status, live numbers, final URLs, CallRail tracker numbers, mismatch text, mismatch count, last checked).
- `PhoneRegistryAdsRefresh` (new Queueable with callouts) and its test class.
- `MarketingSpendGoogleAdsPull`: launches the refresh after its own daily run, and now catches its own failure so the refresh still launches.
- The `Registry_Mismatches` list view, the `Phone_Registry_Google_Ads` permission set, a "Google Ads (DDT-1272)" section on the registry layout (PR #713), and a remote site for the second landing-page host.
- A one-time script that filled the IDs on the 7 Paid Search rows.

**Out of scope**

- Changing any campaign: the Google Ads side is read-only. Swapping a call asset in Google Ads is a click in the UI, handled by firm leadership.
- Meta and LSA: "same idea later, Google first". The 3 LSA rows were left out because LSA campaigns sit in separate accounts with no call assets.
- The failure alarm for the spend pull: DDT-1326, deployed first.

## 4. How it works (background)

A Paid Search call travels like this: the number on the ad is a CallRail tracking number, CallRail forwards it to a RingCentral line, and the registry maps that RingCentral line to a Campaign. So registry rows are the destination lines, not the ad numbers. This was found on 2026-10-08 and explains why some ad numbers had no row of their own: they forward to lines that are registered.

The daily Google Ads spend pull (`MarketingSpendGoogleAdsPull`, from DDT-1167) already ran every morning at 06:27 Chicago time through the shared dispatcher. Production was at its scheduled-job cap, so the new work rides on that run.

## 5. Root cause

The registry held only a label typed by a person. Nothing linked it to the campaign, its call assets or its landing pages, so a rename, a reused number or a wrongly deactivated row went unnoticed. Evidence from the read-only check of 2026-10-08/09 against Google Ads, CallRail and production:

- One campaign's ad groups carried another campaign's CallRail call asset, so its calls were credited to the other campaign (September leads: 104 against 3 by `Automated_Source__c`).
- That campaign's own CallRail tracker, created in February 2026, was on no ad, and its registry row was the inactive one.
- One RingCentral line was the destination of three CallRail trackers, all credited to one campaign.
- A call asset on a long-paused ad group was not any active CallRail tracker (most likely a retired tracker).

## 6. Solution

**The join (PR #704).** Rows carry the Google Ads customer ID and campaign ID; names are labels, IDs are the join. Each morning, right after the spend pull, `PhoneRegistryAdsRefresh` reads each campaign by ID, reads CallRail's tracker list, fetches the landing pages, counts recent inbound calls per row, and writes what it found and any mismatch codes onto the row. Findings that belong to no row go into the run's `Syncing_Log__c` row.

Mismatch codes written to `Registry_Mismatch__c` include:

| Code | Meaning |
|---|---|
| `INACTIVE_TAKING_CALLS` | Inactive row with inbound calls in the last 7 days |
| `INACTIVE_ON_LIVE_CAMPAIGN` | Inactive row whose campaign is enabled and serving |
| `INACTIVE_NUMBER_LIVE` | Inactive row whose number (or a tracker forwarding to it) is on a live ad or landing page |
| `CAMPAIGN_NOT_SERVING` / `CAMPAIGN_NOT_FOUND` | Active row whose campaign is paused, removed, ended or unknown |
| `LABEL_DIFFERS` | The registry label differs from Google's current name; the row still matches by ID |
| `AD_NUMBER_OTHER_LINE` | A number on this campaign's ads rings a different registry line |
| `AD_NUMBER_UNROUTED` / `LANDING_NUMBER_UNROUTED` | A number on the ads or landing page rings no registry line |
| `MULTIPLE_TRACKERS` | More than one active CallRail tracker forwards to this line |

Why these choices:

- **Flag, never fix:** a wrong automatic fix would move credit silently; a person decides.
- **No new scheduled job:** the refresh is enqueued from the spend pull, after a check that there is queueable headroom.
- **Failures are never silent, and never look like "everything is wrong":** if one source fails (a Google account, CallRail, one page), only that source's checks are skipped and yesterday's values and flags are kept, for up to 3 days. A Google Ads account is used only if all of its queries answered; a CallRail list is used only if every page answered.
- **Budgets:** the run measures callout time and count, and stops starting callouts before the platform limits.

**Redirects (PR #720).** The first production run skipped a landing page that answered with a permanent redirect. The job now follows up to 3 redirects, over https, to the firm's own website hosts only. A redirect it does not follow is noted and the page counts as unread, so carried flags stand.

**Placeholder text (PR #809).** The check flagged a reserved fictional example number as "rings no line" on three pages. It was the grey example text inside the phone field of the PPC form, not a printed number. The page scan now reads only `tel:` links and visible text, never other attributes, comments, scripts or styles, and ignores the reserved fictional number range.

## 7. Testing

- The done-when conditions are tests: `aRenamedRegistryLabelStillMatchesByIdAndIsFlagged` and `deactivatingALiveRowIsFlaggedOnTheNextRun`.
- Validated with production check-only deploys (the partial-copy sandbox lacks the spend objects): 86 tests at merge; `PhoneRegistryAdsRefresh` at 97% coverage. Nine deliberately broken copies (for example, joining by name instead of ID, or never expiring carried flags) each failed as they should.
- PR #720 added 6 redirect tests; a broken copy failed as expected. PR #809 added 7 page-scanning tests, using fake numbers only.
- Live: the first run (2026-10-14) checked all 7 Paid Search rows against Google Ads, CallRail and the pages. On 2026-10-16 the run read 51 rows, linked 7 by campaign ID and read 5 landing pages with no errors. After PR #809, the 2026-10-24 run no longer flagged the placeholder on the three rows.

## 8. Step by step

1. Map how a call reaches the registry (ad number → forwarding service → destination line) before deciding a missing row is a bug.
2. Add stable ID fields to the registry; keep names as labels.
3. Fill the IDs with a dry-run-first script, cross-checked against the live ad platform.
4. Run a daily read-only job from an existing scheduled run; read the ad platform by ID, the forwarding service's tracker list and the landing pages.
5. Write mismatch codes onto each row and expose them in a list view; never change the source systems.
6. Grant the new fields through a permission set and add them to the page layout.
7. Check the first runs and tune the page reader (redirects, placeholder text).

## 9. Rollback

Redeploy `PhoneRegistryAdsRefresh` and `MarketingSpendGoogleAdsPull` from before the change, or remove the enqueue from the spend pull. The fields and flags are additive and can stay. For PR #809 alone, redeploying the previous class only brings back the placeholder flags.

## 10. Known limits

- A page fetch only sees numbers in the HTML; a number swapped in by JavaScript shows as the page's default number.
- Only the firm's own landing-page hosts are fetched, and only the first 300,000 characters of a page.
- If a failed spend pull is caught, the job ends Completed, so DDT-1326's alarm watches the failure row instead.
- Flags raised and left for people to resolve on 2026-10-16: two inactive rows still taking calls (both reactivated by firm leadership on 2026-10-22), lines receiving more than one CallRail tracker (one of them handled in DDT-1286), and a paused campaign still serving.

## 11. Reuse for similar issues

- Signs of the same problem: calls land Unclassified, a "retired" number still rings, or one campaign gets credit for another's calls. Compare the registry with the ad platform by ID, not by name.
- Pattern: store the external system's stable IDs, refresh daily read-only, and flag mismatches for a person rather than auto-correcting.
- When scraping pages for phone numbers, read only `tel:` links and visible text; skip form placeholders and ignore reserved fictional numbers.
- Make every partial read explicit: keep yesterday's findings for a bounded time instead of clearing them when a source did not answer.
