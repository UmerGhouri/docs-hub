# Measure what Local Services Ads returns, and say how much is assumed


| | |
|---|---|
| **Ticket** | DDT-1014 |
| **Project** | Marketing Attribution |
| **Status** | Done 2026-09-24 |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-09-14 |
| **Resolved by** | Support team |
| **Systems** | Google Ads (Local Services Ads), Salesforce, RingCentral, CallRail |
| **Area** | Marketing reporting: return on the largest ad channel |
| **Related** | DDT-816, DDT-817, DDT-818, DDT-822, DDT-824, DDT-826, DDT-962, DDT-1082 |
| **Code** | No code change |


---


## 1. Summary

Google Local Services Ads (LSA) was about 87% of the firm's Google spend and nothing measured what it returned. This ticket asked for one written answer: what does the firm get back for LSA spend, and how much of that answer is measured versus assumed? The answer was written on 2026-09-22 from re-read Google Ads and Salesforce figures, with the gap stated alongside it. No uploader to Google was built, because Google offers no conversion upload for LSA. Charged-lead feedback turned out to be handled already by a lead-rating tool built by an outside partner.

## 2. What was reported

- 2026-09-14: LSA is the firm's single largest marketing line and the only major one with no ticket. Conversion tracking reads "not conversion tracked" on all three accounts, and the cost per signed client looks like the best in the firm while 50 of August's 270 signings carry no channel at all. Measure LSA's return inside Salesforce, decide on charged-lead feedback, and do not finish with a clean cost-per-client figure without the gap attached.

## 3. Scope

**In scope**

- Re-reading spend, charged leads and signings for August and the trailing 30 days.
- Writing down how an LSA credit is measured end to end in Salesforce, with its blind spots.
- A recommendation on charged-lead feedback (mark booked / dispute).

**Out of scope**

- Any upload of outcomes to Google: Google accepts no offline conversion upload for LSA (settled on 2026-08-23 with three Google sources). Feedback affects credits, not ranking.
- A scope line about accepting "customer data terms" on the LSA accounts: removed on 2026-09-22. No source supported it; that requirement belongs to the PPC offline-conversion upload, a different system.
- Platform signal policy for Google and Meta feeds: DDT-962.

## 4. How it works (background)

Three phone lines in the tracking-number registry carry LSA calls, one per office (Chicago, Stone Park, Villa Park), each mapped to a `Google LSA - <office>` Campaign. A Lead or Court Case created from one of those calls gets that token on `Automated_Source__c` or its Campaign. The attribution engine reads the token through `Channel_Mapping__mdt` and stamps `Attributed_Channel__c = LSA` with basis Dictionary Token. Since 2026-09-15 each call also writes an `Inbound_Contact__c` touch (DDT-818), and since 2026-09-22 each inbound call carries a lookup to its tracking line (DDT-1082).

91% of Chicago LSA calls never touch CallRail: they go from Google straight to RingCentral. Salesforce is therefore the only place LSA return can be measured.

## 5. Root cause

Not applicable: this was a measurement question, not a defect. The underlying gap was that no ticket owned LSA return, and that the one quotable number (cost per signed client) rested on attribution that a fifth of signings did not carry.

## 6. Solution

The written answer, posted 2026-09-22:

- **Spend and leads (Google Ads API, read 2026-09-22).** August: 310 charged leads. Trailing 30 days: 234 charged leads. Trailing spend was 31% below the ticket's figure because two offices were cut after August (Stone Park spend down 89%, Villa Park down 70%).
- **Signings (Salesforce, by hire date).** August: 270 signings, 74 with LSA credit. Trailing 30 days: 259 signings, 60 with LSA credit. The August spend per LSA signing re-read exactly; the trailing figure was about 15% lower. Money collected to date on those signings already exceeded spend (1.7 times and 2.1 times), but that is money in the door, not lifetime value.
- **Calls.** 299 inbound calls on the three LSA lines in the trailing window against Google's 234 charged leads; Google does not charge repeat callers, Salesforce counts every call.
- **How credit is earned.** Of LSA-credited records in the last 60 days, 333 had basis Dictionary Token and 8 Returning Client; nothing in the LSA bucket was a pattern guess.
- **Measured:** spend, charged leads, calls per line, signings carrying the LSA token, money collected to date. **Assumed:** that a client whose first token was an LSA line was won by LSA (first touch, no assist model), that nobody in the Unknown bucket is an LSA caller, and that Google's "charged" is right.
- **The gap.** 41 of 259 trailing signings (16%) carry no channel; August was 39 of 270 after a recompute (down from 50). Of the 41: 7 returning-client re-hires, 6 "Google / Online Search" on manually created Court Cases, 4 referred by an employee, 4 referred by an outside attorney with no source. If all 6 Google ones were LSA, the trailing spend per signing would fall by about another 9%: that is the width of the uncertainty. Per-office figures flatter the cut offices because signings lag calls. Nothing before 2026-05-01 is used.
- **Charged-lead feedback.** Recommended a monthly review by the intake lead. Firm leadership answered on 2026-10-12 and 2026-10-13: an outside partner had built LSA lead rating through Google's lead feedback API (an AI drafts each rating and a person approves it), rolling out to intake. Google allows one rating per lead within 30 days, so a monthly pass would miss part of every month. No monthly pass, nothing to build.

## 7. Testing

Not applicable: no code. Every figure carries its read date; the account conversion-tracking status could not be re-read with the available API tool and is stated as an assumption.

## 8. Step by step

1. Re-read spend and charged leads from the Google Ads API for fixed windows; write the read date next to each number.
2. Count signings by hire date for the same windows, all channels and LSA-credited.
3. Count inbound calls on the LSA tracking lines for the same windows.
4. Check the basis of every LSA credit (dictionary token versus guess).
5. Count signings with no channel and break them down by how-heard answer.
6. Write the answer with "measured" and "assumed" listed separately and the gap stated in the same message.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- First-touch by construction: there is no assist model.
- The tracking-line lookup on calls was one week deep at the time, and the history backfill was still pending.
- Spend per signing by office is distorted for offices whose budget was cut mid-window.

## 11. Reuse for similar issues

- When a channel looks like the best performer, check how many signings carry no channel before quoting it.
- Always state the window and read date with a cost figure; a monthly cost against a 90-day count is a common error.
- Before building a feed to an ad platform, confirm the platform accepts one.
- Check whether a requested manual process is already covered by another tool before scheduling staff time.
