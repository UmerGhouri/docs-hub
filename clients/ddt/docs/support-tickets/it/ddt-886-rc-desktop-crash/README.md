# RingCentral desktop app crashes on launch

| | |
|---|---|
| **Ticket** | DDT-886 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-16 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-09-01 |
| **Resolved by** | Support team |
| **Systems** | RingCentral desktop app (Windows) |
| **Area** | IT: staff phone software |
| **Related** | None |
| **Code** | Not applicable: no code or configuration change |

---

## 1. Summary

A staff member's RingCentral desktop app vanished mid-day and threw a JavaScript error on relaunch saying a `package.json` file was missing from its install folder. Shortly after reporting it, the staff member said the app was back on its own. No fix was applied.

## 2. What was reported

The app had worked after a morning restart, then disappeared. Reopening it showed:

```
Uncaught Exception:
Error: ENOENT: no such file or directory, open
'...\AppData\Local\Programs\RingCentral\resources\app.asar.unp...\package.json'
   at Object.readFileSync (node:fs:448:20)
```

The staff member could not make or take calls from the desktop app. One user was affected.

## 3. Scope

**In scope:** getting the one user's desktop app working.

**Out of scope:** any wider disk or antivirus investigation. Not recorded as carried out.

## 4. How it works (background)

The RingCentral desktop app is an Electron application installed per user under `%LOCALAPPDATA%\Programs\RingCentral`. On launch it reads files from `resources\app.asar.unpacked`. An `ENOENT` error there means a file the app expects is missing at the moment it starts.

## 5. Root cause

Not recorded in the ticket. The error shows a file missing from the install folder at launch; the ticket's working theory was a corrupted install, possibly from an antivirus quarantine, but neither was confirmed. The staff member reported the app was back on its own shortly afterwards, so the missing file was evidently a temporary state rather than lasting damage.

## 6. Solution

None was needed. The app recovered on its own and the ticket was closed on 2026-09-16.

The fix the ticket prepared, in case it recurred:

1. Uninstall RingCentral.
2. Delete the leftover folders `%LOCALAPPDATA%\Programs\RingCentral` and `%APPDATA%\RingCentral`.
3. Reinstall the latest desktop client from RingCentral's download page.
4. Check Windows Defender protection history for a quarantined RingCentral file.

## 7. Testing

Not applicable: no change was made. The staff member confirmed the app was working again.

## 8. Step by step

If the same error appears:

1. **Wait a few minutes and relaunch once.** In this case the app came back on its own.
2. **If it still fails,** quit RingCentral fully (including the system tray icon).
3. **Uninstall** it from Windows Settings → Apps.
4. **Delete** `%LOCALAPPDATA%\Programs\RingCentral` and `%APPDATA%\RingCentral`.
5. **Reinstall** the latest desktop client and sign in.
6. **Check Windows Security → Protection history** for any RingCentral file that was quarantined; restore or allow it if so.
7. **Check other apps** on the machine for similar errors, to rule out a disk problem.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- The cause was never confirmed, so it may recur.
- A reinstall clears local app settings; the user may need to sign in and set preferences again.

## 11. Reuse for similar issues

**How to recognise one:** an Electron desktop app (RingCentral, Slack, Teams and similar) fails at launch with `ENOENT` on a file inside its own install folder.

- **Relaunch once before acting;** in this case the problem cleared without any fix.
- **A clean reinstall,** including the per-user AppData folders, is the standard fix for a corrupted install.
- **Check antivirus history** when a file inside an app's own folder disappears.
