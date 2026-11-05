# Pull Discovery button missing on inbox email

| | |
|---|---|
| **Ticket** | DDT-1223 |
| **Project** | Support tickets |
| **Status** | Done 2026-10-05 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-10-05 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Shared Inbox, Pull Discovery LWC), Axon evidence portal |
| **Area** | Discovery intake: pulling disclosure files from a portal email into the case |
| **Related** | None recorded |
| **Code** | Not applicable: no change was made |

---

## 1. Summary

A staff member linked an incoming Axon discovery email to a client in the Shared Inbox, but the **Pull Discovery** button did not appear on it. Before anyone investigated, the requester found the answer herself: for this disclosure she first had to click the link in the email and request the download link from the portal. Nothing was changed in Salesforce.

## 2. What was reported

- **2026-10-05 (UTC):** an Axon disclosure email, forwarded into the office inbox by a staff member, was linked to the client's Account from the Shared Inbox. The Pull Discovery button did not show on the email. The ticket expected it to appear once the email was linked to a Court Case or client. Discovery intake for the case was blocked until it worked.
- **About 20 minutes later:** the requester wrote "Disregard. I realized I need to click the link and request the download link from the portal."

One email on one case was affected.

## 3. Scope

**In scope**

- Confirming whether anything in Salesforce needed to change. Nothing did.

**Out of scope**

- **Changing when the Pull Discovery button shows.** The ticket asked whether the trigger was the Court Case link, the client link or both. Since the requester resolved it herself, no change was investigated or made.

## 4. How it works (background)

The **Pull Discovery** button is the `pullDiscovery` Lightning component on an inbox email. Its render rule in `pullDiscovery.js`:

```js
get showButton() {
    return !!this.detectedUrl && !!this.opportunityId && !this.activePullId && !this.caseResolved;
}
```

So the button shows only when **all** of these hold:

1. A supported discovery portal link (for example Axon or NICE) was detected in the email (`DiscoveryPullController.detectPortalUrl`).
2. The email is linked to a **Court Case** (the Opportunity object, relabelled "Court Case" in this org). A link to the client's Account alone does not set this.
3. No pull is already running for it.
4. The inbox email is not resolved or closed.

The **Logins** utility next to it only needs condition 1.

## 5. Root cause

The requester's own finding: this disclosure needed a manual step in the portal first. She had to click the link in the email and request the download link from the portal.

Which of the four conditions in section 4 was not met on this email was **not recorded in the ticket**, because the ticket was closed on the requester's word before it was investigated.

## 6. Solution

Not applicable: no change was made. The requester clicked the link in the email and requested the download link from the portal.

## 7. Testing

Not applicable: nothing was changed, so nothing was tested.

## 8. Step by step

When the Pull Discovery button is missing on a discovery email:

1. **Check the email itself.** If the portal email asks you to request a download link, click its link and request it in the portal first, as the requester did here.
2. **Check that the email is linked to the Court Case**, not only to the client's Account. The button needs a Court Case.
3. **Check that the email is not resolved or closed.** The button hides on resolved emails.
4. **Check whether a pull is already running** on it. The status panel shows instead of the button while one is.
5. **Check that the Logins button shows.** If Logins is missing too, no portal link was detected in the email, which points at the email content rather than the case link.
6. If all of these hold and the button is still missing, raise a new ticket with the email and the case it is linked to.

## 9. Rollback

Not applicable: no change was made.

## 10. Known limits

- The cause on this specific email was never pinned to one of the four conditions, so this page cannot say which one was missing.

## 11. Reuse for similar issues

**How to recognise one:** a "button missing" report on a component that shows conditionally.

- **Find the component's render rule first** (here `showButton` in `pullDiscovery.js`) and check each condition against the record before assuming a bug.
- **Use the sibling control as a probe.** Logins needs only the portal link, so whether it shows splits "no link detected" from "not linked to a Court Case".
- **Ask whether the source system needs a manual step.** Portal emails sometimes need a download link requested before there is anything to pull.
