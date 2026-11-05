# Email addresses converted to tracking links in texts

| | |
|---|---|
| **Ticket** | DDT-931 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-30 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-09-03 |
| **Resolved by** | Support team |
| **Systems** | Heymarket (outbound SMS, per-inbox "Auto shorten SMS URLs" setting) |
| **Area** | Texting clients from Heymarket |
| **Related** | DDT-986 (same Heymarket link-shortening behaviour, mangling payment links on another inbox) |
| **Code** | None. Not a Salesforce issue |

---

## 1. Summary

When a staff member typed an email address into a Heymarket text, Heymarket rewrote it into a short tracking link on Heymarket's own link domain, with the email's domain tacked on the end. The recipient got a broken link instead of the address. The support team established this was Heymarket's link shortener, not Salesforce, found it did not reproduce from their own inbox, and reported it to Heymarket support. The ticket was closed on 2026-09-30 after the reporter was asked to verify and to send a recording if it still happened.

## 2. What was reported

- **2026-09-03:** a staff member sending a government office's email address in a text saw it turn into a short tracking link both in the composer and in the sent message. Screenshot provided. The ticket asked whether email addresses could be excluded from shortening and whether all users were affected.
- **2026-09-07:** the support team tested from their own inbox. A plain email address arrived as plain text. Only text that was already a URL was linked. They noted that the shortening setting is **per inbox, not per user**, and asked which inbox the reporter used.
- **2026-09-07:** the reporter replied that it was still happening and that the address converted to a link as soon as it was typed, with a new screenshot.
- **2026-09-15:** the support team recorded that this is a Heymarket issue, reported it to Heymarket support, and linked DDT-986 as the same issue.
- **2026-09-21:** the reporter was asked to verify whether it was resolved, and to share a recording if it still occurred.

Affected: the reporter's inbox. Not recorded in the ticket how many other inboxes showed it.

## 3. Scope

**In scope:** how Heymarket handles an email address typed into an outbound text, and the inbox setting that controls link shortening.

**Out of scope:** Salesforce. The support team concluded this was a Heymarket issue, not a Salesforce one; the reporter saw the conversion in Heymarket's composer before sending.

## 4. How it works (background)

```
Staff member types a message in a Heymarket inbox
  -> if the inbox has "Auto shorten SMS URLs" on,
     Heymarket replaces anything it reads as a URL with a short tracking link
  -> the message is sent with the short link in place of the original text
```

The setting lives on each Heymarket inbox (Settings, "Auto shorten SMS URLs"), so two people texting from different inboxes can see different behaviour from the same text.

## 5. Root cause

Heymarket's automatic link shortener treated the email address as a URL and replaced part of it with a tracking link. The evidence:

- The rewrite appears in the Heymarket composer as soon as the address is typed, before the message is sent.
- From the support team's inbox, the same address went through as plain text, which fits a per-inbox setting.
- DDT-986 shows the same shortener mangling real URLs on another inbox, joining several links into one broken string.

Why it shortened an email address on one inbox and not another is not recorded in the ticket. It was handed to Heymarket support.

## 6. Solution

- Confirmed the behaviour is inside Heymarket, not Salesforce, so no Salesforce change was made.
- Reported it to Heymarket support on or before 2026-09-15 and tracked their response.
- Asked the reporter to verify on 2026-09-21 and to send a screen recording if it persisted. The ticket was closed on 2026-09-30.

Not recorded in the ticket: what Heymarket changed, or whether "Auto shorten SMS URLs" was switched off on the reporter's inbox.

## 7. Testing

| Test (2026-09-07, support team's inbox) | Result |
|---|---|
| Send a plain email address | Arrived as plain text, no link |
| Send a short tracking link followed by an email domain | Arrived as a link, which is correct for a URL |

**Not covered:** a test from the reporter's own inbox after Heymarket's response. Not recorded in the ticket.

## 8. Step by step

1. Get a screenshot and the **inbox name** the user was sending from. The setting is per inbox, so the inbox matters more than the user.
2. Send the same text yourself from that inbox, and from a second inbox, to a test phone.
3. In Heymarket, open the affected inbox's settings and check **Auto shorten SMS URLs**.
4. If switching it off is acceptable for that inbox, turn it off and resend the test. Note that this also stops click tracking on real links from that inbox.
5. If it must stay on, or if it still happens with the setting off, raise it with Heymarket support with the screenshots and the inbox name.
6. Ask the user to confirm with a fresh text and, if it persists, a screen recording.

## 9. Rollback

Not applicable: no setting was recorded as changed. If "Auto shorten SMS URLs" is switched off as in step 4, rolling back is switching it on again in the same place.

## 10. Known limits

- Heymarket's shortener is outside the firm's control. A Heymarket update can change what it treats as a URL.
- Turning the setting off per inbox fixes that inbox only.
- DDT-986, the same behaviour on another inbox, was still open as of 2026-11-05.

## 11. Reuse for similar issues

**How to recognise one:** text that the user typed comes out changed in the sent message, especially anything with a dot in it (email addresses, web addresses, file names).

- **Find out where the text changes.** If it changes in the composer, it is the messaging tool, not Salesforce.
- **Ask for the inbox, not just the user.** Heymarket settings are per inbox.
- **Test from more than one inbox** before concluding it cannot be reproduced.
- **Link the duplicates.** DDT-986 was the same shortener on a different inbox, and treating them as one issue with the vendor is faster than two.
