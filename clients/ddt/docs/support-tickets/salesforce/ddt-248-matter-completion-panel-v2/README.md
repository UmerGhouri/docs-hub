# Matter Completion panel v2 with in-panel feedback

| | |
|---|---|
| **Ticket** | DDT-248 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-30 |
| **Priority** | High |
| **Reported by** | Firm operations, 2026-06-30 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC, custom fields, custom object, permission set, flexipage) |
| **Area** | PI matters: checking a matter is complete and demand-ready |
| **Related** | DDT-682 (later re-pointed the panel at the Incident and trimmed it) |
| **Code** | PR #41 in the ddt-salesforce repo; deploy doc `docs/deploys/matter-completion-panel.md` |

---

## 1. Summary

The Matter Completion panel (`matterGapChecklist`) on the PI Matter page was upgraded into a sectioned completion and demand-readiness tool, with a button that lets staff send feedback on a question from inside the panel. It was already **live in production for 36 matter-working staff** when this ticket was raised. The ticket was the review-and-merge step to get it into git, plus one piece of drift the pull request did not contain: the page visibility rule, which had been changed directly in production.

## 2. What was reported

- **2026-06-30:** a review request. The panel had been deployed straight to production with the owner's approval and was live to 36 staff. The pull request existed to record it in `main` and get it reviewed, not to gate the deploy. Production was ahead of `main` on these components.

The ticket flagged that the record page's visibility change was **not in the pull request**, so merging it alone would still leave git out of step with production.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `matterGapChecklist` | LWC | Rewrite: conditional sections, % complete bar, demand-ready strip, inline edit of blank fields only, checkbox fix, graceful "needs access" state, feedback button |
| `PI_Incident__c.Has_Health_Insurance__c` | Picklist field (Yes / No / Unknown) | New |
| `Matter__c.Is_Client_Employed__c` | Picklist field (Yes / No / Unknown) | New |
| 16 `PI_Incident__c` fields | Field labels | Plain-English labels; API names unchanged |
| `Matter_Completion_Feedback__c` | Custom object, 7 fields, tab, `New_Feedback` list view | New |
| `Matter_Completion_Panel` | Custom permission | New. Controls whether the panel shows on the page |
| `Matter_Wizard_Access` | Permission set | FLS on the three not-universally-readable fields, the custom permission, and feedback object access |
| `scripts/apex/assign-matter-completion-permset.apex` | Anonymous Apex | Bulk assigns the permission set by profile |
| `Matter_Record_Page` | Flexipage | Visibility rule changed in production. **Not in the PR** |

**Out of scope**

- **Apex.** There is none. The panel reads with `getRecord` and saves with `lightning-record-edit-form`.
- **The flexipage file in git at the time.** It was stale, so it was deliberately left out of the PR and changed against a fresh production retrieve instead.

## 4. How it works (background)

```
User opens a PI Matter
  -> Matter_Record_Page shows matterGapChecklist only if the user has the Matter_Completion_Panel custom permission
  -> matterGapChecklist reads ~55 Matter__c / PI_Incident__c fields with getRecord
  -> groups them into sections, hides sections that don't apply to this case type
  -> shows % complete and a demand-ready strip
  -> the user expands a section, fills only the blank fields, saves (one record-edit-form submit per section)
  -> the feedback button writes a Matter_Completion_Feedback__c record with createRecord
```

This panel replaced the older Matter Completion Wizard screen flow, which is now permanently inactive.

**The key Salesforce rule.** `getRecord` is all-or-nothing. If the user lacks read access to **any** field the component asks for, the whole component gets an error, not just that field.

## 5. Root cause

Not applicable as a fault: this was a feature that had already shipped. The reason for the ticket was **drift**.

- The panel was deployed to production directly, so production was ahead of `main`.
- The flexipage visibility rule was then changed **directly in production** against a fresh retrieve, and never added to the PR.

The ticket noted this was the fourth direct-to-production deploy in a row that git had to catch up to.

## 6. Solution

### 6.1 What the panel does

- **Conditional sections.** Only sections relevant to the case show. A non-vehicle matter hides the vehicle, at-fault, police and scene sections; a pedestrian case hides Client Vehicle; a hit-and-run drops at-fault insurance from demand-ready.
- **Skip-if-populated editing.** Expanding a section shows only the fields still blank.
- **Checkbox fix.** Unchecked boxes were counted as blank, so a new matter looked entirely incomplete. Booleans are no longer counted as gaps.
- **Graceful "needs access" state.** If `getRecord` errors on field access, the panel says so instead of spinning forever. Keep this in any future refactor.

### 6.2 Why the two new fields are three-value picklists

`Has_Health_Insurance__c` and `Is_Client_Employed__c` are **Yes / No / Unknown**, not checkboxes. A checkbox cannot tell "we asked and the answer is no" from "nobody asked yet". Setting **No** counts as answered and clears the item from the demand-ready strip; leaving it blank keeps it there.

### 6.3 One lever: the permission set

`Matter_Wizard_Access` (labelled "Matter Completion Panel Access") grants, together:

1. FLS on the three fields not every user can read: `Employer_Account__c`, `Is_Client_Employed__c`, `Has_Health_Insurance__c`.
2. The `Matter_Completion_Panel` custom permission, which makes the panel visible.
3. Create and read on `Matter_Completion_Feedback__c`, its fields and its tab.

Because visibility and field access arrive in the same permission set, a user can never see the panel without the access that makes it work.

### 6.4 The flexipage visibility rule (applied in production, not in the PR)

On the `matterGapChecklist` component on `Matter_Record_Page`, the owner-only rule was replaced with a permission check:

```xml
<visibilityRule>
  <criteria>
    <leftValue>{!$Permission.CustomPermission.Matter_Completion_Panel}</leftValue>
    <operator>EQUAL</operator>
    <rightValue>true</rightValue>
  </criteria>
</visibilityRule>
```

After that, rolling the panel out to someone is just assigning the permission set. This rule reached git in the 2026-07-03 production backfill.

### 6.5 Feedback from inside the panel

A feedback button opens a form (Type, Section, Question, Details) and writes a `Matter_Completion_Feedback__c` record. It captures the Matter, the section, the question's label and the exact field API name, so feedback can be acted on directly. Types: Remove question, Add question, Bug, Should tie to a record, Confusing label, Other. Status runs New → Reviewed → Actioned → Dismissed. Review it from the **Matter Completion Feedback** tab, **New Feedback** list view.

## 7. Testing

- **Validation:** the ticket records **27 of 27 tests passing** against production. The change has no Apex of its own.
- **Smoke tests listed in the ticket and deploy doc:** the panel renders for a user with the permission set; conditional sections behave per case type; the % bar updates after a save; the demand-ready strip renders; a new matter is not reported as fully incomplete because of unchecked boxes; the feedback button creates a record; a user without the permission set sees the "needs access" state rather than a hung panel.
- **Gate test:** set `Has_Health_Insurance__c = No` and confirm it clears from demand-ready; leave it blank and confirm it stays.
- **Cascade check:** each section save is one update, which fires the usual Matter triggers and flows. Snapshot task and stage-tracking counts on a low-traffic matter, fill several sections, confirm no change.

The ticket does not record the result of each smoke test.

**Not covered:** `Employer_Account__c` did not exist in the partial sandbox, so field access for it could only be validated against production.

## 8. Step by step

To deploy and roll out a panel like this one:

1. **Deploy the metadata:** LWC, new fields, label changes, custom permission, permission set, feedback object and tab. Validate against production first.
2. **Retrieve `Matter_Record_Page` fresh from production.** Do not deploy the repo copy if it may be stale.
3. **Change only the visibility rule** on the component to the custom-permission check in 6.4, then deploy just that flexipage.
4. **Assign the permission set before anyone can see the panel.** Run `scripts/apex/assign-matter-completion-permset.apex`. It assigns by profile, skips users who already have it, and reports failures.
5. **Pilot with one team first**, then widen the profile list in the script and re-run it.
6. **Smoke test** as in section 7, including a user without the permission set.
7. **Record any change made directly in production in git the same day**, including the flexipage.

## 9. Rollback

- **Fastest:** put the old visibility rule back on the flexipage. The panel disappears for everyone it was rolled out to.
- **Full:** revert the PR. The deploy doc notes the new fields, permission set, custom permission and label changes have no Apex or trigger dependents.

## 10. Known limits

- **Every field the panel reads must be readable by the user.** Add a field to the panel without granting its FLS and the panel shows the "needs access" state for everyone missing it.
- **Save fires the normal Matter automation.** Each section save is one DML, far fewer than the old flow, but still runs the Matter triggers and flows.
- **The panel was later changed.** DDT-682 re-pointed it at the Incident and trimmed it. This page describes the v2 state.

## 11. Reuse for similar issues

**How to recognise the pattern:** a feature is already live and the ticket is "get git to match production".

- **Gate visibility and field access with one permission set.** Then a user can never see a component without the access it needs.
- **Use three-value picklists for gating questions** where "No" must be distinguishable from "not asked".
- **Remember `getRecord` is all-or-nothing** and give the component a clear error state.
- **Never deploy a stale flexipage.** Retrieve production, change the one thing, deploy just that page.
- **Anything changed directly in production goes into git the same day**, or the next deploy from git undoes it.
