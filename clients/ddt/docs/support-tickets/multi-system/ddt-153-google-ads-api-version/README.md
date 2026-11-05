# Google Ads lead and conversion sync failing on a retired API version

| | |
|---|---|
| **Ticket** | DDT-153 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-11 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-06-11 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex), Google Ads API |
| **Area** | Marketing attribution: pulling Google Ads and Local Services leads and conversions into Salesforce |
| **Related** | DDT-1226 (later change to how the same class authenticates) |
| **Code** | No pull request. The change was deployed straight to production and reached git later, in a bulk backfill commit dated 2026-07-03 |

---

## 1. Summary

The Google Ads sync stopped working because the Apex class that calls Google was still pinned to **Google Ads API v20**, which Google had switched off. Every call came back `UNSUPPORTED_VERSION`. The fix moved the class to **v24**, and the scheduled sync jobs completed again the same day.

## 2. What was reported

On 2026-06-11 the Google Ads lead and conversion sync was failing on every call with `UNSUPPORTED_VERSION`. Leads and conversion events from Google Ads had stopped flowing into Salesforce, so paid-traffic attribution and follow-up data were being lost for as long as it stayed broken.

How long it had been failing before it was noticed is not recorded in the ticket.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `GoogleAdsSyncService` | Apex class | `API_VERSION` constant changed from `v20` to `v24` |

**Out of scope**

- **Backfilling the leads and conversions missed while the sync was down.** The ticket asked for it; whether it was done is not recorded in the ticket.
- **An alert on `UNSUPPORTED_VERSION` or deprecation warnings, and a quarterly version review.** Also asked for in the ticket; not recorded as done.

## 4. How it works (background)

```
Scheduled Apex job
  -> GoogleAdsSyncService builds a Google Ads query (GAQL)
  -> sends it to the Google Ads searchStream endpoint, with the API version in the path
  -> parses Local Services leads and conversations from the response
  -> writes them to Salesforce Leads
```

The API version lives in one constant at the top of the class, so the whole class moves version together:

```apex
private static final String API_VERSION = 'v24';
// ...
String url = NAMED_CREDENTIAL + '/' + API_VERSION + '/customers/' + customerId + '/googleAds:searchStream';
```

(That second line is today's form. In June 2026 the class built the URL from a literal host rather than a Named Credential; DDT-1226 changed that later.)

**The Google rule behind this:** Google Ads releases a new API version several times a year and turns old ones off on a published date. Before the cutoff, calls still work. After it, they fail outright with `UNSUPPORTED_VERSION`. There is no grace period at the point of failure.

## 5. Root cause

`GoogleAdsSyncService` had `API_VERSION = 'v20'`, and Google had retired v20. Git confirms the before and after: the version committed on 2026-05-15 reads `v20`, and the next version of the file in git (the 2026-07-03 backfill of production state) reads `v24`.

Nothing in the class itself had changed. The version it depended on expired underneath it.

## 6. Solution

The fix comment on the ticket: Google blocked v20; `GoogleAdsSyncService` was updated to v24, deployed to production, and the scheduled jobs were verified to be completing successfully.

```apex
// before
private static final String API_VERSION = 'v20';
// after
private static final String API_VERSION = 'v24';
```

Why it's done this way:

- **One constant, one edit.** Every request the class makes reads the version from `API_VERSION`, so there is no second hard-coded version to miss inside the class.
- **Check for other callers too.** Today a second class, `PhoneRegistryAdsRefresh`, also calls the Google Ads API and has `v24` written directly into its endpoint constant. The next version bump has to change both.

Whether any request or response shapes needed changes between v20 and v24 is not recorded in the ticket. The committed diff of the class from v20 to v24 is mixed in with other changes in the bulk backfill commit, so it cannot be read as this fix on its own.

## 7. Testing

- **Live check:** after the deploy, the scheduled sync jobs were checked and were completing successfully (from the fix comment on the ticket). No counts were recorded.
- **Apex tests:** which tests were run with the deploy is not recorded in the ticket.

**Not covered:** no automated test can catch this. A unit test uses mocked callouts, so it passes whatever version string the class sends. Only a real call to Google fails.

## 8. Step by step

1. **Confirm the error.** Look at the sync job's failure output or debug log for `UNSUPPORTED_VERSION` from Google Ads.
2. **Find every place the version is set.** Search the repo for the Google Ads host and for version strings such as `v20` or `API_VERSION`. Today that means `GoogleAdsSyncService` and `PhoneRegistryAdsRefresh`.
3. **Choose the target version** from Google's Ads API release notes: the current version that is not itself close to its sunset date.
4. **Read Google's migration notes** between the old and new version for removed or renamed fields used in the class's GAQL queries and response parsing.
5. **Change the version** in each place found in step 2.
6. **Validate against production** with the class's own tests, get the change approved, then deploy.
7. **Check the next scheduled run** completes without errors, and that new Google Ads leads are arriving in Salesforce.
8. **Open a pull request the same day** so git has a record of what is now in production.

## 9. Rollback

Not applicable: rolling back would put the class back on a version Google no longer accepts, so the sync would fail again. If the new version breaks something, fix forward or move to a different supported version.

## 10. Known limits

- **It will happen again.** v24 will also be retired. Unless someone moves the class before Google's cutoff date, the sync fails the same way.
- **No alert exists for it** in the sources. The failure is noticed when someone notices missing leads.
- **Data lost during an outage stays lost** unless someone backfills it, and no backfill is recorded for this outage.

## 11. Reuse for similar issues

**How to recognise one:** an integration that worked for months suddenly fails on every call, with an error naming the API version (`UNSUPPORTED_VERSION`, "version deprecated", "sunset").

- **Look for the version string first**, before suspecting credentials or code. Search the repo for it and fix every occurrence, not just the first one you find.
- **Keep the version in one constant per class**, and know how many classes call the same API.
- **Put the sunset date in a calendar** when you upgrade, so the next move happens before the cutoff rather than after it.
- **Record the outage window.** Reporting needs to know which dates are missing data, whether or not it can be backfilled.
- **Same-day pull request.** A change deployed straight to production that only reaches git in a later bulk backfill is hard to tell apart from the other changes in that commit, as this one was.
