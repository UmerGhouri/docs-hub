# Calendar event hover shows client, hearing type, judge and location

| | |
|---|---|
| **Ticket** | DDT-54 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-01 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-05-26 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (compact layout, Court Event automation) |
| **Area** | Calendar: court events seen by attorneys and staff |
| **Related** | None |
| **Code** | None. Configured in Salesforce; not in the ddt-salesforce repo |

---

## 1. Summary

Attorneys and staff had to open each court event on the calendar to see who it was for and where it was. The event's compact layout now shows **Client, Hearing Type, Judge and Location** when an event is clicked, and court event subjects are now formatted automatically so the key details show even when events are stacked on a busy day.

## 2. What was reported

Hovering over a calendar event showed none of the case context. The request was for four fields at a glance, client, hearing type (status, motion, trial, sentencing and so on), judge, and location (courthouse, courtroom or virtual link), without clicking into the event. Empty fields should be left out, not break the layout.

## 3. Scope

**In scope**

- **The `LegalRequest` compact layout**, which now shows Client, Hearing Type, Judge and Location.
- **Automatic formatting of the Court Event subject**, so the key details are visible in the subject on the calendar.

**Out of scope**

- **A custom hover component.** The ticket asked which calendar was in scope (DayBack, the standard Salesforce calendar or a custom LWC). The fix uses standard Salesforce configuration instead of building one.
- **A click-through link** from the preview to the Matter, an open question in the ticket that is not recorded as answered.

## 4. How it works (background)

```
Staff open the calendar
  → each court event shows its Subject on the calendar grid
  → clicking an event opens a preview built from the object's compact layout
```

A **compact layout** is the short list of fields Salesforce shows in record previews and highlights. Changing which fields it lists changes what every preview of that record shows, with no code.

## 5. Root cause

Not applicable: this was a feature request, not a fault. The preview simply listed fields that didn't help staff tell hearings apart, and stacked events on a busy day showed subjects without the key details.

## 6. Solution

Recorded on the ticket:

1. **The `LegalRequest` compact layout** was updated to show Client, Hearing Type, Judge and Location when an event is clicked.
2. **Court Event subjects are now formatted automatically**, so the important details are readable in the calendar even when several events are stacked.

Why a compact layout rather than a custom hover: it is configuration, needs no code or deploy, and applies wherever Salesforce shows a preview of that record.

The exact subject format and the automation that sets it are not recorded in the ticket, and neither component is in the ddt-salesforce repo.

## 7. Testing

Not recorded in the ticket. The acceptance criteria asked for checks across event types (criminal, civil, PI) and with empty fields; no results are recorded.

**Not covered:** no automated test. These are configuration changes.

## 8. Step by step

1. **Find the object behind the calendar's court events**, and its compact layout: Setup → Object Manager → the object → Compact Layouts.
2. **Edit the compact layout** (here `LegalRequest`) to list Client, Hearing Type, Judge and Location, and make sure it is the one assigned as primary.
3. **Set up the subject formatting** in the automation that creates or updates court events, so the subject carries the key details.
4. **Check on the calendar**: click an event of each kind and confirm the four fields appear, and that an event with a blank field still displays cleanly.
5. **Retrieve the changed components into git** in a PR, one component at a time, so the repo records them.

## 9. Rollback

Set the compact layout's field list back, or assign the previous compact layout as primary. Turn off the subject formatting in its automation. Neither change touches data already saved, except subjects already reformatted, which stay as they are.

## 10. Known limits

- **Not in git.** Neither the compact layout nor the subject automation is in the ddt-salesforce repo, so there is no record of exactly what was changed. A future production-to-git sync should pick them up.
- **Subjects are formatted when the automation runs.** Events created before it, or changed in a way the automation doesn't catch, keep their old subject.

## 11. Reuse for similar issues

**How to recognise one:** "I have to open every record to see X".

- **Try the compact layout first.** It controls previews and highlights and needs no code.
- **For calendars and other crowded lists, put the key details in the field the list displays**, usually Subject or Name, since only that is visible without clicking.
- **Record Setup changes in git** the same day, or the next person can't tell what was configured.
