# Red Flag Panel component with inline fixes


| | |
|---|---|
| **Ticket** | DDT-794 |
| **Project** | Red Flag Panel |
| **Status** | Live since 2026-10-08 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-08-21 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC, Apex) |
| **Area** | Matter red flags: record page |
| **Related** | DDT-793, DDT-795, DDT-796, DDT-797, DDT-1354 |
| **Code** | PR #652 and PR #663 in the ddt-salesforce repo |


---


## 1. Summary

The `redFlagPanel` Lightning Web Component replaces the old "Red Flags" field section. It groups flags by Court, Case Setup, Client Data and Money, sorts them Blocker, Warning, Info, colours each by severity, and gives each flag the button its rule names: Fix (edit the field in place), New (create a Court Appearance or Citation with the matter prefilled) or Open (go to the invoice). It renders nothing when a matter has no flags. A follow-up PR fixed three issues found in the first live test.

## 2. What was reported

- 2026-08-21: build the panel with grouping, severity sorting, inline fix and action buttons; render nothing when empty; support desktop and phone.

## 3. Scope

**In scope**

- `redFlagPanel` bundle and its Jest tests (PR #652).
- Follow-up (PR #663): stable group order, hide New buttons from users who cannot create the record, keep the icon beside its text on phone, and correct the `Red_Flag_Panel_Access` description.

**Out of scope**

- Placing the panel on pages (DDT-796, DDT-797).
- Reloading when data changes elsewhere (DDT-1354).

## 4. How it works (background)

The panel calls `RedFlagPanelController.getFlags` (DDT-793) through a cacheable wire. Each flag's rule says what action it supports. Before the panel, every flag was the same red and the section flagged 58% of matters, so staff ignored it.

## 5. Root cause

Not applicable for the build. For the follow-up, found in the live test on 2026-10-08:

1. Groups jumped around after a save, because ties between groups fell back to rule sort numbers.
2. A read-only user saw "New Court Appearance", a form they could not save.
3. On phone, a long message pushed the severity icon onto its own line.

## 6. Solution

PR #652:

- Inline edit uses a record edit form to render the right input for the field type (picklist, rich text, text), but its own save is cancelled and the value goes through the controller, where the rule metadata is the allowlist.
- `refreshApex` is awaited before the success toast, so the toast never shows against stale data; a test asserts the order.
- No Salesforce Date helper on DOB: the ticket asked for one, but the DOB field is an 8-character text field in production, so nothing parses a date.
- The card border sits on an inner element, so a clean matter leaves no empty box.
- Large and Small form factors, so the panel can go on the phone page.

PR #663:

- Groups still lead by their most urgent flag, but ties fall to a fixed order: Court, Case Setup, Client Data, Money.
- The controller returns a New button only when the user can create that object, the same pattern as the Fix button's edit check. In production, 8 panel users could not create Court Appearances or Citations; for them the button opened a form they could not save.
- Icon and message share one row that does not wrap; only the button drops below.

## 7. Testing

- PR #652: 7 Jest tests; breaking the sort or the refresh order made the matching tests fail. Check-only validation against production with the controller test, 20 of 20. It rendered nothing, as it was on no page yet.
- First live test on 2026-10-08 (desktop as admin, as a case management user, as a read-only user, and phone view): correct flags, colours and groups, empty on a clean matter, and five inline saves (Case Number, DOB, Address, rich-text Case Description, Language) confirmed in production on a test matter that was then deleted.
- PR #663: Jest 13 of 13, including a tied-group test that fails against the previous component and a test proving severity is the first sort key. Apex 21 of 21, including a read-only user who sees both flags but no New buttons. Jest does not lay out CSS, so the phone layout change was checked on screen.

## 8. Step by step

1. Wire the controller and group and sort the result in the component.
2. Map each action type to a control: inline edit, navigate to new record with the parent prefilled, navigate to a record, or none.
3. Save inline edits through the controller and await the refresh before the toast.
4. Render nothing when there are no flags.
5. Declare Large and Small form factors.
6. Test on screen as admin, as a normal user and as a read-only user; fix what the test shows.

## 9. Rollback

Redeploy the bundle and controller from the commit before the change. Before DDT-796, the bundle could be deleted outright because no page used it.

## 10. Known limits

- A user who cannot create a Court Appearance or Citation sees the flag with no button and no hint; a hint was left as an open question.
- The panel only refreshed after its own saves until DDT-1354.

## 11. Reuse for similar issues

- Await the data refresh before showing a success toast.
- Hide action buttons the user cannot complete, using the same create/update checks the server uses.
- Give groups a fixed tie-break order so the layout does not move after an edit.
- Always test a new panel on screen as a non-admin and a read-only user.
