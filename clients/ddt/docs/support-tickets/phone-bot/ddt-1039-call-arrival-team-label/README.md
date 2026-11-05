# Team and door labels follow where a call arrived

| | |
|---|---|
| **Ticket** | DDT-1039, DDT-1041 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-17 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-09-16 and 2026-09-17 |
| **Resolved by** | Support team |
| **Systems** | Phone bot (lost-call alerts, daily recap, SPOKE-TO-HUMAN report), RingCentral call log |
| **Area** | Call reporting: which team a call belongs to, and which door it came in by |
| **Related** | DDT-942 (mainline handoff), DDT-1040 (recap table layout) |
| **Code** | Phone bot code, outside the `ddt-salesforce` repo. No PR number is recorded in the tickets. |

---

## 1. Summary

The phone bot labelled calls by where they ended up rather than where they arrived. A call to an attorney's direct line that rolled over to Case Management was counted as CMT, and every inbound call was filed as a direct dial because the mainline marker was only looked for on the first leg, where RingCentral almost never puts a name. Both functions now walk every leg of the call in route order, so the team and the door come from the point of arrival.

## 2. What was reported

- **DDT-1039, 2026-09-16.** A lost-call alert read `Route: <attorney> → Case Management` and `Team: CMT`. The call had arrived on an attorney's direct line and only reached Case Management because nobody picked up. Labelling it CMT credited that team with demand never aimed at them and understated the attorneys'.
- **DDT-1041, 2026-09-17.** The SPOKE-TO-HUMAN report showed `Mainline → user - (0 of 0)` next to `Direct dial → user 74% (166 of 225)`. Every inbound call was being filed as a direct dial, although 160 of those 225 came through the mainline.

## 3. Scope

**In scope**

| Component | Change |
|---|---|
| Team classifier in the phone bot | Walks the route in order; the first hop that identifies a team wins |
| `_classify_door` in the phone bot | Scans every leg for the mainline marker, not just the first |
| Phone bot tests | 9 new tests for the team classifier, 10 new for the door classifier |

**Out of scope**

- RingCentral routing itself. Calls still roll over the same way; only the labels changed.
- Report layout. The table formatting was fixed separately in DDT-1040.

## 4. How it works (background)

RingCentral records each call as a series of legs, one per hop:

```
Inbound  Accepted  864s  to.name=(empty)              <- legs[0]
Inbound  Missed      5s  to.name=Recording Announcement
Inbound  Missed      0s  to.name=DDT Router
```

The phone bot reads these legs to decide two things for every call:

- **Team**: which team the call was aimed at (Intake, CMT, Attorney, PI, Collections, or NONE).
- **Door**: whether it came in through the mainline or as a direct dial to a person.

## 5. Root cause

### Team label (DDT-1039)

Two bugs in the same function:

1. **It did not respect route order.** A call that hit an attorney's direct line and then a queue was labelled with the queue's team.
2. **It looped teams first, then legs.** So the winner was the first team in the code's *hint list*, not the first hop in the *route*. Intake sat first in that list, so a call that hit CMT and later touched anything Intake-shaped was filed under Intake whatever order the call actually took. This second bug was found while fixing the first.

### Door label (DDT-1041)

`_classify_door` read only `legs[0].to.name`. RingCentral leaves the first leg's name empty on nearly every call (5 of 225 carried one on 2026-09-16) and puts the greeting and router a hop or two further in. So the mainline marker matched **zero** times:

```
mainline marker on leg[0]   0
mainline marker ANYWHERE    160
```

## 6. Solution

### Team label

The route is walked **in order**, and the first hop that identifies a team wins, whether that hop is a queue or a person's direct line.

Real data forced two distinctions:

- **A queue hop counts even at zero seconds.** The call was routed there, however briefly. A first attempt that skipped all zero-duration hops cost a production call its team and sent it to NONE.
- **A person hop at zero seconds does not count.** That is the 55-millisecond mainline handoff from DDT-942, which carries a person's name. With the first hop deciding, letting it through would have put that artifact in charge of the whole call.

The reverse case still holds: a queue call answered by an attorney stays with the queue, because the queue was the first hop.

### Door label

Every leg is now scanned for the mainline marker, not just the first.

## 7. Testing

**Team label, measured over 225 production calls: 16 moved.**

```
CMT    -> ATTORNEY      x14
CMT    -> COLLECTIONS   x1
INTAKE -> CMT           x1    the ordering fix

ATTORNEY   5 -> 19      CMT   87 -> 73      NONE   10 -> 10
```

926 tests passing, 9 new, including the reported call (now reads `Team: ATTORNEY`) and the reverse case.

**Door label, same day after the fix:**

```
Mainline -> user          71%  (114 of 160)
Direct dial -> user       80%  (52 of 65)
```

952 tests passing, 10 new.

**Not covered:** whether each new test fails without the fix is not recorded in the tickets.

## 8. Step by step

1. Take a reported call and list its legs in order from the RingCentral call log.
2. For the team: find the first hop that identifies a team. Count a queue hop at any duration; skip a person hop at zero seconds.
3. For the door: search every leg's `to.name` for the mainline marker, not just the first.
4. Change the classifier to match, and add a test for the reported call and for its reverse.
5. Re-run the classifier over a full day of production calls and list every call whose label moved. Each move should be explainable.
6. Deploy the phone bot and check the next recap.

## 9. Rollback

Redeploy the previous phone bot version. That brings back both mislabelling bugs, so do it only if the new classifier itself misbehaves.

## 10. Known limits

- **The attorneys' share in the team breakdown roughly quadrupled** (5 to 19 calls in the sample). That is the correction landing, not a jump in call volume: those calls were always arriving on attorney direct lines. Anyone comparing recaps across 2026-09-17 will see a step change.
- The zero-second person-hop rule depends on the mainline handoff staying under a second. If RingCentral ever lengthens it, that hop would start deciding the team again.
- The door classifier depends on the mainline marker appearing in a leg name. Renaming the greeting or router in RingCentral would break it.

## 11. Reuse for similar issues

**How to recognise one:** a label that is right for where the call ended but wrong for where it started, or a row that is always zero.

- **Order matters.** When classifying a multi-leg call, walk the legs in route order and let the first decisive hop win. Never loop over categories first.
- **Never trust `legs[0]` alone.** RingCentral leaves the first leg's name empty on almost every call.
- **A row reading `0 of N` or `0 of 0` is usually a bug, not a measurement.** Count how often the marker appears anywhere before trusting it.
- **Re-run the classifier over a real day and list every call that moved.** That is how the ordering bug and the zero-second queue hop were found.
