# Salesforce caller lookup on every phone alert


| | |
|---|---|
| **Ticket** | DDT-391 |
| **Project** | #Phone-Bot Tracker |
| **Status** | Live since 2026-07-20 (reached production) |
| **Priority** | None set |
| **Reported by** | Firm operations, 2026-07-20 |
| **Resolved by** | Support team |
| **Systems** | Phone bot service (Python, hosted on Railway), Salesforce (SOSL search), Slack |
| **Area** | Phone monitoring: caller identification |
| **Related** | DDT-382, DDT-385, DDT-389, DDT-392 |
| **Code** | No PR: direct commits in the phone bot service repo |


---


## 1. Summary

Every lost-call, callback-recovery, rang-out and callback-reminder alert now carries a "Client" line saying who the caller is in Salesforce, or that they are not in Salesforce. The bot searches Salesforce phone fields for the number. A follow-up made the line always appear, with a clear reason when the lookup could not run, so "not in Salesforce" and "we did not check" no longer look the same. The commits are dated 2026-07-10 and 2026-07-13; the ticket was logged on 2026-07-20 as a record.

## 2. What was reported

- 2026-07-20: record the Salesforce enrichment. Operations needs to know at a glance whether the missed caller is a client (route to case management), a converted lead (existing case), an unconverted lead (nurture) or unknown (possibly spam). The rule carried over from the previous phone monitoring bot is that the CRM match is the real spam filter.

## 3. Scope

**In scope**

- `app/clients/salesforce.py`: `find_client_by_phone(phone)`.
- `app/core/phonebot_lost_calls.py`: the per-number cache and `_sf_lookup_line()`.

**Out of scope**

- Any Salesforce change. The bot only reads.

## 4. How it works (background)

`find_client_by_phone` runs a Salesforce SOSL search on the caller's digits in phone fields. The ticket lists the objects as Contact, Lead and Account; the commit says the existing helper also searches `Court_Case__c`. Results are cached per normalised number: the ticket says within a tick, the commit says for 1 hour, so a burst of repeat callers costs one query.

## 5. Root cause

The first version is new work. The follow-up fix had this cause, from the commit: the lookup returned nothing both when the Salesforce client failed or was not configured and when there was simply no match, and in the failure cases the code left the line out. A lost-call post with no Client line was flagged, and operations could not tell "not in Salesforce" from "not checked".

## 6. Solution

1. **Add the line (2026-07-10).** Each alert type gets a caller line from the lookup. Errors inside the Salesforce client are swallowed, so a Salesforce problem never stops a phone alert from posting.
2. **Always show a result (2026-07-13).** Every path now returns a line:

| Situation | Line shown |
|---|---|
| Match found | Client: the record's name and object type |
| No match | New lead (not in Salesforce) |
| Salesforce error | Client lookup: unavailable (Salesforce error) |
| Not configured | Client lookup: Salesforce not configured |
| Bad number | Client lookup: no valid phone |

Why: a missing line was ambiguous; a reason on every alert is not.

## 7. Testing

- Automated tests in `tests/test_phonebot_lost_calls.py` with the first commit (shared with the Spoke-to-Human report, 16 test functions across both) and 2 with the follow-up (counted from the commit diffs).
- Whether those tests fail without the change is not recorded.
- Live check: the follow-up came from a live post with no Client line. No other live result is recorded in the ticket.

## 8. Step by step

1. Normalise the caller's number.
2. Check the cache; if not cached, search Salesforce phone fields.
3. Catch any error from the lookup.
4. Turn the result (match, no match, error, not configured, bad number) into one line and add it to the alert.

## 9. Rollback

Revert the commits and redeploy the service on Railway. Removing the Salesforce settings shows "Salesforce not configured" on alerts instead of failing.

## 10. Known limits

- SOSL matches on digits in phone fields, so a number stored in a non-phone field, or with a different number of digits, is not found.
- With the cache, a record created in Salesforce after the first lookup is not seen until the cache expires.
- The sources disagree on the objects searched and on the cache lifetime (section 4).

## 11. Reuse for similar issues

- Signs: an alert with no caller line, or "not in Salesforce" for a known client (check the error and configuration lines first, then how the number is stored).
- Patterns: never let an enrichment call block the main alert; always print the lookup outcome, including errors, instead of leaving it out; cache by normalised number.
