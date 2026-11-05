# Creating a new Salesforce user by cloning a profile

| | |
|---|---|
| **Ticket** | DDT-115, DDT-117, DDT-204, DDT-266, DDT-267 |
| **Project** | Support tickets |
| **Status** | Done 2026-07-07 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-06-05 to 2026-07-03 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Setup: users, profiles, permission sets, managed-package licences) |
| **Area** | Staff onboarding: giving a new hire the same access as an existing colleague |
| **Related** | None recorded |
| **Code** | None. All work was done in Setup |

---

## 1. Summary

Five new hires (one bilingual case-team member, two attorneys and two case managers) each needed a Salesforce login before their first day, set up "the same as" a named colleague in the same job. Each user was created in production with the same profile and role as that colleague. The case-manager requests also named four managed-package licences to assign.

The tickets record the request and the deadline. They do not record the exact steps taken, so this page gives the steps the tickets asked for, what production shows today, and queries to check a new user against the colleague they were copied from.

## 2. What was reported

- **2026-06-05, DDT-115.** A bilingual (English and Spanish) case-team member starting 2026-06-08, to be set up like an existing colleague. The ticket also asked for the user's language to be set for routing.
- **2026-06-08, DDT-117.** A new attorney starting that morning, to be set up like an existing attorney.
- **2026-06-19, DDT-204.** A second new attorney starting 2026-06-22, copied from the same attorney.
- **2026-07-03, DDT-266.** A new case manager starting 2026-07-06, copied from an existing case manager, plus four managed-package licences: S-Docs, SSign, XFilesPro (the CloudFiles package) and Payment Center.
- **2026-07-03, DDT-267.** A second case manager, same source user and same four licences.

Each had a hard start date. A new hire who cannot log in, or who logs in and hits permission errors, loses their first morning.

## 3. Scope

**In scope**

| Item | Where |
|---|---|
| New user record (name, username, email, time zone, locale) | Setup → Users |
| Licence, profile and role, matching the source user | User record |
| Permission sets and permission set groups, matching the source user | User record → Permission Set Assignments |
| Managed-package licences (DDT-266 and DDT-267) | Setup → Installed Packages → Manage Licenses, or the user record |
| Queue and public group membership, matching the source user | Setup → Queues / Public Groups |
| Welcome / set-password email | Sent when the user is saved with "Generate new password and notify user" ticked |

**Out of scope**

- **Systems outside Salesforce** (email, Slack, phone system, calendar tools, court portals). The tickets listed these as open questions; none was handled here.
- **Changing what the source user can do.** If the colleague being copied was missing a permission, the copy is missing it too. Fixing that is a separate ticket.
- **Moving cases to the new user.** Reassigning work is a separate request.

## 4. How it works (background)

What a user can do in Salesforce comes from several places, and **copying the profile copies only the first of them**:

```
User
  ├── Licence            (Salesforce, Platform, ...)      on the user record
  ├── Profile            object, field and app access     on the user record
  ├── Role               record visibility via the hierarchy   on the user record
  ├── Permission sets / permission set groups             separate assignment records
  ├── Managed-package licences (fw1, SDOC, ...)           separate assignment records
  └── Queue and public group membership                   separate membership records
```

Salesforce has no "clone user" button. You create a new user and set each of these to match the source user yourself. The items on separate records are the ones most often missed, because nothing on the user record shows them.

## 5. Root cause

Not applicable: these were requests for new users, not faults.

The one recurring risk the tickets point out is real, though: **a copied user inherits the source user's gaps.** DDT-115 noted that the colleague being copied had just hit a missing Apex class permission, and asked for it to be fixed on both users together so the new hire did not start with the same error.

## 6. Solution

Each user was created in production with the same profile and role as the named colleague. Read back from production on 2026-11-05:

| Ticket | New user's profile | Matches source user's profile | Role | Time zone |
|---|---|---|---|---|
| DDT-115 | DDT Matter Management | Source user not identified in this check | All Employees | America/Chicago |
| DDT-117 | DDT Standard User | Yes | All Employees | America/Chicago |
| DDT-204 | DDT Standard User | Yes | All Employees | America/Chicago |
| DDT-266 | DDT Case Management | Yes | All Employees | America/Chicago |
| DDT-267 | DDT Case Management | Yes | All Employees | America/Chicago |

The users were created on 2026-06-05, 2026-06-08, 2026-06-19, 2026-07-03 and 2026-07-06 respectively. Three of the tickets (DDT-117, DDT-204, DDT-266) were logged after the user already existed, so the ticket is the record of the request rather than a work queue.

**Managed-package licences.** On 2026-11-05 the case-manager user created under these tickets who is still active held the same managed-package licences as the case manager they were copied from (Payment Center `fw1` and Dochly). Which of the four requested licences were assigned in July is not recorded in the ticket, and today's licences reflect later changes too.

**Language for routing (DDT-115).** Whether a language field was set, and which field, is not recorded in the ticket.

## 7. Testing

Not recorded in the ticket. None of the five tickets records a login test or a check against the source user.

The read-back in section 6 is the only verification on file. Use the queries in section 8 to check a new user properly.

## 8. Step by step

These are the steps the tickets laid out, in the order that avoids missing one.

1. **Check the source user first.** Make sure the colleague being copied has everything they should. If they are missing a permission, fix it on both users together.
2. **Read the source user's setup** (replace the placeholders):

    ```sql
    -- Licence, profile, role, time zone
    SELECT Profile.Name, UserRole.Name, Profile.UserLicense.Name,
           TimeZoneSidKey, LocaleSidKey, LanguageLocaleKey
    FROM User WHERE Id = '<source-user-id>'

    -- Permission sets and groups (leave out the hidden one that belongs to the profile)
    SELECT PermissionSet.Name, PermissionSetGroup.DeveloperName
    FROM PermissionSetAssignment
    WHERE AssigneeId = '<source-user-id>'
      AND PermissionSet.IsOwnedByProfile = false

    -- Managed-package licences
    SELECT PackageLicense.NamespacePrefix
    FROM UserPackageLicense WHERE UserId = '<source-user-id>'

    -- Queues and public groups
    SELECT Group.Name, Group.Type
    FROM GroupMember WHERE UserOrGroupId = '<source-user-id>'
    ```

3. **Check licence seats are free** before you start: Setup → Company Information (Salesforce licences) and Setup → Installed Packages → Manage Licenses (each package). Two hires at once need two free seats of each.
4. **Create the user.** Setup → Users → New User. Set name, email, username, the same licence, profile and role as the source user, and time zone Central (America/Chicago). Leave "Generate new password and notify user immediately" ticked if they should get the welcome email now.
5. **Assign the same permission sets and permission set groups** from the list in step 2.
6. **Assign the managed-package licences** the request names, from each package's Manage Licenses page.
7. **Add the user to the same queues and public groups.** Routing that assigns work by queue will not see the new user otherwise.
8. **Set any routing fields** on the user record that the request mentions, such as language.
9. **Run the queries in step 2 again for the new user** and compare the two results side by side. Any difference should be one the request asked for.
10. **Tell the requester the user is ready**, and record it on the ticket.

## 9. Rollback

Deactivate the user: Setup → Users → Edit → untick Active. Salesforce users cannot be deleted. Deactivating frees the Salesforce licence. Remove managed-package licences separately if the seat is needed for someone else.

One of the five users created under these tickets has since been deactivated, which is the normal end of a user's life in Salesforce.

## 10. Known limits

- **A copy is only as good as its source.** Missing permissions on the source user are copied too.
- **The source user can leave.** One of the users used as a source in these tickets was inactive by 2026-11-05. Before copying, check the source user is still active and still in the same job.
- **Package licences change.** Packages get installed, replaced and retired, so the licences a source user holds may not be the ones a new hire in the same job needs. Check with the requester.
- **The hidden profile permission set.** A query on `PermissionSetAssignment` without `PermissionSet.IsOwnedByProfile = false` also returns a hidden row for the user's profile. Comparing raw counts between two users can disagree because of it.

## 11. Reuse for similar issues

**How to recognise one:** "set up X the same as Y", "clone Y's profile", or a new user who can log in but gets permission errors that a colleague in the same job does not.

- **Compare against a named colleague**, not against a memory of what the job needs. Run the four queries in step 2 for both users and read the differences.
- **Profile is not the whole picture.** Permission sets, package licences, queues and public groups are separate records and are the parts usually missed.
- **Batch hires who share a source user.** Check the source once, check seats for all of them at once, then create them together.
- **Check seats first.** A managed-package licence with no free seat stops the setup halfway.
- **Record what you assigned on the ticket.** None of these five tickets did, which is why this page cannot say exactly what was done.
