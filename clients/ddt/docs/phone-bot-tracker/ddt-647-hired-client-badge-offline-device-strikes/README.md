# Hired clients flagged high-value, and reps blamed for offline phones


| | |
|---|---|
| **Ticket** | DDT-647 |
| **Project** | #Phone-Bot Tracker |
| **Status** | Live since 2026-08-05 (reached production) |
| **Priority** | None set |
| **Reported by** | Firm leadership, 2026-08-05 |
| **Resolved by** | Support team |
| **Systems** | Phone Bot service (Python, hosted on Railway), RingCentral, Salesforce (read only), Slack |
| **Area** | Lost-call badges and NO ANSWER strikes |
| **Related** | DDT-649, DDT-599 |
| **Code** | Direct commit to the sf-ops-bot repo (no PR) |


---


## 1. Summary

Two fairness fixes shipped together. First, a signed client calling through an intake line got the high-value badge meant for new leads; the badge is now dropped when Salesforce shows the caller already has a Court Case. Second, a rep whose desk phone was offline received a public NO ANSWER strike; device-unreachable reasons are now read from all the fields RingCentral uses, and such legs never produce a strike.

## 2. What was reported

- 2026-08-04: firm leadership: a call "will show that it's a high value call — aka intake — when it's a hired client."
- 2026-08-04: in a live thread, a rep received a public NO ANSWER strike while the bot's own follow-up showed her device was reporting phone offline / no digital line.

Filed 2026-08-05 after the fix.

## 3. Scope

**In scope**

- `app/core/phonebot_lost_calls.py`: the Salesforce client cache `_sf_client_cache`; new helper `_is_hired_client`; money decision in `_format_lost_call`; `_PHANTOM_RESULT_TAGS`; the leg check in `_extract_rang_out_events`.
- `tests/test_phonebot_lost_calls.py`.

**Out of scope**

- Reporting offline devices to IT: noted as a possible separate ticket, not built.

## 4. How it works (background)

The high-value badge comes from `_is_money_call(call)`, which looks only at routing: calls through intake, PI, paid-source or website lines get it. Separately, each LOST CALL looks up the caller in Salesforce to show a client line at the bottom of the post, and caches the result for an hour. A NO ANSWER strike is raised when a rep's leg of a call rings out; "phantom" legs (the device could not ring at all) are meant to be skipped.

## 5. Root cause

- Badge: signed clients reach their case manager through the same intake queues, so routing alone marks them high-value. The Salesforce lookup that knows they are clients ran after the badge decision and its answer was never used for it.
- Strike: the extractor checked only the leg's `result` field against the phantom list. RingCentral often reports the device problem in `reason` or `reasonDescription` while `result` just says "Missed". The affected leg was "Missed" for 12 seconds with a reason of phone offline, which was ignored.

## 6. Solution

- The cache now keeps the raw Salesforce matches beside the display line (same one-hour cache, no extra queries). `_is_hired_client(phone)` is true when any cached match is a `Court_Case__c`, because Court Cases exist only for signed clients in the firm's data model. The badge is now `_is_money_call(call) and not _is_hired_client(phone)`. A new lead with no match keeps the badge.
- The rang-out extractor checks `result`, `reason` and `reasonDescription`. The phantom list gained the camelCase spellings RingCentral uses in `reason` (`ipphoneoffline`, `nodigitalline`, `digitalline`, `internalaction`, `callnotaccepted`) beside the spaced forms used in `reasonDescription`.

Why: a rep should not be named publicly for a device fault, and a hired client is not a new lead.

## 7. Testing

- 4 new tests: Court Case match drops the badge; no match keeps it; reason "phone offline" gives no strike; reasonDescription "no digital line" gives no strike. 673 tests passed.
- Live checks (a signed client's miss without the badge; no strike for an offline device) are listed as verification steps; results are not recorded.

## 8. Step by step

1. Keep the raw lookup matches in the existing cache.
2. Add a helper that says whether the caller has a record type that only signed clients have.
3. Combine it with the routing check when choosing the badge.
4. Check every field the telephony provider uses for device-unreachable reasons, in each spelling.
5. Add tests for both paths.

## 9. Rollback

Revert the commit on the bot's deploy branch and push; Railway redeploys.

## 10. Known limits

- Callers with a Salesforce record but no Court Case still get the badge if routing says so.
- The badge depends on the Salesforce lookup having run for that caller.

## 11. Reuse for similar issues

- If a label is decided before a lookup that could change it, move the decision after the lookup or reuse the cached result.
- When filtering by a provider's status, check every field that can carry the reason, in every spelling the provider uses.
