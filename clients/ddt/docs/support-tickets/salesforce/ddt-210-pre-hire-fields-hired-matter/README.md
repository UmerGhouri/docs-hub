# Pre-hire fields on hired matters and urgent flags on assignment email

| | |
|---|---|
| **Ticket** | DDT-210 |
| **Project** | Support tickets |
| **Status** | Done 2026-06-22 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-06-22 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Lightning page, Flow, email) |
| **Area** | Handing a newly hired matter to the legal team |
| **Related** | DDT-211, DDT-213 (same batch of matter-management tickets) |
| **Code** | Not recorded in the ticket; not in the `ddt-salesforce` repo |

---

## 1. Summary

Two improvements at the point of hire. First, four time-critical fields that were visible on the potential matter but not on the hired matter were added to the Hired Matter page. Second, the New Case Assignment email to the legal team gained an **URGENT FLAGS** section that appears only when the court date is close, a warrant is active, or a DUI arrest is 8 or more days old.

## 2. What was reported

Written as planned work on 2026-06-22 and closed the same morning with a note of what had been done.

- **Pre-hire context was lost at hire.** Information captured during intake was not visible to attorneys and case managers on the hired matter, so they had to go back to intake or the original record.
- **Every assignment email looked the same.** A routine case and a case with a court date in two days arrived looking identical, and attorneys triage their inbox by eye.

## 3. Scope

**In scope** (from the closing comment)

| Component | Change |
|---|---|
| **Hired Matter v2** Lightning page | New **Pre-Hire Notes** section with Upcoming Court Date, Warrant, Date of Arrest and Days Since Arrest |
| Field visibility on that section | Date of Arrest and Days Since Arrest shown only when Matter Category is DUI |
| `Email_To_Legal_POD` flow | Evaluates three urgency conditions at assignment and adds an URGENT FLAGS section when any is true |

**Out of scope**

- **Copying data between records.** The potential and hired matter are the same Opportunity record with different Lightning pages, so the values were already on the record. No mapping was needed.
- **Other candidate fields** the ticket listed (lead source, intake notes, quoted fee, language needs). Not recorded whether they were considered.
- **A manual "urgent" checkbox and urgency tiers**, both suggested in the ticket. Not mentioned in the closing comment.

**Where this lives.** The ticket does not record the org or a pull request, and none of these components are in the `ddt-salesforce` repo. Confirm the target org before repeating it.

## 4. How it works (background)

```
Potential matter (Opportunity, Potential page)  -- shows court date, warrant, arrest date
  -> matter is hired: same record, now shown on the Hired Matter v2 page
       (the four fields were not on that page)
  -> Email_To_Legal_POD flow sends the New Case Assignment email to the legal team
       (same layout whatever the urgency)
```

## 5. Root cause

- **Fields missing after hire:** the Hired Matter page simply did not include them. The data was there; the page did not show it.
- **Email:** the flow had no urgency logic.

## 6. Solution

### 6.1 Pre-Hire Notes section

Added to the Hired Matter v2 Lightning page in App Builder:

- Upcoming Court Date
- Warrant
- Date of Arrest, visible only when Matter Category = DUI
- Days Since Arrest, visible only when Matter Category = DUI

The DUI-only fields use component visibility rules on the page, so they do not clutter non-DUI matters.

**Why a page change and not a new component:** the ticket offered two options, copying fields onto the hired matter or a component that loads them from the lead. Neither was needed, because both stages are the same record.

### 6.2 URGENT FLAGS in the assignment email

The `Email_To_Legal_POD` flow evaluates three conditions when the matter is assigned:

| # | Condition |
|---|---|
| 1 | Court date is today or within 3 days |
| 2 | An active warrant is on file |
| 3 | Matter is DUI and the arrest was 8 or more days ago (the license-hearing deadline is approaching) |

If any is true, the email includes an URGENT FLAGS section listing which. If none is true, the section is left out entirely, so routine emails look as they did before.

## 7. Testing

The closing comment says the flow was "deployed and verified in org". Test cases and results are not recorded.

**Not covered:** not recorded whether each condition was tested on its own, or the case where none applies.

## 8. Step by step

1. **Confirm the fields exist on the record at both stages.** If the potential and hired stages are one record, it is a page change only.
2. **Edit the Hired Matter Lightning page** in App Builder: add a section and the fields. Retrieve production's current page first; never deploy a sandbox copy of a page over production.
3. **Set component visibility** on the DUI-only fields: Matter Category equals DUI.
4. **Open the assignment flow** and add a decision before the email that evaluates each urgency condition.
5. **Build the flag text** from the conditions that are true, and include the section only when it is not empty.
6. **Test** with one matter per condition, one with several, and one with none.
7. **Activate** the new flow version after approval.

## 9. Rollback

- **Page:** remove the section in App Builder, or reactivate the previous page assignment.
- **Flow:** activate the previous version of `Email_To_Legal_POD`.

Both are safe; neither changes data.

## 10. Known limits

- **The thresholds are fixed in the flow** (3 days, 8 days). Changing them means editing the flow.
- **Urgency is evaluated once, at assignment.** A matter that becomes urgent later gets no new email.
- **Page visibility rules only control this page.** They do not change who can read the fields; field-level security does that.

## 11. Reuse for similar issues

**How to recognise one:** "the information is on the record but people can't see it at this stage".

- Check whether the two stages are the same record before building any copy or sync.
- Use page visibility rules for fields that only matter for one case type.
- For urgency in notifications, add the section only when a condition is true, and say which condition fired.
