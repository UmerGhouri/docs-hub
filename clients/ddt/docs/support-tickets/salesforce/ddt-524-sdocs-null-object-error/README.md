# S-Docs generation fails with a null object error

| | |
|---|---|
| **Ticket** | DDT-524, DDT-565 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-28 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-07-27 |
| **Resolved by** | Support team |
| **Systems** | Salesforce, S-Docs document templates |
| **Area** | Generating opening documents (the Cook PTRSSS petition) on a newly hired DUI court case |
| **Related** | DDT-1437 (later removal of the dead S-Docs buttons) |
| **Code** | None. The fix was made in the org, not in the repo. |

---

## 1. Summary

Clicking **Generate** on the S-Docs template **Cook PTRSSS (NEW)** failed with "Attempt to de-reference a null object" on the S-Docs page `sdoc:sdcreate2`. It was first reported on one court case, fixed in production, and then reported again the next morning on two more newly hired DUI cases. The support team fixed it again the same day. The exact cause and the change made are not recorded in either ticket.

## 2. What was reported

- **DDT-524, 2026-07-27.** A staff member could not generate the opening documentation for a newly hired DUI client. Refreshing the page did not help. Asked which template failed, they named **Cook PTRSSS (NEW)**. The support team reported it fixed on 2026-07-28.
- **DDT-565, 2026-07-28.** A second staff member hit the same error on the same template for **two** more DUI cases, both hired on 2026-07-27, and described it as a repeat of the earlier report. The support team reported it fixed about an hour after the ticket was opened.

The full error text, identical in both tickets:

```
Error occurred while attempting to write data back to Salesforce: Attempt to de-reference a null object
Error is in expression '{!step3}' in component <apex:commandButton> in page sdoc:sdcreate2. (SDOC)
An unexpected error has occurred. Your solution provider has been notified. (SDOC)
```

Why it matters: the PTRSSS petition is part of the paperwork for a newly hired DUI client, so a failed generation blocks case setup.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Cook PTRSSS (NEW) | S-Docs template | Fixed so it generates. The exact change is not recorded in the ticket. |

**Out of scope**

- **The ddt-salesforce repo.** S-Docs templates are stored as records in the org, not as metadata, so there is nothing in git for this fix.
- **Other S-Docs templates.** When asked whether any other template failed, the reporter confirmed it was the PTRSSS template.

## 4. How it works (background)

```
Staff member opens the court case (Opportunity) in Salesforce
  -> starts S-Docs and picks a template
  -> S-Docs page sdoc:sdcreate2 merges the record's fields into the template
  -> clicking Generate runs the page's step3 action
  -> S-Docs writes the generated document back to Salesforce
```

`sdoc:sdcreate2` is a Visualforce page inside the S-Docs managed package. Its code cannot be read or changed by the firm. What the firm controls is the template and the data on the record.

## 5. Root cause

Not recorded in the ticket. The ticket listed the likely causes to check: a missing template mapping, a broken merge field, or a related record (Contact, Account or Matter) not linked to the court case.

What the error itself tells you: it names the button's `{!step3}` action and says the failure happened "while attempting to write data back to Salesforce". So the template opened and merged, and something null was hit at the generate and save step. Which field or setting was null is not recorded.

The second report the next morning shows that the first fix did not cover every case using the template. Whatever was changed on 2026-07-28 for the first case had to be applied again for the next two.

## 6. Solution

The support team corrected the **Cook PTRSSS (NEW)** template (or the data it reads) in production on 2026-07-28, twice: once for the first case and again after the second report. The reporter of the first ticket confirmed that afterwards the template was generating.

The exact change is not recorded in the ticket.

## 7. Testing

- **Live check:** the first reporter confirmed on 2026-07-28 that Cook PTRSSS (NEW) was working.
- **Not covered:** no test was recorded for the two cases in DDT-565 beyond the support team's "fixed" comment, and no regression check across other cases using the template was recorded.

## 8. Step by step

The ticket does not record the exact fix, so this is the order to diagnose the same error again.

1. **Get the template name.** Ask which S-Docs template was being generated. The error page doesn't say.
2. **Reproduce it** on the reported record, then on a second record that uses the same template. If only some records fail, compare what those records have blank that the working ones have filled in.
3. **Open the template** in S-Docs and check every merge field and related-object reference against the record. A lookup that is blank on the record, or a field that has been renamed or deleted, is the usual source of a null value.
4. **Check the template's write-back settings**, since the error says it failed while writing back to Salesforce.
5. **Fix the template or the data**, then generate on every reported record, not only the first one.
6. **Ask the reporter to confirm** with a freshly generated document.

## 9. Rollback

Not applicable: the change was a correction to a template that was failing. There was no working state to go back to.

## 10. Known limits

- **A follow-up was raised on the same thread and is not recorded as resolved.** After the fix, the reporter said the template ran, but the field for the officer's star number did not appear, and the document filled that spot with information from the sworn report instead.
- **The fix isn't recorded**, so if the error returns there is no record of what was changed last time.
- **S-Docs has since been retired.** The firm moved document generation to Dochly, and DDT-1437 removed the dead S-Docs buttons, so this template no longer runs.

## 11. Reuse for similar issues

**How to recognise one:** an S-Docs error naming `sdoc:sdcreate2` and `{!step3}`, with "Attempt to de-reference a null object".

- **The managed page is not the problem to fix.** The error is raised inside the S-Docs package, but the cause is almost always in the template or the record it reads.
- **Test more than the record in the report.** Here a fix for one case did not fix the next two the following morning. Check every reported record, and a couple of others using the template, before closing.
- **Write down what you changed.** Template fixes live in the org and leave no trace in git. A short note on the ticket is the only record the next person will have.
