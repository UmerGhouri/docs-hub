# Inbound faxes landing on the wrong extension

| | |
|---|---|
| **Ticket** | DDT-1464 |
| **Project** | Support tickets |
| **Status** | Done 2026-10-28 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-10-27 |
| **Resolved by** | Support team |
| **Systems** | RingCentral (fax, fax-to-email notifications) |
| **Area** | PI team: receiving inbound faxes such as medical records and liens |
| **Related** | None recorded |
| **Code** | Not applicable: RingCentral admin setting, no code |

---

## 1. Summary

The PI team stopped receiving faxes sent to the PI fax number. RingCentral showed the faxes arriving, but since about 12 October they were being delivered to the "PI: Recording Announcement" extension instead of the "PI Department" extension, and that extension emailed its faxes to one staff member only. The fix added the PI team inbox as a fax notification recipient on that extension, so every new fax now emails the PI inbox with the PDF.

## 2. What was reported

On 2026-10-27 a PI staff member reported that faxes to the PI fax number were no longer reaching the PI email inbox and were not visible under Received in the RingCentral app. They work on liens for settlements, which mostly arrive by fax.

RingCentral's fax log showed:

- faxes delivered to the **PI Department** extension up to 8 October 2026;
- from 12 October 2026, faxes delivered to the **PI: Recording Announcement** extension instead;
- at least six complete legitimate-looking faxes from medical providers on the wrong extension between 14 and 23 October, plus three partial receives;
- outbound faxes from the number working normally.

The reporter also thought faxes might be going to other departments. That was not borne out: the log showed only the PI: Recording Announcement extension as recipient.

## 3. Scope

**In scope:** getting new inbound faxes to the PI team inbox.

**Out of scope:**

- **Moving the fax line back to the PI Department extension.** The fix works with the extension the faxes now land on.
- **Faxes received 12 to 27 October.** They had been emailed to the one staff member on that extension's notification list; the PI team was to get any they needed from that person.

## 4. How it works (background)

```
Fax sent to the PI fax number
  -> RingCentral delivers it to the extension that owns the fax line
  -> that extension's fax notification settings email the fax (as a PDF)
     to each recipient on its list
```

A fax is only visible in the RingCentral app to someone signed in as that extension, and only emailed to the addresses on that extension's notification list.

## 5. Root cause

Since about 12 October the faxes were delivered to the PI: Recording Announcement extension, and that extension's fax email notifications went to a single staff member, not to the PI team inbox. So the PI inbox received nothing, and nobody on the PI team was signed in as that extension to see them in the app.

Why delivery moved from the PI Department extension to the PI: Recording Announcement extension around that date is not recorded in the ticket.

## 6. Solution

In the RingCentral admin portal, on the **PI: Recording Announcement** extension, the PI team inbox was added as a **fax notification recipient**. Every new inbound fax to the PI number now emails the PI team inbox with the fax attached as a PDF.

## 7. Testing

**Live check:** confirmed on the next real inbound fax, on 2026-10-27, which reached the PI team inbox.

**Not covered:** viewing received faxes in the RingCentral app, which still needs a sign-in as that extension.

## 8. Step by step

1. **Open the fax log** for the number in RingCentral and find which extension each recent inbound fax was delivered to, and from what date.
2. **Compare with the last fax that reached the team**, to date the change.
3. **Open that extension's notification settings** for fax and see who is on the email list.
4. **Add the team's shared inbox** as a fax notification recipient, with the attachment included.
5. **Wait for the next live fax** and confirm it arrives in the team inbox.
6. **Tell the team where earlier faxes went**, so they can recover any they need from the person who received them.

## 9. Rollback

Remove the team inbox from the extension's fax notification recipients. Faxes go back to emailing only the existing recipient.

## 10. Known limits

- **The fax line still delivers to the PI: Recording Announcement extension.** If the line is moved back to the PI Department extension, the notification setting must be checked there too, or faxes stop reaching the inbox again.
- **The cause of the move on about 12 October is not known**, so it could happen again unannounced.
- **The RingCentral app's Received list** still shows these faxes only to someone signed in as that extension.

## 11. Reuse for similar issues

**How to recognise one:** a team says faxes or voicemails stopped arriving, but the vendor's log shows them received.

- **Read the vendor log first.** It shows which extension got each item, which separates "not arriving" from "arriving somewhere else".
- **Date the change** from the last good delivery and the first bad one.
- **Check the notification list on the receiving extension**, not just who owns the number.
- **Confirm with a real item** after the fix, not just the setting.
- **Account for the items in the gap**: say where they went and who can recover them.
