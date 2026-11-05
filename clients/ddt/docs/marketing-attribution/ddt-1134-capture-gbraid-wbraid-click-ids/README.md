# Capture Google's iPhone click ids (gbraid, wbraid) in the firm's own cookies


| | |
|---|---|
| **Ticket** | DDT-1134 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-11-02 (reached production) |
| **Priority** | Medium |
| **Reported by** | Firm leadership, 2026-09-26 |
| **Resolved by** | Support team |
| **Systems** | Firm website (Cloudflare Workers), Salesforce (Lead fields) |
| **Area** | Website attribution: Google click ids |
| **Related** | DDT-1133, DDT-1131, DDT-1132, DDT-820, DDT-821, DDT-1332, DDT-1432, DDT-1172 |
| **Code** | PR #291 in the driver-defense-site repo (supersedes PR #215, closed) |


---


## 1. Summary

On some iPhone ad clicks Google sends a `gbraid` or `wbraid` instead of a `gclid`. The website's own attribution cookies only kept `gclid` and `fbclid`, so those clicks reached Salesforce through the firm's own path with no Google click id. The only copy sat inside a vendor package (Heeet) that the firm planned to remove. The site now stores both ids inside the existing `ddt_first_touch` and `ddt_last_touch` cookies and writes them to `GBRAID__c` and `WBRAID__c` on the Lead. A week after deploy, the firm's capture saw a gbraid on exactly the same 13 submissions as the vendor's cookie.

## 2. What was reported

- 2026-09-26: capture gbraid and wbraid in the firm's own cookies and send them to Salesforce, behind a switch that ships off until DDT-1133 has created the fields and the website's integration user can write them. Done when (1) a live test visit with a test gbraid creates a Lead with that value in `GBRAID__c`, (2) a normal gclid still fills `GCLID__c`, and (3) a week after deploy the audit log from DDT-1131 shows the firm's gbraid flag roughly matching the vendor's.

## 3. Scope

**In scope**

- Website tracking code: read `gbraid` and `wbraid` from the address, store them in both touch cookies, count either as a campaign signal; carry them in the touch and snapshot types; read them back on the server.
- Building the Lead payload: send `GBRAID__c` and `WBRAID__c` when present, only when both click-id switches are on.
- The click-id retry list used when Salesforce rejects a field (the bug fixed in PR #291).
- The audit line from DDT-1131: `has_gbraid` and `has_wbraid` from the firm's cookies.
- The five form handlers pass the switch through.

**Out of scope**

- Creating the Salesforce fields and access (DDT-1133).
- The first-touch fallback for click ids (DDT-820) and the server hand-back of cookies (DDT-1132).
- The booking button path, which forwards only gclid and fbclid; noted as a separate lead path.

## 4. How it works (background)

When a visitor lands from a campaign, page code writes the touch cookies. When a form is submitted, the server reads the cookies and creates a Lead in Salesforce. If Salesforce rejects click-id fields (for example the integration user cannot edit them), the server drops those fields and retries so the lead still lands. The Google offline-conversion upload in Salesforce needs a click id on the record to tell Google about hires.

## 5. Root cause

The cookies were written with `gclid` and `fbclid` only, so a gbraid-only click carried no Google id through the firm's path. A measurement before the work found, among leads since 2026-06-01, 28 that had a vendor-captured gbraid and no `GCLID__c`.

## 6. Solution

1. **Keep the same cookies.** New fields go inside the existing `ddt_` cookies, not new cookie names, so the server hand-back in DDT-1132 carries them automatically.
2. **Either id counts as a campaign signal.** Without that, a direct visit carrying only a gbraid would write no cookie at all, which is the exact click being targeted.
3. **A second switch, shipped off** (PR #215). Sending a field the integration user cannot edit fails the whole Lead insert, so capture started at merge and only the Salesforce write waited.
4. **Retry-list fix** (PR #291). The retry path that drops rejected click-id fields did not list the two new fields, so a rejected braid field would have failed the whole lead. Both were added, with tests.
5. **Switch on** in the same PR, after the Salesforce side confirmed on 2026-09-30 that the website user could edit both fields. Merged and deployed 2026-10-26.

## 7. Testing

- PR #291: 708 tests passed; type check and build clean. Removing either braid name from the retry list fails both new rejection tests; removing the reader lines fails both reader tests.
- Done-when 1 and 2 (2026-10-28, live site, fresh browsers): the gbraid test Lead had `GBRAID__c` equal to the test value and `WBRAID__c` empty (read back in production on 2026-10-29); the gclid test Lead had `GCLID__c` filled. Test Leads were named TEST with fictional contact details.
- Done-when 3 (2026-11-02): 56 audit lines since the deploy. The firm's cookies and the vendor's cookie both saw a gbraid on the same 13 submissions and neither on the other 43. One of the 13 is probably the test Lead. No submission carried a wbraid that week.
- Not covered: whether Safari's link tracking protection strips gbraid in private browsing.

## 8. Step by step

1. Add the new ids to the existing cookies and types; count them as a campaign signal.
2. Add the Salesforce fields and confirm the integration user can edit them in production.
3. Add the fields to the payload behind a switch, and to any retry list that drops rejected fields.
4. Test that a rejected new field still produces a lead.
5. Turn the switch on, deploy, and test on the live site with a fresh browser; read the Lead record, not the browser.
6. After a week, compare the new capture against an independent source.

## 9. Rollback

Set the braid write switch back to off: the Salesforce write stops and capture in the cookies continues. Reverting PR #291 removes the capture as well.

## 10. Known limits

- A rejection naming one click-id field sheds all of them on the retry, because Salesforce reports fields one at a time; a braid rejection would also drop a valid gclid and UTMs on that lead. This predates the ticket.
- The comparison shows both sides saw a gbraid on the same submissions; the log never records the value itself.
- Removing the vendor package was not unblocked by this ticket alone: the website still sent two vendor package fields on every lead (handled in DDT-1432 and DDT-1332).

## 11. Reuse for similar issues

- Signs of the same problem: paid clicks from iPhones arrive in Salesforce looking unpaid. Check which click-id parameters the site actually stores.
- Ship a new field write behind a switch that defaults off until field access is proven in production.
- Any list of "fields safe to drop on retry" must be updated whenever a new optional field is sent.
- Prove capture by comparing against an independent source over a week, not by one test.
