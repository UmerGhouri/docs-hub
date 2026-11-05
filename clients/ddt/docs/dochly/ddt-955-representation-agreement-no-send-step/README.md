# Representation Agreement generates but offers no way to send it

| | |
|---|---|
| **Ticket** | DDT-955 |
| **Project** | Dochly |
| **Status** | Done 2026-09-09 |
| **Priority** | High |
| **Reported by** | Case staff, 2026-09-08 |
| **Resolved by** | Firm staff (a colleague who had just written the user guide) |
| **Systems** | Dochly document generation and e-signature |
| **Area** | Sending the Representation Agreement for e-signature from the Account |
| **Related** | None recorded |
| **Code** | Not applicable: no code or configuration was changed |

---

## 1. Summary

A user generated the Representation Agreement and Affidavit for one client, but the screen then offered only Download, Preview and Back to record, with no way to send for e-signature. The templates they picked were the versions that **cannot** be sent for e-signature. The fix was to pick the e-signature version of the template, which is marked with a small envelope icon. Nothing in Salesforce or Dochly was changed.

## 2. What was reported

On one client's Account, the user opened the Create Dochly Templates component and generated "Representation Agreement (ESIGN)" and "Affidavit - NO SIGNATURE". Generation succeeded ("PDF generated successfully") and both PDFs were attached to the record. The next screen, "Choose a next step", showed only Download, Preview and Back to record. There was no Send or e-sign option, so the agreement could not go out, which blocked onboarding for that client.

The ticket assumed it would affect any user sending this template, and asked whether the cause was template configuration, permissions or the Dochly component.

## 3. Scope

**In scope:** guidance to the user on which template to select. No component was changed.

**Out of scope:** template, permission and component changes. None were needed.

## 4. How it works (background)

```
User opens Create Dochly Templates on the record
  -> picks one or more templates
  -> Dochly generates the PDF(s) and attaches them to the record
  -> "Choose a next step"
       e-signature template selected:  Send / e-sign is offered
       non-e-signature template:       only Download / Preview / Back
```

In the template picker, a template that can be sent for e-signature shows a **small envelope icon** next to its name. A combined template holding both the Representation Agreement and the Affidavit exists with that icon.

## 5. Root cause

User selection, not a defect. The user had picked the entries just below the e-signature one. A colleague replied in the Slack thread: "You need to select the ones with the little envelope next to it, there is one with the rep agreement and affidavit together. If it doesn't have the little envelope it cannot be sent via e-sign." The user confirmed: "You were right I am hitting the one above it."

The ticket's first guesses (template configuration, permissions, component fault) were all wrong. No investigation in Salesforce is recorded; the colleague recognised the problem from the report.

## 6. Solution

Select the template marked with the envelope icon: the combined Representation Agreement and Affidavit e-signature template. Its next step then offers sending for e-signature.

The colleague had written a user guide covering this the day before the ticket was raised.

## 7. Testing

**Live check:** the user confirmed in the Slack thread that they had been selecting the wrong entry and asked for the ticket to be closed. Whether the send then completed for that client is not recorded in the ticket.

**Not covered:** no change was made, so there is nothing to test.

## 8. Step by step

1. Open the record and launch **Create Dochly Templates**.
2. In the template list, find the entry with the **envelope icon**: the combined Representation Agreement and Affidavit template.
3. Select it and generate.
4. On "Choose a next step", send for e-signature.
5. If the next step shows only Download, Preview and Back, go back and check that the template you chose has the envelope icon.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- **Similar template names sit next to each other**, and only the envelope icon tells them apart. The same mistake is easy to repeat.
- A downloaded PDF from a non-e-signature template is not set up for e-signature. The user asked about this in the thread; the answer is not recorded in the ticket.

## 11. Reuse for similar issues

**How to recognise one:** a Dochly document generates fine but the next step has no Send or e-sign option.

- **Check which template was picked first.** Ask for a screenshot of the template list, and look for the envelope icon.
- **Before investigating permissions or configuration**, confirm the user chose an e-signature template.
- **Point the user at the user guide** for the template picker.
