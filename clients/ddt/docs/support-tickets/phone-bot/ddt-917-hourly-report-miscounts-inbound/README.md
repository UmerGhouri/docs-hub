# Hourly staffing report miscounts inbound and answered calls

| | |
|---|---|
| **Ticket** | DDT-917 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-04 |
| **Priority** | High |
| **Reported by** | Firm operations, 2026-09-02 |
| **Resolved by** | Support team |
| **Systems** | Phone Bot (Python), RingCentral call log and active-calls feed |
| **Area** | The hourly staffing recap the Phone Bot posts to Slack |
| **Related** | None recorded |
| **Code** | Commit `998f4d8` in the Phone Bot repo (`driverdefenseteam-sf-ops-bot`); no PR number recorded |

---

## 1. Summary

The Phone Bot's hourly staffing recap listed answered calls as lost. The ticket blamed internal calls being counted and the wrong call leg being read. **Neither was the cause.** Every disputed call was still connected when the recap ran at the top of the hour, so it had no final result yet, and the recap counted anything not yet answered as lost. A call still connected at report time is now shown as **"still in progress"**, outside both answered and lost.

## 2. What was reported

On 2026-09-02 the operations lead checked three of that morning's recaps against the RingCentral call log and found each one wrong:

| Hour | Recap said (in / answered / lost) | Actual (in / answered / lost) |
|---|---|---|
| 8 to 9 AM | 16 / 12 / 4 | 17 / 15 / 2 |
| 9 to 10 AM | 24 / 22 / 2 | 24 / 23 / 1 |
| 10 to 11 AM | 23 / 18 / 5 | 24 / 20 / 4 |

In each hour, calls named on the lost list had in fact been answered, with recordings and long talk times. She also asked separately whether calls that ended in voicemail were being left out of "lost".

The recap is the main signal management uses to judge staffing and to chase lost calls, so a wrong lost list sends people to call back clients who were already helped.

## 3. Scope

**In scope:** `app/core/phonebot_stats.py`, the hourly recap. A new `_split_still_running` step, and the headline line gains "N still in progress". New tests in `tests/test_phonebot_calls_in_progress.py`.

**Out of scope**

- **The internal-call filter and the call classifier.** Both were checked and both were already correct (section 5).
- **Re-editing a posted recap** once its long calls finish. Possible, not built (section 10).
- **Voicemail handling.** Already correct (section 5).

## 4. How it works (background)

```
Top of the hour (e.g. 11:00)
  → fetch the RingCentral call log for the hour that just closed
  → keep inbound calls, drop internal extension-to-extension calls
  → classify each call from its master result: answered, abandoned, rang out, voicemail
  → lost = total inbound − answered
  → post the recap to Slack
```

The recap runs the moment the hour closes, and is written once and never revisited.

## 5. Root cause

The ticket's diagnosis was **wrong on both counts**, and that is the most useful thing to know before touching this code:

- **"Internal calls are counted as inbound."** The filter found exactly the two internal calls the reporter found in the 10 to 11 AM hour.
- **"The classifier reads an intermediate 'Missed' leg instead of the master result."** All the disputed calls were checked; the classifier reads the master result correctly and returns "answered" for every one of them now.

**What was actually happening:** every disputed call was still connected when its recap ran.

```
8 to 9 AM recap fires 9:00    call started 8:53, ran 32 min, ended 9:24
                              call started 8:53, ran 38 min, ended 9:31
9 to 10 AM recap fires 10:00  call started 9:59, ran 14 min, ended 10:13
10 to 11 AM recap fires 11:00 call started 10:57, ran 5 min (318 s)
                              call started 10:59, ran about 3 min (185 s)
```

A call still in progress has no final result, so it was not "answered", and `lost = total − answered` swept it into lost.

The one remaining gap in the 10 to 11 AM hour (23 inbound instead of 24) was a record RingCentral had not yet written when the recap ran. Nothing computed at report time can see it.

**Voicemail** was also checked: over the previous three days, all 10 calls that ended in voicemail were counted as lost and shown on the voicemail line. The hour showing "0 voicemail" was accurate for that hour.

## 6. Solution

At report time, ask RingCentral which calls are still active, and set those aside:

```python
inbound, running = _split_still_running(inbound)
answered = sum(1 for c in inbound if _classify_call(c) == "answered")
lost = len(inbound) - answered
pct = _pct(answered, len(inbound))   # answer rate over finished calls only
```

`_split_still_running` matches each call's session id against RingCentral's active-calls feed. The headline becomes:

```
Overall: 24 inbound calls · 18 answered · 4 lost · 2 still in progress
```

Why it's done this way:

- **The live call stays in the inbound total** but is excluded from both answered and lost, so the parts still add up to the total. That arithmetic is what let the reporter spot the error in the first place.
- **The answer rate is over finished calls,** so an unfinished call no longer drags it down.
- **RingCentral's active-calls feed is the authority,** not the call's duration, which is itself still counting up. The feed was already used and cached by the lost-call alerts, so this adds no meaningful traffic.
- **If the feed can't be reached, every call is treated as finished,** which is the old behaviour, rather than guessing.
- **A delay before posting was considered and rejected.** Two of the disputed calls ran over 30 minutes; no delay worth having would cover them, and every recap would be late.

## 7. Testing

Nine new tests; 865 passing in the suite at that commit, 880 at closure. They cover: a connected call is not settled; nothing active means everything settled; empty input; a call with no session id is treated as settled; a failed active-calls fetch falls back to the old behaviour; and, on a fixture modelled on the reported hour, live calls are not counted as lost, the numbers reconcile, the live call is absent from the lost list, and the answer rate excludes unfinished calls.

**Why re-running the reported hour proves nothing.** The acceptance criterion "re-run 10 to 11 AM and get 24 / 20 / 4" passes today whatever the code does, because the data is final now. It would have passed before the fix too. A real test needs a call that is live at the moment the report runs, which is what the new tests simulate.

**Not covered:** a record RingCentral has not written yet at report time (section 10).

## 8. Step by step

1. For each disputed call, compare its start time and duration with the time the recap ran. If the call was still connected at that moment, this is the cause.
2. Check the internal-call filter and the classifier against the same hour before changing either. Here both were already right.
3. Add `_split_still_running` and call it before counting answered and lost.
4. Compute lost and the answer rate over settled calls only; keep the total over all inbound calls.
5. Add "N still in progress" to the headline when N is above 0.
6. Fall back to treating everything as settled if the active-calls fetch fails.
7. Add tests that hold a call live at report time.
8. Deploy and watch the next few recaps.

## 9. Rollback

Revert commit `998f4d8` in the Phone Bot repo and redeploy. Safe, but live calls will be counted as lost again.

## 10. Known limits

- **A recap is written once.** A call in progress at the top of the hour stays "in progress" in that post forever. On the 8 to 9 AM example that reads "12 answered · 2 in progress" where the reporter counts 15 answered. The fix would be to edit the message once the calls finish; the mechanism exists but is not built.
- **Late records.** If RingCentral has not yet written a call's record when the recap runs, the call is missing from the total entirely. That needs a re-read later, not a classifier change.

## 11. Reuse for similar issues

**How to recognise one:** a report built "as of now" disagrees with the source system when someone checks it later, and every disputed item was still open at report time.

- Before blaming the parsing code, compare each disputed item's timing against when the report ran.
- Never compute a category as "total minus the others" when some items have no outcome yet. Give unfinished items their own bucket so the parts still sum.
- Use the source system's live-state feed to tell finished from unfinished, not an elapsed time.
- An acceptance test that re-runs a past period on now-final data cannot catch a timing bug. Test with an item that is still open.
