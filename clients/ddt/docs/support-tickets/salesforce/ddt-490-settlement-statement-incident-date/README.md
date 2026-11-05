# Settlement statement incident date wrong and PDF download for non-attorneys

| | |
|---|---|
| **Ticket** | DDT-490 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-27 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-07-22 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, Flow, permission set) |
| **Area** | PI settlements: the settlement statement and its PDF |
| **Related** | None recorded |
| **Code** | Commit `5b761d85a` on `main` in the `ddt-salesforce` repo, 2026-07-23 (no PR number) |

---

## 1. Summary

Two separate problems on the PI settlement statement.

1. **The incident date was wrong.** The statement read the Matter's `Date_of_Incident__c`, which had drifted a day from the Incident record because the matter-creation flows converted the incident's date and time in GMT instead of the firm's Central time. The statement now takes the date from the Incident record in Central time, and the flows now stamp the Matter's date in Central time too.
2. **Only some users could download the PDF.** Access came from a permission set only admins had. It was assigned to the case team in the org.

## 2. What was reported

On 2026-07-22 a staff member showed, on one PI matter, that the settlement statement printed the wrong incident date, and asked for non-attorney case staff to be able to download the statement PDF without asking an attorney.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `SettlementStatementController` | Apex class | Statement incident date comes from the linked Incident, in Central time |
| `IncidentDateService` | Apex class (invocable) | New. Converts an incident date/time to the firm's Central calendar date |
| `IncidentDateServiceTest`, `SettlementStatementControllerTest` | Apex tests | New tests (see 7) |
| `Matter_Before_Save` | Flow | Calls `IncidentDateService` before stamping `Date_of_Incident__c` |
| `New_PI_Matter`, `Intake_to_Matter_Flow` | Flows | No longer set `Date_of_Incident__c` directly |
| `Settlement_Statement` | Permission set | Assigned to the case team in the org (no code change) |

**Out of scope**

- **Rewriting `Date_of_Incident__c` on existing Matters.** The commit stops new drift and makes the statement ignore the drifted value. A bulk correction of old Matters is not recorded.
- **The rule that a statement must be Finalized before its PDF can be generated.** Unchanged.

## 4. How it works (background)

```
PI_Incident__c.Incident_Date_Time__c   (a Date/Time: when the crash happened)
  -> matter-creation flows copy it into Matter__c.Date_of_Incident__c   (a Date)
  -> SettlementStatementController reads Matter__c.Date_of_Incident__c
  -> the statement and its PDF print that date
```

**The Salesforce rule behind this:** when a flow assigns a Date/Time to a Date field, it takes the calendar day in GMT or the running user's zone, not the firm's. The firm works in Central time, which is behind GMT, so an incident in the evening Central time is already the next day in GMT.

## 5. Root cause

1. The statement read `Matter__c.Date_of_Incident__c`.
2. That field was filled by `New_PI_Matter` and `Intake_to_Matter_Flow`, which assigned `Incident_Date_Time__c` straight into the Date field.
3. The implicit conversion used GMT or the running user's time, not Central time, so evening incidents landed on the next calendar day.
4. So the Matter's date had drifted from the Incident record, and the statement printed the drifted date.

**The PDF:** download access came from the `Settlement_Statement` permission set (it grants the settlement statement Apex classes, including the PDF controller), and only admins had it.

## 6. Solution

### 6.1 The statement reads the Incident, in Central time

```apex
private static final String FIRM_TIME_ZONE = 'America/Chicago';

out.dateOfIncident = resolveIncidentDate(m);

// The linked PI_Incident__c is the system of record, so its
// Incident_Date_Time__c wins when present, converted to Central time.
// Falls back to the Matter's own Date_of_Incident__c when no Incident
// is linked, or its date/time is blank.
private static Date resolveIncidentDate(Matter__c m) { ... }
```

The query now also selects `Incident__r.Incident_Date_Time__c`.

### 6.2 Stop new drift

`IncidentDateService` is an invocable method that makes the conversion explicit:

```apex
r.localDate = (req == null || req.incidentDateTime == null)
    ? null
    : Date.valueOf(req.incidentDateTime.format('yyyy-MM-dd', FIRM_TIME_ZONE));
```

`Matter_Before_Save` calls it and stamps its result into `Date_of_Incident__c`. `New_PI_Matter` and `Intake_to_Matter_Flow` no longer set the field themselves, so there is one place that sets it.

Why it's done this way:

- **Named time zone, not an offset.** `America/Chicago` handles daylight saving; a fixed offset would be wrong for half the year.
- **The same rule in both places.** The statement and the flow use the same conversion, so the stored date and the printed date agree.
- **Fallback to the Matter field** keeps matters with no Incident record working as before.

### 6.3 PDF download for the case team

No code change. The existing `Settlement_Statement` permission set was assigned to the case team (named in the ticket as DDT Matter Management). The statement still has to be **Finalized** before a PDF can be generated.

## 7. Testing

**Apex tests:**

| Test | What it proves |
|---|---|
| `IncidentDateServiceTest.midnightCentralKeepsSameDay` | A midnight Central time stays on its own day |
| `IncidentDateServiceTest.eveningIncidentResolvesToLocalDay` | An evening incident resolves to the Central day, not the GMT day |
| `IncidentDateServiceTest.daylightSavingAfternoonKeepsSameDay` | Daylight saving time is handled |
| `IncidentDateServiceTest.nullReturnsNull` | A blank date/time returns blank |
| `IncidentDateServiceTest.bulkPreservesOrder` | Several requests come back in order |
| `SettlementStatementControllerTest.getStatementData_incidentDatePrefersIncidentRecord` | The statement uses the Incident's date |
| `SettlementStatementControllerTest.getStatementData_incidentDateFallsBackToMatterField` | With no Incident date, it uses the Matter's field |

**Live check:** verified on the matter shown in the report; the statement then showed the Incident's correct date. `IncidentDateService` exists in production, created 2026-07-23 (queried 2026-11-05).

**Not covered:** existing Matters whose `Date_of_Incident__c` already drifted still hold the drifted value; other screens that read that field are not fixed by this change.

## 8. Step by step

1. **Compare the dates.** On the affected matter, compare `Matter__c.Date_of_Incident__c` with the linked Incident's `Incident_Date_Time__c` in Central time.
2. **Add `IncidentDateService`** and its test, as in 6.2.
3. **Change `SettlementStatementController`** to read the Incident's date through `resolveIncidentDate`, as in 6.1, and add the two controller tests.
4. **Edit the flows:** `Matter_Before_Save` calls `IncidentDateService` and stamps its output; remove the direct `Date_of_Incident__c` assignment from `New_PI_Matter` and `Intake_to_Matter_Flow`.
5. **Validate against production** with the new tests, get approval, deploy.
6. **Check the reported matter's statement** shows the Incident's date.
7. **For the PDF,** assign the `Settlement_Statement` permission set to the case staff who need it.

## 9. Rollback

- **Code and flows:** redeploy the previous versions of the class and the three flows. That brings the wrong date back.
- **Permission set:** remove the assignments. Case staff then lose the PDF download again.

## 10. Known limits

- **Old drifted values remain** in `Matter__c.Date_of_Incident__c` until corrected.
- **The time zone is hardcoded** to `America/Chicago` in two classes. If the firm's time zone ever changes, change both.
- **The PDF still needs a Finalized statement.**

## 11. Reuse for similar issues

**How to recognise one:** a date that is one day off, and only for events late in the day.

- **Check whether a Date/Time is being copied into a Date field.** Flows do this implicitly, in GMT or the running user's zone.
- **Convert explicitly** with `Datetime.format('yyyy-MM-dd', 'America/Chicago')`, not by casting.
- **Read the source record**, not a copy of it, when the copy can drift.
- **Set a field in one place** so two flows cannot disagree.
- **A "can't download" report is often a permission set assignment**, not code. Check which permission set grants the controller class.
