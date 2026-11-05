# Missed calls not reported by the phone bot

| | |
|---|---|
| **Ticket** | DDT-710, DDT-746, DDT-1336 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-12 (DDT-710) and 2026-08-17 (DDT-746); DDT-1336 Done 2026-10-15, no counting defect |
| **Priority** | High (DDT-710, DDT-746); none set (DDT-1336) |
| **Reported by** | Firm operations, 2026-08-11 (DDT-710) and 2026-08-16 (DDT-746); Firm staff, 2026-10-14 (DDT-1336) |
| **Resolved by** | Support team |
| **Systems** | Phone bot (hosted on Railway), RingCentral call log API, Slack |
| **Area** | Missed-call (LOST CALL) alerts and hourly call recaps |
| **Related** | DDT-709 (lost-call analysis, same theme, different bug) |
| **Code** | Phone bot repository, not the `ddt-salesforce` repo. No PR in `ddt-salesforce`. |

---

## 1. Summary

Three reports, over two months, that the phone bot was not reporting every missed call. They were not one bug:

- **DDT-710:** a missed call never reached the bot's classifier. Two ingestion defects were found and fixed: RingCentral's phone-number filter under-matched numbers written with a leading `+`, and the bot's call-log window was too small to survive a delayed poll.
- **DDT-746:** a rang-out call was counted in the hourly summary but got no LOST CALL alert, because the bot treated the hand-off to the call router as an answer.
- **DDT-1336:** not a defect. The recap that looked short was a team channel's recap, which only covers that team's calls; the firm-wide recap for the same hour matched RingCentral exactly.

## 2. What was reported

- **2026-08-11, DDT-710.** An inbound call that rang out at about 2:48 PM CT was missing from the bot's missed-call reporting, although it was in RingCentral. Attempts to look it up live returned `RingCentral 429: rate-limited` repeatedly, raising the question of whether the bot's own polling was being throttled and dropping calls.
- **2026-08-16, DDT-746.** The 2 PM to 3 PM hourly summary reported 1 lost call (rang out, 21 seconds, new caller), but no individual LOST CALL notice was posted for it. Without the individual notice, nobody knows to call the lead back.
- **2026-10-14, DDT-1336.** A 12 PM to 1 PM hourly recap said 6 inbound, 2 answered, 3 missed and 1 abandoned. RingCentral's call log showed 10 missed, voicemail or abandoned calls in that hour, and only 2 missed-call alerts appeared in the channel the reporter was watching.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `search_call_log` | Phone bot function | Strips the phone-number filter to digits before calling RingCentral (DDT-710) |
| `tick_sync` | Phone bot sync loop | Call-log window raised from 50 to 200 records; watermark check logs `INGESTION GAP` (DDT-710) |
| Missed-call analyzer | Phone bot | Reads the staffing engine's cached queue topology first, so monitored queues cost no RingCentral calls (DDT-710) |
| `analyze_missed_call` | Phone bot tool | Answers "why was this call not reported?" by reference id or by phone and time (DDT-710) |
| Call classification | Phone bot | A call counts as answered only when a person or queue picks up, not when it is handed to the router (DDT-746) |
| Bot's reply when comparing recaps | Phone bot | Corrected after it compared a team recap to a firm-wide list (DDT-1336) |

**Out of scope**

- **The `ddt-salesforce` repo.** None of these tickets changed Salesforce.
- **A recap-versus-RingCentral sanity-check script** (an acceptance criterion on DDT-1336). Not recorded in the ticket as built; the ticket was closed once the counts were shown to agree.
- **A backfill of other calls missed on 2026-08-11.** Not recorded in the ticket.

## 4. How it works (background)

```
RingCentral call log
  -> tick_sync polls the most recent N call-log records       (N was 50, now 200)
  -> _seen_call_ids skips calls already posted                (stops double-posting, not never-posting)
  -> classify() decides: answered / rang_out / abandoned / voicemail ...
  -> LOST CALL alert to the channel of the team the call belongs to
  -> hourly recap per channel (a team channel covers only that team's calls;
     the main channel covers the whole firm)
```

A call that never lands in the polled window is never classified, and nothing goes back for it later.

## 5. Root cause

### DDT-710: the call was dropped before classification

Run through the bot's own classifier, the call came out as `rang_out` with no leg connected, so **it should have posted**. It was lost earlier, in ingestion. Two defects were found:

1. **RingCentral's `phoneNumber` filter under-matches on `+`.** Measured against the live account, the same number returned 0 matches in E.164 form (`+1...`) and several in digits-only form, and the failure was inconsistent: some numbers returned a partial set, others nothing. The bot passed E.164 straight off call-log records. The clearest casualty was the REMINDER freshness check ("has this caller been reached since the miss?"): this caller had been called back ten minutes after the miss, and the bot could not see it.
2. **The ingestion window had no floor and no alarm.** `tick_sync` pulled 50 records per tick. At the volume that afternoon (about 1.5 calls a minute, 180 calls in the two hours around this one), 50 records covered roughly 33 minutes. Any tick delayed past that by RingCentral 429 backoff, slow Salesforce enrichment, lock contention or a redeploy (there were twelve deploys that day) let calls scroll out unexamined, and nothing logged it.

**The 429s were real and partly self-inflicted.** Widening the monitored queues from 4 to 20 took the topology refresh from 4 member fetches to 20 every 15 minutes, and the new missed-call analyzer added one more per route hop.

**What is not claimed:** that either defect caused this specific drop. That day's logs had rotated out of the hosting provider's buffer before the investigation. Both defects are in the ingestion path and either could have swallowed the call.

### DDT-746: the router hand-off counted as an answer

The bot treated the hand-off to the call router as an answered call, so a call that then rang out unanswered was not treated as lost and no LOST CALL notice fired. The resolution note does not record more detail than this.

### DDT-1336: two different scopes compared

**The ticket's guessed causes (timezone window, number filter, de-duplication by caller) were all wrong.** The recap in question was posted in a team channel, so it covered only that team's calls: 6 of the 27 that came in that hour. The reporter's list of 10 was firm-wide. The main channel's recap for the same hour showed 6 missed and 4 abandoned, exactly the 10, and all 10 alerts had gone to the main channel. Each team channel receives only its own team's misses. Nothing was dropped.

The one real fault was the bot's own reply in the thread, which compared the team recap to the firm-wide list and agreed it was wrong. That reply logic was fixed.

## 6. Solution

### DDT-710

- **Digits-only filter.** `search_call_log` strips the phone-number filter to digits. Digits-only returned a superset of the `+` form in every case tested.
- **Bigger window plus a gap alarm.** `tick_sync` now pulls 200 records (about two hours of headroom at that volume), and a watermark check logs `INGESTION GAP` when one polled window starts after the previous one ended. Before this, a gap was completely silent, so under-reporting could only be found by a person noticing one specific missing call.
- **Fewer RingCentral calls.** The missed-call analyzer reads the staffing engine's topology cache first.
- **Investigation tooling.** `analyze_missed_call` answers the question that the live lookup could not.

### DDT-746

Call classification counts a call as answered only when a person or a queue actually picks up. A router hand-off is no longer an answer.

### DDT-1336

No change to counting or alerting. The bot's comparison reply was corrected, and the explanation was given to the reporter, who accepted it.

## 7. Testing

- **DDT-710:** 737 phone bot tests passed. Verified live: the freshness query now returns both the miss and the callback, and `analyze_missed_call` was checked against the reported call by reference id and by phone and time. After the fix, one burst of 73 RingCentral 429 responses in a single minute was seen and recovered on its own.
- **DDT-746:** Not recorded in the ticket.
- **DDT-1336:** the main channel's recap for the hour was reconciled call by call against RingCentral's log (6 missed plus 4 abandoned equals the 10 listed).

**Not covered:** proof that either DDT-710 defect caused the original drop; a standing automated check that recap totals match RingCentral.

## 8. Step by step

When someone reports a missed call the bot did not post:

1. **Find the call in RingCentral's call log** and note its direction, result, duration and the number it came in on.
2. **Check which channel should have received it.** Team channels get only their team's calls; the main channel gets the whole firm. Compare like with like (DDT-1336).
3. **Run `analyze_missed_call`** with the call's reference id, or the phone and time, to see how the bot classified it.
4. **If it classified as lost but never posted**, look in the bot's logs for `INGESTION GAP` around that time (DDT-710).
5. **If it classified as answered**, check which leg the bot took as the answer. A router or announcement leg must not count (DDT-746).
6. **If a lookup by phone number returns nothing**, retry with digits only before concluding the call is missing.

## 9. Rollback

Redeploy the previous phone bot release on Railway. Doing so brings back the under-reporting, so there is no reason to unless a new release itself fails. DDT-1336 changed nothing to roll back beyond the bot's reply.

## 10. Known limits

- **RingCentral 429 throttling still happens under load.** It self-recovered during validation but was flagged as worth watching.
- **200 records is still a fixed window.** At much higher call volume, or a long outage, calls can still scroll out. The `INGESTION GAP` warning makes that visible but does not recover the calls.
- **Logs rotate quickly** on the hosting provider, so investigate a missed call the same day.

## 11. Reuse for similar issues

- **Before believing a count is wrong, confirm both sides cover the same scope**: same channel, same team, same time window.
- **Never trust an external API's filter on a formatted value.** Test the `+1` form against the digits-only form before relying on either.
- **A polling window needs an alarm.** De-duplication protects against double-posting; only a watermark check protects against never-posting.
- **"Answered" must mean a human or queue picked up.** Announcements, routers and IVR legs connect but are not answers.
- **Self-inflicted rate limits:** when you widen what the bot monitors, count the extra API calls it causes and cache where you can.
