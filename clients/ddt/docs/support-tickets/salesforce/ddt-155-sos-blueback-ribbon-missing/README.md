# SOS cases missing the Blueback approval-stage ribbon

| | |
|---|---|
| **Ticket** | DDT-155 |
| **Project** | Support tickets |
| **Status** | Done 2026-06-16 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-06-11 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (report, Court Case record page) |
| **Area** | Blueback approval process for SOS (Secretary of State hearing) cases |
| **Related** | None recorded |
| **Code** | None. Handled by explanation and a report change in Setup |

---

## 1. Summary

SOS Court Cases were piling up in the "You're Up Next Blueback Report" with no ribbon on the record to move them through Blueback approval. The ribbon was not missing by mistake: the new Blueback process no longer uses the **Blueback Stage** field, but that report was still built on it. The firm asked for SOS cases to be taken out of the report, and that was done.

## 2. What was reported

On 2026-06-11 staff reported that SOS Court Cases (for example the SOS Formal Hearing type) had no Blueback approval-stage ribbon of the kind other case types have. On one example case the Blueback tab showed Copies of Tickets and Description done and the Representation Agreement signed, but **Intake Complete** was blank with no way to set it from the page. The path at the top of the record was the SOS Formal Hearing stage path, not the Blueback ribbon.

These cases were accumulating in the "You're Up Next Blueback Report" with no next step anyone could take. How many cases were affected is not recorded in the ticket.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| You're Up Next Blueback Report | Report | SOS cases removed from the report |

**Out of scope**

- **Adding a Blueback ribbon to SOS record types.** The ticket asked for it, but the investigation showed the report, not the page, was out of step with the new process.
- **Redesigning the report around the new Blueback process.** The firm was asked how the report should work under the new process; the answer recorded was to remove SOS cases.

## 4. How it works (background)

```
Old Blueback process
  -> Blueback Stage field on the case
  -> "You're Up Next Blueback Report" lists cases by Blueback Stage
  -> a ribbon on the case moves Blueback Stage forward

New Blueback process
  -> Blueback Stage field no longer used
  -> the report still filtered on Blueback Stage
```

## 5. Root cause

The support team's finding on 2026-06-15: the new Blueback process no longer uses the **Blueback Stage** field, but the "You're Up Next Blueback Report" was still based on that field. That is why the SOS cases appeared in the report, and why there was no ribbon on them to move them through approval stages: the stage the report was waiting on is not one anything moves any more.

**The ticket's first guess was wrong.** It assumed the SOS record types were missing a ribbon that other case types had. The page was fine; the report was reading a retired field.

## 6. Solution

The firm replied that if SOS is not going through Blueback approval, all SOS cases should be removed from the "You're Up Next Blueback Report". The support team confirmed on 2026-06-16 that this was done.

The exact filter added to the report is not recorded in the ticket.

## 7. Testing

The completion was confirmed in the ticket thread on 2026-06-16. No query or count was recorded.

**Not covered:** whether other reports or list views still filter on the retired Blueback Stage field is not recorded in the ticket.

## 8. Step by step

1. **Open the report** the cases are stuck in and read its filters. Note which field drives it.
2. **Confirm whether that field is still part of the current process**, and whether anything on the record page still updates it.
3. **Ask the firm how the report should behave** under the current process. Record the answer on the ticket.
4. **Edit the report filter** in Setup's report builder, here to exclude SOS case types, and save.
5. **Re-run the report** and confirm the SOS cases no longer appear.

## 9. Rollback

Remove the added filter from the report. Safe: it is a report change only, and no record data was changed.

## 10. Known limits

- **The report still depends on the retired Blueback Stage field** for the case types it still shows, as far as the sources say.
- **Any other report built on Blueback Stage** would show the same stuck cases.

## 11. Reuse for similar issues

**How to recognise one:** records "stuck" in a report or queue with no button to move them on, shortly after a process change.

- **Check what field the report filters on** before adding UI to the record. A report built on a retired field shows records as waiting for a step that no longer exists.
- **When a process stops using a field, search for every report, list view and flow that still reads it.**
- **Ask how the report should work** rather than guessing; here the answer was a filter, not a feature.
