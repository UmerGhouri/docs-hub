# Recap credits pickups to the wrong rep or team

| | |
|---|---|
| **Ticket** | DDT-783, DDT-834, DDT-942, DDT-1258 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-06 |
| **Priority** | High (DDT-834); Medium (DDT-783, DDT-942, DDT-1258) |
| **Reported by** | Firm operations, 2026-08-20 |
| **Resolved by** | Support team |
| **Systems** | Phone Bot (Python service), RingCentral call log |
| **Area** | Hourly and daily phone recaps: the "Top pickups" leaderboard |
| **Related** | DDT-754 (first-leg ordering used for recovery credit) |
| **Code** | Four commits in the `driverdefenseteam-sf-ops-bot` repo, one per ticket; DDT-1258 shipped as PR #17 in that repo |

---

## 1. Summary

The Phone Bot posts an hourly and a daily recap of inbound calls, with a "Top pickups" line naming the reps who answered the most. Over seven weeks that line was wrong four different ways: it credited an outside party, it missed almost half of all pickups, it credited a zero-second hand-off as a pickup, and it looked as if it was mixing teams.

Each was a separate flaw in how the bot reads a RingCentral call's legs to decide who answered. Each was fixed in the bot's pickup-credit logic, with tests, and the last one was a wording change rather than a logic change.

## 2. What was reported

- **DDT-783, 2026-08-20.** The hourly Top pickups listed a name in all capitals that matched no RingCentral user or extension. The reps who actually answered calls in that hour did not include anyone by that name.
- **DDT-834, 2026-08-25.** The daily recap credited a person with 5 pickups on a day they answered none, which RingCentral's call log confirmed. The same day a supervisor asked why a rep who takes calls on his own direct line led the hourly board when Intake had answered 14 of 15 calls and no Intake rep appeared at all. The firm's owner said in the thread to disable the feature if it could not be made right.
- **DDT-942, 2026-09-07.** An hourly recap showed 1 call, answered, attributed to Intake, with the pickup credited to an attorney who is not in any Intake queue.
- **DDT-1258, 2026-10-06.** An hourly alert with an Intake-only team breakdown listed three Case Management (CMT) reps in Top pickups, alongside one Intake rep.

Why it matters: the recap is used to see who is carrying the phones. A wrong name, or a correct team total next to the wrong names, makes the whole recap untrustworthy for staffing and performance conversations.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `app/core/phonebot_lost_calls.py` | Python module | DDT-783: `_is_person_leg_name` rejects carrier caller-ID names via a new `_looks_like_carrier_cnam`. DDT-942: new `_leg_duration_is_zero`; `_any_leg_connected` skips explicit zero-second legs |
| `app/core/phonebot_stats.py` | Python module | DDT-834: new `_pickup_rep_name` replaces the inline name matcher in `_per_rep_leaderboard`; extension-to-name map cached for every user. DDT-942: zero-second legs skipped in pickup credit. DDT-1258: label changed to "Top pickups (any team):" on hourly and daily |
| `tests/test_phonebot_cnam_names.py` | Tests | 11 new (DDT-783) |
| `tests/test_phonebot_pickup_credit.py` | Tests | 13 new (DDT-834) |
| `tests/test_phonebot_zero_duration_legs.py` | Tests | 10 new (DDT-942) |
| `tests/test_phonebot_team_channels.py` and others | Tests | 10 new, plus label updates in existing tests (DDT-1258) |

**Out of scope**

- **Making Top pickups match the "By team" totals.** They measure different things (where a call was routed against who answered it). Forcing them to match would hide real overflow coverage. See section 6.4.
- **Restricting Top pickups to reps who belong to the team.** Rejected for the same reason in DDT-1258.
- **Republishing past recaps.** DDT-783 and DDT-834 asked for a backfill or correction of earlier recaps. Whether one was posted is not recorded in the ticket.
- The other changes that rode on PR #17 (per-team daily recaps, team channel routing, a marker on calls that reach no queue) are Phone Bot features, not part of this fix.

## 4. How it works (background)

RingCentral records each inbound call as a session with several **legs**: the main number, the router, each queue it rang, each person or device it rang, any transfer. The recap has to pick out which leg represents a human answering.

```
Caller dials the main number
  -> greeting / router leg
  -> queue leg ("All Intake Members"), result Accepted
  -> rep's device leg, result "Call connected", extension only, no name
  -> (optional) transfer leg to someone else or an outside number
```

Before these fixes, the leaderboard walked the legs **backwards** and took the last leg whose `to.name` looked like a person and whose result contained "accepted" or "answered". If none matched it fell back to the last person-shaped name on the call.

## 5. Root cause

Four separate flaws in reading the legs.

### 5.1 Outside parties counted as reps (DDT-783)

When a rep transfers a caller to an outside line, RingCentral fills that leg's `to.name` from the **telephone carrier's caller-name (CNAM) database**. Those strings look like person names, so they passed the person-name check and were credited. Reproduced on the hour that was reported: the bogus entry also pushed a real rep's genuine pickup out of the top five.

Evidence recorded in the commit: across 3 days of production inbound legs, the only all-capitals names were three CNAM strings (one person's name, a county facility, and `WIRELESS CALLER`), and all 21 real reps were Title Case.

### 5.2 Queue pickups were invisible (DDT-834)

The ticket's first guess was that the recap credited anyone whose extension was rung. The real cause was the opposite: the bot could not see most real pickups. A queue-delivered pickup is logged like this:

```
Inbound   Accepted        name="All Intake Members"   ext=-
Outbound  Call connected  name=-                      ext=<rep's extension>
```

The rep's leg has **no name, only an extension**, and its result is "Call connected", not "Accepted". The matcher read only `to.name` and only matched accepted or answered, so it missed that leg and fell back to the last person-shaped name on the call.

Measured over two days of production calls: **94 of 200 answered calls (47%) credited nobody.** The calls that did credit someone were mostly reps answering their own direct line, where RingCentral does put a name on the leg. That is why a rep who takes direct calls led the board while most of Intake was invisible.

### 5.3 A zero-second "Accepted" leg (DDT-942)

The ticket asked whether the problem was team bucketing or pickup attribution. It was neither in the expected sense: **nobody answered that call.** It rang 8 seconds and rang out. One leg on the main number was `Accepted` for 55 milliseconds, and RingCentral labels that step with the name of the extension that owns it, which happened to be the attorney. The recap read "Accepted plus a person's name" as an answered call.

The DDT-834 fix made this worse in one direction: it credits the **first** connected leg, so a zero-second artifact earlier in the call beat the rep who actually talked to the caller. Measured over three days of production data:

- **4 genuinely lost calls had been counted as answered**, so no lost-call alert ever posted for them. One caller waited 72 seconds and reached voicemail.
- **8 pickups were credited to the wrong person.** 4 moved to the rep who actually answered; 4 moved to nobody because the calls were lost.

The commit records that requiring talk time had been considered during DDT-834 and rejected to avoid over-filtering, and that this was the consequence.

### 5.4 Cross-team names under a team heading (DDT-1258)

Not a counting bug. The Intake pickups added up exactly to the Intake answered figure beside them. The CMT reps really did answer Intake calls: Intake overflowed into the shared Intake + CMT queue and CMT picked up. The call still counts as Intake because that is where it arrived. There was no rep-to-team mapping to fix; nothing in the bot maps reps to teams. The label simply did not say that the names could come from any team.

## 6. Solution

### 6.1 Reject carrier caller-ID names (DDT-783)

```python
def _looks_like_carrier_cnam(name: str) -> bool:
    """True if `name` looks like a telco CNAM string rather than an RC
    user record: all-caps with at least two letters."""
    if not name:
        return False
    letters = sum(1 for ch in name if ch.isalpha())
    if letters < 2:
        return False
    return not any(ch.islower() for ch in name)
```

`_is_person_leg_name` returns `False` when this is true.

Why this way:

- **Case is the separator.** RingCentral user records are entered Title Case; CNAM is uppercase by convention.
- **Leg shape cannot do it.** Reps also produce outbound, no-extension, no-number legs, the same shape as the CNAM legs. `to.extensionNumber` cannot do it either: real reps appear with and without it. The commit records this so nobody tries again.
- **Filtered in the shared helper**, not in the leaderboard, so the same names cannot leak into the rang-out lines.
- **Fails safe for a leaderboard.** A real rep entered in all capitals would be skipped rather than mis-credited.

### 6.2 Credit the extension that connected, first leg wins (DDT-834)

New `_pickup_rep_name` in `phonebot_stats.py`:

- Treats `call connected`, `accepted` and `answered` as connected results.
- Resolves the leg's **extension** to a person using an extension-to-name map, falling back to the name only where there is one.
- Takes the **first** connected leg, not the last. A warm transfer adds a later connected leg, and the pickup belongs to whoever answered, not whoever it was handed to. This is the same ordering DDT-754 settled for recovery credit.
- Keeps the extension-to-name map for **every** user, whether or not their RingCentral department maps to a team, so a blank department (4 users had one) cannot cost someone their credit.

The DDT-783 rule still applies: a CNAM name on a connected leg is never credited.

### 6.3 A pickup needs talk time (DDT-942)

```python
def _leg_duration_is_zero(leg):
    raw = leg.get("duration")
    if raw is None:
        return False
    try:
        return int(raw) <= 0
    except (TypeError, ValueError):
        return False
```

Applied in two places: `_pickup_rep_name` skips such legs, and `_any_leg_connected` no longer treats them as proof the call was answered.

Why only an **explicit** zero: a missing duration is unknown, not zero. An existing override relies on missing durations to stop answered calls being reported as missed while RingCentral's data is still settling, and a test pins that case. A call whose master result has settled to `Accepted` is still treated as answered regardless of its legs.

### 6.4 Say what the line means (DDT-1258)

The label now reads `Top pickups (any team):` on both the hourly and the daily recap. Scoping it to reps who belong to the team was rejected: it would drop the overflow pickups, stop the line reconciling with the answered count, and hide the useful signal, which is CMT covering Intake.

## 7. Testing

Recorded in the commits:

| Ticket | Tests | What they prove |
|---|---|---|
| DDT-783 | 807 pass, 11 new | CNAM strings are rejected; **all 21 production rep names** still pass, so a future tightening that drops a real person fails loudly |
| DDT-834 | 835 pass, 13 new | A nameless "Call connected" leg is credited via its extension; first connected leg wins |
| DDT-942 | 890 pass, 10 new | A zero-second Accept is neither a pickup nor proof of an answer; a missing duration keeps its old meaning |
| DDT-1258 | 1083 pass, 10 new | New label and the team-channel changes on the same branch |

Live checks, replayed on production call data:

- **DDT-783**, on the reported hour (27 calls): the bogus name was replaced by the real rep it had displaced; the rest of the top five was unchanged.
- **DDT-834**, over two days: calls credited to nobody went from **94 to 0**, coverage from **53% to 100%**, and Intake reps moved to the top of the board.
- **DDT-942**, over three days: answered 130 to 126, rang out 66 to 69, abandoned 11 to 12, voicemail unchanged. The reported hour now reads 0 answered, 1 lost, credited to nobody.

**Not covered:** whether past recaps were corrected is not recorded. The firm's owner noted on DDT-942 that one fix is not proof the bot is ready, and asked staff to keep flagging wrong numbers.

## 8. Step by step

To investigate a "wrong name in Top pickups" report:

1. Get the recap window and the name that looks wrong.
2. Pull the RingCentral call log for that window and list every leg of each answered call: direction, result, duration, `to.name`, `to.extensionNumber`.
3. For each call, find the leg the bot credited and the leg where a human actually talked (real duration, "Call connected" or "Accepted").
4. Classify the mismatch:
   - all-capitals name on a transfer leg: carrier caller-ID (6.1)
   - rep's leg has an extension but no name: extension resolution (6.2)
   - credited leg has duration 0: hand-off artifact (6.3)
   - counts add up but names cross teams: overflow, working as designed (6.4)
5. Fix in the shared helper in `phonebot_lost_calls.py` or `phonebot_stats.py`, not in the formatter, so lost-call alerts and recaps agree.
6. Add a test built from the real leg shape, then replay a few days of production calls and compare before and after counts.
7. Deploy the bot and check the next hourly recap.

## 9. Rollback

Revert the relevant commit in the bot repo and redeploy. Each fix is independent code, but they build on each other: reverting DDT-834 also removes the code path DDT-942 patches. Reverting brings back the wrong attribution described in section 5. No data is written, so nothing needs cleaning up.

## 10. Known limits

- **A rep entered in RingCentral in all capitals** would never be credited. All 21 reps were Title Case when this shipped; the test of production names is the guard.
- **Top pickups will not match the By-team totals**, by design. On the DDT-834 sample, Intake was routed 108 answered calls but Intake reps picked up 76; CMT picked up 62 against 35 routed. Adding a line that shows this overflow was offered to the supervisor; the answer is not recorded in the ticket.
- **A leg with no duration field** is still trusted. If RingCentral ever starts omitting durations on hand-off legs, the DDT-942 artifact would come back.

## 11. Reuse for similar issues

**How to recognise one:** a report built from telephony legs names someone who did not do the thing, or misses someone who did.

- **Read the raw legs before believing the ticket's theory.** DDT-834's guess (crediting rung extensions) was the reverse of the truth (missing answered extensions).
- **Match on the extension, not the display name.** Queue-delivered legs often have no name.
- **Require evidence of a conversation.** A result of "Accepted" with zero duration is a hand-off, not a pickup.
- **Treat missing as unknown, not zero.**
- **Pin real data in tests**, including the full list of real names a filter must not drop.
- **Replay a few days of production calls** and publish before-and-after counts; a single reported hour proves little.
- **When two numbers disagree, check whether they measure different things** before forcing them to match.
