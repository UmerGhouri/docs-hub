# Attorney's ARDC number missing from appearance forms

| | |
|---|---|
| **Ticket** | DDT-1408 |
| **Project** | Dochly |
| **Status** | Live since 2026-10-29 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-10-23 |
| **Resolved by** | Support team |
| **Systems** | Dochly document generation (template editor), Salesforce attorney data |
| **Area** | Court filings: appearance forms, by county |
| **Related** | DDT-1413 (preview and edit custom input fields before routing), DDT-1237 (same Attorney No. pattern on Motion to Withdraw) |
| **Code** | None. Dochly template changes and one data entry, so nothing is in git |

---

## 1. Summary

An attorney's ARDC number was added so it prints on appearance forms generated from Dochly. The first fix did not reach the Cook County forms, and the ticket was reopened to cover every appearance template where that attorney is the Primary Attorney. Along the way a second attorney turned out to have no ARDC number in Salesforce, so her forms printed a blank box; her number was entered in Salesforce and the forms now print it.

## 2. What was reported

Appearance forms generated from Dochly needed to include an attorney's ARDC number so they were complete and accurate.

How it developed on the ticket:

| Date | What happened |
|---|---|
| 2026-10-23 | The ARDC number was added in Dochly |
| 2026-10-23 | The requester reported it was not working for Cook County courthouses |
| 2026-10-26 | The Cook County templates were updated |
| 2026-10-26 | Firm staff asked for the ticket to be reopened: the number had to appear on **all** appearance forms where this attorney is the Primary Attorney |
| 2026-10-27 | All appearance templates checked and the missing ones updated (see 6) |
| 2026-10-29 | A second attorney's ARDC number was entered in Salesforce |

## 3. Scope

**In scope**

| Template | Before this ticket | Change |
|---|---|---|
| Appearance - Cook County (Short) | Did not show the attorney's ARDC | Added. The firm numbers on the form were kept |
| Appearance - DuPage County | Did not show the attorney's ARDC | Added. The firm numbers on the form were kept |
| Appearance - Lake County | Already showed it | None |
| Appearance - Kane County | Already showed it | None |
| Will County appearance | Already showed it | None |
| Appearance - Cook County (Long) | Already showed it | None in the final pass; see 10 |
| A second attorney's record in Salesforce | No ARDC number | ARDC number entered |

**Out of scope**

- **Previewing and editing custom input fields before routing.** That is DDT-1413, linked to this ticket.

## 4. How it works (background)

```
Staff generate an appearance form for the court's county
  -> each county has its own Dochly appearance template
  -> the "Atty. No." box prints either a fixed firm number, or the Primary Attorney's
     ARDC number read from Salesforce, depending on how that template is built
```

Because each county has its own template, a change to one county's form does not reach the others.

## 5. Root cause

Two separate gaps:

1. **Not every county's template printed the attorney's ARDC number.** Lake, Kane, Will and Cook (Long) already did. Cook (Short) and DuPage did not, so the first fix appeared to work in some counties and not in Cook.
2. **A second attorney had no ARDC number in Salesforce.** The templates that print the Primary Attorney's number had nothing to print for her, so the box was blank. That was missing data, not a template fault.

**The lesson.** "It works for one county" proves nothing about the others when each county is a separate template.

## 6. Solution

### 6.1 Templates

On 2026-10-27 every appearance template was checked. The attorney's ARDC number was added to the two that lacked it, Cook (Short) and DuPage, and the firm numbers already on those forms were left in place.

### 6.2 Data

The second attorney's ARDC number was entered on her record in Salesforce. Because the templates read the number from the Primary Attorney, no template change was needed for her forms to print it. The field used is not named on this ticket; DDT-1237 names the Primary Attorney's ARDC field as `ARCD_Number__c`.

## 7. Testing

**Recorded on the ticket:** Cook (Short) was generated on one of the attorney's cases and showed his ARDC number. The requester was then asked to generate one and confirm. That confirmation is not recorded in the ticket.

**Not covered:** a generated form from every county for every attorney.

## 8. Step by step

1. **List every appearance template**, one per county and form length.
2. **For each, check what the "Atty. No." box prints:** a fixed firm number, or the Primary Attorney's ARDC number.
3. **Add the Primary Attorney's ARDC number** to every template that lacks it. Keep any firm number already on the form.
4. **Query the attorneys** who can be Primary Attorney and find any with no ARDC number in Salesforce. Ask the firm for the missing numbers and enter them.
5. **Generate one form per county** on a case for each attorney, and read the box.

## 9. Rollback

Not applicable: removing the number would make the forms incomplete again.

## 10. Known limits

- **An open question on Cook (Long).** Its "Atty. No." box used to show a firm number and now shows each attorney's ARDC number, for all attorneys. The ticket asks whether to put the firm number back and show the ARDC number on a separate line. The answer is not recorded in the ticket.
- **A new attorney with no ARDC number in Salesforce** will print a blank box again. Enter the number on the attorney's record when they join.
- **A new county template** has to be built with the same box, or it will repeat gap 1.

## 11. Reuse for similar issues

**How to recognise one:** a value prints in some counties and not others, or for some attorneys and not others.

- **Separate template gaps from data gaps.** If the same template prints the value for one attorney and not another, the data is missing. If one county's template never prints it for anyone, the template is missing it.
- **Check every sibling template**, not just the one in the report. Each county is a separate template.
- **Prefer reading the number from the attorney's record** over typing it into the template, so a new attorney needs a data entry, not a template edit.
- **Keep the values already on the form** (here, the firm numbers) while adding another.
