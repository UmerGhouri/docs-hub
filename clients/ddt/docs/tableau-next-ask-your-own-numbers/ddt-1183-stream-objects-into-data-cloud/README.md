# Bring the Salesforce objects Tableau Next needs into Data Cloud


| | |
|---|---|
| **Ticket** | DDT-1183 |
| **Project** | Tableau Next: Ask-Your-Own-Numbers |
| **Status** | Live since 2026-10-21 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-10-02 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Data Cloud data streams, connector permission set, Digital Wallet) |
| **Area** | Self-serve reporting: data feed for Tableau Next |
| **Related** | DDT-1182, DDT-1185, DDT-1187 |
| **Code** | PR #749 and PR #788 in the ddt-salesforce repo (records of the connector's read access; nothing deployed) |


---


## 1. Summary

Tableau Next reads Data Cloud, not Salesforce directly, so every object behind an approved business term had to be streamed in first. Data streams were set up for Lead, Opportunity, Matter, Account, Disposition, the two Marketing Spend objects, the fw1 billing objects, Call Session, Call Leg, User, Task and a filtered slice of the trust ledger. Row counts in Data Cloud matched production exactly. Streams refresh about every 15 minutes. Credit-usage tracking was installed, and the yearly credit allotment was confirmed to be far above what the project uses.

## 2. What was reported

- 2026-10-02: stream Lead, Opportunity, Matter, Account, Contact, User, Call Session and Task into Data Cloud; leave out trust and payment data until firm leadership says otherwise. Done when each object shows a recent refresh, Data Cloud counts match production for Lead, Opportunity and Matter, and the refresh schedule is written down. Confirm streams do not use Scheduled Apex slots.

## 3. Scope

**In scope**

- Data streams created 2026-10-14: Lead, Opportunity, Matter, Account, Disposition, Marketing Spend, Marketing Spend Campaign, fw1 Payment, Invoice and Installment.
- Data streams created 2026-10-16: Call Session, Call Leg, Task, User and Trust Ledger Entry.
- Read access for the Data Cloud connector permission set on Call Session (19 fields), Call Leg (all 10 permissionable fields) and Trust Ledger Entry (only the fields Revenue needs). Later: View All Fields on Opportunity (PR #788).
- Credit tracking: the usage-event object added to the default data space and Salesforce's Consumption Tagging app installed.

**Out of scope**

- `Call_AI_Insights__c` (call transcripts and AI summaries): left out on purpose, as privileged client conversations.
- Task was streamed and then removed (see section 6).
- The 5 Google Ads streams added for marketing work share the same credits but belong to other work.

## 4. How it works (background)

A Data Cloud data stream copies one Salesforce object into a data lake object on a schedule. Data Cloud reads Salesforce through a connector permission set: an object or field it cannot read cannot be picked for a stream. A data space filter can limit which rows of a stream Tableau sees. Data Cloud bills in credits (AI credits and data-loading credits) against a yearly allotment.

## 5. Root cause

Not applicable: this was build work. Points that shaped it:

- The connector had no access to Call Session, Call Leg or Trust Ledger Entry, so they could not be streamed until read access was granted (2026-10-16).
- Firm leadership approved streaming the fw1 billing objects and a limited trust-ledger slice on 2026-10-15, because the approved Revenue term needs them. Normal users still cannot ask about them (DDT-1187).
- A plain SOQL count of Task leaves out archived and deleted activities, so the side-by-side count was kept to Lead, Opportunity and Matter as the ticket asked.

## 6. Solution

1. Granted the connector read access to the three new objects, field by field. Recording links and AI summaries on Call Session were left out. On the trust ledger, memo, client name and payee fields were not shared.
2. Created the streams. The trust-ledger stream has a data space filter (Debit, earned-fee payee category, not void), so Tableau sees only the 18 Revenue rows.
3. Installed credit tracking before any AI feature was switched on.
4. Removed the Task stream on 2026-10-20 on firm leadership's instruction: no approved term reads Task, and CRM streams have no pause option. Tasks in Salesforce were untouched.
5. An encrypted identity-number field on Account had come in through View All Fields. It was disabled on the stream the same day; in DDT-1187 the Account stream was rebuilt without it so no values remain.
6. After View All Fields was added on Opportunity, the Opportunity stream failed every refresh with a Data Cloud internal error (its staging table did not pick up the new columns). Salesforce Support asked for a Full Refresh; it ran twice on 2026-10-21 and the 15-minute refreshes resumed.

Why field-by-field access for the new objects: it keeps privileged and personal data out of a self-serve AI tool by default.

## 7. Testing

Row counts, Salesforce vs Data Cloud (Lead, Opportunity, Matter counted 2026-10-15; the rest 2026-10-16):

| Object | Salesforce | Data Cloud |
|---|---|---|
| Lead | 35,830 | 35,830 |
| Opportunity | 42,148 | 42,148 |
| Matter | 314 | 314 |
| Call Session | 68,726 | 68,726 |
| Call Leg | 261,821 | 261,821 |
| User | 197 | 197 |
| Trust Ledger Entry, Revenue rows | 18 | 18 |
| Task (SOQL count) | 449,592 | 449,591 |

- Refresh: batch streams every 15 minutes; the trust ledger and Marketing Spend Campaign stream on change.
- No Scheduled Apex slots used.
- Credit use after the first full day: about 3,160 credits for the year so far, 91% AI and 9% data loading. Most AI credits were used when the model was first indexed, not by questions (about 8 credits per question). That is about 1.2% of the yearly AI allotment.

## 8. Step by step

1. Agree with firm leadership which objects may go into a self-serve AI tool, and which stay out.
2. Grant the Data Cloud connector permission set read on each object, field by field where data is sensitive.
3. Create one data stream per object; add a data space filter where only some rows may be seen.
4. Compare row counts with SOQL for the main objects.
5. Write down each stream's refresh schedule.
6. Install credit tracking before switching on AI features.
7. Record every connector permission change in git the same day.

## 9. Rollback

Remove the connector's object and field permissions and delete the data streams in Data Cloud. Deleting a stream removes its loaded data. Nothing in Salesforce itself changes.

## 10. Known limits

- The connector permission set in git is a partial record: a Metadata API retrieve does not list every field grant. It is excluded from deploys.
- The cause of the Opportunity stream failure was asked of Salesforce Support; the answer is not recorded in the ticket.
- Adding many fields or objects to the model costs noticeably more credits than asking questions.

## 11. Reuse for similar issues

- Signs of the same problem: an object cannot be picked for a stream, or a stream loads 0 rows. Check the connector permission set first.
- A stream that loaded zero rows looks the same as a working one until you count. Always compare counts.
- After adding many fields to an existing stream, watch the next refreshes. If they fail with a new-columns error, a Full Refresh fixes it.
