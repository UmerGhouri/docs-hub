# Caller lookup showed "unavailable" after the Salesforce session expired


| | |
|---|---|
| **Ticket** | DDT-403 |
| **Project** | #Phone-Bot Tracker |
| **Status** | Live since 2026-07-20 (reached production) |
| **Priority** | None set |
| **Reported by** | Salesforce lead, 2026-07-20 |
| **Resolved by** | Support team |
| **Systems** | Phone Bot service (Python, hosted on Railway), Salesforce (SOSL search) |
| **Area** | Salesforce client line on phone alerts |
| **Related** | None |
| **Code** | Direct commit to the sf-ops-bot repo (no PR) |


---


## 1. Summary

After about two hours of uptime, every Phone Bot alert showed "lookup unavailable" instead of the caller's Salesforce client, until the service restarted. The phone-number search did not renew an expired Salesforce session. It now catches the expired-session error, renews the session and retries the search once.

## 2. What was reported

- 2026-07-20 (work done 2026-07-14): service logs showed "SF lookup returned error ... Expired session" after about 2 hours, and Slack showed "lookup unavailable" on every alert until a restart.

## 3. Scope

**In scope**

- `app/clients/salesforce.py`: `find_client_by_phone`.
- `tests/test_rc_caller_enrichment.py`: 2 new tests.

**Out of scope**

- A separate credential-configuration error seen in the logs needed a manual configuration fix and is not handled by this retry (recorded in the commit).

## 4. How it works (background)

For each alert, the bot searches Salesforce by the caller's phone number (SOSL) and adds the matching client to the post. Salesforce sessions expire, so long-running clients must renew them. Two other functions in the same client already renewed and retried on expiry.

## 5. Root cause

`find_client_by_phone` caught the general Salesforce error but never renewed the session, so one expired session made every later lookup fail. The expired-session error is a subclass of the general error, so it has to be caught first.

## 6. Solution

- Catch the expired-session error before the general error.
- Rebuild the Salesforce client with a forced refresh and run the same search once more.
- If the retry also fails, return a clear "re-auth failed" error instead of looping.
- Why: same pattern the client already used elsewhere, so behaviour is consistent.

## 7. Testing

Two new tests: the first search raises an expired session and the retry with a forced refresh returns records; both attempts fail and the function returns an error result rather than raising. A live check is not recorded in the ticket.

## 8. Step by step

1. Find every Salesforce call in a long-running service.
2. Catch the expired-session error first (it is a subclass), refresh the session, retry once.
3. Return a clear error if the retry fails; never loop.
4. Test both paths with a mocked client.

## 9. Rollback

Revert the commit and redeploy; lookups would again fail after the session expires until a restart.

## 10. Known limits

- Only one retry; a second failure still shows "lookup unavailable" for that alert.
- Credential errors other than expiry are not covered.

## 11. Reuse for similar issues

- Signs: an integration that works after a restart and fails hours later with "expired session".
- Pattern: wrap every API call of a long-running client in "on expiry, refresh and retry once". Check the order of exception handlers when one error type is a subclass of another.
