# Linking a user to the firm Google account for Drive

| | |
|---|---|
| **Ticket** | DDT-1196 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-06 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-10-02 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex), Google Drive |
| **Area** | Files tab on the Matter: the Drive panel that shows a case's Google Drive folder |
| **Related** | DDT-1121 (Drive panel identity and delete mode) |
| **Code** | PR #635 in the `ddt-salesforce` repo |

---

## 1. Summary

A new member of staff got "Your Salesforce user is not linked to a firm Google account" on the Files tab of every Matter. The ticket asked for her Salesforce email to be corrected, but it was already right: her firm Google account is on a **second email domain the firm uses inside the same Google Workspace**. Salesforce's own rule for "which Google person is this user" only accepted the firm's main domain. The fix adds the second domain to that rule.

## 2. What was reported

- **2026-10-02:** a new staff member reported the error "Documents could not be loaded — Your Salesforce user is not linked to a firm Google account, so Google Drive cannot be opened as you" on the Files tab of a client's Matter. Any Matter gave the same error, since it is a user-level problem. It blocked her from working the matter.
- **2026-10-04:** an administrator added her to the shared drive in Google Drive as a Content Contributor.
- **2026-10-05:** she reported the same error was still showing.
- **2026-10-06:** after the fix was deployed she confirmed it was working.

One user was affected. The fix also covers anyone added later on the same second domain.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `GoogleDriveService` | Apex class | `personSubject()` accepts a set of firm Workspace domains instead of a single one |
| `GoogleDriveServiceTest` | Apex test | New test `personSubjectAcceptsTheFirmsSecondWorkspaceDomain` |
| `Drive_Delete_Mode` | Permission set (assignment only) | Assigned to the user after deploy, as every other case manager has it (DDT-1121) |

**Out of scope**

- **Changing her Salesforce email.** It already matched her firm Google account. There was no address on the main domain to change it to.
- **`EmailInboxController`'s own firm-domain constant.** It still treats only the main domain as "the firm", so an email this user sends or is copied on is threaded as if she were an outside party. Raised separately.
- **Moving the domain list into custom metadata.** Kept as a code constant on purpose (see 6).

## 4. How it works (background)

```
User opens the Files tab on a Matter
  -> the Drive panel (DriveFolderController) asks GoogleDriveService.personSubject(User.Email)
       "which firm Google person is this Salesforce user?"
  -> personSubject lowercases the email, strips the sandbox suffix,
     and returns it ONLY if it ends with a firm Workspace domain
  -> no match: the panel shows "not linked to a firm Google account"
  -> match: the panel opens Drive as that person
```

Since DDT-1121, `personSubject()` is the single rule for this. It is used by the Drive panel (`DriveFolderController`) and by the folder actions (`DriveFolderActions`).

## 5. Root cause

The ticket's stated cause (her Salesforce email does not match her Google account) was **wrong**. Checked against production and Google on 2026-10-06:

| Check | Result |
|---|---|
| Her Salesforce email | Her real firm Google address, on the second domain |
| Is the second domain on Google Workspace? | Yes |
| Same Workspace as the main domain? | Yes |
| Can she see the firm's shared drives as herself in Google? | Yes (after the 2026-10-04 Content Contributor change) |
| Does she have an address on the main domain? | No, Google says it does not exist |
| Salesforce users on the second domain | One, her |

What blocked her was Salesforce's own rule. `personSubject()` compared the email against one constant, the main domain, so any address on the second domain returned `null`, which the panel reports as "not linked".

**The lesson.** The error message pointed at the user record, so the ticket did too. The Google side was already fine; the gap was in our code. Check the ticket's diagnosis against the data before building its fix.

## 6. Solution

`personSubject()` now checks a set of firm Workspace domains. Domains are shown as placeholders here.

```apex
private static final Set<String> WORKSPACE_DOMAINS = new Set<String>{
    '@<main-domain>',
    '@<second-domain>'
};

public static String personSubject(String userEmail) {
    if (String.isBlank(userEmail)) {
        return null;
    }
    String normalised = userEmail.toLowerCase().removeEnd(SANDBOX_EMAIL_SUFFIX);
    for (String domain : WORKSPACE_DOMAINS) {
        if (normalised.endsWith(domain)) {
            return normalised;
        }
    }
    return null;
}
```

Why it's done this way:

- **Every domain is anchored on `@`.** A bare suffix check would also accept a look-alike such as `someone@fake<main-domain>`. The `@` makes the match exact.
- **One rule, one change.** Because every Drive feature goes through `personSubject()`, adding the domain here fixed both the panel and the folder actions.
- **A code constant, not a custom metadata field.** A custom metadata deploy writes the whole record, and the existing Drive configuration record has already nearly switched provisioning off once that way. One new domain was not worth that risk.
- **A domain belongs in the list only if it is in the same Workspace.** That was proven for this domain before adding it. A domain from a different Google organisation would simply fail at Google, so adding one buys an error, not access.

## 7. Testing

**Apex.** `GoogleDriveServiceTest.personSubjectAcceptsTheFirmsSecondWorkspaceDomain` checks that:

- an address on the second domain is accepted and lowercased;
- the sandbox suffix is stripped for the second domain too;
- three look-alikes are refused: a domain with a prefix in front of it, the firm domain in the middle of a longer domain, and a near-miss spelling.

The existing `personSubjectNamesFirmStaffAndNobodyElse` still holds for the main domain.

Validated against production with `GoogleDriveServiceTest`, `DriveFolderControllerTest` and `DriveFolderActionsTest`.

**Live check.** The user confirmed on 2026-10-06 that the Drive files load.

**Not covered:** whether the test fails with the change removed is not recorded.

## 8. Step by step

1. **Read the user's email** on their Salesforce user record and note the domain.
2. **Confirm it is the right Google account.** The user signs in to Google with that address and can see the firm's shared drives. If they cannot see the shared drive, a Drive administrator adds them to it.
3. **If the domain is not the main domain,** confirm it belongs to the firm's Google Workspace before going further.
4. **Add the domain to `WORKSPACE_DOMAINS`** in `GoogleDriveService`, anchored on `@`.
5. **Add a test** for the new domain and for look-alikes of it.
6. **Validate against production** with `GoogleDriveServiceTest`, `DriveFolderControllerTest` and `DriveFolderActionsTest`, get the PR approved, then deploy.
7. **Assign `Drive_Delete_Mode`** to the user if their role should have it, as other case managers do.
8. **Ask the user to open a Matter's Files tab** and confirm the files load.

## 9. Rollback

Redeploy the previous `GoogleDriveService`. Users on the second domain lose the Drive panel again; nobody else is affected.

## 10. Known limits

- **Each new email domain needs a code change.** A user on a third domain will see the same error until it is added.
- **Email threading still treats the second domain as external** (`EmailInboxController`), as noted in 3.
- **The error message still says to check the email on the user record.** In this case the email was correct, so the message can mislead the next person too.

## 11. Reuse for similar issues

**How to recognise one:** a single user gets "not linked to a firm Google account" while everyone else's Drive panel works.

- **Check the domain of the user's email first.** If it is not the main domain, this page is the fix.
- **If the domain is the main domain,** the email probably does not match a real Google account. Correct the email on the user record.
- **If the user cannot see the shared drive in Google itself,** that is a Drive sharing problem, not a Salesforce one.
- **Allow-lists of domains must anchor on `@`.** Test the look-alikes every time you add one.
