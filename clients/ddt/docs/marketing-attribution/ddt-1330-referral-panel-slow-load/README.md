# Lead referral panel briefly showed the full picker on a slow load


| | |
|---|---|
| **Ticket** | DDT-1330 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-10-16 (reached production) |
| **Priority** | Low |
| **Reported by** | Firm operations, 2026-10-14 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC) |
| **Area** | Lead page: Referral Source panel |
| **Related** | DDT-1178 |
| **Code** | PR #745 in the ddt-salesforce repo |


---


## 1. Summary

On the Lead page, the Referral Source panel hides its card when the "how did you hear about us" answer is a channel that needs no referrer. On a slow first load, the panel briefly showed the full referrer picker with its instructions instead. The panel has a fallback that shows the full picker if the data has not loaded within a set time; that wait was 3 seconds, too short for a slow but healthy cold load. It now waits 8 seconds.

## 2. What was reported

- 2026-10-14: in a read-only production click-through after DDT-1178 went live, the first cold load of a Lead whose answer was a channel showed the full picker; three later loads hid the card correctly. Likely cause: the 3-second fallback in `inboundContactReferrerPicker`. Low impact, because nothing is written on page load. Done when a load slower than the timeout ends with the card hidden, proven by a Jest test that fails without the fix.

## 3. Scope

**In scope**

- `inboundContactReferrerPicker`: `LOAD_FALLBACK_MS` from 3000 to 8000.
- Three new Jest tests.

**Out of scope**

- The fallback behaviour itself: a data load that never arrives still shows the full picker, now after 8 seconds.
- The partial-copy sandbox copy of this component, which differs from `main`; it was left alone.

## 4. How it works (background)

The panel reads the Lead and any saved referral. Its rules decide whether to hide the card, show "Referred by ...", or show the full picker. If either read has not answered after the fallback time, it shows the full picker, so a read that never arrives cannot hide the card for good.

## 5. Root cause

The fallback time was too short. The ticket expected the panel to stay stuck in fallback; in Jest it does not: once data arrives late, the panel returns to the right view. So the production symptom was the picker showing until the data landed on a load slower than 3 seconds. Evidence: a new test of a 5-second cold load fails on the 3-second code ("Received" the picker) and passes with 8 seconds.

## 6. Solution

Raise the wait to 8 seconds. Why: the trade-off is an empty slot for 5 more seconds in a real failure, against a wrong view on every slow but healthy load. The second is more common and more confusing for staff.

## 7. Testing

- New Jest tests: a 5-second cold load on a channel answer never shows the full picker (fails on the old code); a channel answer arriving after the wait hides the card again; a named answer plus saved referral arriving after the wait show "Referred by ..." with no dropdown, instructions or false "disagrees" flag. The last two pass on the old code too and now guard against regression.
- Jest suites for the panel and the picker: 92 of 92 passing.
- Live check: a cold load of a vendor-answer Lead in production no longer flashes the picker.

## 8. Step by step

1. Reproduce the slow load in a Jest test with fake timers and a delayed wire.
2. Confirm the test fails on the current timeout.
3. Raise the timeout; keep the fallback for reads that never arrive.
4. Add tests for data that arrives after the timeout, so recovery stays guaranteed.

## 9. Rollback

Redeploy `inboundContactReferrerPicker` from `main` before PR #745 (timeout back to 3 seconds).

## 10. Known limits

- A load slower than 8 seconds still shows the full picker until the data arrives.

## 11. Reuse for similar issues

- Signs of the same problem: a component shows a "safe default" view for a moment on first load, and then corrects itself.
- Check fallback timers against real cold-load times, and test both "late but arrives" and "never arrives".
