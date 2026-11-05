# Google Drive blocked with an unusual traffic prompt

| | |
|---|---|
| **Ticket** | DDT-1025 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-16 |
| **Priority** | High |
| **Reported by** | Case Manager, 2026-09-16 |
| **Resolved by** | Support team |
| **Systems** | Google Workspace (Google Drive) |
| **Area** | Staff access to case documents in Drive |
| **Related** | None recorded |
| **Code** | Not applicable: no change made |

---

## 1. Summary

A staff member could not open folders or upload files in Google Drive, and Google showed an "unusual traffic on your account" prompt asking for a verification code. Before IT could look at it, the problem cleared on its own. No change was made, and the staff member was asked to send a screen recording if it happens again.

## 2. What was reported

- **2026-09-16:** a case manager could not click into Drive folders or upload files. Google also showed an "unusual traffic" prompt asking for a verification code. They had restarted their machine with no change and were not on a VPN. Suggested steps (hard refresh, private window, completing the check) may not all have been tried.
- **Later the same day:** asked for a screenshot, the staff member said it seemed to be working now, but that earlier it would not let them add anything.

One user was affected.

## 3. Scope

**In scope:** one user's Google Drive access in the browser.

**Out of scope:** account or domain-level Google Workspace settings. Nothing was changed.

## 4. How it works (background)

Google shows an "unusual traffic" check when it sees requests from a browser or network that look automated. Until the check is passed, Google may block actions such as opening folders and uploading. It is usually tied to the browser session or network, not to the account's permissions.

## 5. Root cause

Not recorded in the ticket. The problem stopped before IT investigated, so it was not established whether it was a session, browser or network issue.

## 6. Solution

No change. The issue resolved itself. IT asked the staff member to follow up, with a screen recording, if it happens again.

## 7. Testing

The staff member confirmed Drive was working again. Nothing else was tested.

## 8. Step by step

There was no fix to repeat. The steps suggested on the ticket, if this comes back:

1. Ask for a screenshot or recording of exactly what fails.
2. Confirm whether the user is on a VPN (here they were not) and whether a restart changes anything (here it did not).
3. Try a hard refresh, then a private (incognito) window.
4. Complete Google's verification check if it is offered.
5. If it still fails, investigate whether it is an account-level lockout, a verification issue, or a browser or session issue, as the ticket's requirements list.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- The cause is unknown, so it may recur.

## 11. Reuse for similar issues

**How to recognise one:** Google blocks normal actions and shows an "unusual traffic" or verification prompt.

- Ask for a screenshot or recording straight away. These problems often clear before anyone can look.
- Separate browser or session, network and account causes by changing one at a time.
- Do not change account settings for a problem that has cleared on its own.
