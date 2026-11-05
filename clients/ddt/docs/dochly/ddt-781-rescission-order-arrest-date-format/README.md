# Cook County Rescission order printed the arrest date as YYYY-MM-DD

| | |
|---|---|
| **Ticket** | DDT-781 |
| **Project** | Dochly |
| **Status** | Live since 2026-08-21 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-08-20 |
| **Resolved by** | Support team |
| **Systems** | Dochly document generation |
| **Area** | Court filings: Cook County Rescission orders |
| **Related** | None recorded |
| **Code** | None. The fix was made in the Dochly template editor. |

---

## 1. Summary

The Dochly template for Cook County Rescission orders printed the arrest date in ISO format, `YYYY-MM-DD`. Court filings use the US format, `MM-DD-YYYY`. The template's arrest date merge field was changed to output `MM-DD-YYYY`.

## 2. What was reported

Every Cook County Rescission order generated from Dochly showed the arrest date as `YYYY-MM-DD`, so staff had to correct the date by hand before filing. The ticket says it reproduced on any generation of this template.

## 3. Scope

**In scope**

| Component | System | Change |
|---|---|---|
| Cook County Rescission order | Dochly template | Arrest date merge field now outputs `MM-DD-YYYY` |

**Out of scope**

- **Other templates.** Only this template was reported. The ticket asked that the other dates on the same template be checked; the result of that check is not recorded.

## 4. How it works (background)

```
User generates the Cook County Rescission order from a case record
  -> Dochly reads the arrest date from the record
  -> the merge field prints it in whatever format the template specifies
```

## 5. Root cause

Not recorded in the ticket. For context, `YYYY-MM-DD` is the ISO form in which Salesforce returns a Date value to other systems, so a merge field that prints the raw value without a display format would show exactly this.

## 6. Solution

The completion note on the ticket reads: "The arrest date on Cook County Rescission orders is now displayed in the MM-DD-YYYY format." The exact merge-field syntax used is not recorded.

## 7. Testing

Not recorded in the ticket. The acceptance criteria asked for a sample generation confirmed by the requester; whether that was done is not recorded.

## 8. Step by step

1. Open the Cook County Rescission order template in Dochly's template editor.
2. Find the arrest date merge field.
3. Set its output format to `MM-DD-YYYY`.
4. Check the other date fields on the same template use the same format.
5. Generate the order from a case and check the printed date.

## 9. Rollback

Restore the previous merge field in the template editor. That brings back the ISO date, so there is no reason to.

## 10. Known limits

- **The format is set per merge field.** Any other template that prints the arrest date, or any new date field added to this one, needs its own format set.

## 11. Reuse for similar issues

**How to recognise one:** a generated document shows a date as `2026-08-10` style instead of `08-10-2026`.

- **Set a display format on every date merge field** in a court document. Don't rely on the default.
- **When you fix one date on a template, check every date on it**, and on the sibling templates for other counties.
- **A Date and a Date/Time field can behave differently.** A Date/Time is converted to a time zone, so near midnight it can print a different calendar day. Check which type the template reads before you change it.
