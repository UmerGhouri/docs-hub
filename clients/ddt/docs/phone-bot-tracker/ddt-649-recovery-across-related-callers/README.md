# Reminder fired after a relative of the caller was already reached


| | |
|---|---|
| **Ticket** | DDT-649 |
| **Project** | #Phone-Bot Tracker |
| **Status** | Live since 2026-10-05 (reached production) |
| **Priority** | None set |
| **Reported by** | Firm leadership, 2026-08-05 |
| **Resolved by** | Support team |
| **Systems** | Phone Bot service (Python, hosted on Railway), RingCentral, Salesforce (read only), Slack |
| **Area** | Missed-call recovery detection |
| **Related** | DDT-646, DDT-647, DDT-648, DDT-704 |
| **Code** | PR #14 in the sf-ops-bot repo (shared with DDT-704) |


---


## 1. Summary

The bot only treated a missed call as recovered when the same phone number called back or was called. In one case a rep reached a relative of the missed caller on the relative's own number, so the 15-minute REMINDER still fired and staff chased a call that was already handled. The bot now also closes a miss when the callback number belongs to a Salesforce contact on the same Account as the missed caller, and the RECOVERED post names both numbers.

## 2. What was reported

- 2026-08-05: on 2026-08-03 a LOST CALL fired for a client, then a REMINDER 15 minutes later. The rep replied that she had already spoken with the client's relative, who had called from a different number. Firm leadership listed this as complaint 5 of 2026-08-04. The ticket asked to wait until the real-time events and rep registry (DDT-646, DDT-648) had run in production for a week.

## 3. Scope

**In scope**

- `app/clients/salesforce.py`: surface the `AccountId` already returned by the existing client search.
- `app/core/phonebot_lost_calls.py`: store the Account on the miss; use it as a fallback match in recovery detection; add a "Same client as the missed call from ..." line to the RECOVERED post.
- New `tests/test_phonebot_reminder_age_and_related_recovery.py` (shared with DDT-704).

**Out of scope**

- Custom relationship objects between contacts, raised in the ticket as a possible complication: not used; the Account link is the only correlation.
- A feature flag, listed in the acceptance criteria: not recorded as built.

## 4. How it works (background)

Unrecovered misses are kept by the caller's normalised phone number. Recovery is detected when an outbound call goes to that number, or an inbound call from it is answered. When a LOST CALL posts, the bot searches Salesforce for the caller and caches the result. In the firm's data model, contacts in the same household or matter share an Account.

## 5. Root cause

Recovery matched strictly on the missed phone number, so a callback from or to a relative's number was never linked. The Salesforce search already returned `AccountId` in its result fields, but it was dropped when the match was built (PR #14).

## 6. Solution

The `AccountId` is kept on the match and stored on the miss. In recovery detection:

| Guard | Why |
|---|---|
| Exact phone match always wins | The Account path is only tried after an exact match fails |
| A miss with no Account is unchanged | Behaves as before |
| Misses rebuilt at boot never match | They carry no caller context; matching one could swallow a real callback |
| The Account comes from the cache the alert already filled | No extra Salesforce queries |

The RECOVERED post gains a second line, "Same client as the missed call from" the original number, because the number that called back is not the one that was missed and the post would otherwise look like it closes an unlogged call.

## 7. Testing

- PR #14: 1055 tests passed, 17 new in the shared test file. For this ticket: the related-caller case end to end, inbound and outbound; the RECOVERED names the number it closes; an unrelated caller recovers nothing; an exact match wins when both apply; boot-rebuilt and Account-less misses never match; a test fails if the Salesforce search is called during the Account check.
- Live check suggested: next time a relative calls back from their own number, no REMINDER should fire and the RECOVERED should name both numbers. The result is not recorded.
- The acceptance item to replay a real production case is not recorded as done.

## 8. Step by step

1. Return the Account id from the existing client search instead of dropping it.
2. Save it on the miss when the LOST CALL posts.
3. In recovery detection, try the exact number first; if no match, compare the callback's Account (from cached results) with the miss's Account.
4. Skip misses with no Account and misses rebuilt at boot.
5. Name both numbers in the RECOVERED post.
6. Test each guard, including that no new query is made.

## 9. Rollback

Revert PR #14 on the bot's deploy branch; Railway redeploys. That also reverts DDT-704, which shares the PR.

## 10. Known limits

- Works only when both people are contacts on the same Account. The check uses cached Salesforce results and makes no new query; how a callback number with no cached result is handled is not recorded.
- Two unrelated people on one Account would close each other's misses; the ticket flagged this as a rare false positive.

## 11. Reuse for similar issues

- Sign: a REMINDER or "unresolved" alert after staff say they spoke to the family. Check whether matching is by phone number only.
- Before adding a query, check whether the data is already in an existing result and being dropped.
- Add a related-record match only as a fallback after the exact match, and say in the message why it matched.
