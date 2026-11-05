# AI Summary component: missing Generate button and summary errors

| | |
|---|---|
| **Ticket** | DDT-42, DDT-49 |
| **Project** | Support tickets |
| **Status** | Live since 2026-05-26 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-05-26 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC, Apex, permission sets) |
| **Area** | The AI "Record Summary" component on record pages |
| **Related** | DDT-47 (AI summaries for hired, open cases), DDT-43 (mentioned in DDT-49 as a possible permission-set drift) |
| **Code** | Not applicable: fixed by a permission set assignment, no code change |

---

## 1. Summary

Two users hit problems with the AI summary component on the same morning. One got an error when creating a summary, even though a summary was then produced. The other, a user the component had just been shared with, saw no **Generate** button, and the prompt set-up screen offered no fields to pick.

Both tickets were closed the same day as **a permission problem**, fixed by assigning the relevant permission set to the user. No code changed.

## 2. What was reported

- **DDT-42, 2026-05-26.** A user clicked to create a summary report on one client's record and got an error. A detailed summary was still produced: it covered the latest gameplan update, the mitigation letter status, and the notes from the welcome call. It also said there were no call recordings on file, and the user asked whether the component was really reading the case file.
- **DDT-49, 2026-05-26.** When the component was shared with a new user, that user saw **no Generate button**. Clicking **Create Summary Prompt** asked her to select a field, but **no fields were available to select**.

One user each. No record counts were affected.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Permission set assignment for each affected user | Setup / user record | The relevant permission set was assigned |

**Out of scope**

- **The component's code.** Nothing was changed in `caseBriefGenerator` or `BriefGeneratorController` for these tickets.
- **Whether the summary reads the full case file** (the DDT-42 question about call recordings). Not recorded in the ticket.

## 4. How it works (background)

The component is the `caseBriefGenerator` LWC, labelled **Record Summary** in App Builder. It can sit on any record page. Its configuration is one `Summary_Prompt__c` record per object, keyed by the object's API name, which holds the fields to include, related-record subqueries, the system prompt, the model, and an optional eligibility filter.

```
Record page loads the Record Summary component
  -> BriefGeneratorController.getConfig(objectApiName)     reads Summary_Prompt__c
     config found     -> state "idle"        -> "Generate Summary" button shows
     no config found  -> state "needsConfig" -> prompt set-up form shows,
                         field list loaded by BriefGeneratorController.describeFields
  -> Generate Summary
  -> BriefGeneratorController.generateSummary   inserts a Brief_Feedback__c (Pending)
  -> BriefGenerationQueueable calls the Claude API and writes the summary back
```

In the repo, the permission set that grants this component's Apex classes, its two objects (`Summary_Prompt__c`, `Brief_Feedback__c`) and the named credential for the Claude API is **`Claude_Brief_Generator`** (label "Claude Brief Generator"). It held 52 assignments in production when this page was written (2026-11-05).

## 5. Root cause

The tickets record only this: **"There was a permission issue"**, fixed by assigning **"the relevant permission set"**. They do not name the permission set.

The component's code (as recorded in git on 2026-05-22) shows why a missing permission produces exactly the DDT-49 symptoms, without any clear error:

1. On load, the component calls `getConfig`. If that call fails, the error is swallowed and the configuration is treated as **missing**.
2. With no configuration, the component skips the **idle** state, so the **Generate Summary** button never renders. It shows the prompt set-up form instead.
3. The set-up form loads the field list through `describeFields`. That call fails the same way, so the form has **nothing to pick** (the code shows a "Could not load fields" toast), and `describeFields` also leaves out any field the user cannot read.

So a user who cannot reach the controller or read `Summary_Prompt__c` looks, to the component, exactly like an object that has never been set up.

**What is not recorded:** which permission set was assigned, and what exactly caused the DDT-42 error message.

## 6. Solution

The relevant permission set was assigned to each affected user, in production, on 2026-05-26.

Why a permission set and not a code change: the component worked for users who already had access. The problem was the new user's access, not the component.

## 7. Testing

- **Live check:** DDT-49 was closed with a note that the issue was fixed for the user, for both the Generate button and the fields. The ticket does not record how that was checked.
- **Automated tests:** Not applicable: nothing in the code changed.

**Not covered:** the DDT-42 question of whether the summary reads every part of the case file. Not recorded in the ticket.

## 8. Step by step

1. **Open the record as the affected user**, or check their access in Setup, and confirm the symptom: no Generate button, or an empty field list in the prompt set-up.
2. **Check that a configuration exists for the object.** In production, query:

    ```sql
    SELECT Name, Eligibility_Filter__c FROM Summary_Prompt__c
    ```

    If the object is listed, the set-up form appearing for the user means the user cannot read it, not that it is missing.
3. **Check the user's permission sets.** Setup → Users → the user → Permission Set Assignments. Look for `Claude_Brief_Generator`.
4. **Assign the missing permission set.** Production assignments need human approval first.
5. **Reload the record page as the user.** The Generate Summary button should appear.

## 9. Rollback

Remove the permission set assignment from the user. That brings the problem back, so there is no reason to do it unless the user should not have the feature.

## 10. Known limits

- **A missing permission looks like missing configuration.** In the version of the component in use on 2026-05-26, a failed `getConfig` call fell back silently to the set-up form, which hides the real cause.
- **A later version of the component changes what non-admins see.** The version recorded in git on 2026-07-03 shows non-admin users a "Record Summary is not yet configured for this object type. Ask a System Administrator to set it up." message instead of the set-up form, and shows the edit button only to admins. A missing permission will now show that message, which still does not say "permission".
- **Eligibility filter.** A record that does not match the object's eligibility filter cannot be summarised (see DDT-47). That is a different message, "This record does not meet the eligibility criteria for an AI summary."

## 11. Reuse for similar issues

**How to recognise one:** a component works for most users but, for a newly onboarded user, shows an empty state, a missing button, or a picker with no options.

- **Compare the user with one for whom it works.** The difference is almost always a permission set.
- **Read the component's error handling.** A `catch` that sets a value to null and carries on turns "no access" into "nothing here". Look for it before suspecting the data.
- **Field pickers built on describe calls skip fields the user cannot read.** An empty picker can mean no field-level security, not no fields.
- **When onboarding a user to a feature**, assign the feature's permission set at the same time as sharing the page, and record which permission set it was on the ticket.
