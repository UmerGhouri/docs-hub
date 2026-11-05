# Setting or fixing a user's outbound caller ID

| | |
|---|---|
| **Ticket** | DDT-219, DDT-238, DDT-495, DDT-566, DDT-577, DDT-619, DDT-634 |
| **Project** | Support tickets |
| **Status** | Done (DDT-219 2026-06-29, DDT-238 2026-06-30, DDT-495 2026-07-29, DDT-566 2026-07-29, DDT-577 2026-07-30, DDT-634 2026-08-04, DDT-619 2026-08-21) |
| **Priority** | High (DDT-238, DDT-495, DDT-566); Medium (DDT-219, DDT-577, DDT-619, DDT-634) |
| **Reported by** | Attorney and Firm staff, 2026-06-22 to 2026-08-04 |
| **Resolved by** | Support team |
| **Systems** | RingCentral (admin settings, RingCentral app) |
| **Area** | Phones: which number clients see when staff call out |
| **Related** | None outside this group |
| **Code** | Not applicable: every change was made in RingCentral's admin settings |

---

## 1. Summary

Staff kept calling out with the wrong caller ID: a general firm number instead of the Personal Injury (PI) mainline, a personal extension instead of the PI line, or a number still labelled with its previous owner's name. In RingCentral, a user's caller ID is not one setting. It is set separately **per device** (such as the RingCentral app) and **per feature** (RingOut, Call Flip, Delegated calls and others), and a user can only choose from the numbers an administrator has made available to them. Each ticket was fixed by an administrator correcting those settings in RingCentral. Two users reported the wrong number coming back after a fix; the tickets do not record what caused that.

## 2. What was reported

- **DDT-219 (2026-06-22).** An attorney's direct number appeared on his app's dial pad, but searching RingCentral admin for his name did not find it. The number was assigned to his extension, but its **label** still carried the previous owner's name.
- **DDT-238 (2026-06-29).** The same attorney said his caller ID was still not updated, and that signing in to RingCentral showed `Error: 404 Page not found`.
- **DDT-495 (2026-07-23).** A user could no longer pick the PI line or a second firm line as his caller ID. His picker showed only his direct number, a blocked number, the mainline and the company fax. The same day a second user reported the same symptom: she should call from the PI mainline, the app showed a different number, and the "My caller ID" dropdown had no option to change it.
- **DDT-566 (2026-07-28).** A client-facing user's app showed the general number instead of the PI mainline. He had noticed it at the end of his shift, so every call that day went out with the wrong number. Another staff member noted it was the same issue the two DDT-495 users had the week before.
- **DDT-577 (2026-07-29).** A user was calling out with her personal extension and asked for the PI mainline instead.
- **DDT-619 (2026-08-03).** The DDT-566 user's caller ID had gone back to the general number. The same day, a second user (one of the DDT-495 users) said hers had "switched again".
- **DDT-634 (2026-08-04).** The DDT-577 user's app showed the general number again; she said it had "switched again".

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Phone number label | RingCentral admin setting | Label changed from the previous owner's name to the current owner's (DDT-219) |
| Caller ID by device (RingCentral app) | RingCentral admin setting, per extension | Set to the PI mainline where it was not |
| Caller ID by feature (RingOut, RingMe, Call Flip, Fax number, Mobile app, Additional desktop app, Common phone, Delegated) | RingCentral admin setting, per extension | Set to the PI mainline (DDT-566, DDT-619) |
| Numbers available in the user's caller ID picker | RingCentral admin setting | PI mainline made selectable (DDT-495, DDT-577) |

**Out of scope**

- **Salesforce.** Nothing in Salesforce changed. DDT-619 found that the DDT-566 user's calls placed from Salesforce were already going out as the PI mainline.
- **The second firm line in DDT-495.** After the fix the user could call from the PI mainline "but still not" the second line. The tickets do not record whether that was resolved.
- **Why the setting reverts.** Not identified (see 5 and 10).

## 4. How it works (background)

```
User places an outbound call
  -> how the call starts decides which caller ID setting applies:
       RingCentral app dialer            -> "by device" entry for the RingCentral app
       RingOut, RingMe, Call Flip,
       Delegated, softphone, fax ...     -> the matching "by feature" entry
       (Alternate: an optional extra entry, often left "Not specified")
  -> the number shown to the person called is the one in that entry
```

Two more things matter:

- **A user can only choose a number an administrator has made available.** If the PI mainline is not in a user's list, the picker cannot show it, and in several of these tickets the picker was disabled ("greyed out") for the user.
- **A number has a label** in RingCentral admin, separate from the person it is assigned to. Admin search uses that label.

## 5. Root cause

### DDT-219 and DDT-238: stale label and the wrong sign-in page

- **DDT-219:** the number was correctly assigned as the attorney's primary direct number, but its label still named the previous owner, so a search by his name could not find it.
- **DDT-238:** the page he was signing in to was the RingCentral **admin portal**. His account is a standard user, so that page may not load for him. The support team pointed him at the RingCentral app's own web sign-in instead. He replied that "both options 1 and 2 do not work"; what happened after that is not recorded in the ticket.

### DDT-495 and DDT-577: the PI mainline was not available to the user

The users' caller ID lists did not include the PI mainline (DDT-495: "No number resembling a PI ... line is currently assigned"). An operations lead reported "we have identified the issue" and an administrator fixed it; the exact setting changed is not recorded in the ticket.

### DDT-566, DDT-619 and DDT-634: feature-level entries pointing at the general number

The ticket's first guess in DDT-566 was that the app header "picks up one of the feature-level defaults". DDT-619 checked the extension's caller ID settings and found:

| Entry | Value |
|---|---|
| By device: RingCentral app | PI mainline (correct) |
| By feature: Fax number, Call Flip, RingOut, RingMe, Additional softphone, Common phone, Delegated | General number (wrong) |
| Mobile app, Alternate | Empty |

The app device was right and every feature entry was wrong, so the number a client saw depended on how the call was started.

**What this does not explain.** The second DDT-619 user's settings were **already correct everywhere**: every device and feature on the PI mainline. She still saw the general number, and her picker was disabled. These were ruled out:

- **Queue membership.** Both users are Primary Agents on the PI Department queue that owns the PI mainline.
- **A queue-level caller ID setting.** RingCentral has none.
- **Caller ID restrictions on her extension.** All five permissions were on.

**Why it kept reverting is not recorded.** DDT-619 set out the test: if the fix held the next day, DDT-566 had simply never been fully applied; if it reverted, something was resetting it. The result of that re-check is not recorded, and DDT-634 records only "Resolved."

## 6. Solution

### DDT-219: rename the number's label

The label on the attorney's direct number was changed from the previous owner's name to an abbreviated form of his own name. **RingCentral limits a label to 15 characters**, so a full name may not fit. Searching admin by his first name, surname or the number then returned the assignment.

### DDT-238: use the right sign-in page

A standard user signs in through the RingCentral app's own sign-in, not the admin portal. The ticket treated the caller ID request as the earlier DDT-219 request still being open; what was changed for it this time is not recorded in the ticket.

### DDT-495, DDT-566, DDT-577 and DDT-634: make the PI mainline available and selected

An administrator made the PI mainline available and set it as the user's caller ID. The user then **reloaded the app, or signed out and back in** (DDT-577: "Yes please restart"). DDT-566's user confirmed "all good now" after reloading.

### DDT-619: align every entry, not just the app

On the reverted user's extension, **eight** caller ID entries were moved from the general number to the PI mainline: RingOut, RingMe, Call Flip, Fax number, Mobile app, Additional desktop app, Common phone and Delegated. The RingCentral app entry was already correct. Only Alternate was left "Not specified", matching the second user's configuration.

Why all eight: setting only the app's entry (as in DDT-566's first pass) leaves every other way of starting a call on the old number. Aligning all of them removes the dependence on how the user dials.

For the second user, whose admin settings were already correct, the next steps were to sign out and back in, and to read the number under **Settings → Phone → Default Caller ID** in the app (the settings gear, not the dialer). If she still saw the general number after a fresh sign-in with correct settings, the ticket's plan was to open a **RingCentral support case**. Whether that was needed is not recorded.

## 7. Testing

The check in every ticket was the user's own view of the app, or a test call:

| Ticket | Check | Result |
|---|---|---|
| DDT-219 | Admin search by name and by number | Both returned the assignment. The attorney's test call is not recorded. |
| DDT-495 | User dials out from the PI mainline | Worked; the second firm line still did not. The second user said "still the same" after signing out and in. |
| DDT-566 | User reloads and checks the picker | "All good now." |
| DDT-577 | User restarts the app | Not recorded in the ticket. |
| DDT-619 | Re-check both extensions the next day | Not recorded in the ticket. |
| DDT-634 | Not recorded in the ticket | "Resolved." |

**Not covered:** a test call answered on a separate phone to confirm the number the recipient actually sees, for any of these tickets.

## 8. Step by step

1. **Find out which number the user should show.** For PI staff it has been the PI mainline. Confirm with the user or their lead if unsure.
2. **Open the user's extension in RingCentral admin** and go to its caller ID settings.
3. **Check the "by device" entry** for the RingCentral app.
4. **Check every "by feature" entry**: RingOut, RingMe, Call Flip, Fax number, Mobile app, Additional desktop app, Common phone, Delegated. Set each one to the correct number. Leave Alternate as it is unless asked.
5. **If the number is not offered in the list**, make it available to the user first (DDT-495, DDT-577).
6. **If the problem is search or display of the number itself**, check the number's **label** in admin and update it (15 characters at most).
7. **Ask the user to reload the app, or sign out and back in.** Then have them check **Settings → Phone → Default Caller ID** in the app, not only the dialer header.
8. **Ask for a test call** to a phone they can see, and record the result on the ticket.
9. **Re-check the extension a day later** if the user says it has "switched again". If the admin settings are still right and the app still shows the wrong number, open a RingCentral support case.
10. **If the user cannot sign in at all**, confirm they are using the RingCentral app sign-in and not the admin portal.

## 9. Rollback

Each change is a single setting in RingCentral admin and can be changed back the same way. There is no reason to undo it: the old values were the wrong caller IDs. **Record the old values on the ticket before changing them**; DDT-619 did this with before and after screenshots.

## 10. Known limits

- **The revert is unexplained.** At least two users had their caller ID go back to the general number after a fix (DDT-619, DDT-634). DDT-619 also reported a user whose settings were correct but whose app still showed the wrong number. The cause, whether a template, a reset, or the app, is not recorded.
- **Partial fixes look complete.** Setting only the app entry leaves RingOut, Call Flip, Delegated and the other paths on the old number. Calls from Salesforce can look right while those paths are wrong.
- **The second firm line in DDT-495** is not recorded as fixed.
- **Labels are capped at 15 characters**, so a long name has to be shortened.

## 11. Reuse for similar issues

**How to recognise one:** a user says "my caller ID switched", "it shows a different number", or "I can't pick the PI line", or the caller ID dropdown is greyed out.

- **Check device and feature entries together.** One correct entry proves nothing about the others.
- **Ask how the user placed the call** (app dialer, RingOut, Salesforce, delegated). That tells you which entry applies.
- **"Not in the list" means admin, not the user.** A user can only pick numbers made available to them.
- **When search by name fails, check the number's label**, which can still carry a former owner's name.
- **Correct settings, wrong display:** sign out and in, read Settings → Phone → Default Caller ID, then escalate to RingCentral support.
- **When it says "again", re-check the next day** and record whether it held, so the cause can be found next time.
