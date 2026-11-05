# Attorney Welcome Call option on the Calendly scheduling screen

| | |
|---|---|
| **Ticket** | DDT-1058 |
| **Project** | Dochly Template Migration |
| **Status** | Done 2026-09-21 |
| **Priority** | High |
| **Reported by** | Support team, 2026-09-21 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Screen Flow), Calendly managed package, Calendly |
| **Area** | Scheduling client calls from the Court Case |
| **Related** | DDT-1057 (welcome email that links to this event), DDT-735 (same "missing from the widget" failure for the Case Manager Welcome Call) |
| **Code** | PRs #413, #414 and #418 in the ddt-salesforce repo |

---

## 1. Summary

Calendly already had an "Attorney Welcome Call" event assigned to 8 accounts, but the Court Case's **Schedule Calendly Meeting** screen offered no button for it, so an attorney had no welcome-call option. Three PRs added the button, fixed its look, and corrected its label from ACW to AWC. The Calendly event itself was still named **ACW**, so the button and the booking page it opens disagreed. This ticket recorded that mismatch and the order needed to fix it.

## 2. What was reported

- Picking Primary Attorney on the scheduling screen showed only Case Discussion and two Case Update lengths. The existing Welcome Call button is limited to the Case Manager host.
- 7 of the 10 active users with the title Attorney held the Calendly event. For the other 3 the button opens a booking path that does not exist, because the flow does not check that the event exists before building the link.
- After PR #418 the button read **AWC** while the Calendly page read **ACW**. The client sees both.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Calendly_Court_Case_Schedule_Calendly_Meeting_v3` | Screen flow | New `attorney_welcome_call` formula, new `DisplayAttorneyWelcomeCall` screen field, and a visibility rule copied from the 30-minute button (PR #413) |
| Button image | File shared to all users | New image (PR #413), replaced to match its neighbours (PR #414), replaced again with the AWC label (PR #418) |

PR #413 was purely additive: 42 lines added, none removed, and no existing formula, field, rule, decision, lookup or variable touched.

**Out of scope**

- **Renaming the event in Calendly.** That is an admin action in Calendly, not a deploy.
- **The formula's URL path.** Left on `attorney-welcome-call-acw` on purpose (section 6).
- **A known phone-number defect** in the formulas, shared by all six existing buttons. The new formula copies it on purpose, because fixing it here would change what the other buttons send.

## 4. How it works (background)

```
User opens Schedule Calendly Meeting on a Court Case
  -> picks a host (Case Manager, Primary Attorney, ...)
  -> the screen shows one image button per event type that host offers
  -> each button is a link built by a formula:
       the host's Calendly page + event path + the case's name, phone and record Id
  -> Calendly opens with those details filled in
```

The Calendly managed package syncs each host's event links into `Calendly__CalendlyLink__c`. That sync is where the event's real path can be checked.

## 5. Root cause

Two separate gaps.

1. **No button.** The event existed in Calendly from 2026-08-27, but nobody had added it to the Salesforce screen. An event existing in Calendly is not the same as it being offered in Salesforce. DDT-735 was the same failure for the Case Manager Welcome Call.
2. **Two names.** "Attorney Welcome Call" abbreviates to AWC, but the event was created in Calendly as "Attorney Welcome Call - ACW" on the path `attorney-welcome-call-acw`. The synced link records, last synced 2026-09-18, still read ACW on all 8 accounts.

**The button's look (PR #414).** The first image rendered larger, in a different colour and at a different offset from its neighbours. The cause was a width attribute that PR #413 had added to make it fit:

| | Existing buttons | First image (PR #413) |
|---|---|---|
| Canvas | 875 x 100 | 726 x 66 |
| Background | opaque white | transparent |
| Text colour | `#0A2540` | `#444444` |
| Text height | 27 px | 41 px |
| Width attribute | none | 242 |

The six existing images are all 875 px wide with no width attribute, so the browser scales them all by the same factor. Pinning the new one to 242 px scaled it differently, so its text came out about 30% taller. The attribute added to make it match is what stopped it matching.

## 6. Solution

**PR #413:** added the formula, screen field and visibility rule. The image was shared to all users the same way as the six existing images; a plain file upload shares only to the uploader and would show as broken for everyone else.

**PR #414:** replaced the image with one built to the existing buttons' exact geometry (875 x 100, Arial Bold, the same text colour), and dropped the width attribute.

**PR #418:** replaced the image again with the label **AWC**, and moved the link's title text to AWC. **The formula's path was deliberately left as `attorney-welcome-call-acw`.** Changing it before Calendly is renamed would point all 8 attorneys' buttons at a path that does not exist.

**What the ticket set out for the rest, in this order:**

- [ ] Rename the event in Calendly to the final name, once the firm confirms AWC or ACW.
- [ ] Find out whether the rename changes the URL path or only the display name. Calendly can do either.
- [ ] If the path changes, change the Salesforce formula in the same hour, or all 8 links break.
- [ ] Re-sync `Calendly__CalendlyLink__c` and confirm all 8 accounts carry the new value.
- [ ] Confirm staff can book the event from the Salesforce Calendly widget.
- [ ] Hand the final link to DDT-1057 for the welcome email.

Whether the Calendly rename was done is not recorded in the ticket. As of 2026-11-05 the repo's copy of the flow still builds the link on the `attorney-welcome-call-acw` path.

## 7. Testing

**Verified:**

- Before editing, production's version of the flow was retrieved and matched the repo byte for byte.
- A production dry run succeeded.
- One attorney's ACW booking link was opened by hand and booked correctly.
- The PR #414 and #418 images measure identical to an existing button on canvas size, dot position and size, text position, text height and colour.

**Live check (2026-11-05):** production holds three versions of the flow saved on 2026-09-18, matching the three PRs, and a later active version from 2026-10-14 (DDT-1309).

**Not verified:** nothing rendered the screen. A dry run compiles metadata and runs no browser code, so nobody had seen the button on screen when the PRs merged. The image measurements are of the source files, not of what the browser shows, and PR #414 is the proof that those can differ. The ticket asks that someone open a record and click the button.

## 8. Step by step

To add a new Calendly event to the scheduling screen:

1. Confirm the event exists in Calendly and which accounts hold it. Read the synced `Calendly__CalendlyLink__c` records for its exact path.
2. Retrieve production's current version of the screen flow and confirm it matches the repo.
3. Add a formula for the link, copying the closest existing button's formula.
4. Add a display field and a visibility rule for the hosts who should see it.
5. Build the button image to the existing images' geometry. Add no width attribute.
6. Upload the image and share it to all users the same way as the existing images. Check it as a different user.
7. Dry-run against production, then deploy.
8. Open a Court Case, pick the host and click the button.

To rename the event later, follow the order in section 6. The Calendly rename and any formula change go together.

## 9. Rollback

Redeploy the previous version of the screen flow. The old images are left in place and unreferenced, so an earlier version still finds its image. Each image is referenced by its own version, so a corrected image cannot reuse the old one.

## 10. Known limits

- **The button opens a missing page for hosts without the event.** The formula does not check that the host holds the event. At the time, 3 of 10 active attorneys did not. That is a Calendly assignment fix, not a code fix.
- **The button and the Calendly page disagree (AWC against ACW)** until the event is renamed.
- **If Calendly's path changes without the formula,** every attorney's link breaks at once.

## 11. Reuse for similar issues

- **An event in Calendly is not an option in Salesforce.** Check the screen offers it for the right host.
- **Read the real path from the synced link records** before hard-coding it in a formula.
- **Rename in the right order.** When a name lives in two systems and one builds links to the other, change the target first, and the link in the same hour if the path moves.
- **Match images by their files, not by forcing a size.** If the existing images carry no width attribute, the new one must not either.
- **Share flow images to all users,** and check them as a second user. A file the uploader can see may be broken for everyone else.
- **A green dry run on a screen change is silent, not clean.** Open the screen.
