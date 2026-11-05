# Marking a signed representation agreement as received

| | |
|---|---|
| **Ticket** | DDT-889, DDT-1265 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-14 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-09-01 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Lightning record page, Opportunity) |
| **Area** | Hired Court Case (Opportunity): Blueback tab, representation agreement status |
| **Related** | PR #352 (Discovery panel on SOS Formal/Informal Hearing cases) |
| **Code** | PR #377 in the ddt-salesforce repo |

---

## 1. Summary

Staff could not mark a signed representation agreement as received on most hired Court Cases, and kept getting a "We need a signed Representation Agreement" popup they could not clear. The field for it, `Representation_Agreement__c`, already existed, but it sat in the Intake section of the Blueback tab, which only shows in the first three Blueback stages. We moved that one field into its own section at the top of the Blueback tab, visible at every stage. No new field was built.

## 2. What was reported

- **DDT-889 (2026-09-01).** A request for a new checkbox or date field on the Opportunity to mark the signed rep agreement as received. The requester then said it was not a new feature: "the buttons" disappear when the case type is SOS Formal Hearing and show when it is SOS Hearing. With an example where the popup kept firing despite a signed agreement on file.
- **DDT-1265 (2026-10-07).** A "missing representation agreement" popup on another hired case although the signed agreement was on file. Resolved within minutes in Slack: the staff member had not known to change the Representation Agreement dropdown to Signed. Once set, the popup stopped. Closed 2026-10-08 with no change.

## 3. Scope

**In scope:** `Opportunity_Record_BB_Default_Hired` (Lightning record page). One field moved from a stage-gated section into a new section.

**Out of scope**

- **A new `Signed_Rep_Agreement_Received__c` field.** Held twice at approval: a second place to record the same fact lets two places disagree.
- **"Bug 1", the Discovery toggles** blank on SOS Formal Hearing and SOS Informal Hearing cases. A separate cause, fixed in PR #352.
- **Auto-sending the representation agreement** after hire and first payment. Named as the long-term fix and scoped separately.
- **Permissions.** Moving an existing field between sections grants and revokes nothing.
- **The mobile page** `Opportunity_Hired_BB_Mobile_Only`, which does not include the field.

## 4. How it works (background)

`Opportunity.Representation_Agreement__c` is a restricted picklist: Not Sent, Sent, Signed, N/A. A popup component on the page (`S360_LN:PopUp`) shows "We need a signed Representation Agreement" whenever the value is **Not Sent** or **Sent**.

Before the fix, the field sat in the **Intake** section of the Blueback tab, whose visibility rule is:

```
1 AND (2 OR 3 OR 4)
  1  BB_Blocks__c       CONTAINS  Intake
  2  Blueback_Stage__c  EQUAL     Intake
  3  Blueback_Stage__c  EQUAL     Case Manager
  4  Blueback_Stage__c  EQUAL     Attorney
```

The rule sits inside the field section's component rather than on its wrapper, which makes it easy to miss when reading the page file.

## 5. Root cause

"The buttons" turned out to be two different bugs:

1. **Discovery toggles (Bug 1):** the case management component matched case type by exact text and only handled "SOS Hearing", so SOS Formal and Informal Hearing cases rendered blank. Case type **was** the trigger for that one (PR #352).
2. **Representation Agreement (Bug 2, this fix):** **Blueback stage**, not case type, hid the field. Past the Attorney stage, the Intake section disappears, so the field cannot be reached, while the popup keeps firing because it reads the same field. Staff were told to fix a value they could not get to.

**Evidence.** Four example cases from the thread, checked in production on 2026-09-14: the two reported as broken were at the Case Closed and Filings stages with the value Sent; the two that worked were at the Intake stage with the value Signed. One of the working ones was an **SOS Formal Hearing** case, which rules out case type for this bug. All four carried Intake in `BB_Blocks__c`, so stage was the only thing hiding the field.

**Scale.** 21,926 hired Opportunities; only 486 could reach the field.

```sql
SELECT COUNT(Id) FROM Opportunity WHERE Hire_Status__c = 'Hired'
  AND BB_Blocks__c LIKE '%Intake%'
  AND Blueback_Stage__c IN ('Intake','Case Manager','Attorney')
```

The ticket's first framing (build a new field) was wrong: the field already existed and was hidden.

## 6. Solution

1. **Removed** the `Representation_Agreement__c` field from the Intake section. `Hired_by_User__c` stays in that column, so the Intake section is otherwise unchanged.
2. **Added** a new field section, **Representation Agreement**, as the first item on the Blueback tab, holding only that field.
3. **Kept criterion 1 only** on the new section: `BB_Blocks__c CONTAINS Intake`. The three stage criteria are dropped, so the section shows **at every stage**.

Why this way:

- **Move the field, don't relax the Intake rule.** Removing the Intake section's rule would also expose `Intake_Complete__c`, `Copies_of_Tickets__c`, `Description__c` and `Hired_by_User__c` at every stage, which nobody asked for.
- **Keep criterion 1.** The instruction was "every stage"; `BB_Blocks__c` is the firm's per-case setting for whether the Intake block applies, not a stage. The first draft dropped it too, which would have shown the field on 2,005 hired cases configured with no Intake block.
- **The field keeps its identifier and behaviour,** so it stays inline-editable and appears in exactly one place on the page.

The edit was made on top of a fresh production retrieval of the page (the repo copy was identical to production), and deployed on 2026-09-14.

## 7. Testing

No Apex changed. A check-only deploy against production succeeded. The sandbox could not validate this page at all because of an unrelated component missing there; the unmodified page failed the same way.

**What that does not cover:** a check-only deploy proves the metadata deploys, not that the page renders. The acceptance check was the requester opening the two broken cases, seeing the new section above Intake, and setting the value to Signed so the popup stops. Its result is not recorded in the ticket.

## 8. Step by step

1. **Find which control the user means.** Ask for a screenshot; "the buttons" here covered two separate components.
2. **Compare a working and a broken record** on case type, stage and `BB_Blocks__c` together. Change one variable at a time to find the real gate.
3. **Retrieve the record page from production** and find the field's section and its visibility rule (look inside the component, not only on the wrapper).
4. **In Lightning App Builder** (or the page file), move the field into a new section at the top of the tab with only the rule you intend to keep.
5. **Validate against production** with a check-only deploy, then deploy.
6. **Open a broken record** at a late stage and confirm the section shows and the value can be set to Signed, and the popup stops.

For a DDT-1265-style report: open the case, find the **Representation Agreement** section on the Blueback tab, and set it to **Signed** (or **N/A** where no agreement applies). The popup only fires on Not Sent or Sent.

## 9. Rollback

Reverse the diff: move the field back into the Intake section and delete the new section. No data is touched. Rolling back makes the field unreachable again past the Attorney stage.

## 10. Known limits

- **1,651 hired cases are still not reached.** They have no Intake in `BB_Blocks__c` and sit at Not Sent or Sent, so the popup fires and the field is hidden. Not a regression; they could not reach it before either. Left for the auto-send work.

  ```sql
  SELECT COUNT(Id) FROM Opportunity WHERE Hire_Status__c = 'Hired'
    AND (BB_Blocks__c = null OR (NOT BB_Blocks__c LIKE '%Intake%'))
    AND Representation_Agreement__c IN ('Not Sent','Sent')
  ```

- **The field has other writers.** The Case Management Discovery panel (made visible on SOS Formal/Informal cases by PR #352) also edits it, so those cases have two editable places for the same fact. Two e-signature flows also set it, but only when it is not already Sent or Signed, so a person's Signed is not overwritten; a deliberate N/A can be.

## 11. Reuse for similar issues

**How to recognise one:** a popup or validation asks users to fix a value, and they say they cannot find it.

- **Check the field's visibility rule on the record page** before building anything new. Dynamic Forms rules hide existing fields per stage or status.
- **When a requester names a cause, test it against a counter-example.** One working SOS Formal Hearing case disproved "case type" here.
- **Don't build a second field for the same fact;** two places to record one answer will disagree.
- **Move one field rather than loosening a section's rule,** so nothing else becomes visible by accident.
- **Separate per-case configuration from stage** when relaxing a visibility rule, and count what each choice exposes.
