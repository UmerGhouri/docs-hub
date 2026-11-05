# Reports and opportunity page failing to load

| | |
|---|---|
| **Ticket** | DDT-1506 |
| **Project** | Support tickets |
| **Status** | Done 2026-11-04 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-11-04 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Lightning reports, Court Case record page) |
| **Area** | Opening billing reports and Court Case (Opportunity) records |
| **Related** | DDT-1505 (similar page error for another user, reported two minutes earlier) |
| **Code** | None (no PR) |

---

## 1. Summary

A staff member could not open two billing reports, and a Court Case record opened from a report tab failed with a "Sorry to interrupt" error saying the user had no access to the record. The issue was resolved within about 15 minutes of the ticket being raised, and the user confirmed it worked. The ticket does not record the cause or what was changed.

## 2. What was reported

- **2026-11-04:** one user saw "Looks like there's a problem. Unfortunately, there was a problem. Please try again." with an error ID on the "Past Due Installments" report, and then on a second report, "Invoice Due Today or in the Next 3 Days".
- In the same session, a "Sorry to interrupt" dialog appeared while opening a Court Case record from a report workspace tab:

```
Failed to get generated module for forceGenerated:flexipage_RecordPage___Opportunity_Record_BB_Default_Hired__Opportunity___VIEW___LMT___...:
ui.force.api.act.javascript.JavaScriptGenerationException: You don't have access to this record.
Ask your administrator for help or to request access.
```

The user could not open reports needed for daily billing work.

## 3. Scope

**In scope:** whatever was changed to restore access. Not recorded in the ticket.

**Out of scope:** Not recorded in the ticket.

## 4. How it works (background)

```
User opens a report in a workspace tab
  -> clicks a Court Case (Opportunity) record
  -> Lightning generates code for the record page "Opportunity_Record_BB_Default_Hired"
  -> every component on that page loads for the current user
```

The error names the step where Lightning builds the record page's generated code, and says the user has no access to "this record". That can mean the Court Case itself, or something a component on the page loads.

## 5. Root cause

Not recorded in the ticket.

The ticket listed the possibilities to check: record access or sharing on the Court Case, report folder and report type access, field or object permissions behind a component on the record page, or a platform issue. All four acceptance criteria were ticked (root cause identified, both reports load, the record page opens or the gap is documented, the user confirms), but the findings were not written down.

## 6. Solution

Not recorded in the ticket. The user reported that it worked, including when they tried again on their own profile.

## 7. Testing

The user confirmed both that it worked and that it worked again on their own profile. No other testing is recorded in the ticket.

## 8. Step by step

The steps actually taken are not recorded. The checks below are the ones the ticket listed.

1. Record the error ID, the report names and the record the user was opening.
2. Have the user hard-refresh, clear the cache, try a private window or another browser, and close any stuck workspace tabs.
3. Check whether other users see it. DDT-1505 was a similar error for another user at the same time.
4. Check the user's access to both reports, their folders and their report types.
5. Check the user's access to the Court Case record, and the field and object permissions behind every component on the Court Case record page.
6. Look for changes made that day to the record page or its components.

## 9. Rollback

Not applicable: no change is recorded to roll back.

## 10. Known limits

- **The cause is unknown**, so this page cannot say whether it could happen again.
- **The record page changed that day.** On 2026-11-04 the same Court Case record page named in the error was edited in production (DDT-1386 added a tab with an archived-records component, and Archive access was assigned to the case managers), and a managed package was uninstalled (DDT-1332). DDT-1506 does not connect either change to this error, so treat this as a place to look, not a finding.

## 11. Reuse for similar issues

**How to recognise one:** a "You don't have access to this record" error inside a "Failed to get generated module for ...flexipage..." message. It is raised while the page is being built, so check the page's components as well as the record's sharing.

- **Read the flexipage name in the error.** It tells you exactly which record page failed.
- **Check for same-day page edits.** A newly added component that some users cannot access is a common cause of a whole page failing for them.
- **Two users with similar errors at the same time** suggests a shared change, not two separate permission problems.
- **Write the root cause on the ticket** when you close it.
