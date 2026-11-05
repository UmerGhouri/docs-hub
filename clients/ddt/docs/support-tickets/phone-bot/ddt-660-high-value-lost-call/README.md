# High-value label on lost-call alerts

| | |
|---|---|
| **Ticket** | DDT-660, DDT-742 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-17 |
| **Priority** | Medium |
| **Reported by** | Intake team, 2026-08-06 |
| **Resolved by** | Support team |
| **Systems** | Phone Bot service (hosted on Railway), Slack, RingCentral, Salesforce (caller lookup) |
| **Area** | Lost-call alerts posted to the phone-bot Slack channel |
| **Related** | DDT-661, DDT-754 (other lost-call alert fixes) |
| **Code** | Phone Bot service; not part of the `ddt-salesforce` repo |

---

## 1. Summary

The Phone Bot posts a Slack alert for every lost call, and some alerts carried a "**LOST CALL — high-value**" banner. Two reports showed the banner was misleading: first on a call from an insurance company, then on an ordinary traffic case. The label was never based on the case at all; it came from the line the call arrived on. Insurance and similar non-lead callers were filtered out of the alert, and the banner wording was changed to "new lead".

## 2. What was reported

- **DDT-660, 2026-08-06:** a lost call routed to PI was posted as high-value for the intake team. The caller was an insurance company held in Salesforce as a Contact, not a lead. Insurance calls on PI matters are not intake work, so the alert was noise.
- **DDT-742, 2026-08-14:** a lost call from an intake caller was posted as high-value although the case was a traffic misdemeanour. The reporter raised that if every intake call is "high-value", reps stop reacting to the label and real high-value misses get lost.

## 3. Scope

**In scope**

| Component | Change |
|---|---|
| Phone Bot high-value lost-call detection | Insurance companies, claims adjusters, underwriters and similar callers no longer trigger the alert (DDT-660) |
| Phone Bot lost-call alert template | Banner wording changed from "high-value" to "new lead" (DDT-742) |

**Out of scope**

- **A real definition of "high-value".** DDT-742 asked for one (case type, practice area, estimated fee). It was not built: at the moment a call is lost the case details are not known yet, so the label was made accurate instead of made smarter.
- **Routing PI-side calls to a separate channel.** Suggested in DDT-660 as an alternative; not recorded as done.

## 4. How it works (background)

```
Caller rings a firm line -> RingCentral routes the call
  -> nobody answers (abandoned / voicemail)
  -> Phone Bot sees the lost call, looks the number up in Salesforce
  -> posts a LOST CALL alert in Slack
       banner chosen from the line the call came in on
```

## 5. Root cause

The two tickets were the same problem seen from two sides.

- **The banner was keyed to the incoming line, not the case.** Every lost call that came in on an intake line got the high-value banner, whoever the caller was and whatever the matter. That is why a traffic misdemeanour was labelled high-value (DDT-742).
- **Non-lead callers were not excluded.** A caller already known in Salesforce as an insurance company still produced the alert (DDT-660).

The ticket's first framing (DDT-660) treated this as a PI routing edge case. The later ticket showed the label itself had no case information behind it.

## 6. Solution

### 6.1 Exclude non-lead callers (DDT-660)

The detection now skips callers that are insurance companies, claims adjusters, underwriters and similar. New leads are still flagged. The filter works from the caller's **Salesforce name**: to suppress another false positive, the support team adds that name to the filter.

### 6.2 Change the wording (DDT-742)

The banner now reads "new lead" instead of "high-value". The same calls are flagged as before. Why not build a real high-value test: the alert fires when the call is lost, before anyone has spoken to the caller, so the case type and value are not known yet. "New lead" says only what the bot actually knows.

## 7. Testing

Not recorded in the ticket. Neither ticket records automated tests or a sample review of recent alerts, although DDT-742 listed one as an acceptance criterion.

**Not covered:** confirmation from the intake team that the noise stopped. Not recorded.

## 8. Step by step

1. **Find what drives the label.** Read the Phone Bot's lost-call alert code and see which input selects the banner (here, the incoming line).
2. **Decide whether the bot can know what the label claims** at the moment it posts. If not, change the wording to what it does know.
3. **For specific false positives**, add the caller's Salesforce name to the exclusion filter.
4. **Deploy the Phone Bot** (Railway deploys it on push).
5. **Watch the next few lost-call alerts** in the phone-bot channel to confirm the banner and the exclusions.

## 9. Rollback

Revert the Phone Bot change and redeploy. Safe: it only changes Slack message text and which callers produce an alert. No data is written.

## 10. Known limits

- **The exclusion list is name-based.** A new insurance company or adjuster not yet on it will still produce an alert until someone reports it.
- **"New lead" is still a guess.** A known client calling about an existing matter on an intake line may still be labelled a new lead.

## 11. Reuse for similar issues

**How to recognise one:** an alert label that is "always on" for a whole category, or that staff say does not match reality.

- **Ask what the label is computed from.** If the input cannot carry the meaning of the label, the label is wrong, not the data.
- **Prefer an accurate plain label over a clever one** when the system does not have the facts yet.
- **A label that fires on everything trains people to ignore it.** Treat false positives as a real defect, not cosmetic.
