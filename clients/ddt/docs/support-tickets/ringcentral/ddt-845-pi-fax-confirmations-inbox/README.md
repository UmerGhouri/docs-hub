# PI fax confirmation emails routed to the injury inbox

| | |
|---|---|
| **Ticket** | DDT-845 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-28 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-08-26 |
| **Resolved by** | Support team |
| **Systems** | RingCentral (fax notification settings) |
| **Area** | PI department: outbound fax confirmations |
| **Related** | None recorded |
| **Code** | None. RingCentral admin configuration only |

---

## 1. Summary

Fax transmission result emails for the PI fax line were going only to the personal inbox of whoever sent the fax, so nobody else could see which faxes had failed. The request assumed the fax line's own settings were wrong; they were not. RingCentral sends the confirmation to the **sending user**, not to the fax line.

Support added the PI shared inbox as an extra recipient of **fax transmission results only**, on each of the five PI users' own notification settings, and kept their personal email on as well.

## 2. What was reported

Fax confirmations (success and failure) landed in individual users' personal inboxes. When the sender was out, the rest of the PI team could not tell which pages had gone through or which needed resending. The example given was a multi-file fax where one of the files failed. The request was to send these confirmations to the PI shared inbox instead.

## 3. Scope

**In scope**

| Setting | Where | Change |
|---|---|---|
| Notifications, Fax Transmission Results | Each of the five PI department users in RingCentral | PI shared inbox added as a custom recipient; personal email kept |

**Out of scope**

- **Voicemail, inbound fax and text notifications.** Left going to each user's personal email only, so voicemail transcripts and text alerts do not land in the shared client inbox.
- **The fax line's own notification settings.** Already pointed at the shared inbox; nothing to change.

## 4. How it works (background)

```
PI user sends a fax from the RingCentral app using the PI fax line
  -> RingCentral transmits it
  -> RingCentral emails the result (sent / failed)
       to the notification addresses of the USER who sent it
       (not the addresses configured on the fax line)
```

## 5. Root cause

The ticket's first assumption was that the fax line's notification recipient needed changing. **That setting was already correct**: the PI fax line already had the shared inbox configured. The confirmations did not use it because RingCentral routes fax transmission results to the person who sent the fax, using that person's own notification settings.

## 6. Solution

The fix was proposed on the ticket, approved by firm management, and applied in the RingCentral admin portal on 2026-08-28.

For each of the **five PI department users**:

1. Open the user's **Notifications** settings.
2. Switch from **Default** to **Custom recipients**.
3. Add the PI shared inbox as a recipient of **Fax Transmission Results** only.
4. Keep the user's personal email on the same notification.

Why it was done this way:

- **All five users, not just the two who send most faxes.** Two users do most of the faxing, but the approval asked for all five so that the next person who starts faxing does not silently fall outside the routing and produce a second ticket. Adding the address to someone who rarely faxes costs nothing.
- **Custom recipients, not Default.** With Default, every notification type would go to the shared inbox, including voicemail transcripts and text alerts. Custom lets only fax results go there.
- **Personal email kept**, so each sender still sees their own result immediately.

## 7. Testing

The approval made a point of this: **a settings page showing the address saved is not proof that the notification arrives there.** It asked for a real fax to be sent by a user who is **not** one of the two regular senders, so the test would cover one of the newly configured users.

A real fax was sent from the desktop app, and its confirmation email arrived in the PI shared inbox. **Success path live-verified** on 2026-08-28.

**Not covered:** a failed transmission was not tested, so the failure notification reaching the shared inbox is not separately confirmed. It is the same notification setting.

## 8. Step by step

1. **Check the fax line's notification settings** in RingCentral first. If the shared inbox is already there and confirmations still go to individuals, the routing is per user.
2. **List every user who can send from the fax line**, not just the ones who usually do.
3. **Get approval** for the recipient list.
4. **For each user:** Notifications, then Custom recipients, add the shared inbox to Fax Transmission Results only, and keep the personal email.
5. **Send one real fax** from a user who was newly configured, and confirm the result email lands in the shared inbox.
6. **Record on the ticket** who was changed and how it was verified.

## 9. Rollback

In each user's Notifications settings, remove the shared inbox from Fax Transmission Results, or switch back to Default. Safe; it only stops the extra copy.

## 10. Known limits

- **New PI users are not covered automatically.** Anyone added to the PI department later needs the same setting, or their fax confirmations go only to them.
- **The failure path was not tested live** (see section 7).

## 11. Reuse for similar issues

**How to recognise one:** a notification for a shared line (fax, voicemail, queue) is "going to the wrong person" even though the line's own settings look right.

- **Find out whose settings the notification actually uses.** In RingCentral, fax results follow the sender, not the line.
- **Use custom recipients** to send only the notification type you need to a shared inbox.
- **Configure everyone who could trigger it**, not only the current heavy users.
- **Verify with a real event from a newly configured user**, not by reading the settings page back.
