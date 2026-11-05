# "Upload & Send for Signature" button on the Matter page

| | |
|---|---|
| **Ticket** | DDT-1352 |
| **Project** | Dochly |
| **Status** | Live since 2026-10-16 |
| **Priority** | Low |
| **Reported by** | Firm staff, 2026-10-15 (ticket opened 2026-10-16) |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow / Quick Action / Lightning page), Dochly e-signature |
| **Area** | Sending one-off letters for signature from a Matter |
| **Related** | DDT-1387 (completion email for Dochly signatures) |
| **Code** | PRs #741, #753 and #754 in the `ddt-salesforce` repo |

---

## 1. Summary

Staff asked for a shortcut on the client's Matter to send an envelope for signature, like the Dochly template buttons, for letters their department receives that can't be made into templates. We added an **Upload & Send for Signature** quick action to the Matter page. It opens Dochly E-Sign on its New envelope screen with the Matter already linked. There's no custom code: it wraps a Dochly package component in a one-screen flow.

## 2. What was reported

The request, made in Slack on 2026-10-15: "Is it possible to create a shortcut on our client's matter to send an envelope? Similar to our Dochly templates?"

Before this, the only way to send a one-off letter was to open the E-Sign tab directly, upload the PDF, add the signer, place fields and send. Envelopes sent that way from the tab carry no source record, so they aren't linked to the Matter.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Matter_Upload_and_Send_for_Signature` | Flow (screen) | New |
| `Matter__c.Upload_and_Send_for_Signature` | Quick action (flow) | New |
| `Matter_Record_Page` | Lightning record page | Existing. One action added to the highlights panel, right after the existing Dochly button |

`dochly:dhFlowOpenUploadSign` is a Dochly package component, not ours. Its inputs, read from production: `recordId` (required, the record the new envelope is linked to) and `finishFlow` (so a quick-action window closes by itself).

**Out of scope**

- **Who can send.** The button only opens E-Sign. Sending still needs a Dochly e-signature permission set; no permissions were changed.
- **Creating or sending anything automatically.** The flow creates nothing. The envelope exists only once the user sends it in E-Sign.

## 4. How it works (background)

```
Matter page -> "Upload & Send for Signature" (quick action)
   -> screen flow Matter_Upload_and_Send_for_Signature
        -> dochly:dhFlowOpenUploadSign (recordId = Matter Id, finishFlow = true)
             -> Dochly E-Sign, New envelope, Matter linked
```

Staff then:

1. upload the letter as a PDF,
2. add the client as signer,
3. place the signature and date fields,
4. send.

## 5. Root cause

Not applicable: this was a feature request, not a defect.

## 6. Solution

**A screen flow around Dochly's own component.** Dochly 1.91 ships `dhFlowOpenUploadSign`, which opens E-Sign's New envelope screen linked to a record. A one-screen flow passes it the Matter Id with `finishFlow = true`, and a flow quick action puts it on the Matter page. This is the same set of manual steps staff had already been given, started from the Matter instead of the tab. The difference is the link to the Matter.

**The page was reconciled with production first.** `Matter_Record_Page` was retrieved from production on 2026-10-16 and compared with `main`: identical apart from line endings. The only change is the one new action entry.

**Rolled out in three steps, all on 2026-10-16:**

1. **PR #741:** the flow, the quick action and the page change, deployed as a **one-user pilot** (the action had a visibility rule showing it to one support-team user only). Production deploys flows as Draft, so version 1 was activated by hand straight after the deploy.
2. **PR #753:** git updated to record the flow as Active, from production's own retrieved copy. The only behavioural difference was `Draft` to `Active`.
3. **PR #754:** the pilot's visibility rule removed, so everyone who sees the Matter page gets the button. Page-only change.

**Who can actually send**, measured in production on 2026-10-16: of 49 active standard users (about 10 of them integration accounts), 40 hold `Dochly_eSignature_Admin` and 10 hold `Dochly_eSignature_Sender`. The flow needs Run Flows, which every profile has except admin, read-only and integration profiles.

## 7. Testing

**It couldn't be tested in the sandbox.** The sandbox runs Dochly 1.83 and production runs 1.91, and only 1.91 has `dhFlowOpenUploadSign`. A sandbox dry run failed on exactly that component, while the quick action and page passed. Upgrading the sandbox to 1.91 was attempted on 2026-10-16 and failed on the custom object limit (the sandbox carries 37 custom objects production doesn't have); the install rolled back.

**Production deploy:** the deploy doc records the PR #741 deploy with `RunSpecifiedTests` on `GlobalMinuteSchedulerTest`, 33 of 33 passing. There is no Apex in this change; a test is named only because a production deploy needs one.

**Live check (2026-11-05):** in production, `Matter_Upload_and_Send_for_Signature` has its latest version active, last modified 2026-10-16.

**Not covered:**

- The pilot click-through on a real Matter (button opens E-Sign, a test envelope sends and signs, the envelope is linked to the Matter) is listed on the ticket and in the deploy doc, but its result is not recorded.
- The requester's live check after rollout is not recorded.
- Nothing in CI or a validation opens a screen flow, so the deploy proves the metadata is valid, not that the button works.

## 8. Step by step

1. **Retrieve `Matter_Record_Page` from production** and confirm it matches git before adding anything.
2. **Create the flow**: one screen holding `dochly:dhFlowOpenUploadSign`, with `recordId` set to the Matter Id and `finishFlow` set to true.
3. **Create a flow quick action** on `Matter__c` that runs it.
4. **Add the action** to the Matter page's highlights panel. For a pilot, add a visibility rule limiting it to one user.
5. **Validate check-only against production.** The sandbox can't, if its Dochly version lacks the component.
6. **Deploy** the flow, quick action and page after approval.
7. **Activate the flow** in Flow Builder. Production deploys it as Draft, and the button errors until it is active. Record it as Active in git the same day.
8. **Re-retrieve the page** from production and confirm it matches git.
9. **Click through** on a test Matter: E-Sign opens, an envelope sends and signs, and it is linked to the Matter.
10. **Roll out**: remove the visibility rule and deploy the page.

## 9. Rollback

Remove the `Matter__c.Upload_and_Send_for_Signature` entry from `Matter_Record_Page` and deploy the page. The flow and quick action can stay; nothing calls them without the button. Safe: it creates no data.

## 10. Known limits

- **Everyone sees the button; not everyone can send.** A user without a Dochly e-signature permission set gets to E-Sign but can't send from it.
- **It depends on Dochly 1.91 or later.** A sandbox on an older Dochly version can't hold this flow.
- **Scope approval.** The deploy doc records that the feature-scope approval for the all-user rollout was still owed when it went out.

## 11. Reuse for similar issues

**How to recognise one:** staff want a managed package's screen opened from a record, already linked to that record.

- **Look for a flow component in the package first.** Packages often ship one for exactly this, and a one-screen flow plus a quick action needs no code.
- **Read the component's inputs from production**, not the sandbox, when the two run different package versions.
- **Pilot with a visibility rule** on the page action, then remove the rule to roll out. It's a page-only change either way.
- **Activate flows by hand after a production deploy**, and record the Active status in git the same day.
