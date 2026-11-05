# Court Appearance result modal, path and panel enhancements

| | |
|---|---|
| **Ticket** | DDT-319 |
| **Project** | Support tickets |
| **Status** | Done 2026-07-13 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-07-13 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC) |
| **Area** | Court appearances: logging outcomes and scheduling the next date |
| **Related** | None recorded |
| **Code** | Not recorded in the ticket. No commit for DDT-319 exists in the ddt-salesforce repo. |

---

## 1. Summary

Attorneys and case managers had no clean way to log the result of a court appearance and set up the next one. We added a result modal that logs an outcome from the panel, pre-fills the next appearance from that result, and can close the case when the result resolves it. The Court Appearance panel now shows the next upcoming appearance on top, sorts past ones newest first, and only offers **Enter Result** where a result is missing.

## 2. What was reported

The ticket asked for three things around court appearances: a modal to capture the result of an appearance, a case path showing past and upcoming appearances, and improvements to the read-only Court Appearances panel. The UI for capturing outcomes and seeing a case's trajectory was described as fragmented.

## 3. Scope

**In scope** (from the completion comment)

| Component | Type | Change |
|---|---|---|
| `courtAppearanceResultModal` | LWC | Log a result for a past appearance; pre-fill and schedule the next appearance; close the case when the result resolves it; layout fixes |
| `courtAppearancePanel` | LWC | Upcoming appearance on top; past appearances newest first; **Enter Result** only where needed; record navigation; stage gate |

**Out of scope, or not recorded as done:** a separate visual "case path" stepper, filtering by outcome or attorney, and sortable columns were in the ticket but are not mentioned in the completion comment.

## 4. How it works (background)

```
Court Appearance panel on the case
  → upcoming appearance shown at the top
  → past appearances listed newest first
  → past appearance with no result shows "Enter Result"
      → result modal: log the outcome
      → optional "Schedule Next Appearance": reopens the New Appearance modal, pre-filled
      → optional close: case marked Resolved
```

## 5. Root cause

Not applicable: this was an enhancement, not a defect.

## 6. Solution

**Result modal (`courtAppearanceResultModal`)**

- Users log the outcome of a past appearance from the modal, without opening the record.
- Pre-fill logic for the next court date: next date and time, court action, courtroom and judge are filled in from the result.
- **Schedule Next Appearance** re-opens the New Appearance modal with those values after the result is saved.
- When the result resolves the case, a close option marks the case **Resolved** and shows a confirmation toast.
- Modal size and layout fixed so it renders the same on all screen sizes.

**Panel (`courtAppearancePanel`)**

- The nearest future appearance is shown at the top, separate from past ones.
- Past appearances are sorted by date, newest first.
- **Enter Result** appears only on past appearances with no result; it turns to the destructive (red) style when overdue.
- Clicking through to the Court Appearance record uses `NavigationMixin`.
- **Stage gate:** the component is hidden on cases in the Pre-file stage, and a banner explains why, with reason text by matter type (Criminal, Federal, Civil, Post-Case Resolution).

## 7. Testing

Not recorded in the ticket. **Not covered:** no Jest or Apex test is named.

## 8. Step by step

1. Build the result modal with the outcome fields and the save to the Court Appearance record.
2. Add the pre-fill logic that maps the result to the next appearance's date, time, action, courtroom and judge.
3. Add **Schedule Next Appearance**, which opens the New Appearance modal with those values.
4. Add the close option that sets the case to Resolved and shows a toast.
5. Update the panel: split upcoming from past, sort past newest first, show **Enter Result** only where the result is blank, and style it as overdue when the date has passed.
6. Add the Pre-file stage gate and its banner text per matter type.
7. Open the screen on a real case and check each path by hand.

## 9. Rollback

Remove the component from the record page in App Builder, or redeploy the previous version of the bundles. Results already logged stay on their records.

## 10. Known limits

- Neither `courtAppearanceResultModal` nor `courtAppearancePanel` appears in the ddt-salesforce repo. The repo holds a different bundle, `courtAppearancesPanel` (plural); how the two relate is not recorded.
- The case-path stepper and the extra filters from the original ticket are not recorded as built.

## 11. Reuse for similar issues

- When a result usually leads to a next step, pre-fill that next step from the result rather than asking the user to retype it.
- Show an action button only where it is needed (here, past appearances with no result), and make overdue items visibly different.
- When a component is hidden by a gate, show a banner saying why, so users do not report it as missing.
- On any LWC change, open the screen: nothing in CI renders the component.
