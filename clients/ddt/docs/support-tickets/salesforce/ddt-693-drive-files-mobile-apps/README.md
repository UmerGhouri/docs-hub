# Drive files not visible in the Salesforce mobile apps

| | |
|---|---|
| **Ticket** | DDT-693 |
| **Project** | Support tickets |
| **Status** | Done 2026-08-17 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-08-10 |
| **Resolved by** | Support team |
| **Systems** | Salesforce mobile app (Android, iPad), Google Drive integration |
| **Area** | Viewing a case's Google Drive files from a phone or tablet |
| **Related** | None recorded |
| **Code** | None. No code change is recorded for this ticket. |

---

## 1. Summary

One user could no longer see Google Drive files in Salesforce on the Android app or the iPad app. An admin changed something in production the same day, and after the user logged out and back in, the files showed again. The ticket does not record which setting was changed.

## 2. What was reported

- **2026-08-10:** a user reported that Google Drive files were no longer visible in the Salesforce mobile app on Android and in the iPad app. Desktop behaviour was not stated in the report.
- **2026-08-10:** after a first admin change, the user checked again and still saw no files. They sent screenshots.
- **2026-08-10 (evening, Central time):** after a second check and a log-out and log-in, the user confirmed it worked.

Only one user is named in the ticket. Whether other mobile users were affected is not recorded.

## 3. Scope

**In scope:** restoring Drive file visibility in the Salesforce mobile apps for the reporting user.

**Out of scope:** not recorded in the ticket.

## 4. How it works (background)

In early August 2026 the firm moved case folders from the CloudFiles package to a native Google Drive integration. The `driveFilePanel` Lightning component shows a record's Drive folder on the Account, Opportunity (Court Case) and Matter record pages (PR #91 and PR #118 in the ddt-salesforce repo). The ticket does not say which of the two (the old CloudFiles widget or the new panel) the user was looking at on mobile.

## 5. Root cause

Not recorded in the ticket. The ticket listed the possible causes to check: Files Connect or Drive authorisation, the mobile app's rendering, or the user's permissions. None of these was confirmed in writing.

## 6. Solution

An admin made a change in production and asked the user to check again. The user then logged out of the app and back in, and the files appeared. **The specific change is not recorded in the ticket.**

## 7. Testing

The reporting user checked on their device and confirmed it worked. No other test is recorded.

## 8. Step by step

The exact fix was not written down. For the next report like this, these are the steps the ticket asked for:

1. Reproduce on the user's Android or iPad app.
2. Check whether desktop Salesforce shows the same Drive files for the same user. This tells you if the problem is mobile-only or affects the whole integration.
3. Check the user's permission set assignments for Drive or Files Connect access.
4. Check whether other users are affected.
5. After any change, have the user **log out of the mobile app and back in**. In this ticket the change only showed after a fresh log-in.

## 9. Rollback

Not applicable: the change made is not recorded.

## 10. Known limits

Because the root cause is not recorded, it cannot be ruled out that the same problem affects other users or comes back.

## 11. Reuse for similar issues

- **Ask the user to log out and back in** after any permission or page change before you decide a fix didn't work. Here the first check failed and the second, after a fresh log-in, worked.
- **Split mobile-only from integration-wide** early by comparing desktop and mobile for the same user.
- **Write down the setting you changed** in the ticket, so the next person can repeat it.
