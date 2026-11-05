# User gets a no-access-to-Apex-class error

| | |
|---|---|
| **Ticket** | DDT-101, DDT-229, DDT-237, DDT-264, DDT-279, DDT-578, DDT-654, DDT-774, DDT-1048, DDT-1296 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-12 (each fix was made in production on the day its ticket closed; DDT-1296 was the last) |
| **Priority** | High (DDT-229: Medium) |
| **Reported by** | Firm staff, including Case Managers and the Intake team, 2026-06-03 to 2026-10-09 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex class access, permission sets, profiles, LWC) |
| **Area** | Any page component backed by an Apex controller: Treatment Calls, Court Case, Court Appearance disposition entry, Client page, Referral Source and How They Found Us panels |
| **Related** | DDT-752 (automatic RingCentral call linking, the source of the DDT-774 banner), DDT-1285 (How They Found Us panel, the source of the DDT-1296 gap), DDT-509 (Referral Source attribution) |
| **Code** | PR #683 in the `ddt-salesforce` repo (DDT-1296 only). The other nine were fixed in Setup with no code change. |

---

## 1. Summary

Ten times between June and October 2026 a user opened a record and got a red banner or toast reading "You do not have access to the Apex class named '...'". Lightning components call Apex controllers, and a user can only run a controller if their profile or one of their permission sets grants access to that class. Each time, the user was missing that grant.

Nine of the ten were fixed in Setup by assigning the user the permission set that already grants the class (or, in one case, by adding the class to every profile). The tenth, DDT-1296, was different: the class had never been added to any permission set, so it failed for every non-admin user, and it was fixed in code (PR #683).

## 2. What was reported

| Ticket | Reported | Class named in the error | Where the user saw it |
|---|---|---|---|
| DDT-101 | 2026-06-03 | `TreatmentCallController` | Treatment Calls ("Tx Call Log") component on a client record, reported by a PI Case Manager |
| DDT-229 | 2026-06-24 | `Path_lwcsvc_PathWrapper` | A pop-up on every Court Case record load; the Path component on the Court Case page |
| DDT-237 | 2026-06-29 | `DispositionEntryController` | "Enter Disposition" component on a Court Appearance record; blocked closing out court dates |
| DDT-264 | 2026-07-02 | `EnhancedRelatedList_CC` | A related list on the Person Client page (seen on mobile); the rest of the page loaded |
| DDT-279 | 2026-07-07 | `CourtCaseManagement` | Banner on the Court Case record page. The user said it was happening "again" |
| DDT-578 | 2026-07-29 | `CourtCaseManagement` | Same banner on a Court Case record, a different user (a Case Manager) |
| DDT-654 | 2026-08-05 | `TreatmentCallController` | Logging a Treatment Call from the Communication tab on a PI Matter |
| DDT-774 | 2026-08-19 | `LinkRcCallsAction` | Warning "Could not link RingCentral calls automatically: ..." A second user replied in the same thread that she was getting it too |
| DDT-1048 | 2026-09-17 | `ReferralSourceController` | "Could not save referrer" when saving "Who referred you" on an Account |
| DDT-1296 | 2026-10-09 | `HowTheyFoundUsController` | The "How They Found Us" panel showed the error instead of the picker, reported by the Intake team |

In every case except DDT-1296 the component worked for other users, which is the tell that the problem is one user's access and not the code.

Why it matters: in most of these the page still loads, but the one component that needs the class is dead. For DDT-237 that blocked a daily workflow (entering dispositions). For DDT-1048 and DDT-1296 it blocked recording referral and marketing attribution.

## 3. Scope

**In scope**

| Ticket | Component | Type | Change |
|---|---|---|---|
| DDT-654 | `Treatment_Call_Management` | Permission set | Assigned to the user (recorded in the ticket) |
| DDT-101 | Not recorded | Permission set / access | "Updated the permissions." Which grant was used is not recorded in the ticket |
| DDT-229, DDT-578, DDT-1048 | Not named in the ticket | Permission set | The user was assigned the permission set they were missing |
| DDT-237 | Not named in the ticket | Permission set / access | Access granted. The firm asked that all users get access to the Disposition Entry component |
| DDT-264, DDT-279 | Not recorded | Access | Access given; the user confirmed |
| DDT-774 | `LinkRcCallsAction` | Profile class access | Class access added to the profiles (see 6.2) |
| DDT-1296 | `How_Heard_Attribution` | Permission set (metadata, PR #683) | `HowTheyFoundUsController` added to its class accesses |
| DDT-1296 | `docs/deploys/how-they-found-us.md` | Deploy doc | Corrected the claim that users without the permission set see a locked line |

For reference, the permission sets that grant each class in production today (read from `SetupEntityAccess` on 2026-11-05):

| Class | Permission set that grants it |
|---|---|
| `TreatmentCallController` | `Treatment_Call_Management` |
| `DispositionEntryController` | `Disposition_Entry_Access` |
| `CourtCaseManagement` | `Court_Case_Managment` (the API name is misspelled; search for it as written) |
| `ReferralSourceController`, `HowTheyFoundUsController` | `How_Heard_Attribution` |
| `Path_lwcsvc_PathWrapper` (managed package, `pgrp` namespace) | `BBT_Path_User`, `BBT_Path_Admin_User` |
| `EnhancedRelatedList_CC` (managed package, `CMTD` namespace) | `CMTD_ERL_Permission_Set` |
| `LinkRcCallsAction` | No permission set; granted on the profiles |

Some of these classes are also granted on individual profiles, including System Administrator.

**Out of scope**

- **Why DDT-279 recurred.** The ticket asked for the cause of the recurrence. It is not recorded in the ticket.
- **Users with neither the permission set nor an admin profile (DDT-1296).** A small number of users on read-only or similar profiles still see the error. Left as a known gap, see 10.
- **Where the How They Found Us panel appears.** DDT-1296 found the panel was never on the Potential Court Case page, although the ticket assumed it was. Raised separately.

## 4. How it works (background)

```
User opens a record page
  -> a Lightning component on the page (LWC, Aura or a managed-package component) loads
  -> it calls a method on its Apex controller class
  -> Salesforce checks: does this user's profile, or any of their permission sets,
     grant access to that Apex class?
       yes -> the method runs, the component renders
       no  -> "You do not have access to the Apex class named '<ClassName>'."
```

The rest of the page loads normally, which is why the error often shows up as a banner over an otherwise working record.

Most custom features in this org ship their own **additive permission set** that grants the controller class plus the object and field access the feature needs. For example, `Treatment_Call_Management` grants `TreatmentCallController` and the `Treatment_Call__c` fields, and nothing else. A new user does not get these automatically; someone has to assign them.

## 5. Root cause

There are two different causes behind the same message.

### 5.1 The user was never assigned the permission set (DDT-101, 229, 237, 264, 279, 578, 654, 1048)

The class was granted by a permission set that other users already held. The reporting user did not hold it. Several of these came shortly after the feature shipped:

- The Treatment Call Logger and its permission set were added on 2026-06-01. DDT-101 was reported two days later.
- The Disposition Entry component and its permission set were added on 2026-06-18. DDT-237 was reported on 2026-06-29.

For DDT-1048 the reply on the ticket says it directly: the user "was missing the permission set, which grants access to ReferralSourceController."

For DDT-279 the user said the error was back after an earlier fix. What removed the access is not recorded in the ticket.

### 5.2 The class was called for a new audience (DDT-774)

`LinkRcCallsAction` had existed since May 2026 behind a "Link RC Calls" button. On 2026-08-19, DDT-752 (PR #193) added a headless component, `linkRcCallsAuto`, to the Client page that calls the class **automatically on every page load**. The banner text "Could not link RingCentral calls automatically" comes from that component. DDT-774 was reported the same day.

Before that day the class was granted on only three profiles. As soon as it ran on page load, every other user who opened a Client record hit the error.

### 5.3 The class was never granted to anyone but admins (DDT-1296)

This is the one where the ticket's first guess was wrong. The ticket assumed the reporting user was missing a permission set. She already had it. The real cause, from the DDT-1296 commit:

> PR #669 shipped the How They Found Us panel and its Apex controller but never granted the class to any permission set. In prod only the System Administrator profile could execute it, so every other user saw "You do not have access to the Apex class named 'HowTheyFoundUsController'" instead of the panel.

`How_Heard_Attribution` already granted the panel's other two classes, `ReferralSourceController` and `IntakeHowHeardService`, but not the new one. The resolution comment on the ticket counted the effect: **32 of 36 active holders** of the permission set, across 6 profiles, got the error, not just the one who reported it.

The deploy doc also said users without the permission set would see a locked, read-only line. They saw the error instead, because the panel cannot load at all without the class.

**Evidence query.** This shows every profile and permission set that grants a class. Run it before assuming which permission set to assign:

```sql
SELECT Parent.Name, Parent.Label, Parent.IsOwnedByProfile, Parent.Profile.Name
FROM SetupEntityAccess
WHERE SetupEntityType = 'ApexClass'
  AND SetupEntityId IN (SELECT Id FROM ApexClass WHERE Name = 'HowTheyFoundUsController')
```

Rows with `IsOwnedByProfile = true` are profiles. Rows with `false` are real permission sets. If the only row is the System Administrator profile, no assignment will help: the class is missing from the permission set (5.3).

## 6. Solution

### 6.1 Assign the existing permission set (eight tickets)

Find which permission set grants the class (query in 5.3, or Setup → Apex Classes → the class → Security, which lists profiles only). Then assign it to the user: Setup → Users → the user → Permission Set Assignments → Edit Assignments.

Recorded examples:

- **DDT-654:** assigned `Treatment Call Management`, then asked the user to log out and back in and log a call.
- **DDT-229:** "you were missing a permission set", fixed, and asked the user to fully refresh or log out and in.
- **DDT-1048:** assigned the permission set that grants `ReferralSourceController`, then verified the save worked under the user's own login.

Why a permission set and not the profile: the tickets ask for it (DDT-101: "Apex access should NOT be granted on the Profile directly going forward"; DDT-237 and DDT-578: grant it "via the standard perm set (not a one-off), so future case managers inherit it"). A permission set can be assigned to any user on any profile and is kept in the repo as metadata.

### 6.2 Grant the class on every profile (DDT-774)

Production shows `LinkRcCallsAction` class access added to **36 profiles at one moment on 2026-08-19**, about a minute before the "It's fixed. Please refresh the page" reply on the ticket. Before that, only three profiles had it.

This is the exception to 6.1. Because the component runs on every Client page view, every user who can open a Client needs the class. The component itself handles users who cannot edit call records: per the DDT-752 deploy doc, `autoLinkForRecord` checks whether the user can update `Call_Session__c` and returns `'no-access'` without querying or writing anything. So granting the class widely does not let read-only users change data.

Why profiles were used rather than a permission set is not recorded in the ticket.

### 6.3 Add the class to the permission set (DDT-1296, PR #683)

The change to `How_Heard_Attribution.permissionset-meta.xml`:

```xml
<classAccesses>
    <apexClass>HowTheyFoundUsController</apexClass>
    <enabled>true</enabled>
</classAccesses>
```

Why it was done this way:

- **The set that already grants the panel's other classes.** `How_Heard_Attribution` already covered `ReferralSourceController` and `IntakeHowHeardService` and was assigned to 36 active users, including the reporter. Adding the class there fixed all of them with no new assignments.
- **Compared against production before deploying.** Permission sets deploy by replacement: the file in the deploy becomes the whole permission set. Production's copy matched `main` on all 47 entries before the edit, so deploying the file changed only the one line. After the deploy, a fresh production retrieve showed 40 field permissions and 2 object permissions unchanged and the one class added.
- **The deploy doc was corrected in the same PR**, so it no longer promises a locked line to users without access.

## 7. Testing

**Setup fixes (nine tickets).** Checked by the reporting user after a refresh or log out and in. Confirmations are recorded on DDT-264 ("Looks good!"), DDT-279 ("Looks good now!") and DDT-1048. DDT-1048 was also checked by the Support team under the user's own login before closing. No test is recorded for DDT-101, DDT-229, DDT-237, DDT-578, DDT-654 or DDT-774 beyond the fix being reported done.

**DDT-1296.**

- A check-only deploy against production passed `HowTheyFoundUsControllerTest`, 12 of 12. That proves the class compiles and its tests pass. It does not render the panel as a real user, and the PR said so.
- **Live check:** the reporting user opened a Client record, the panel rendered, she changed the answer and saved, then set it back. Both writes were stamped with her as the person who recorded it.

**Not covered:** there is no automated check that a new Apex controller is granted to a permission set. Apex tests run as a system or test user and pass whether or not real users can call the class. See 11.

## 8. Step by step

1. **Read the class name from the error.** It is the exact API name of the Apex class.
2. **Check whether it fails for everyone or one user.** Ask whether colleagues on the same page see it. If only one user does, it is 5.1. If everyone except admins does, it is 5.3.
3. **Find who grants the class.** Run the `SetupEntityAccess` query in 5.3 with the class name. Note the permission sets (`IsOwnedByProfile = false`).
4. **If a permission set grants it:** check the user's assignments.

    ```sql
    SELECT PermissionSet.Name FROM PermissionSetAssignment
    WHERE AssigneeId = '<user id>'
    ```

    If the set is missing, assign it in Setup → Users → the user → Permission Set Assignments. Production changes need approval first.
5. **If no permission set grants it (only System Administrator):** add the class to the feature's permission set in the repo, retrieve production's copy of that permission set first and confirm it matches, then open a PR, validate against production and deploy (as in 6.3).
6. **If the class is called by a component that runs for every user on page load** (like DDT-774), grant it to everyone who opens that page, and make sure the code itself does nothing harmful for users without edit access.
7. **Ask the user to refresh or log out and back in,** then retry the exact action that failed.
8. **Check the other classes the same component calls.** DDT-237, DDT-1048 and DDT-1296 all asked for this: a component often calls more than one class, and fixing the first error can reveal the next.

## 9. Rollback

- **Permission set assignment:** remove it in Setup → Users → the user → Permission Set Assignments. Safe, but it brings the error back.
- **Profile class access (DDT-774):** remove the class from each profile's Apex Class Access. That brings back the banner on every Client page view for those profiles.
- **PR #683:** redeploy `How_Heard_Attribution` without the class entry. That breaks the panel again for every non-admin, so there is no reason to do it unless the class itself is being retired.

## 10. Known limits

- **New users need their feature permission sets assigned.** The additive permission set pattern means nothing is automatic. DDT-101 asked for an onboarding checklist so new PI case managers get the right set by default; whether it was created is not recorded in the ticket.
- **Some users still lack `HowTheyFoundUsController`.** On 2026-10-10 three active users had neither `How_Heard_Attribution` nor an admin profile, and they still get the error rather than a locked line.
- **The cause of the DDT-279 recurrence is unknown,** so it could happen again.
- **`LinkRcCallsAction` is granted on profiles, not in a permission set,** so it is not captured by a permission set file in the repo. A new profile created later will not have it.

## 11. Reuse for similar issues

**How to recognise one:** the exact wording "You do not have access to the Apex class named '...'", usually on a page that otherwise loads.

- **One user or everyone?** Fails for one user: a missing assignment. Fails for everyone but admins: the class was never granted (DDT-1296). Find out which before doing anything; the second needs a code change, and assigning permission sets will not help.
- **Look up the grant, don't guess it.** `SetupEntityAccess` tells you exactly which permission sets and profiles grant a class.
- **When you ship a new Apex controller, add it to the feature's permission set in the same PR.** Apex tests pass whether or not anyone can call the class, so nothing else catches it. DDT-1296 is what happens when this step is skipped.
- **When a class moves from a button to page load, its audience changes.** DDT-752 turned a button that few people clicked into a call on every Client page view, and DDT-774 followed the same day. Check who can run the class before a change like that goes live.
- **Check the deploy doc's claims about users without access.** DDT-1296's doc promised a read-only line; users actually got an error. If the component cannot load without the class, there is no graceful fallback.
- **Ask the user to refresh or log out and back in** after any access change before deciding it didn't work.
