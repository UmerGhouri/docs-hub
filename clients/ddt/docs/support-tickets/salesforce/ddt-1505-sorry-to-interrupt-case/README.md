# Sorry to interrupt error opening case records

| | |
|---|---|
| **Ticket** | DDT-1505 |
| **Project** | Support tickets |
| **Status** | Done 2026-11-04 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-11-04 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Lightning record page) |
| **Area** | Opening client case records |
| **Related** | DDT-1506 (similar page error for another user, reported two minutes later) |
| **Code** | None (no PR) |

---

## 1. Summary

A staff member got a "Sorry to interrupt — This page has an error" dialog every time she opened a client case record. The Support team fixed it within about 20 minutes of the ticket being raised and asked her to hard-refresh, after which she could open the cases again. The ticket does not record what the cause was or what was changed.

## 2. What was reported

- **2026-11-04:** one user got "Sorry to interrupt" with a Salesforce error ID on every client case she tried to open (a couple so far), so it did not look limited to one record. She could not work on those clients.

The ticket was raised at about the same time as DDT-1506, where another user got a similar error on the Court Case record page and two reports.

## 3. Scope

**In scope:** whatever was changed to restore access. Not recorded in the ticket.

**Out of scope:** Not recorded in the ticket.

## 4. How it works (background)

A Lightning record page is assembled in the browser from the page's components. If any component, or the generated code for the page, fails for the current user, Salesforce shows the "Sorry to interrupt" dialog with an error ID instead of the page. Lightning also caches page code in the browser, so a fix on the server may not reach a user until they reload.

## 5. Root cause

Not recorded in the ticket.

The ticket's checklist named the things to look at: the error ID, whether other users were affected, recent changes to Case flows, layouts and Lightning pages, and the user's profile, permission sets and record access. All three acceptance criteria (root cause identified, the user can open cases, whether other users were affected) were ticked, but the findings themselves were not written down.

## 6. Solution

The Support team reported the issue fixed and asked the user to do a hard refresh (`Ctrl+Shift+R`) or log out and back in, then open the case again. She confirmed she could open the case files again.

What was changed in Salesforce is not recorded in the ticket.

## 7. Testing

The user confirmed she could open case files again. No other testing is recorded in the ticket.

## 8. Step by step

The steps actually taken are not recorded. The checks below are the ones the ticket listed, plus the refresh that closed it.

1. Note the error ID and the exact page or record the user was opening.
2. Ask whether anyone else sees it. One user points to permissions or that user's browser; everyone points to a page or platform change.
3. Look for changes made that day to the record page, its components, layouts or flows.
4. Check the user's profile, permission sets and record sharing for the object, and for anything the page's components read.
5. After fixing, ask the user to hard-refresh (`Ctrl+Shift+R`) or log out and back in, so the browser drops the cached page code.

## 9. Rollback

Not applicable: no change is recorded to roll back.

## 10. Known limits

- **The cause is unknown**, so this page cannot say whether it could happen again.
- **Something changed the same day.** On 2026-11-04 the Court Case record page was edited in production (DDT-1386 added a tab with an archived-records component, and Archive access was assigned to the case managers), and a managed package was uninstalled (DDT-1332). DDT-1505 does not connect either change to this error, so treat this as a place to look, not a finding.

## 11. Reuse for similar issues

**How to recognise one:** a user gets "Sorry to interrupt" on every record of one type, not on a single record.

- **Ask who else sees it** before changing anything.
- **Check what changed on that page today.** A new component the user cannot access can break the whole page for them.
- **Always end with a hard refresh.** Lightning caches page code, so a fix can look like it didn't work.
- **Write the root cause on the ticket** when you close it. This one cannot be learned from because it wasn't.
