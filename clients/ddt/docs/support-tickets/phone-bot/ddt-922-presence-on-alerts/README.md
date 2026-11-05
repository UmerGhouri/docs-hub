# Rep presence state on lost-call alerts

| | |
|---|---|
| **Ticket** | DDT-922, DDT-1129 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-30 (DDT-922); DDT-1129 done 2026-09-28 |
| **Priority** | Medium |
| **Reported by** | Firm operations, 2026-09-02 (DDT-922); Case Manager, 2026-09-25 (DDT-1129) |
| **Resolved by** | Support team |
| **Systems** | Phone Bot (Python), RingCentral presence and Audit Trail |
| **Area** | Phone Bot lost-call alerts: who was available when a call rang |
| **Related** | DDT-1075 (device-offline check on the same alert line) |
| **Code** | PR #13 in the Phone Bot repo (`driverdefenseteam-sf-ops-bot`), commit `6ac2449` |

---

## 1. Summary

**DDT-922** asked for lost-call alerts that name a rep to say *why* the rep didn't pick up, by adding their presence at the moment the call rang: "On a Call", "Was Available", "On DND", "In a Meeting" or "Offline". It shipped on 2026-09-30, with a firm rule: **no snapshot at ring time means no label**, never a guess.

**DDT-1129** questioned the presence data behind it: a snapshot recorded a rep as DND when she says she was available. Checking RingCentral's Audit Trail showed **the snapshot was right**. Her queue status had been set to "Do not accept any calls" the day before, which is a separate setting from the DND switch she was looking at.

## 2. What was reported

- **DDT-922, 2026-09-02.** The operations lead asked for a presence suffix on alerts such as "NO ANSWER — (rep)", so readers can tell "rep was on another call" from "rep was free and didn't answer". Being a new feature, it waited for approval, which came on 2026-09-25 with two conditions (section 6).
- **DDT-1129, 2026-09-25.** On a missed call that afternoon, the Phone Bot's presence snapshot for a defense case-management queue recorded one rep as `dnd`, about six seconds before the call. She reported that her RingCentral status was not DND at the time. The concern: presence data is used to hold reps accountable, so a wrong snapshot flags the wrong person.

## 3. Scope

**In scope**

| Component | Change |
|---|---|
| `app/core/phonebot_lost_calls.py` | `_PRESENCE_LABELS` map and `_presence_suffix`; the suffix is added to each "rang out" rep on the alert |
| `tests/test_phonebot_presence_suffix.py` | New tests (section 7) |

**Out of scope**

- **Which extensions are polled for presence.** Only reps in the monitored queues have snapshots. Widening that is separate work; the formatter needs no change when it happens.
- **DDT-1129:** no change. It was not a Phone Bot fault.

## 4. How it works (background)

```
Phone Bot polls RingCentral presence for monitored queue members, about every 45 seconds per extension
  → stores each snapshot with its timestamp
Call is lost
  → the alert lists each rep the call rang and for how long
  → (new) for each rep, read the stored snapshot at the call's start time
  → append the label, or nothing if there is no snapshot in the tolerance window
```

**Two RingCentral settings both mean "don't ring me":** the user's DND switch, and the queue status set through Shadow Agent ("Do not accept any calls"). The second does not change the first.

## 5. Root cause

### DDT-922

Not a fault; a feature request. The alert named the rep but had no way to say whether they were busy or free.

### DDT-1129

The ticket's suspicion (polling lag, stale cache, or a misread presence code) **was wrong**. RingCentral's Audit Trail showed:

1. The rep's queue status had last been set the previous afternoon, via Shadow Agent, to **"Do not accept any calls"**.
2. It had no status change on the day of the call, so at the time of the call it was still blocking all calls.
3. The Phone Bot's `dnd` was therefore accurate, and the queue correctly skipped her; only one other rep was rung.
4. Her own RingCentral app showed "Available" and "DND: Off" because Shadow Agent does not update that setting.

## 6. Solution

### DDT-922

```python
_PRESENCE_LABELS = {
    "on-call":    "On a Call",
    "available":  "Was Available",
    "dnd":        "On DND",
    "in-meeting": "In a Meeting",
    "offline":    "Offline",
}

def _presence_suffix(ext_id, call) -> str:   # simplified
    # no extension id, no start time, unparseable time, lookup error,
    # no snapshot, or an unknown state  ->  return "" (no suffix)
    snap = state_store.get_presence_at(ext_id, start_timestamp)
    label = _PRESENCE_LABELS.get(_derive_state(snap))
    return f" ({label})" if label else ""
```

Rendered, for example: `Rang out: (rep) 1s (On a Call)`.

The two conditions set at approval, both enforced:

- **Presence at ring time, never presence now.** The lookup reads the historical snapshot for the call's start time. Current presence would look authoritative and be wrong for every alert more than a minute old.
- **No snapshot in the window means no suffix.** No fallback to "Was Available" or "Offline", and no guessing from the call outcome. A missing label reads as "we don't know", which is true. A wrong label ends up in a conversation about someone's performance.

One extra guard: **if the rep's extension number cannot be resolved, no label is added.** In that case the device-offline check from DDT-1075 cannot run either, so it is not known whether the phone even rang, and "Was Available" on a phone that never rang is exactly the wrong claim.

Human labels, not raw API values, because the label should say what happened, not what the system stores.

### DDT-1129

No change. The finding was recorded on the ticket and it was closed as not a Phone Bot bug.

## 7. Testing

**DDT-922.** Seventeen new tests; 1,038 passing in the suite. They cover: the human labels; each state rendering its label; no snapshot, no extension id, no start time, an unparseable start time and an unknown state each producing no suffix; a lookup failure never costing the alert; the lookup using the call's start time; the suffix landing on the rang-out line; the line being exactly as before when there is no snapshot; a device-offline rep getting no presence suffix; and an unresolvable extension getting no claim.

**Coverage, measured against production rather than estimated:** 32% of named-rep lost-call legs would carry a suffix, over a 28-leg sample. That is above the 20% bar set at approval. The limit is **not snapshot frequency**, which was effectively complete (about one per extension every 45 seconds over 30 days, 1 gap in 28). It is which extensions are polled: 64% of the misses were reps outside the monitored queues.

**DDT-1129.** Checked against RingCentral's Audit Trail for the rep's status changes over the two days around the call (section 5).

**Not covered:** the approval asked for two real rendered alerts in the PR, one "On a Call" and one "Was Available". Whether those were captured is not recorded in the ticket.

## 8. Step by step

**Adding the suffix (DDT-922)**

1. Find where the alert lists rang-out reps.
2. For each, read the stored presence snapshot at the call's start time, not the rep's current presence.
3. Map the raw state to a human label; return nothing for any state not in the map.
4. Return nothing on every missing input or lookup error.
5. Skip the label when the rep's extension can't be resolved.
6. Measure how many recent alerts would get a label before shipping; if it is low, fix polling coverage first.
7. Deploy the Phone Bot.

**Investigating a disputed presence value (DDT-1129)**

1. Note the rep, the snapshot value and its timestamp from the missed-call analysis.
2. Open RingCentral's Audit Trail for that user and look for status changes before the call, including queue status set through Shadow Agent.
3. Check which reps the queue actually rang. A rep that was skipped was blocked, whatever their own app showed.
4. Explain the difference between the DND switch and queue status to the rep.

## 9. Rollback

**DDT-922:** revert PR #13 and redeploy the Phone Bot. Safe; alerts go back to naming reps without a label. **DDT-1129:** nothing to undo.

## 10. Known limits

- **Only monitored queue members have snapshots,** so most named reps get no label today (about a third do).
- **The label is the presence at ring time,** not over the whole ring. A rep who changes state during a long ring shows the state at the start.
- **"On DND" can mean either setting.** A rep blocked by queue status set through Shadow Agent shows as DND even though their own app says DND is off. Expect that question again.

## 11. Reuse for similar issues

**How to recognise one:** a request to add context to an alert, or a dispute that a recorded state was wrong.

- Read state **as of the event**, from history, never current state.
- When the data is missing, show nothing. A blank is honest; a guessed label about a person's performance is not.
- Measure how often the enrichment would actually appear before shipping it.
- When someone disputes a recorded state, check the source system's audit trail before suspecting the poller. Two settings that both block calls can disagree on screen.
