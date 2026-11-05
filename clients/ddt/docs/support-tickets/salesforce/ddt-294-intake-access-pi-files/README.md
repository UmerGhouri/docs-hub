# Intake team access to PI case files

| | |
|---|---|
| **Ticket** | DDT-294 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-10 |
| **Priority** | High |
| **Reported by** | Firm staff, on behalf of the Intake team, 2026-07-09 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (permission sets) |
| **Area** | PI cases: letting the Intake team read case files |
| **Related** | None recorded |
| **Code** | Built in production Setup. The permission set reached the `ddt-salesforce` repo later, in PR #115, which also added read access to `PI_Incident__c`. |

---

## 1. Summary

The Intake team started handling PI cases at the intake stage, but could see only the personal information page of a PI case, not its notes and related records. The requester asked for them to see everything. A read-only permission set, **Intake - PI Case Access**, was created in production and assigned to the seven Intake users, giving them view access to all PI case files.

## 2. What was reported

On 2026-07-09 the Intake team was blocked on PI cases: they could open the personal information page but not the case notes, related lists or other case-level detail. Seven Intake users were named. When asked which sections they needed, the requester answered "let them see everything". The change was reported done on 2026-07-10: the seven users had full **view** access to PI case files.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Intake_PI_Case_Access` (label "Intake - PI Case Access") | Permission set | New. Read + View All on `Matter__c`, `Account` and `Contact`; `Matter__c` tab visible |
| Seven Intake users | Permission set assignments | Assigned the new permission set |

Added later, under PR #115 (2026-08-05): Read + View All on `PI_Incident__c` in the same permission set.

**Out of scope**

- **Edit access.** The request was to see the files. Every object permission is read-only: no create, edit or delete.
- **Field-level security.** The permission set grants object access only and carries no field permissions. What fields a user sees still comes from their profile and other permission sets.

## 4. How it works (background)

A PI case lives on the `Matter__c` object, related to the client's `Account` and `Contact`. What access the Intake users had before the change is not recorded in the ticket; what they saw was only the personal information page.

Two Salesforce rules matter here:

- **Read** on an object lets a user open records they can already reach through sharing. **View All** lets them read every record of that object, regardless of owner or sharing rules.
- **A report is hidden entirely** if the user lacks Read on any object in its report type.

## 5. Root cause

Not recorded in the ticket. The ticket listed possible causes to check (field-level security, page layout, record type visibility or sharing), but the resolution does not say which one applied. The fix that was applied, granting object-level Read + View All, implies the users lacked object access or sharing to the PI case records.

## 6. Solution

### 6.1 The permission set

A new permission set, created in production Setup, as later retrieved into the repo:

| Object | Read | View All | Create / Edit / Delete |
|---|---|---|---|
| `Matter__c` | yes | yes | no |
| `Account` | yes | yes | no |
| `Contact` | yes | yes | no |
| `PI_Incident__c` (added 2026-08-05, PR #115) | yes | yes | no |

Plus the `Matter__c` tab set to Visible.

Why it was done this way:

- **A separate, additive permission set**, not a profile edit. It can be assigned to exactly the people who need it and removed without touching anyone else.
- **View All, not sharing rules.** The request was "everything", and View All grants it in one place without building sharing rules per record owner.
- **Read-only.** Intake needed to see the files, not change them.

### 6.2 The follow-up in PR #115

On 2026-08-04 a user holding this permission set could not see a PI report in a public folder. The report's type spans `Account`, `Matter__c` and `PI_Incident__c`, and the permission set had no access to `PI_Incident__c`, so Salesforce hid the report. PR #115 added Read + View All on `PI_Incident__c`. At that point the permission set had 8 assignees, all of whom already had View All on `Matter__c` and `Account`, so the addition exposed no case data they could not already reach.

PR #115 is also where the permission set first entered git: it was retrieved from production, which was ahead of `main`.

## 7. Testing

**Live check:** the change was reported to the requester on 2026-07-10 and acknowledged. The ticket asked for access to be verified with one Intake user; whether that check was done is not recorded in the ticket.

To check who holds the permission set, read-only:

```sql
SELECT Assignee.Name, Assignee.IsActive
FROM PermissionSetAssignment
WHERE PermissionSet.Name = 'Intake_PI_Case_Access'
```

**Not covered:** no automated test. Permission sets are configuration.

## 8. Step by step

1. **Confirm what the users need** with the requester: read only, or edit too, and which records. Here it was read-only, everything.
2. **Create the permission set** in Setup → Permission Sets → New, label "Intake - PI Case Access".
3. **Object settings:** grant Read + View All on `Matter__c`, `Account`, `Contact` and `PI_Incident__c`. Leave Create, Edit and Delete off.
4. **Tab settings:** set the `Matter__c` tab to Visible.
5. **Assign** the permission set to each named user (Manage Assignments → Add Assignment).
6. **Verify** with the query in section 7, then have one user open a PI case and a PI report.
7. **Record it in git** the same day: retrieve the permission set from production and open a PR. This one reached the repo only in PR #115, almost a month later.

## 9. Rollback

Remove the assignments, or remove the object permissions from the permission set. Both are immediate and safe: nothing else depends on this permission set, and removing it returns users to whatever their profile grants.

## 10. Known limits

- **View All is broad.** Holders can read every `Matter__c`, `Account` and `Contact` record, not only PI ones and not only intake-stage ones. Assign it only to people who should see all of them.
- **No field permissions.** A field hidden by a user's profile stays hidden; this permission set will not fix that.
- **Any new object on the PI case page or in PI report types needs its own Read.** PR #115 was exactly this: a report type gained an object the permission set did not cover.

## 11. Reuse for similar issues

**How to recognise one:** a team can open part of a record but not its related lists, notes or reports, and the request is "let them see it", not "let them edit it".

- **Prefer an additive, read-only permission set** named for the team and purpose, over editing a shared profile.
- **List every object the users will touch**, including objects that only appear inside report types. A missing one hides the whole report.
- **Decide Read versus View All on purpose.** Read follows sharing; View All bypasses it for that object.
- **Retrieve the permission set into git the same day** it is created in Setup.
