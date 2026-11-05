# Slack link previews no longer show client and matter names

| | |
|---|---|
| **Ticket** | DDT-925 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-07 (no org change; affected one user's Slack app) |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-09-03 |
| **Resolved by** | Support team |
| **Systems** | Slack, Salesforce for Slack integration (link previews) |
| **Area** | Sharing Salesforce record links in Slack channels |
| **Related** | None recorded |
| **Code** | None |

---

## 1. Summary

A staff member reported that Salesforce record links pasted into Slack had stopped expanding into a preview card showing the client or matter name, and stayed as bare links. On checking, previews were working for everyone else, so the problem was isolated to that one person's Slack app. The fix offered was to reinstall the Slack app on their side. No change was made to Salesforce or to the integration.

## 2. What was reported

- **2026-09-03:** links to Salesforce records (an Account in the example) pasted into Slack no longer showed the record name. The reporter shared a phone screenshot of the bare link. The team uses these previews to see at a glance which client or matter a message is about, without opening Salesforce.
- **2026-09-07:** the support team replied that previews were working for everyone else and asked the reporter to reinstall the Slack app and check again. The reporter then posted a test link to a Court Case record.

One user was affected. Not recorded in the ticket whether the reinstall was confirmed to fix it.

## 3. Scope

**In scope:** whether the Salesforce-to-Slack link preview was working, and for whom.

**Out of scope:** the integration's authorisation, its integration user and its permissions. The ticket listed these as things to check, but once previews were confirmed working for other users there was no org-wide fault for them to explain.

## 4. How it works (background)

```
User pastes a Salesforce record link into Slack
  -> Slack recognises the Salesforce domain
  -> the Salesforce for Slack app looks the record up
  -> Slack shows a preview card with the record name under the link
```

The preview depends on the Slack app connection on the Salesforce side and on the Slack client showing the preview. If it fails for everyone, the integration is the suspect. If it fails for one person, their own client or connection is.

## 5. Root cause

Not recorded in the ticket beyond this: the fault was specific to the reporter's Slack app, because the same kind of link previewed correctly for other users. The ticket's first guesses (integration authorisation, integration user permissions) did not apply, because those would have broken previews for everyone.

## 6. Solution

No change to Salesforce or to the integration. The reporter was asked to reinstall the Slack app and try again.

Why that and not an integration fix: testing the same kind of link as other users showed the integration itself was working. Re-authorising or changing the integration would have touched every user to fix one.

## 7. Testing

- **Checked:** record links previewed correctly for other users (stated in the support team's reply on 2026-09-07).
- **Not covered:** the ticket's acceptance criteria asked for a check across Account, Court Case, Contact and Case links with a non-admin user. Not recorded in the ticket that each was checked individually, or that the reporter confirmed the reinstall fixed it.

## 8. Step by step

1. Ask the reporter for one example link and a screenshot of what they see.
2. Paste the same kind of record link into Slack yourself, and ask one other non-admin user to do the same.
3. **If previews work for others,** the problem is the reporter's client. Ask them to sign out and back in, or reinstall the Slack app, and test again.
4. **If previews fail for everyone,** check the Salesforce for Slack app's connection and its integration user's access in Salesforce.
5. Ask the reporter to confirm with a fresh test link.

## 9. Rollback

Not applicable: nothing was changed in Salesforce or Slack's workspace settings.

## 10. Known limits

- A preview only appears for a record the integration can read. A record type, object or field the integration cannot see will show as a bare link for everyone, which looks the same as this problem but has a different cause.
- Not recorded in the ticket whether the reporter's previews came back.

## 11. Reuse for similar issues

**How to recognise one:** "it stopped working for me" with no matching report from anyone else.

- **Reproduce as another user first.** One person failing and everyone else working points at the person's device or app, not the integration.
- **Do not change a shared integration to fix one user.** Re-authorising or editing the integration user affects everyone.
- **Ask for one concrete example** (the link and a screenshot) so the test you run matches what the user saw.
