# Google Calendar reminder sent to the wrong attorney

| | |
|---|---|
| **Ticket** | DDT-1324 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-21 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-10-13 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (record-triggered Flow), Google Calendar |
| **Area** | Court appearance calendar invites |
| **Related** | DDT-1342 |
| **Code** | PR #730 in the `ddt-salesforce` repo |

---

## 1. Summary

When a court appearance was reassigned to a different attorney, its Google Calendar event never updated. The previous attorney stayed on the guest list and kept getting reminders, and the new attorney was never added. The flow that pushes updates to Google only re-ran when a **formula** field changed, and Salesforce cannot detect a change on a formula field, so it never re-ran. The fix makes the flow watch the real attorney lookup field instead.

## 2. What was reported

On 2026-10-13 firm staff reported that Google Calendar had sent a reminder for a court appearance to an attorney who was neither the case's primary attorney nor the attorney assigned to that appearance. They asked whether this was a stale guest or a bug in how invites are built, and whether the right attorney was getting the reminder.

The reported appearance had been created assigned to one attorney, then reassigned to another about three weeks later with no other field changing. No calendar update fired, and the first attorney received the reminder.

On 2026-10-15, **114 of 662** upcoming court appearances had had their assigned attorney changed since they were created, so each could be carrying a stale guest.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Google_Calendar_Push` | Record-triggered flow on `Court_Appearance__c` | Entry-criteria filter 4: `dbk_AttorneyName__c` IsChanged → `Assigned_Attorney__c` IsChanged |

`filterLogic` (`1 AND 9 AND (2 OR 3 OR 4 OR 5 OR 6 OR 7 OR 8)`) and every other filter are unchanged.

**Out of scope**

- **Backfilling the 114 stale appearances.** Held: each correction emails a real attorney a calendar notification, so the firm was asked how it wanted that done (small batches rather than all at once). The decision is not recorded in the ticket.
- **Event titles not updating when a case is renamed.** Filter 7 has the same formula-field problem, but no entry criterion on this object can fire on a rename at all (see 10). Left for a separate ticket.
- **Other defects found while investigating.** Listed in section 10, each for a separate ticket.

## 4. How it works (background)

```
Court_Appearance__c saved
  -> Google_Calendar_Push checks its entry criteria
     (one of the watched fields must have changed)
  -> calls Sync_Google_Calendar_Court_Appearance
  -> PATCHes the Google event with the full attendees list,
     built from the appearance's assigned attorney
```

Because the update sends the **full** attendee list, re-running the sync replaces the guests. The old attorney drops off and the new one is added. The problem was only that the sync never re-ran.

## 5. Root cause

Filter 4 of the flow's entry criteria was `dbk_AttorneyName__c` **Is Changed**. `dbk_AttorneyName__c` is a formula field. Salesforce cannot evaluate Is Changed on a formula: the prior-record snapshot is not recalculated, so the condition never becomes true.

So reassigning `Assigned_Attorney__c` changed the formula's value, but the flow never saw a change, never re-synced, and the Google event kept whichever attorney was assigned when it was first created.

The ticket asked whether this was a stale guest or a logic bug. It was both: the guest was stale because of a logic bug.

## 6. Solution

One line in `Google_Calendar_Push.flow-meta.xml`. The field was `dbk_AttorneyName__c`:

```xml
<filters>
    <field>Assigned_Attorney__c</field>
    <operator>IsChanged</operator>
    <value>
        <booleanValue>true</booleanValue>
    </value>
</filters>
```

Why it's done this way:

- **Watch the lookup, not the formula built from it.** `Assigned_Attorney__c` is a real field, so Is Changed works on it.
- **Filter 1 stays as it is.** It is `dbk_AttorneyName__c` Not Equal To blank, a value check, which works on a formula.
- **Filter 7 was left alone after review.** The first version also changed filter 7 (`Case_Display_Name__c`, another formula) to `Opportunity__c`. Review found `Opportunity__c` is a master-detail that cannot be reparented, so it never changes after insert, and the swap replaced one dead condition with another. It was reverted, leaving the PR as the single line that fixes this ticket.

**Activation.** This org deploys flow versions inactive regardless of the status in the file (`enableFlowDeployAsActiveEnabled` is false). A Draft version means no calendar sync at all, so the deploy doc makes activating the new version in Setup a same-day step. The fix was reported live in production on 2026-10-15.

## 7. Testing

- **No Apex test covers this.** It is a flow entry-criteria change. The deploy used `RemoveHtmlMarkup_Test` only as the gate class required by `RunSpecifiedTests`.
- `CourtAppearancePanelControllerTest` was tried first and failed all six methods for an unrelated reason: it stamps `Court_Date__c` with the current time, and a validation rule only accepts court times within business hours, so it fails whenever it runs outside them. Logged separately.
- **Live check:** reassign the attorney on one upcoming appearance and open the Google event; the old attorney should drop off and the new one appear.
- On 2026-10-21 the reporter said they had seen no further wrong reminders.

## 8. Step by step

1. Confirm production's `Google_Calendar_Push` matches the repo before editing.
2. Change entry-criteria filter 4 from `dbk_AttorneyName__c` to `Assigned_Attorney__c`, operator still Is Changed.
3. Validate against production with `--test-level RunSpecifiedTests --tests RemoveHtmlMarkup_Test`.
4. Get the PR approved, then deploy `Flow:Google_Calendar_Push`.
5. **Activate the new version** in Setup → Process Automation → Flows → Google Calendar Push.
6. Confirm the newest version is Active and the previous one Obsolete:

    ```
    SELECT VersionNumber, Status FROM Flow
    WHERE Definition.DeveloperName = 'Google_Calendar_Push'
    ORDER BY VersionNumber DESC LIMIT 2
    ```

    (Tooling API query.)
7. Reassign the attorney on one appearance that genuinely needs it, and check the Google event's guests.
8. For stale events, re-run the sync per record in small batches, only once the firm agrees, because each one notifies a real attorney:

    ```apex
    Map<String, Object> inputs = new Map<String, Object>{ 'SelectedCourtAppearanceId' => recId };
    new Flow.Interview.Sync_Google_Calendar_Court_Appearance(inputs).start();
    ```

## 9. Rollback

Redeploy `Google_Calendar_Push` from the previous commit and activate that version. The change writes no data, so there is nothing to unwind. Rolling back brings the bug back.

## 10. Known limits

- **Appearances reassigned before the fix** keep their stale guests until each is re-synced (114 measured on 2026-10-15; re-measure before acting).
- **Event titles do not update when a case is renamed.** Renaming the case does no update on `Court_Appearance__c`, so no entry criterion on that object can catch it. It needs a different design, for example a flow on the case that touches its appearances.
- **No primary-attorney fallback.** If an appearance has no assigned attorney, the invite goes only to the firm's shared office calendar address.
- **Inactive users can still be chosen** as the assigned attorney.
- **Duplicate Salesforce `Event` records** exist, and some are not linked to the case.

## 11. Reuse for similar issues

**How to recognise one:** an automation that should run "when X changes" never runs, and X is a formula.

- **Never use Is Changed on a formula field** in flow entry criteria. Watch the real field the formula reads.
- **Check that the field you switch to can actually change.** A master-detail that cannot be reparented never changes after insert.
- **A rename of a parent** does not save its children, so a child-object trigger cannot react to it.
- **Remember flows deploy inactive here.** Activate the new version the same day, or the automation stops entirely.
- **When a fix needs a backfill that notifies people**, hold it and agree the pace with the firm first.
