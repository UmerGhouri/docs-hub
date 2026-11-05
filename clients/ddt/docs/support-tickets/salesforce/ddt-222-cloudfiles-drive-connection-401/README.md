# CloudFiles Google Drive connection fails with Unauthorized 401

| | |
|---|---|
| **Ticket** | DDT-222 |
| **Project** | Support tickets |
| **Status** | Done 2026-06-24 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-06-23 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (CloudFiles managed package), Google Drive, CloudFiles mobile/tablet app |
| **Area** | Case files: the CloudFiles Files widget on the case record page |
| **Related** | DDT-225 (CloudFiles flow that moves files to Drive) |
| **Code** | Not applicable: no metadata changed; the fix was a licence assignment and an app restart |

---

## 1. Summary

A staff member who had just been given a CloudFiles licence could not connect his Google Drive account. The connect step ended on CloudFiles' own server with `Invalid state!` / `Unauthorized` / `401`, so the Files widget could not show client documents on case records. The user closed and relaunched the app on his iPad and the connection then worked; the support team recorded that the licence was now in place and closed the ticket.

## 2. What was reported

- **DDT-222 (2026-06-23).** A staff member followed the in-app prompt to connect Google Drive to CloudFiles. The browser landed on CloudFiles' API server showing:

```json
{"message":"Invalid state!","error":"Unauthorized","statusCode":401}
```

He could not open client files from the Files widget on a case record, which blocked his day-to-day case work. One user was affected. His CloudFiles licence had been assigned earlier the same day, in response to a separate request.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| CloudFiles licence for the affected user | Managed package licence | Assigned (before and around the ticket) |
| The user's CloudFiles app session on his iPad | Client app | Closed and relaunched by the user |

**Out of scope**

- **Remote site settings, CSP trusted sites, Google Workspace SSO or consent settings.** The ticket listed them as things to check; nothing in the ticket says any were changed, and the connection worked without changing them.
- **CloudFiles support.** Not contacted; the issue cleared before escalation was needed.

## 4. How it works (background)

```
User opens a case record
  -> CloudFiles Files widget loads
  -> if the user has no linked Drive account, the widget prompts "connect Google Drive"
  -> browser goes to Google for consent, then back to CloudFiles' server with a "state" value
  -> CloudFiles checks that the returned state matches the one it issued for this user's session
       match    -> Drive account linked, widget lists the case folder's files
       no match -> "Invalid state!" 401, nothing linked
```

The user needs a CloudFiles licence before the connection can complete.

## 5. Root cause

Not fully recorded in the ticket. The two sources give two accounts:

- **The user (relayed from Slack, 2026-06-23):** he was using the app on an iPad; he closed and relaunched it, and it then worked.
- **The support team (2026-06-24):** "The license was provided now it's fixed."

The `Invalid state!` message itself means CloudFiles could not match the returned connect request to a session it had started. A connect attempt begun in an app session that was opened before the licence took effect, then completed after it, would fit both accounts, but the ticket does not confirm that.

**The ticket's first guesses** (stale OAuth state, browser cache, incomplete provisioning after licence assignment, SSO consent, remote site or CSP settings) were not individually tested; the problem cleared before any of them was examined.

## 6. Solution

1. Confirm the user holds a CloudFiles licence (it had been assigned earlier that day).
2. Have the user fully close the CloudFiles app (or browser tab) and start the Google Drive connection again in a fresh session.

No Salesforce configuration was changed.

## 7. Testing

- **Live check:** the user confirmed in Slack that, after relaunching the app, "it now appears to be working".
- **Not recorded:** whether the Files widget was checked on a specific case record afterwards, as the acceptance criteria asked.
- **Not covered:** nothing automated; this is a vendor OAuth flow.

## 8. Step by step

1. **Check the licence.** In Setup, open Installed Packages, find CloudFiles, and use Manage Licenses to confirm the user is listed.
2. **If the licence was only just assigned,** ask the user to close the app or browser tab completely, not just go back.
3. **Start the connection again** from the Files widget's connect prompt, in a fresh session (on desktop, a private/incognito window also gives a fresh session).
4. **Complete the Google consent screen** with the user's firm Google account.
5. **Open a case record** and confirm the Files widget lists the case folder's documents.
6. **If the 401 persists after a fresh session,** escalate to CloudFiles support with the time of the failed attempt.

## 9. Rollback

Not applicable: nothing was changed that would need undoing. Removing the licence would disconnect the user from the widget again.

## 10. Known limits

- **The root cause was not confirmed**, so the same error could have a different cause next time.
- **CloudFiles is being replaced.** The repo's own notes say CloudFiles is being replaced by the firm's own Google Drive integration; once that is complete, this connect flow no longer applies.

## 11. Reuse for similar issues

**How to recognise one:** an OAuth connect step for a managed package ends on the vendor's server with `Invalid state`, `state mismatch` or `401` straight after a licence or permission was granted.

- **Licence first.** A user who was just licensed often needs a completely new session before the vendor sees the licence.
- **Restart the session before debugging configuration.** A state mismatch is almost always a session that started in one context and finished in another (a different tab, an app that was backgrounded, a session older than the licence).
- **Ask which device.** Tablet and mobile apps hold sessions longer than browser tabs; "close the app and reopen it" is the tablet equivalent of an incognito window.
- **Record which fix worked.** This ticket closed with two different explanations; writing down which step actually cleared the error makes the next one faster.
