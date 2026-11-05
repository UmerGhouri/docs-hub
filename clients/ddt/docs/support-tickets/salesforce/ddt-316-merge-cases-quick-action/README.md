# Merge Cases quick action to deduplicate opportunities

| | |
|---|---|
| **Ticket** | DDT-316 |
| **Project** | Support tickets |
| **Status** | Done 2026-07-13 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-07-13 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC, quick action) |
| **Area** | Court Case (Opportunity) records: cleaning up duplicates |
| **Related** | None recorded |
| **Code** | Not recorded in the ticket. No commit for DDT-316 exists in the ddt-salesforce repo. |

---

## 1. Summary

Duplicate Court Case (Opportunity) records were being created for the same client, which split tasks, court appearances and notes across two records. The only fix was a manual reparent-and-delete that only an admin could do safely. We built a **Merge Case** quick action: a modal compares the two records side by side, lets the user choose the winning value for each conflicting field, and moves the loser's activities, Chatter posts and files onto the master record.

## 2. What was reported

Duplicate Opportunities happen when intake creates one on a first call and another is created on a callback, or when the contract-signing conversion fires twice. The result was split tasks, split court appearances and split notes on two records for one matter. No count of duplicates is recorded in the ticket.

## 3. Scope

**In scope** (as listed in the ticket's completion comment)

| Component | Type | Change |
|---|---|---|
| `MergeCasesController` | Apex class | Compares the two records field by field, detects conflicts, reparents children, copies Chatter posts, reparents files, resolves lookup labels for display |
| `MergeCasesControllerTest` | Apex test | Unit tests for the controller |
| `mergeCasesModal` | LWC | Side-by-side comparison, conflict highlighting, per-field choice, submits the merge |
| `Opportunity.Merge_Case` | Quick action | Launches the modal from any Opportunity record |

**Out of scope, or not recorded as done**

The original ticket also asked for reparenting of court appearances, matter costs, payment, bonus, disposition and coverage records; a soft-delete `Merged_Into__c` field; a merge audit log; and restricting the action to intake leads and managers. The completion comment does not mention any of these, so treat them as not built unless you confirm otherwise in the org.

## 4. How it works (background)

```
User opens a Court Case (Opportunity) record
  → clicks the Merge Case quick action
  → mergeCasesModal opens; user picks the other Opportunity
  → MergeCasesController returns both records' fields, flags conflicts
  → user chooses the winning value per conflicting field
  → controller updates the master, moves children, copies Chatter posts
```

## 5. Root cause

Not applicable: this was a feature request. The underlying cause of the duplicates (intake re-creating a record on a callback, or the signing conversion re-firing) was not addressed by this ticket.

## 6. Solution

From the completion comment:

- **Dynamic field comparison.** The controller reads the fields of both Opportunities and marks a field as a conflict when both records hold different non-matching values.
- **Child records reparented to the master:** Tasks, Events, Emails and Notes.
- **Chatter posts copied** to the master, keeping their rich-text formatting. Chatter posts cannot simply be reparented, which is why they are copied.
- **Files** are reparented by moving the `ContentDocumentLink` rows to the master.
- **Lookup fields** show the related record's label in the modal rather than a raw Id.

The code itself is not in the ddt-salesforce repo, so the exact implementation cannot be quoted here.

## 7. Testing

The acceptance criteria in the completion comment:

- The action launches from any Opportunity record.
- The modal shows both records side by side, conflicts highlighted, and the user can choose a value.
- After the merge the master has the chosen values; Tasks, Events and Emails sit on the master; Chatter posts are visible there with formatting; files are accessible there.
- `MergeCasesControllerTest` passes.

Test results are not recorded in the ticket. **Not covered:** the child objects listed as out of scope above.

## 8. Step by step

1. Build the Apex controller with a method that returns both records' fields and the conflict list, and a method that performs the merge.
2. Reparent Tasks, Events, EmailMessages and Notes; copy Chatter posts; move `ContentDocumentLink` rows for files.
3. Build `mergeCasesModal` with the side-by-side comparison and a per-field choice.
4. Create the `Merge_Case` quick action on Opportunity pointing at the modal, and add it to the page layout.
5. Write `MergeCasesControllerTest` and run it.
6. Commit to the ddt-salesforce repo and open a PR before deploying.

## 9. Rollback

Remove the quick action from the page layout to hide the feature. A merge that has already run cannot be undone by the tool: moved children would have to be moved back by hand.

## 10. Known limits

- **Not found in production or in git.** A query of production Apex classes on 2026-11-05 returned no class named `MergeCasesController`, and none of the components appears in the ddt-salesforce repo. Whether this ever reached production is not recorded.
- Court appearances, matter costs, payments and other custom children are not listed as reparented, so a merge could leave them on the losing record.
- No audit trail or permission gate is recorded.

## 11. Reuse for similar issues

- Before building a merge tool, list **every** child relationship on the object (Setup → Object Manager → the object → Related Lists, or a describe call). A merge that misses one leaves orphans.
- Chatter posts are copied, not reparented; files are moved by their `ContentDocumentLink`.
- Decide early whether the loser is deleted or marked as merged, and log every merge.
- Fix the source of duplicates too, or the merge tool becomes a permanent chore.
- Commit the code to git the same day it ships; this feature cannot be found in the repo today.
