# Assigned email opens the whole thread

| | |
|---|---|
| **Ticket** | DDT-1143 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-01 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-09-28 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC), Email-to-Case |
| **Area** | Shared Inbox: opening an assigned email |
| **Related** | DDT-970 (merging a correspondent's emails into one chain) |
| **Code** | PR #564 and PR #570 in the `ddt-salesforce` repo |

---

## 1. Summary

In the Shared Inbox, opening an assigned email showed the whole chain with the newest message on top, even when that newest message had been assigned to someone else. Staff ended up acting on the wrong email.

The ticket's first diagnosis was that a thread has a single owner. The org showed otherwise: every inbound email is its own Case with its own owner, and the inbox merges a client's Cases into one chain. The fix labels each email with its Case and owner, highlights and scrolls to the opened Case's own email, and shows the Case's hand-offs in the header. The first deploy broke every assigned thread and was rolled back within 15 minutes; a fix-forward went live the same day.

## 2. What was reported

On 2026-09-28: one email from a client was assigned to one staff member, and a later email from the same client to another. When either opened their assignment, Salesforce showed the full message history, including the other person's email and older messages from days before. Replies went to the wrong message and older assignments got buried.

The request: open on the specific assigned message, make it visually distinct, and let several assignments in one chain each resolve to their own message.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `EmailInboxController` | Apex class | Each message in a merged chain carries its Case number and owner; the thread carries the Case's owner history |
| `EmailInboxControllerTest` | Apex test | Two new tests, then the fix-forward test |
| `emailThreadView` | LWC | Per-message labels, highlight of the opened Case's emails, one-time scroll, hand-off header |
| `emailThreadView.assignment.test.js` | Jest | Seven new tests |

**Out of scope**

- **The data model.** No new fields, no per-message owner. Each email already has an owner through its Case.
- **Permissions and post-deploy steps.** None.

## 4. How it works (background)

```
Client sends an email
  -> Email-to-Case opens a NEW Case for every inbound email
  -> each Case is assigned on its own (Case.OwnerId)
Shared Inbox (DDT-970)
  -> getEmailThread merges every sibling Case on the Matter that involves the same correspondent
  -> into one chain, newest first
```

So "the thread" a user sees is several Cases, each with its own owner, shown as one list.

## 5. Root cause

**The ticket's first reading was wrong, and checking the org is what showed it.** The initial review assumed assignment is `Case.OwnerId` on one Case per thread, so per-message assignment would need a data-model rebuild, and approved a narrower scope on that basis.

Checking the example in the org on 2026-09-30 showed:

- the first email had opened its own Case, assigned to the first staff member;
- the later emails had opened two more Cases, assigned to the second;
- the client's Matter held 8 Cases opened by that client's inbound emails, 20 Cases in all.

So every email already had an owner. The inbox simply merged the Cases into one chain, newest first, and said nothing about which email belonged to which Case. The person who opened the older Case saw a colleague's newer email on top. The revised scope was approved the same day.

## 6. Solution

### 6.1 What staff see (PR #564)

1. **Each email says whose it is.** In a merged chain: `This case` on the opened Case's own emails, `Case <number> · <owner>` on a sibling Case's, `Case <number> · Unclaimed` on a sibling still in a queue, and `Sent from the matter` on a Matter-level email. The opened Case's emails get an orange border.
2. **It opens on the right email.** When a merged chain loads, the view scrolls to the opened Case's newest email, once per Case, so a refresh does not move the reader. The assignment notification already targets the Case, so opening from the bell lands there too.
3. **The header shows hand-offs:** `Assigned: <user>, Sep 23 → <user>, Sep 28`, read from Case owner history. Hand-offs back to a queue are left out; only the latest four are named and older ones collapse to `2 earlier → …`.

A chain that is not merged looks as before, apart from the header line.

### 6.2 How

- `ThreadMessageDTO` gains `sourceCaseNumber` and `sourceOwnerName`, filled from the sibling-Case query DDT-970 already runs (now also selecting `CaseNumber`, `OwnerId`, `Owner.Name`). No new query for this.
- `ThreadDTO` gains `ownerHistory`, from one new query: `CaseHistory` for this Case, `Field = 'Owner'`, newest 50 rows (about 25 hand-offs, since each change writes a name row and an Id row).
- Only the Id row (`DataType = 'EntityId'`) is read, so a user is told apart from a queue with `getSObjectType()`, not by name or Id prefix.
- `CaseHistory` rows cannot be created with values in a test, so the logic that turns rows into the header is split into `ownerHistoryFrom()`, which the tests call.
- The scroll is keyed on the loaded thread's `caseId` and spent only once the Case's own email is found, because `emailInbox` keeps one thread view mounted and swaps its Case.

### 6.3 The incident and the fix-forward (PR #570)

PR #564 was deployed on 2026-09-30. Within minutes, opening any Case with owner history, which is nearly every assigned Case, showed `Error loading thread: List index out of bounds: 0`. It was rolled back about 15 minutes later.

The cause was in the query half, the part tests could not reach:

```apex
// Before: built oldest-first by inserting at the front
newOwners.add(0, h.NewValue);
changedAts.add(0, h.CreatedDate);
```

As deployed, this threw on an empty list, so the thread failed to load. The fix appends in query order and lets `ownerHistoryFrom` sort by time itself:

```apex
public class OwnerChangeDTO implements Comparable {
    @AuraEnabled public String ownerName;
    @AuraEnabled public DateTime changedAt;

    /** Oldest first; a hand-off with no time sorts to the front. */
    public Integer compareTo(Object other) {
        DateTime theirs = ((OwnerChangeDTO) other).changedAt;
        if (changedAt == theirs) return 0;
        if (changedAt == null) return -1;
        if (theirs == null) return 1;
        return changedAt < theirs ? -1 : 1;
    }
}
// ...in loadOwnerHistory
newOwners.add(h.NewValue);
changedAts.add(h.CreatedDate);
// ...at the end of ownerHistoryFrom
out.sort();
```

Why: the header's order no longer depends on the query's `ORDER BY` agreeing with a reverse loop somewhere else. The fix-forward was deployed on 2026-09-30 after validation (68 of 68 tests).

## 7. Testing

- **Apex:** `mergedChainTagsEachEmailWithItsOwnCaseAndOwner` and `ownerHistoryKeepsUserHandOffsInOrderAndDropsQueues`. The fix-forward test feeds rows out of order and checks that an empty history returns nothing without throwing.
- **Jest:** `emailThreadView.assignment.test.js`, 7 tests: labels and highlight, scroll once, scroll survives a stale render, header cap, header text, single-Case thread unchanged, Matter-level label.
- **Before the fix-forward went back to production,** it was **executed against real `CaseHistory` in the Partial sandbox**: five Cases, including ones with 26 and 10 owner rows, loaded without error and listed hand-offs oldest first. That execution is the check that was missing the first time.
- **The assumption behind the query** (an owner change writes an `EntityId` row) was checked against production on 2026-09-30: four rows on one Case, two `Text` and two `EntityId`.

**Not covered:** the query half is untested by Apex by construction. Validation compiles Apex and runs tests; it does not render the component. If `CaseHistory` ever stops writing `EntityId` rows, the header goes blank rather than erroring, so an empty header on an assigned Case is a failure, not "never assigned".

## 8. Step by step

1. **Check the data before accepting the ticket's model.** Query the example's Cases and their `CaseHistory` owner rows. Here each email was its own Case with its own owner.
2. **Confirm `CaseHistory` writes `EntityId` rows** for owner changes in production.
3. **Extend the sibling-Case query** with `CaseNumber`, `OwnerId` and `Owner.Name` and fill the two new message fields.
4. **Add the owner-history query**, appending rows in query order, and a separate `ownerHistoryFrom()` that filters out queues and sorts by time.
5. **Update `emailThreadView`:** labels, highlight, scroll once per Case, header.
6. **Write the Apex and Jest tests.**
7. **Execute the query path against real `CaseHistory` in a sandbox** before deploying. Tests cannot reach it.
8. **Validate against production** naming both `EmailInboxControllerTest` and `InboxSpamServiceTest` (the pre-change class sits at 74.3% per-class coverage, under the 75% bar `RunSpecifiedTests` applies). Validate at deploy time and quick-deploy from that run.
9. **Open an assigned Case** and confirm it lands on its own email, highlighted, with the expected header text.

## 9. Rollback

Redeploy `EmailInboxController`, `EmailInboxControllerTest` and `emailThreadView` from the commit **before PR #564**, not "the commit before" the fix-forward, which is the version that broke the inbox. Name both test classes for the coverage reason in step 8. No data is written, so rolling back is safe.

## 10. Known limits

- **Only the newest 50 owner-history rows** (about 25 hand-offs) are read, and the header names the latest four.
- **Hand-offs to a queue are not shown.**
- **The header depends on `CaseHistory` writing `EntityId` rows.** If that changes, the header goes blank.
- **Unmerged threads** are unchanged apart from the header.

## 11. Reuse for similar issues

**How to recognise one:** staff say "I opened my item and saw someone else's", in a view that combines several records into one list.

- **Query the real records before accepting the ticket's diagnosis.** Here the "one owner per thread" model was wrong, and the right fix was smaller than the one first approved.
- **Label each item with its source record** when a view merges several.
- **Code that tests cannot reach must be executed against real data before production.** A green test run proved nothing about the `CaseHistory` query.
- **Don't rely on list order from somewhere else;** sort where the order matters.
- **Write the rollback target as a fixed commit,** not "the previous one", so a later fix doesn't turn the rollback into a redeploy of the bug.
