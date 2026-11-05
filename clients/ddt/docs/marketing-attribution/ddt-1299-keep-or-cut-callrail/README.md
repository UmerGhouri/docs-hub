# Decision: keep or cut CallRail call tracking


| | |
|---|---|
| **Ticket** | DDT-1299 |
| **Project** | Marketing Attribution |
| **Status** | Done 2026-10-22 |
| **Priority** | Medium |
| **Reported by** | Firm leadership, 2026-10-10 |
| **Resolved by** | Support team |
| **Systems** | CallRail, Google Ads, RingCentral, Salesforce (scheduled click-id sync) |
| **Area** | Phone attribution: click ids for phone clients |
| **Related** | DDT-810, DDT-1133, DDT-1272, DDT-1286, DDT-1362 |
| **Code** | No code change |


---


## 1. Summary

CallRail, a paid call-tracking subscription, had a review date of 2026-11-17 to decide whether to keep or cancel it. This ticket collected what the decision had to weigh. The key finding was that CallRail is how phone clients get an ad click id: CallRail links a call on an ad number to the Google Ads click, and a nightly Salesforce job copies that click id onto signed clients so Google receives the conversion. Firm leadership decided on 2026-10-22 to keep CallRail, and the November review was dropped.

## 2. What was reported

- 2026-10-10: CallRail stays through the Meta launch, then gets a keep-or-cut decision on 2026-11-17. Record what CallRail does that nothing else does, whether phone Leads carry the ad click id, and what would have to move first if it were cut. Done when the decision is written on the ticket, and, if it is a cut, a ticket exists to move every live CallRail number first.

## 3. Scope

**In scope**

- The facts behind the decision and the decision itself.

**Out of scope**

- Any configuration change. Option 1 (keep) needed none.
- Adding number-swapping pools to the landing pages, mentioned as an optional add-on.

## 4. How it works (background)

- **Dynamic number insertion.** CallRail shows each website visitor a number tied to their visit, so a call can be traced to its source. RingCentral cannot swap numbers on a page, and Google's own forwarding numbers only cover Google Ads traffic. Recording, routing and transcripts are already paid for elsewhere.
- **How Salesforce credits a call.** Salesforce never reads CallRail's reports; it credits a call by which RingCentral line rang (see DDT-1272).
- **Click ids on phone clients.** A call to the number in an ad (a CallRail number) is linked by CallRail's Google Ads integration to the ad click, so CallRail holds that call's gclid. Each night `CallRail Click Id Sync` (a Salesforce job from an earlier PR) matches signed clients with no click id to those calls by phone number, and writes `GCLID__c` on the Opportunity, so the signed client can go back to Google as a conversion.

## 5. Root cause

Not applicable: this was a decision ticket, not a fault. One correction is recorded: the first version of the ticket (2026-10-09) said phone Leads carry no click id. That was wrong, and the description was corrected on 2026-10-10. In a one-week sample, 83 of 97 calls to the four ad numbers carried a gclid in CallRail, and in the 30 days before, 16 of the 20 Paid Search clients who signed by phone got a click id this way.

## 6. Solution

The two options weighed:

1. **Keep CallRail**, so phone clients keep getting click ids. Optionally add swapping pools to the landing pages.
2. **Cut CallRail**, after moving every live number (ad call extensions and the website and brand pools) to RingCentral lines or Google forwarding numbers, and accept that phone clients stop sending a click id back to Google. Google's own "Calls from ads" counts the call inside Google Ads but gives Salesforce no click id to send back when the client signs.

**Decision (2026-10-22, firm leadership): keep CallRail for the foreseeable future (option 1).** The cost is small for what it adds to lead tracking and reporting. The 2026-11-17 review is dropped.

Already in place before the decision: the three ad landing pages show a RingCentral line rather than a CallRail number, so they would keep working whichever option was chosen.

## 7. Testing

Not applicable: no change was made. Evidence at decision time: in the 30 days before 2026-10-22, 31 of 37 Paid Search clients who signed went to Google with a click id.

## 8. Step by step

1. List what the tool does that nothing else in the stack does.
2. Trace whether any data the business depends on (here, click ids for phone clients) flows only through that tool.
3. List every live number or asset that would go dead if it were cancelled.
4. Write both options with their consequences on the ticket and get the decision recorded there.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- Calls to a landing page's own fixed number carry no gclid; only calls to ad numbers do.
- If CallRail is ever cut, every live ad call extension and website pool must be moved first, or those numbers go dead on live ads.

## 11. Reuse for similar issues

- Before cancelling a tracking tool, check whether any nightly job or integration reads data only that tool captures. Here the main value was not CallRail's reports (unused) but the click id it holds for each ad call.
- Correct a wrong premise on the ticket itself, dated, so the decision is made on the right facts.
