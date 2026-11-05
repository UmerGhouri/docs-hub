# Adding a recipient to SOS court appearance notices

| | |
|---|---|
| **Ticket** | DDT-589 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-21 |
| **Priority** | Medium |
| **Reported by** | Attorney, 2026-07-30 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow), Google Calendar |
| **Area** | Court appearances: pushing SOS hearings to Google Calendar |
| **Related** | DDT-1365 (duplicate Google events, caused in part by this fix) |
| **Code** | None in the ddt-salesforce repo: the fix was a new flow built and activated in production |

---

## 1. Summary

An attorney asked to be added to the email and Google Calendar notices for SOS court appearances. The investigation found she was already the person who should receive them, so there was no list to add her to. The real problem was that some court appearances never produced a Google Calendar event at all.

The cause was a record-triggered flow that only fired on **update**, so appearances created with the attorney already filled in were skipped. A second flow, `Google_Calendar_Push_on_Create`, was added to cover the insert case.

## 2. What was reported

- **2026-07-30:** an attorney said she was not receiving calendar notices for SOS court appearances, by email or as Google Calendar invites, and asked to be added as a recipient.
- **2026-07-31:** the investigation found two separate problems. She was already set up as the right recipient. Some of her upcoming SOS hearings had **no** Google Calendar entry. The new flow's description records the count: **14 of 34** SOS hearings from July 2026 onward.

Until existing hearings were backfilled, the attorney was pointed to the Daily Docket view. It reads straight from Salesforce, so it shows every hearing on a given day whether or not it has a Google event.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Google_Calendar_Push_on_Create` | Record-triggered flow on Court Appearance (after save) | New. Fires on create when the Assigned Attorney is set, and hands off to the existing sync subflow |
| `Sync_Google_Calendar_Court_Appearance` | Subflow | Unchanged; reused by the new flow |

**Out of scope**

- **Adding the attorney to a recipient list.** Not needed: she was already the recipient.
- **Emailed invites.** Before changing anything that affects every attorney, the attorney was asked whether she had ever received an emailed invite. Her answer is not recorded in the ticket.
- **A daily or weekly email digest of upcoming hearings** was offered as an alternative. Whether it was wanted is not recorded in the ticket.
- **Backfilling events for hearings created before the fix.** The ticket says approval was being sought. Whether the backfill ran is not recorded in the ticket.

## 4. How it works (background)

Before the fix:

```
Court Appearance created or edited
  -> Google_Calendar_Push (record-triggered, after save)
       entry criteria: one of seven "Is Changed" conditions
  -> Sync_Google_Calendar_Court_Appearance (subflow)
  -> Google Calendar event created or updated, with the attorney as guest
  -> event id written back to GoogleCalendarId__c on the appearance
```

`Court_Appearance_After_Save` fills `Assigned_Attorney__c` when a record is created without one.

## 5. Root cause

**"Is Changed" is false while a record is being created.** `Google_Calendar_Push` required one of seven Is Changed conditions, so it only ever fired on an update.

That happened to work for most appearances. They were created with no attorney, `Court_Appearance_After_Save` filled one in, and that change triggered the push. An appearance **created with the attorney already set** never had a qualifying change, so it never got a Google event.

**The ticket's first guess was wrong.** It assumed a missing recipient. The recipient was right; the event didn't exist.

**Evidence.** The cause and the 14-of-34 count are recorded in the new flow's description in production. To see the two flows side by side:

```sql
SELECT ApiName, Label, IsActive, TriggerType, Description, LastModifiedDate
FROM FlowDefinitionView
WHERE ApiName IN ('Google_Calendar_Push_on_Create', 'Google_Calendar_Push')
```

## 6. Solution

A new record-triggered flow, `Google_Calendar_Push_on_Create` (label "Google Calendar Push (Create)"), on Court Appearance, after save:

- **Runs on create**, when `Assigned_Attorney__c` is populated and the appearance has no Google event id yet.
- **Calls the same subflow**, `Sync_Google_Calendar_Court_Appearance`, so event content and guest list match the existing path.

Version 1 was activated on **2026-07-31 at 12:15 UTC**, according to DDT-1365's deploy doc.

Why it was built this way:

- **A second flow rather than editing the first.** The existing flow's update criteria were left alone.
- **Attorney required at insert.** The intent, written into the flow description, was that when the attorney is blank, `Court_Appearance_After_Save` fills it and the original flow fires on that change, so the two paths "cannot both fire for the same record".

**That last assumption turned out to be wrong.** See section 10.

## 7. Testing

Not recorded in the ticket. There is no automated test for the flow, and the ticket's acceptance step (the attorney confirming receipt of the next notice) is not recorded either.

The later measurement in DDT-1365 shows the create path working: appearances created after activation did get Google events.

**Not covered:** emailed invites, and the backfill of hearings created before the fix.

## 8. Step by step

1. **Confirm who should receive the notices** before adding anyone. Check the Assigned Attorney on the affected appearances.
2. **Check whether the events exist.** A blank `GoogleCalendarId__c` on an appearance that already has an attorney means no event was ever created.
3. **Read the entry criteria** of `Google_Calendar_Push` in Flow Builder. If every condition is an Is Changed check, it cannot fire on create.
4. **Build `Google_Calendar_Push_on_Create`**: record-triggered, Court Appearance, after save, on create, with the conditions "Assigned Attorney is not blank" and "GoogleCalendarId__c is blank", calling `Sync_Google_Calendar_Court_Appearance`. Put the reason in the flow's description.
5. **Activate it**, then create a test appearance with an attorney already set and confirm an event appears on the calendar.
6. **Point the requester to Daily Docket** until any missing events are backfilled.

## 9. Rollback

Deactivate `Google_Calendar_Push_on_Create` in Setup → Flows. That brings the gap back: appearances created with an attorney get no event.

## 10. Known limits

- **This fix caused duplicate events.** DDT-1365 measured that, after activation, appearances whose attorney was filled by `Court_Appearance_After_Save` during the insert got **two** Google events. Both flows queued the sync subflow on the same re-save, both saw the event id blank, and both created an event. The twin rate was 77 to 88% every week, against none before. DDT-1365 fixed it by creating the event under an id derived from the appearance, so Google refuses the second create.
- **Emailed invites.** DDT-1365 also found that the create call did not actually ask Google to send notifications. That was inferred from the schemas, not checked in an inbox.
- **Hearings created before 2026-07-31** without an event stay without one unless they are backfilled.

## 11. Reuse for similar issues

**How to recognise one:** an automation works for records that are edited but silently skips records that are created already complete.

- **"Is Changed" is always false on create.** A record-triggered flow whose only entry criteria are Is Changed conditions never fires on insert.
- **Before adding someone to a notice, check the notice exists.** "I'm not getting it" can mean it was never sent to anyone.
- **Two flows calling the same async action on the same record can race.** If you add a second path, check whether both can fire in one transaction, and measure it with real data rather than reasoning about it. DDT-589's reasoning was sound on paper and wrong in practice.
- **Give users a reliable fallback** (here, Daily Docket) while the integration is being repaired.
