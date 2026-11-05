# Move the Google Ads developer token out of the code


| | |
|---|---|
| **Ticket** | DDT-1226 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-10-15 (reached production) |
| **Priority** | Medium |
| **Reported by** | Salesforce lead, 2026-10-05 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, protected credential configuration), Google Ads API |
| **Area** | Integration security: Google Ads |
| **Related** | DDT-1167 |
| **Code** | PR #612 in the ddt-salesforce repo |


---


## 1. Summary

The Google Ads developer token was written into the Apex code that makes every Google Ads call (the lead ingest, the conversation sync and the daily spend pull). It was moved into Salesforce's protected credential configuration, so the code no longer holds or sets it. A test now fails if a token literal reappears in the class. All Google Ads calls ran green on the new setup for a full day before the ticket closed.

## 2. What was reported

- 2026-10-05: the developer token is a string constant in the Google Ads service class, in git and in production. Move it to protected configuration and switch the shared callout to use it. Done when there is no token literal in the repo and the lead ingest and the spend pull both run green for 24 hours on the new setup.

## 3. Scope

**In scope**

- The Google Ads service class: three lines changed so the shared Google Ads callout uses the protected configuration.
- New credential configuration metadata and a permission set granting its use to the users whose jobs make the calls.
- A new guard test class (2 tests).

**Out of scope**

- The OAuth login used alongside the token: unchanged.
- Rotating the old value: reviewed after the move and closed by a firm leadership decision; details are not published.

## 4. How it works (background)

Every Google Ads call in Salesforce goes through one shared method. The lead ingest runs every two minutes, so any break in that method stops lead ingestion quickly; the order of the deploy steps mattered.

## 5. Root cause

The token had been written into the class when the integration was built, so it lived in source control and in production code rather than in protected configuration.

## 6. Solution

1. **Configuration first, code second.** The credential configuration and permission set were deployed first; they change nothing on their own. The token value was then entered inside production by an administrator, never printed or logged, and access was assigned. Only then was the class deployed. Why: deploying the class before the value and access were in place would stop every Google Ads job.
2. **One change covers all callers**, because there is only one Google Ads callout.
3. **A guard test** asserts the class holds no token literal, so it cannot quietly come back.
4. **A validated rollback** was prepared before the deploy (the old class plus removal of the guard test in the same deploy).

## 7. Testing

- Sandbox: check-only and deploy passed 13 of 13. A live call could not be proven there, because the sandbox's Google login lacks the Ads scope and Google checks that first.
- Production deploy (2026-10-05): 50 of 50 tests, 98 percent coverage. In the first 13 minutes, 39 of 39 Google Ads calls succeeded through the new setup and 0 jobs failed.
- After 24 hours (2026-10-06): every Google Ads call used the new setup and the morning spend pull worked. A short burst of overnight failures came from Google's own servers.

## 8. Step by step

1. Find every place the secret is used; route them through one callout.
2. Create protected credential configuration and a permission set for the running users.
3. Deploy the configuration, enter the secret inside the org, and assign access; confirm the assignments.
4. Validate and prepare the rollback package before deploying the code.
5. Deploy the code and watch the next runs of every caller.
6. Add a test that fails if the literal returns.

## 9. Rollback

Deploy the previous class and remove the guard test in the same deploy (otherwise the guard test fails a full test run), then confirm a successful lead-ingest call. The credential configuration can stay; on its own it does nothing.

## 10. Known limits

- Any scheduled job that makes Google Ads calls must run as a user who holds the access permission set; a new job owner needs it assigned.
- The value remains in older git history; the decision on that is recorded on the ticket.

## 11. Reuse for similar issues

- Signs of the same problem: a key or token appears as a string constant in a class. Move it to protected configuration and add a test that fails if a literal returns.
- Order matters: configuration and access first, code last, with a validated rollback ready.
- Prove the change on every caller's next run, not only the one you were working on.
