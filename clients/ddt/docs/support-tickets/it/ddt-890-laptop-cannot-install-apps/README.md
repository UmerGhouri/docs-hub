# Laptop cannot install apps

| | |
|---|---|
| **Ticket** | DDT-890 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-16 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-09-01 |
| **Resolved by** | Support team |
| **Systems** | Windows laptop (operating-system install permissions) |
| **Area** | IT: installing software on a staff laptop |
| **Related** | None recorded. The ticket mentions an earlier desktop-app crash report from the same user as possibly related. |
| **Code** | Not applicable: no code change |

---

## 1. Summary

A staff member could not download or install any apps on her laptop. Every install stopped with an "administrator permission required" prompt, because her account does not have administrator rights on the machine. Administrator permission was granted to her for five hours so she could install what she needed.

## 2. What was reported

On 2026-09-01 a staff member reported through Slack that she was unable to download any apps on her laptop, which blocked her from installing software she needed for work. One person and one laptop were affected. When asked, she confirmed the message she saw: **"administrator permission required"**.

## 3. Scope

**In scope:** the user's install permissions on her own laptop, granted temporarily.

**Out of scope:** a permanent change to her account's rights, and any change to the firm's wider install policy. The ticket asked for "a documented workflow for requesting installs" as an alternative; no such workflow is recorded in the ticket.

## 4. How it works (background)

```
User starts an installer
  → the operating system checks whether the account may install software
  → standard account: the install stops with "administrator permission required"
  → account with administrator permission: the install continues
```

Staff accounts do not normally hold administrator rights on their laptops, so installing new software needs an administrator to allow it.

## 5. Root cause

The prompt the user saw names the cause directly: the install needed administrator permission and her account did not have it. The ticket listed other possible causes to check (group policy, antivirus, device management, a browser download block). None of those was needed once the exact message was known.

## 6. Solution

Administrator permission was applied to the user's account **for five hours**, with a note that it might take a few minutes to take effect on her laptop. This let her install the apps she needed within that window without leaving her account permanently elevated.

Why temporary: the request was to install specific software, not to change her standing access. A time-limited grant covers the need and lapses on its own.

## 7. Testing

The user was told to retry her installs once the permission came through. A follow-up confirmation from the user is not recorded in the ticket; the ticket was closed on 2026-09-16.

**Not covered:** whether the earlier desktop-app crash mentioned in the ticket was related. Not recorded in the ticket.

## 8. Step by step

1. Ask the user for the exact message the installer shows. "Administrator permission required" points straight at account rights.
2. Have an administrator grant the user's account administrator permission for a limited time (five hours was used here).
3. Tell the user it can take a few minutes to apply on their laptop.
4. Ask the user to retry the install and confirm it worked.
5. Let the permission expire on its own.

## 9. Rollback

Nothing to undo: the permission was time-limited and expires by itself. An administrator can also remove it early.

## 10. Known limits

- The grant is temporary. The next time the user needs to install something, the same prompt will appear and the same request is needed.
- Any software installed during the window stays on the laptop after the permission expires.

## 11. Reuse for similar issues

**How to recognise one:** a user says they "can't install anything" and the installer shows an administrator or elevation prompt.

- Get the exact on-screen message first. It usually names the cause and saves checking policy, antivirus and device management one by one.
- Prefer a time-limited administrator grant over a permanent one.
- If the same user keeps needing installs, raise a standing install-request process rather than repeating one-off grants.
