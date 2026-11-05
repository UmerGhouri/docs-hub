# Assigning CloudFiles licence and permission set

| | |
|---|---|
| **Ticket** | DDT-121, DDT-133, DDT-281, DDT-579 |
| **Project** | Support tickets |
| **Status** | Done (DDT-121 2026-06-09, DDT-133 2026-06-12, DDT-281 2026-07-13, DDT-579 2026-07-30) |
| **Priority** | High (DDT-121, DDT-133, DDT-281); Medium (DDT-579) |
| **Reported by** | Firm staff, 2026-06-08 to 2026-07-29 |
| **Resolved by** | Support team |
| **Systems** | Salesforce, CloudFiles (managed package) |
| **Area** | User access: the Files widget on Account and Court Case records |
| **Related** | DDT-222 (CloudFiles Google Drive connection, also fixed by providing a licence) |
| **Code** | Not applicable: access was granted in the org; no metadata changed |

---

## 1. Summary

Four users, one at a time, could not see files in Salesforce. The CloudFiles widget told each of them they needed a CloudFiles licence. The fix each time was to give the user a CloudFiles licence and the CloudFiles permission set; one user also needed access to the Apex class behind the Court Case Files tab.

## 2. What was reported

- **DDT-121 (2026-06-08).** A user got a "needs access to CloudFiles" message when viewing records. The ticket asked for the `CloudFiles_Standard_User_DDT` permission set.
- **DDT-133 (2026-06-09).** A user could not access any files. The widget showed: "You need CloudFiles license to access this widget. Please contact your account admin to assign you the license."
- **DDT-281 (2026-07-07).** A user saw the same licence message on the Files widget of a Person Account. The requester also asked that CloudFiles access be set up when a new team member's Salesforce account is created, rather than after the fact.
- **DDT-579 (2026-07-29).** A user saw the same licence message on the Files tab of a Court Case, plus a toast: "You do not have access to the Apex class named 'CourtCaseManagement'." She needed it to check whether representation agreements had been sent to clients for signature.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| CloudFiles licence | Managed package licence | Assigned to each user |
| `CloudFiles_Standard_User_DDT` | Permission set | Assigned to each user who lacked it |
| `CourtCaseManagement` | Apex class access | Needed for DDT-579; production grants it through the `Court_Case_Managment` permission set (read 2026-11-05) |

**Out of scope**

- Making CloudFiles part of new-user setup (asked for in DDT-281). Not recorded as done.
- Buying more CloudFiles licences.

## 4. How it works (background)

Seeing files through CloudFiles needs two separate things:

```
User opens the Files widget on a record
  -> CloudFiles checks the user holds a CloudFiles licence   (else: "You need CloudFiles license ...")
  -> the user's permissions must allow the CloudFiles app, its objects and its Apex classes
       (CloudFiles_Standard_User_DDT grants these)
  -> on Court Case, the Files tab also calls the DDT Apex class CourtCaseManagement
       (else: "You do not have access to the Apex class named 'CourtCaseManagement'")
```

A permission set alone does not clear the licence message, and a licence alone does not grant the permissions.

## 5. Root cause

Each user was simply missing access:

- **DDT-121:** the support team recorded "It was a license issue." The user had no CloudFiles licence.
- **DDT-133, DDT-281:** the widget message itself names the missing licence.
- **DDT-579:** missing licence, and no permission set granting the `CourtCaseManagement` Apex class.

**Contributing cause, from DDT-281:** on 2026-07-08 the support team recorded that **no CloudFiles licences were available**, so a new licence could not be assigned until one was freed or bought. How it was resolved by 2026-07-13 is not recorded in the ticket.

**Underlying cause:** CloudFiles access was not part of setting up a new Salesforce user, so each new user hit the error first and raised a ticket.

## 6. Solution

For each user:

1. Assign a CloudFiles licence. Where in the org this was done is not recorded in the tickets.
2. Assign the `CloudFiles_Standard_User_DDT` permission set. DDT-133 listed five CloudFiles permission sets in the org at the time (`CloudFiles_Admin_User`, `CloudFiles_External_User`, `CloudFiles_Field_Service_User`, `CloudFiles_Standard_User`, `CloudFiles_Standard_User_DDT`); the DDT one is the firm's standard.
3. **For Court Case access (DDT-579):** also make sure the user can run `CourtCaseManagement`. Which permission set was used is not recorded; `Court_Case_Managment` grants it.
4. Ask the user to refresh the record and confirm.

## 7. Testing

The user's own confirmation was the check:

- **DDT-121:** the user confirmed "it is all set now" after the licence was fixed.
- **DDT-133, DDT-281, DDT-579:** Not recorded in the ticket.

To confirm the permission sets afterwards:

```sql
SELECT Assignee.Name, PermissionSet.Name
FROM PermissionSetAssignment
WHERE PermissionSet.Name IN ('CloudFiles_Standard_User_DDT', 'Court_Case_Managment')
  AND AssigneeId = '<userId>'
```

## 8. Step by step

1. **Read the exact message.** "You need CloudFiles license" means the licence. "No access to the Apex class" means a permission set.
2. **Check a licence is free.** If none are, say so to the requester straight away (DDT-281 waited on this).
3. **Assign the CloudFiles licence** to the user.
4. **Assign `CloudFiles_Standard_User_DDT`:** Setup → Users → the user → Permission Set Assignments → Edit Assignments.
5. **If the error is on a Court Case's Files tab**, also assign a permission set that grants `CourtCaseManagement`, such as `Court_Case_Managment`.
6. **Ask the user to refresh** and confirm the widget loads.
7. **Verify** with the query in section 7.

## 9. Rollback

Remove the permission set assignment and the licence from the user. Both are reversible and affect only that user.

## 10. Known limits

- **Licences are finite.** When they run out, the next new user waits (DDT-281).
- **It recurs for every new user** until CloudFiles access is part of user setup.
- **CloudFiles is being replaced.** A read of production on 2026-11-05 found only one of the five permission sets listed in DDT-133 (`CloudFiles_Standard_User_DDT`) and no package licence record for CloudFiles. Check what the Files widget on a record actually uses today before following these steps.

## 11. Reuse for similar issues

**How to recognise one:** a single user sees a "you need a licence" or "no access to Apex class" message that colleagues in the same role do not.

- **Managed packages need a licence and a permission set.** Fix both.
- **Read the error literally.** It usually names exactly what is missing.
- **Compare with a colleague who works:** list their permission sets and package licences and copy the difference.
- **Check licence headroom before promising a fix.**
- **Fix it at onboarding.** If several tickets ask for the same access for new users, add it to the new-user checklist.
