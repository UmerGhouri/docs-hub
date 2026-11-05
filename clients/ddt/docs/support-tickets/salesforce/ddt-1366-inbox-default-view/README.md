# Per-user default view in the email inbox

| | |
|---|---|
| **Ticket** | DDT-1366 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-23 |
| **Priority** | Medium |
| **Reported by** | Attorney, 2026-10-16 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC, custom setting) |
| **Area** | Shared Inbox: the filters the inbox opens with |
| **Related** | Not recorded in the ticket |
| **Code** | PR #778 in the `ddt-salesforce` repo |

---

## 1. Summary

Two staff members wanted the Shared Inbox to open with more filters on (Open, Unclaimed, Mine, Linked and Unlinked) instead of the built-in default. Rather than hard-coding a default for two people, the request was re-scoped: the inbox now has a **Save as my default** button, so each user saves their own filters and nobody else's inbox changes.

## 2. What was reported

On 2026-10-16 an attorney asked for the inbox's default view to change from "Open, Unclaimed, Unlinked" to "Open, Unclaimed, Mine, Linked, Unlinked", for himself and a colleague. The built-in default hid emails claimed as "Mine" and emails already linked to a record, so both had to change the filters every time they opened the inbox.

The ticket asked whether this could be a per-user preference or would have to be a shared default affecting everyone.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Email_Inbox_Settings__c` | Hierarchy custom setting (new) | Three Text(255) fields: `Default_Status__c`, `Default_Owner__c`, `Default_Linkage__c`, each holding semicolon-separated filter names |
| `EmailInboxController` | Apex class | Adds `getMyDefaultView()` and `saveMyDefaultView(...)` |
| `EmailInboxControllerTest` | Apex test | 8 new tests |
| `emailInbox` | LWC | Loads the saved view on open, adds the button, points Reset at the user's default; 9 new jest tests |

**Out of scope**

- **The "Assigned to" filter** is not saved. Its list of people differs per inbox and is already cleared whenever the user switches inbox, so a saved value could point at someone not on the next queue.
- **Everyone else's default.** Users who haven't saved a view keep the built-in Open + Unclaimed + Unlinked.

## 4. How it works (background)

The Shared Inbox (`emailInbox` LWC) has three filter axes:

- **Status** (e.g. Open)
- **Ownership** (Unclaimed, Mine)
- **Linkage** (Linked, Unlinked)

Before this change, the inbox always opened on a built-in combination, and **Reset** returned to that same built-in combination. There was nowhere to store a user's preference.

## 5. Root cause

Not applicable: this was a feature request, not a bug. The inbox had no per-user storage for filters, so any change to the default would have changed it for everyone.

## 6. Solution

### Where the preference is stored

A **hierarchy custom setting**, `Email_Inbox_Settings__c`, one row per user. The same pattern is already used by `DocketDeckSettings__c`. A hierarchy setting is read with `getInstance()`, which walks user → profile → org, so a per-user row is exactly what's needed.

### What the button does

- **Save as my default** saves the current Status, Ownership and Linkage filters for the person who clicks it. From then on their inbox opens with those filters, on any device.
- **Reset** goes back to *their* saved view instead of the built-in one, and its tooltip names that view.
- The button is greyed out while the filters already match their default.

### Design choices

- **Added to the existing controller, not a new class.** A new class would need class access granted on the inbox permission set and on every profile that uses the inbox. Users who can open the inbox can already call this one.
- **Only the caller's own row is written.** The save upserts the row owned by the running user and never creates a profile-level or org-level row.
- **Bad input is rejected before any database write**, including an axis with nothing turned on, which would show an empty inbox.
- **A half-readable saved row counts as no saved view.** If someone edits the setting in Setup and leaves an axis blank or misspelt, the inbox falls back to the built-in default rather than opening on half a view.
- **The read is not cached**, or the inbox would reopen on the old view right after a save.
- **The first list query waits for the saved view.** If the user clicks a filter before it arrives, their click wins. Reset clicked while loading still lands on the saved view.
- **One wording everywhere.** The "Showing:" line, the Reset tooltip and the save toast all describe a view the same way, for example `Open · Unclaimed + Mine · any linkage`.
- **No permission set needed.** The fields are read and written only by the Apex controller; users never touch them directly.

## 7. Testing

- **Apex:** validated against production, **57 of 57** `EmailInboxControllerTest` tests passing, including the 8 new ones. One test confirms that one user's save leaves another user's view empty. **This ran no browser code.**
- **Jest:** **21 of 21** `emailInbox` tests passing, including the 9 new ones. Both review-round tests fail against the previous code, and removing the line that makes the first query wait for the saved view fails exactly one test, so the race test is real.
- **Live:** deployed to production on 2026-10-20. The requesting staff were sent setup steps; the requester replied with thanks, and the ticket was closed on 2026-10-23.

**Not covered by anything automated:** the button rendering on the real Lightning page, and the toast.

## 8. Step by step

**For a user setting their own default:**

1. Open the Shared Inbox tab and hard-refresh the page once.
2. In the filter row, turn on the filters you want (for this request: **Mine** under Ownership and **Linked** under Linkage, so both turn blue). The line under the filters should read `Showing: Open · Unclaimed + Mine · any linkage`.
3. Click **Save as my default**. A green "Saved as your default view" message appears.
4. Refresh the page, or close the tab and open it again. The inbox should open with the saved filters.

**To repeat the build:**

5. Retrieve production's `EmailInboxController` and `emailInbox` and confirm they match the repo before editing.
6. Create the hierarchy custom setting with the three Text fields.
7. Add the read and save methods to the existing controller; write only the running user's row.
8. In the LWC, load the saved view before the first list query, add the button, and point Reset at the saved view.
9. Validate against production with `EmailInboxControllerTest`, run jest, then deploy after approval.
10. **Open the screen.** Check that the inbox still opens on the built-in default for a user with no saved view, that the button is greyed out, and that the Reset tooltip names the default.

## 9. Rollback

Redeploy the previous `EmailInboxController` and `emailInbox`. The custom setting can stay; with nothing reading it, saved rows have no effect. Safe to undo.

## 10. Known limits

- **An org-level row would become everyone's default.** Nothing in this change creates one, but an admin could add one in Setup to change the firm-wide default without code.
- **"Assigned to" is not saved**, by design.
- **Rows edited by hand in Setup** that are misspelt fall back silently to the built-in default.

## 11. Reuse for similar issues

**How to recognise one:** a user asks for "the default" to change, but other users rely on the current default.

- **Make it per user instead of changing it for everyone.** A hierarchy custom setting gives a per-user row with a natural fallback to profile and org.
- **Write only the running user's row**, and test that one user's save doesn't touch another's.
- **Validate stored preferences on read**, and fall back to the built-in default if they're incomplete.
- **Don't cache a preference read** that the same screen can change.
- **Remember the Apex validation runs no browser code.** Open the screen after deploying.
