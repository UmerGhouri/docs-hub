# Fee Per Court Date agreement missing Send for Signature

| | |
|---|---|
| **Ticket** | DDT-744 |
| **Project** | Support tickets |
| **Status** | Done 2026-08-17 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-08-14 |
| **Resolved by** | Support team |
| **Systems** | Salesforce document generation and e-signature |
| **Area** | Generating the representation agreement from a Court Case (Opportunity) |
| **Related** | None recorded |
| **Code** | None: no code change in the ddt-salesforce repo |

---

## 1. Summary

When staff generated the "Representation Agreement Fee Per Court Date" document from a Court Case, the **Send for Signature** option was not offered, so the agreement had to be downloaded and sent by hand. After the support team's change, the template appeared in the list with an `(ESIGN)` tag and offered Send for Signature, and the requester was asked to try again. The exact configuration change is not recorded in the ticket.

## 2. What was reported

On a hired Court Case (case type Violation of Sentence), choosing the "Representation Agreement Fee Per Court Date" template did not offer Send for Signature. The other Representation Agreement templates did. One template, every Court Case that used it.

## 3. Scope

**In scope:** the "Representation Agreement Fee Per Court Date" template's e-signature setup, in the document generation tool's template configuration.

**Out of scope:** the other Representation Agreement templates, which already offered Send for Signature, and the content of the agreement.

## 4. How it works (background)

```
Court Case record
  -> "Choose a template" screen (search, tick one or more templates)
  -> Preview  |  Generate PDF  |  Send for Signature
```

Send for Signature is offered for templates set up for e-signature. In the template list after the fix, those templates carry an `(ESIGN)` tag in their name, for example "Representation Agreement + Affidavit (ESIGN)".

## 5. Root cause

Not recorded in the ticket. The ticket asked for the template's configuration to be compared with a working Representation Agreement template; the result of that comparison is not written down.

What the screenshot attached to the resolution shows: the template listed as "Representation Agreement Fee Per Court Date (ESIGN) [PDF]", selected, with the **Send for Signature** button available.

## 6. Solution

After the change, the template appears as "Representation Agreement Fee Per Court Date (ESIGN)" and offers Send for Signature, like the other e-sign Representation Agreement templates. The exact setting changed, and where, is not recorded in the ticket.

## 7. Testing

The support team opened the "Choose a template" screen on a Court Case, selected the template, and confirmed Send for Signature was available (screenshot on the ticket, 2026-08-17). The requester was asked to try again; her reply is not recorded in the ticket.

**Not covered:** whether a signature request sent from this template completes and files back correctly.

## 8. Step by step

1. Open a Court Case and start document generation ("Choose a template").
2. Search for "representation agreement" and compare the templates: note which ones carry `(ESIGN)` and offer Send for Signature.
3. Open the configuration of a working e-sign template and of the failing one in the document generation tool, and compare the e-signature settings.
4. Set the failing template up the same way as the working one.
5. Return to the Court Case, select the template, and confirm Send for Signature appears.
6. Ask the requester to send one agreement for signature and confirm it arrives.

## 9. Rollback

Undo the template's e-signature setting in the same screen. Safe; it only removes the option again.

## 10. Known limits

- **New templates do not offer e-signature automatically.** Each one needs the same setup.
- **A copy of a template** may not carry the e-signature setup with it. Check copies the same way.

## 11. Reuse for similar issues

**How to recognise one:** one template is missing an option that its siblings have.

- Compare the failing template's configuration with a working sibling's, side by side, before changing anything else.
- Check the template list for naming tags such as `(ESIGN)` that show which templates are set up for signature.
- Record which setting was changed on the ticket, so the next person does not have to rediscover it.
