# MassMailer install and user access

| | |
|---|---|
| **Ticket** | DDT-162, DDT-187, DDT-504 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-24 |
| **Priority** | Medium (DDT-162, DDT-504), High (DDT-187) |
| **Reported by** | Firm staff, 2026-06-12, 2026-06-17 and 2026-07-24 |
| **Resolved by** | Support team |
| **Systems** | Salesforce, MassMailer managed package |
| **Area** | Client-services mass email: MassMailer access for a staff user |
| **Related** | None recorded |
| **Code** | None. Licence and permission set assignments in Setup |

---

## 1. Summary

One client-services staff member needed MassMailer in production, and hit three separate access walls over six weeks: no access at all, then no access to the Template Builder, then no access to create Campaigns. MassMailer was already installed in production. Each step was fixed by assigning a MassMailer licence or permission set, and in one case by logging out and back in so the new permissions took effect.

## 2. What was reported

- **DDT-162 (2026-06-12):** the staff member could use MassMailer only in the sandbox and asked for it to be installed in production and for her to be added as a user.
- **DDT-187 (2026-06-17):** opening MassMailer Template Builder gave "You don't have access to MassMailer Template Builder". She confirmed she needed both to build templates and to send mass email.
- **DDT-504 (2026-07-24):** creating a MassMailer Campaign gave an access-denied error.

One user was affected each time.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| MassMailer package licence | Managed package licence | Assigned to the user (DDT-162) |
| `MassMailer_User_Permissions` | Permission set | Assigned to the user (DDT-162) |
| Template Builder access | Permission set | Assigned (DDT-187); which set is not recorded in the ticket |
| Campaign creation access | Permission set or licence | Assigned (DDT-504); which one is not recorded in the ticket |

**Out of scope**

- **Installing MassMailer in production.** It was already installed.

## 4. How it works (background)

A managed package like MassMailer gates access in layers, and each must be in place:

```
Managed package licence assigned to the user
  -> permission set(s) the package ships, granting its objects, tabs and pages
  -> the user's session picks up the new permissions (log out and back in)
```

The MassMailer permission sets the DDT-187 ticket lists as present in the org:

| API name | Label |
|---|---|
| `MassMailer_User_Permissions` | MassMailer User |
| `MassMailer_User_Permissions_Clone` | MassMailer User - Additional |
| `MassMailer_Admin_Permissions` | MassMailer Admin |

The ticket noted that `MassMailer_Admin_Permissions` was likely needed for full Template Builder access.

## 5. Root cause

- **DDT-162:** the request assumed the package was missing from production. It was installed; the user simply had no licence or permission set.
- **DDT-187:** the user permission set from DDT-162 did not cover the Template Builder. After further access was assigned the error persisted until she logged out and back in, which then fixed it.
- **DDT-504:** her assignments did not yet allow creating Campaigns. The exact missing permission is not recorded in the ticket.

The common thread: access was granted for the task asked about each time, not for everything the user said she needed to do.

## 6. Solution

- **DDT-162 (2026-06-15):** assigned the MassMailer user licence and the `MassMailer_User_Permissions` permission set, and told her to log out and back in, then open App Launcher → MassMailer Lightning.
- **DDT-187 (2026-06-17):** assigned further MassMailer access for the Template Builder. When she still saw the error, she was asked to log out and back in; on 2026-06-18 she confirmed it worked.
- **DDT-504 (2026-07-24):** assigned the access needed to create Campaigns. She confirmed it worked the same day.

## 7. Testing

The user confirmed access worked after DDT-187 and after DDT-504. Each confirmation is recorded on its ticket. No other check was recorded.

## 8. Step by step

1. **Check the package is installed** before asking for an install: Setup → Installed Packages.
2. **Ask the user what they need to do** in the package: send, build templates, create campaigns, administer. Grant for all of it at once.
3. **Assign the package licence:** Setup → Installed Packages → MassMailer → Manage Licenses → Add Users.
4. **Assign the matching permission set(s):** Setup → Users → the user → Permission Set Assignments. Use `MassMailer_Admin_Permissions` if they build templates.
5. **Have the user log out and back in.** Package permission changes often do not show in an open session.
6. **Ask them to try each task** (open Template Builder, create a Campaign, send) and confirm on the ticket.

## 9. Rollback

Remove the permission set assignments and the package licence from the user. Safe, but she loses MassMailer access.

## 10. Known limits

- **Package licences are seat-limited.** Assigning one may require removing another user.
- **The admin permission set grants more than template building.** Granting it to a non-admin is a judgement call for the firm.

## 11. Reuse for similar issues

**How to recognise one:** "You don't have access" inside a managed package's page, for a user who can otherwise use the package.

- **Check before installing.** "It's only in the sandbox" can mean "I have no access in production".
- **Ask for the full list of tasks**, so one ticket grants everything rather than three tickets each granting a little.
- **Licence, then permission set, then a fresh login.** A grant that "didn't work" often only needs the user to log in again.
