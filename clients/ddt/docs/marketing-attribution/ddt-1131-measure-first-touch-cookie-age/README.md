# Measure how old the first-touch cookie is when a form arrives


| | |
|---|---|
| **Ticket** | DDT-1131 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-11-02 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-09-26 |
| **Resolved by** | Support team |
| **Systems** | Firm website (Cloudflare Workers, Workers Logs) |
| **Area** | Website attribution: cookies and form submissions |
| **Related** | DDT-1132, DDT-1134, DDT-820, DDT-821, DDT-1133, DDT-1355, DDT-1361 |
| **Code** | PR #214 in the driver-defense-site repo |


---


## 1. Summary

The website remembers where a visitor first came from in a first-touch cookie, but Safari can delete cookies written by page JavaScript after 24 hours (when the visit came from an ad) or 7 days (no interaction). Before changing how the cookies are set (DDT-1132), the firm needed a "before" picture. One log line was added to every form handler, recording the browser, the operating system, and how old the attribution cookies were, never their values. After a week of data, the baseline was posted on 2026-10-26: 43 submissions, and most leads submitted on the same day they first arrived, so Safari's limits had little to delete in that week.

## 2. What was reported

- 2026-09-26: measure how old the first touch is when a form arrives, by browser, before changing anything. Done when at least one full week of log lines exists and a baseline comment gives total submissions, share by browser and OS, share with a first touch, first-touch age buckets, click-id shares, and how many Safari leads show a replaced first touch. The cookie fix must not deploy before that comment exists.

## 3. Scope

**In scope**

- New `touch-audit` module that builds and logs one JSON line tagged `touch_audit`, with tests.
- The five form handlers that read tracking: contact, defense, game plan, quote and reinstatement. The call sits after the bot check, so bots are not counted.
- Fields: endpoint, browser (Safari, Chrome, Firefox, Edge, InApp, Other), OS, has first touch and its age in days, has last touch and its age, has a Google or Facebook click id, whether the Torq tracking cookie arrived, its first-seen age, and whether it holds a Google `gbraid`.

**Out of scope**

- No Salesforce change and no change to what any form does.
- The two nurture forms, which read no tracking.
- The cookie fix itself (DDT-1132).

## 4. How it works (background)

The site writes `ddt_first_touch` once (when absent) and `ddt_last_touch` on every visit that carries a campaign tag or comes from another site. Both ask for 90 days. A separate Torq cookie is written on every page view. Form handlers read these cookies on the server and send them to Salesforce with the lead.

Safari's current rules (since a 2022 WebKit change): cookies written by JavaScript on a page reached through an ad click id are deleted 24 hours after being set; other JavaScript cookies are deleted after 7 days with no interaction on the site. A lost first touch can also be re-created on a later campaign visit, so it looks young rather than missing.

## 5. Root cause

Not applicable: this ticket measures a suspected problem rather than fixing one. The concern was that `ddt_first_touch` is written on exactly the landing page the 24-hour rule targets, and nobody knew how many leads that affected.

## 6. Solution

1. One log line per form submission, as JSON, so it is easy to search in Workers Logs. Why: Worker logs were already on and kept, so the numbers land somewhere with no new system.
2. **Never log a value.** Yes/no, counts and ages only: no click ids, URLs, referrers, names, emails or phones. Enforced by a test that plants fake values in every cookie and fails if any appears.
3. **Never throw.** The call is wrapped so it can never stop a lead reaching Salesforce; a test proves it.
4. Log browser and OS separately, and count in-app browsers (Facebook, Instagram, the Google app, WebViews) as their own bucket. Why: every browser on an iPhone runs on Safari's engine, and in-app browsers keep their own cookie jar, so the data should show whether they behave like Safari instead of assuming.
5. A cookie that arrives but cannot be decoded counts as present with a blank age, so corruption is not counted as loss.
6. Review on the pull request found the Safari rule first described was out of date; it was corrected the same day, and the gbraid flag and in-app bucket were added.

## 7. Testing

- 47 tests in the audit test file, with fifteen deliberate breaks each caught and restored (age rounding, browser order, a click id leaking, the try/catch removed, in-app detection, desktop-mode iOS Chrome read as macOS, and more). Full site suite 643 of 643 passed; type check and build clean.
- Run in the local Cloudflare runtime against all five handlers with planted fake values: each wrote exactly one line with the expected values and none of the planted values appeared.
- Live: the PR merged and deployed on 2026-09-26; the baseline comment on 2026-10-26 covered every `touch_audit` line from 2026-10-19 to 2026-10-26 (about seven days, as far back as Workers Logs keeps), read one day at a time so nothing was sampled.
- Not covered: browser detection is from the User-Agent, so it is an approximation.

### Baseline (2026-10-19 to 2026-10-26)

| Browser | Leads | Has first touch | Age 1 day or less | 2 to 7 days | Over 7 days |
|---|---|---|---|---|---|
| In-app | 17 | 16 | 13 | 1 | 2 |
| Chrome | 14 | 9 | 9 | 0 | 0 |
| Safari | 8 | 8 | 8 | 0 | 0 |
| Edge | 4 | 0 | 0 | 0 | 0 |
| All | 43 | 33 | 30 | 1 | 2 |

- 9 of 43 (21 percent) had a Google click id and 8 (19 percent) a Torq gbraid.
- 0 of 8 Safari leads showed a replaced first touch; across all browsers, 1.
- Only 6 of 43 (14 percent) had been seen on an earlier day, all in-app. Safari and Chrome looked the same this week; the sample is small.

## 8. Step by step

1. List every server handler that reads attribution cookies.
2. Add one tagged JSON log line per submission with yes/no flags and ages only.
3. Add tests that fail if any value leaks or if the logger throws.
4. Merge, confirm the deploy ran, then confirm lines actually appear in the logs (a green build is not proof).
5. Check how long the log store keeps data; read it in day-long windows before it ages out.
6. Post the baseline table before shipping the change it will judge.

## 9. Rollback

Revert PR #214. Nothing else depends on the log line except the comparison planned for DDT-1132.

## 10. Known limits

- Workers Logs keeps only recent days, so the baseline covers 2026-10-19 to 2026-10-26, not the first week after deploy.
- A week with 8 Safari leads is a small sample; the visitors who return after a week, whom the cookie fix targets, were rare.
- A Torq first-seen age past the first-touch age can mean a re-created first touch or earlier direct visits; the line cannot tell which.

## 11. Reuse for similar issues

- Before changing tracking behaviour, log a cheap baseline first, so the change can be judged.
- Log flags and ages, never values, and test for leaks with planted values.
- Wrap diagnostic logging so it can never break the main path.
- When a browser policy is suspected, split the data by both browser and OS, and keep in-app browsers separate.
