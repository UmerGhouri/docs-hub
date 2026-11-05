# Disengaged reason and date shown only when disengaged

| | |
|---|---|
| **Ticket** | DDT-1383 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-21 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-10-21 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Lightning record page, Dynamic Forms) |
| **Area** | PI Matters: the Matter record page |
| **Related** | None recorded |
| **Code** | PR #799 in the `ddt-salesforce` repo |

---

## 1. Summary

The Matter page showed **Reason** and **Disengaged Date** on every Matter, although they only mean something on a disengaged one. Both fields now have a visibility rule on the Matter record page: they show only when Hire Status is one of the four Disengaged values. No data changed.

## 2. What was reported

On 2026-10-21 a staff member asked for Reason (`Reason__c`) and Disengaged Date (`Disengaged_Date__c`) to be shown only when Hire Status (`Hire_Status__c`) is one of:

- To Disengage
- Disengaged/Dropped
- Disengaged - Has New Counsel
- Disengaged - Referred Out

Showing them on every Matter cluttered the page. This was a change request, not a fault.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Matter_Record_Page` | Lightning record page (FlexiPage) | A visibility rule on the `Reason__c` and `Disengaged_Date__c` field instances |

**Out of scope**

- **`Matter Layout`** (the only page layout). It still has `Reason__c`. A page layout cannot hide a field conditionally, and removing the field would take it away from disengaged Matters too. While Dynamic Forms drives the record page, the layout is used only in Classic and by anything that renders the layout directly.
- **The fields, their data, and the two validation rules on them.** Unchanged.

## 4. How it works (background)

The Matter page uses Dynamic Forms: each field is its own component on the page, and each can carry a visibility rule. The page already had rules of the same form, for example the one on `Case_Manager__c` driven by `Hire_Status__c`.

`Matter_Record_Page` is the only Matter record page in production (checked with a Tooling query on 2026-10-21). It is the org default for both desktop and phone, and no app assigns a different Matter page, so one rule covers both.

## 5. Root cause

Not applicable: this was a request for new behaviour. The fields simply had no visibility rule.

## 6. Solution

Each of the two field instances gets the same rule, four criteria joined by OR:

```xml
<visibilityRule>
    <booleanFilter>1 OR 2 OR 3 OR 4</booleanFilter>
    <criteria>
        <leftValue>{!Record.Hire_Status__c}</leftValue>
        <operator>EQUAL</operator>
        <rightValue>To Disengage</rightValue>
    </criteria>
    <criteria>
        <leftValue>{!Record.Hire_Status__c}</leftValue>
        <operator>EQUAL</operator>
        <rightValue>Disengaged/Dropped</rightValue>
    </criteria>
    <criteria>
        <leftValue>{!Record.Hire_Status__c}</leftValue>
        <operator>EQUAL</operator>
        <rightValue>Disengaged - Has New Counsel</rightValue>
    </criteria>
    <criteria>
        <leftValue>{!Record.Hire_Status__c}</leftValue>
        <operator>EQUAL</operator>
        <rightValue>Disengaged - Referred Out</rightValue>
    </criteria>
</visibilityRule>
```

Why it's done this way:

- **Picklist API values, not labels.** A visibility rule compares the stored value. Here the four API values equal their labels, which was checked against the production describe.
- **Edited on top of production's current page.** The page was re-retrieved from production, and the branch differs from it by these two inserted blocks only. A sandbox copy of a record page must never be deployed over production's.
- **The rule also shows the fields while editing.** Changing Hire Status to a Disengaged value in the edit form makes both fields appear before saving.

**Two validation rules to know about:**

- `Disengaged_Requires_Reason_And_Date` fires when Hire Status changes to a Disengaged value. The visibility rule shows both fields for those values, so they are on screen when that error appears.
- `Reason_Required_When_Not_Kept` requires `Reason__c` when `Standing__c` is `Not Kept`, whatever the Hire Status. That could ask for a hidden field, but it cannot happen from this page today: `Standing__c` is not on the page, and all 7 Not Kept Matters in production were `Disengaged/Dropped` on 2026-10-21.

## 7. Testing

**Validation:** the page could not be deployed to the partial sandbox because a component it uses is missing there, and production's unmodified copy fails there the same way, so that is sandbox drift. The change was validated check-only against production instead, and it succeeded.

A check-only deploy does not show that the page renders. The live check was manual:

1. A Matter with a non-Disengaged Hire Status: both fields hidden.
2. A Matter with each Disengaged value: both shown, with their existing values.
3. Changing Hire Status to `Disengaged/Dropped` in edit: both fields appear before save.
4. The same in the Salesforce mobile app.

The requester confirmed on 2026-10-21 that it was fixed.

**Not covered:** no automated test exists for page visibility rules.

## 8. Step by step

1. **Retrieve `FlexiPage:Matter_Record_Page` from production.**
2. **Check it is the only Matter page** and which form factors use it.
3. **Add the visibility rule** in 6 to the `Reason__c` and `Disengaged_Date__c` field instances (or do the same in Lightning App Builder: select the field, Set Component Visibility, Hire Status equals each value, "Any condition is met").
4. **Diff against production's copy.** Only the two rule blocks should differ.
5. **Validate check-only against production**, get the PR approved, then deploy.
6. **Run the four checks** in section 7.

## 9. Rollback

Remove the two `visibilityRule` blocks (or the visibility filters in App Builder) and redeploy the page on top of production's current copy. Safe: no data is involved.

## 10. Known limits

- **New Disengaged values.** If a fifth Disengaged Hire Status value is added, it must be added to both rules or the fields stay hidden for it.
- **The page layout still shows `Reason__c` unconditionally** in Classic and anywhere the layout is rendered directly.
- **`Reason_Required_When_Not_Kept`** could demand a hidden field if `Standing__c` is ever added to the page.

## 11. Reuse for similar issues

**How to recognise one:** a request to show or hide fields depending on another field's value on a Dynamic Forms page.

- **Use a component visibility rule, not a new page or layout.**
- **Compare API values**, and check them against the describe before trusting the labels.
- **Check validation rules on the hidden fields**, so nobody is asked to fill in a field they cannot see.
- **Find every page that shows the field** (record pages per app and form factor, layouts, compact layouts) before saying a field is hidden everywhere.
- **Edit production's current page**, never a sandbox copy.
