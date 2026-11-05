# Hourly call summary counts inbound only and uses answer rate

| | |
|---|---|
| **Ticket** | DDT-727 |
| **Project** | Support tickets |
| **Status** | Done 2026-08-19 (inbound-only change reported done 2026-08-13) |
| **Priority** | Medium |
| **Reported by** | Firm operations, 2026-08-13 |
| **Resolved by** | Support team |
| **Systems** | Phone bot, RingCentral call log, Slack |
| **Area** | The hourly call summary posted to the phone-bot Slack channel |
| **Related** | None recorded |
| **Code** | Phone bot repository, not the ddt-salesforce repo. No PR in ddt-salesforce. |

---

## 1. Summary

The phone bot's hourly summary counted inbound and outbound calls together, so its headline percentage was a "connected rate" that outbound dialling inflated. The purpose of the summary is to watch how the firm handles inbound demand, so it was changed to count inbound calls only and report an answer rate. Staff checking the summaries the same day raised further counting issues (faxes, internal extension-to-extension calls, and lost calls that could not be reconciled), which were added to the ticket.

## 2. What was reported

- **2026-08-13, original report.** The 8 to 9 AM summary said "12 calls · 12 answered · 0 lost (100%)", with every team at 100%. Of those 12 calls, 8 were outbound calls placed by staff and only 4 were inbound, all 4 answered. Mixing outbound into the totals inflated the answered percentage and could hide missed inbound calls behind outbound activity.
- **Same day, 10 to 11 AM summary.** "20 inbound calls · 18 answered · 2 lost" included an incoming fax. Staff asked that faxes be excluded from a voice-call summary.
- **Same day, 11 AM to 12 PM summary.** A staff member cross-checked the summary against RingCentral Call Analytics and found two problems:
  - **Missed calls not in the Lost list.** At least two inbound missed calls in that hour, both very short and both from callers marked Spam or Disqualified in Salesforce, did not appear in the summary's Lost count. If spam or short calls are excluded on purpose, the summary should say so, so its numbers reconcile with RingCentral.
  - **Internal calls counted as inbound.** Six extension-to-extension calls (internal transfers and warm-transfer legs) carried `direction: Inbound` in the RingCentral log and were counted in the inbound total.
  - The same staff member also asked why a particular number was listed as the hour's missed calls when that was not correct.
- **Same day, 12 to 1 PM summary.** "16 inbound calls · 14 answered · 2 lost (88%)": staff could not find the 2 lost calls and asked for a way to see which calls were counted as lost. Another incoming fax had been counted as an inbound call.

## 3. Scope

**In scope**

- The hourly summary's call set: inbound only.
- The headline metric: an answer rate on inbound calls, with "inbound" in the wording.
- The per-team breakdown (Other / Intake / Case Management) and the "Top pickups" section: inbound only.
- Follow-up asks added to the ticket: exclude faxes (`call_type=Fax`), exclude internal extension-to-extension calls, and show which calls were counted as lost so staff can reconcile against RingCentral.

**Out of scope**

- Showing outbound activity elsewhere (for example a separate "Outbound: N calls placed" line). The ticket asked for that to be confirmed with management before adding it; no decision is recorded.
- The individual LOST CALL alerts, which are a separate part of the bot.

## 4. How it works (background)

```
Each hour, the phone bot reads the RingCentral call log for the hour
  -> classifies each call (answered, abandoned, rang out, voicemail)
  -> groups calls by team
  -> posts "HOURLY: <start>-<end>" to the phone-bot Slack channel:
       Overall: N calls · answered · lost (percentage)
       Lost: abandoned · rang out · voicemail
       By team: ...
       Top pickups: ...
```

The RingCentral call log records every call leg with a direction. Outbound calls, incoming faxes and internal calls between extensions all appear in it alongside customer calls; internal calls can appear with `direction: Inbound`.

## 5. Root cause

The summary built its totals from the whole call log for the hour, with no filter on direction or call type. So outbound calls were counted as "connected" and raised the percentage, and faxes and internal extension-to-extension legs were counted as inbound customer calls.

Why the two short spam-flagged missed calls were missing from the Lost list is not recorded in the ticket. The staff member's note suggested a short-duration or spam filter might be dropping them, but that was not confirmed.

## 6. Solution

The summary was changed to show **inbound calls only** (reported done 2026-08-13). Management asked that the "recovered: this missed call was called back" note stay in the summary; whether it was kept is not clearly recorded in the ticket.

Which of the follow-up asks (fax exclusion, internal-call exclusion, listing the lost calls) were built before the ticket closed on 2026-08-19 is not recorded in the ticket. The code change is in the phone bot's own repository, which is not available to this write-up.

## 7. Testing

The acceptance criteria asked for the change to be verified against a known hour that mixes inbound and outbound calls. Whether and how that was done is not recorded in the ticket.

The staff cross-checks against RingCentral Call Analytics recorded in section 2 are the closest thing to a test: they compared each summary's counts with the call-level log for the same hour.

## 8. Step by step

To check or repeat this kind of change:

1. **Pick a known hour** that has outbound calls, at least one fax, and at least one internal transfer.
2. **Pull that hour's call log from RingCentral** and list each call with its direction, type and result.
3. **Count by hand:** inbound voice calls from outside numbers only, then how many were answered.
4. **Compare with the bot's summary** for that hour. The Overall count, the answer rate and each team's numbers should match your count.
5. **For each lost call in the summary,** confirm you can identify the call in RingCentral. If the summary drops any category on purpose (spam, very short calls), it should say how many it dropped.

## 9. Rollback

Redeploy the previous version of the phone bot. The summary would then go back to counting outbound calls, faxes and internal calls in its totals.

## 10. Known limits

- Internal calls between extensions can carry `direction: Inbound` in RingCentral, so a direction filter alone does not remove them.
- Very short missed calls and calls from numbers marked as spam may be treated differently from other missed calls; if so, the summary needs to say so for its numbers to reconcile with RingCentral.
- Outbound activity is no longer visible in the hourly summary at all.

## 11. Reuse for similar issues

**How to recognise one:** a summary percentage that looks too good, or staff who cannot find the calls the summary says were lost.

- Decide what the metric is for (here, inbound customer demand) and filter the input to match: direction, call type, and external versus internal.
- Reconcile one real hour call by call against RingCentral before trusting a summary change.
- Any category a report drops on purpose should be counted and shown, so the numbers can be checked.
- Give staff a way to see which calls make up each count; a number nobody can reconcile gets ignored.
