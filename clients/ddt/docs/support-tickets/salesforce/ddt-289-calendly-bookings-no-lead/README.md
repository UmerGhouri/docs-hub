# Calendly bookings not creating Salesforce leads

| | |
|---|---|
| **Ticket** | DDT-289, DDT-375, DDT-1161 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-01 |
| **Priority** | High (DDT-375: Urgent) |
| **Reported by** | Firm staff, 2026-07-08 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow), Calendly managed package |
| **Area** | Intake: turning a Calendly consult booking into a Lead |
| **Related** | None recorded |
| **Code** | PR #574 in the `ddt-salesforce` repo (DDT-1161) |

---

## 1. Summary

Calendly bookings are meant to create a Lead in Salesforce within seconds. Over the summer of 2026 many did not. Two separate causes were found. From late June to late July the Calendly-to-Salesforce connection was down, and it was reconnected (DDT-289). Separately, Calendly renamed the paid-traffic strategy-session event, and the flow that creates Leads matched on the event **name**, so bookings for that event stopped creating Leads until the flow was changed to match on the event's stable **slug** (DDT-1161).

## 2. What was reported

- **DDT-289 (2026-07-08).** A consult booked in Calendly that morning produced an Event in Salesforce but no Lead. A global search on the caller's phone number found only the Event.
- **DDT-375 (2026-07-17).** Bookings on a staff member's SOS consult calendar were not creating records in Salesforce. Each should have created a Lead with a specific `Automated_Source__c` value. Flagged very urgent.
- **DDT-1161 (2026-09-30).** Bookings for the event type with slug `sc-strat-session-90` (the paid-traffic driving consult) still landed in `Calendly__CalendlyAction__c` but produced no Lead or Contact. The ticket counted 5 of 5 April bookers with Leads against 6 of 7 September bookers with none, and `LeadSource = 'Calendly'` volume falling 137 (May), 59 (June), 7 (July), then about 11 a month.

## 3. Scope

**In scope**

| Component | Type | Change | Ticket |
|---|---|---|---|
| Calendly ↔ Salesforce connection | Integration | Reconnected | DDT-289 |
| `CreateEventTemplateNew` | Record-triggered flow on `Calendly__CalendlyAction__c` | `Create_Lead` outcome also matches `Calendly__EventTypeSlug__c = 'sc-strat-session-90'`; activated as version 11 | DDT-1161 |
| `CreateEventTemplateNew` | Flow definition | Active version 10 → 11 | DDT-1161 |

**Out of scope**

- **Backfilling Leads for bookings missed during the outage or the rename.** DDT-1161 asked for it; a backfill is not recorded on any of the three tickets.
- **Routing flow error emails somewhere monitored.** Also asked for on DDT-1161; not recorded as done.
- **Rewriting the event-name allowlist.** The other event names were left as they were; only the slug condition was added.

## 4. How it works (background)

```
Invitee books a Calendly event
  -> Calendly managed package writes a Calendly__CalendlyAction__c record
  -> flow CreateEventTemplateNew (record-triggered)
       looks for an existing record (Lead, Contact, Account, Opportunity ...)
       decision "Do we have a Record?"
         Create_Lead: no record found AND event type is on the allowlist
           -> create the Lead, then the Event on it
         otherwise ("Record Found" path)
           -> create the Event only
```

Before the fix, the allowlist in `Create_Lead` was a list of exact `Calendly__EventTypeName__c` values, one condition per event name.

## 5. Root cause

### DDT-289: the connection was down

The support team found that **the Calendly integration to Salesforce had been down from June 25 to July 22**, so any meeting booked in that window did not create a Lead. It was reconnected and new bookings created Leads again. What broke the connection, and how it was reconnected, is not recorded in the ticket.

### DDT-375: SOS calendar bookings

Raised during the outage window and closed on 2026-07-27, after the reconnection. A separate root cause is not recorded in the ticket.

### DDT-1161: the event was renamed

- **The ticket's suspect was wrong.** It pointed at a flow swap in July (`CreateEventTemplate` deactivated, `CreateEventTemplateNew` activated) and a UTM value change, because the timing lined up with the drop.
- **The real cause:** around 2026-06-12 Calendly renamed the event from "...: 90% Success" to "...: 96% Success". `CreateEventTemplateNew` only creates a Lead when `Calendly__EventTypeName__c` is on its hardcoded allowlist, and the new name was not on it. So, per the PR #574 commit, **every new booker since 2026-06-22** fell through to the "Record Found" path and got an Event with no linked person and no Lead.
- **Why the slug fixes it:** `Calendly__EventTypeSlug__c` stayed `sc-strat-session-90` across both names, on 1,356 Calendly actions.

## 6. Solution

### 6.1 Reconnect the integration (DDT-289)

Reconnected on or about 2026-07-22. The steps used are not recorded in the ticket.

### 6.2 Match on the slug, not the name (DDT-1161)

One condition added to the `Create_Lead` rule of `CreateEventTemplateNew`, and the condition logic extended to include it:

```xml
<conditionLogic>(1 AND 2) AND (3 OR 4 OR ... OR 19 OR 20)</conditionLogic>
...
<conditions>
    <leftValueReference>$Record.Calendly__EventTypeSlug__c</leftValueReference>
    <operator>EqualTo</operator>
    <rightValue>
        <stringValue>sc-strat-session-90</stringValue>
    </rightValue>
</conditions>
```

Conditions 1 and 2 are "no existing Who record and no existing What record". Conditions 3 to 19 are the event-name allowlist. Condition 20 is the new slug match.

- **Slug, not another name.** Adding "96% Success" to the name list would have broken again on the next rename. The slug does not change when the event is renamed.
- **Activated as version 11.** Production's highest version was 10.

## 7. Testing

No automated test is recorded in the ticket or PR.

**Live check (DDT-1161, 2026-10-01):** a real booking created a Lead in 4 seconds, recorded on the ticket.

**Volume check (read from production on 2026-11-05).** Leads with `LeadSource = 'Calendly'` per month, grouped by UTC calendar month:

```sql
SELECT CALENDAR_YEAR(CreatedDate) y, CALENDAR_MONTH(CreatedDate) m, COUNT(Id) n
FROM Lead
WHERE LeadSource = 'Calendly' AND CreatedDate >= 2026-05-01T00:00:00Z
GROUP BY CALENDAR_YEAR(CreatedDate), CALENDAR_MONTH(CreatedDate)
ORDER BY CALENDAR_YEAR(CreatedDate), CALENDAR_MONTH(CreatedDate)
```

| Month (UTC) | Calendly Leads |
|---|---|
| 2026-05 | 137 |
| 2026-06 | 59 |
| 2026-07 | 7 |
| 2026-08 | 21 |
| 2026-09 | 12 |
| 2026-10 | 140 |
| 2026-11 (to the 5th) | 14 |

October, the first month after the fix, is back at May's level.

**Not covered:** whether each Calendly event type on the allowlist still matches its current name in Calendly; and the DDT-375 SOS calendar specifically.

## 8. Step by step

When a Calendly booking creates an Event but no Lead:

1. **Find the booking's `Calendly__CalendlyAction__c` record.** If there is none, the problem is the connection, not the flow (DDT-289). Check the Calendly connection.
2. **If the record exists, read `Calendly__EventTypeName__c` and `Calendly__EventTypeSlug__c`** on it.
3. **Open `CreateEventTemplateNew`** → decision "Do we have a Record?" → outcome `Create_Lead`. Compare the event name with the allowlist.
4. **If the name is not listed**, add a condition on `$Record.Calendly__EventTypeSlug__c` equal to the slug, and add its number to the condition logic.
5. **Save as a new version and activate it.** Note the active version number in the flow definition.
6. **Book a test meeting** on that event type and confirm a Lead appears within seconds.
7. **Re-run the volume query** in section 7 after a few days.

## 9. Rollback

Reactivate flow version 10 of `CreateEventTemplateNew`. That stops Leads being created for `sc-strat-session-90` bookings again, so there is no reason to do it unless version 11 itself fails.

## 10. Known limits

- **Every other event type still matches by name.** Renaming any of them in Calendly stops its Leads with no error.
- **A changed slug** (for example a new event type replacing this one) would also need a new condition.
- **Bookings missed during the outage and the rename** were not recorded as backfilled.
- **No alert** is recorded for the next time Lead volume drops; it was noticed by staff each time.

## 11. Reuse for similar issues

**How to recognise one:** an integration still writes its staging record, but the downstream record is missing, and the drop lines up with a change made outside Salesforce.

- **Check whether the staging record exists** before blaming the flow. No staging record means the connection; a staging record with no Lead means the flow's logic.
- **Match on stable keys** (slugs, ids, external keys), not on display names that someone can edit in another system.
- **A timing coincidence is not a cause.** DDT-1161's ticket blamed a flow swap that happened near the drop; the actual change was a rename weeks earlier.
- **Watch monthly volume** of records an integration creates. A drop from 137 to 7 a month is easy to see in a report; for the renamed event it went unseen from June to September.
