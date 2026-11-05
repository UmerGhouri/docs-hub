# Send for Signature from an existing client account

| | |
|---|---|
| **Ticket** | DDT-364, DDT-365 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-16 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-07-16 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC), DocuSign |
| **Area** | PI intake: sending the retainer for e-signature and creating the Matter |
| **Related** | DDT-1070 (PI contract sender later moved from DocuSign to Dochly) |
| **Code** | DDT-365: direct commits to `main` on 2026-07-16 in the ddt-salesforce repo (no PR number), "PI contract sender: companion matter inherits parent accident details". DDT-364: none found in the repo |

---

## 1. Summary

Two gaps in the PI "Send for Signature" workflow, both closed on 2026-07-16. DDT-364 extended sending to an existing client's page, so a returning client can sign a new retainer and get a new Matter without going through a Lead. DDT-365 fixed the companion step on the Lead-side form: picking an existing client from the same accident now pre-fills the Matter Type, Accident Type and contract sentence from that client's matter.

The two tickets are related but are separate changes, so sections 2, 5 and 6 treat them separately.

## 2. What was reported

### DDT-364: no way to send from a client record (2026-07-16)

The Send for Signature workflow only existed on Leads: create Lead, send agreement, sign, convert the Lead and create the Matter. That did not fit a past client hiring the firm for a new case. Converting a Lead would create a duplicate client account, so staff either sent the retainer by email and created the Matter by hand, or created a throwaway Lead to reuse the automation.

### DDT-365: companion step forgot the parent's details (2026-07-16)

On Step 2 of the Send for Signature form, staff answer whether this is a companion case (another person from the same accident as an existing client). After answering "Yes" and picking the existing client, the form still asked for **Matter Type** and **Accident Type**, both already known from that client's matter, and the **"Contract will read"** preview text was wrong. Staff re-entered data, and the preview did not match what would go out.

## 3. Scope

**In scope**

| Component | Type | Ticket | Change |
|---|---|---|---|
| `HiredMatterConvertController` | Apex class | DDT-365 | Companion search returns the parent matter's accident type, description and incident location |
| `convertToHiredMatter` | LWC on the Lead page | DDT-365 | On companion pick, pre-fills the fields and rebuilds the contract sentence; clears them on remove |
| Client-page Send for Signature | Not found in the repo | DDT-364 | See section 6 |

**Out of scope**

- **Companion handling on the client-page send.** DDT-365 asked whether DDT-364 needed the same pattern. Not recorded in the ticket.
- **Linking the new companion Matter to the parent** beyond what the conversion already did. The existing conversion reuses the parent matter's Incident (`resolveCompanionIncidentId`); no new link was added.

## 4. How it works (background)

```
Lead page -> convertToHiredMatter LWC (Send for Signature)
  Step 1: client details (new client, or pick an existing one)
  Step 2: matter details; "Is this a companion case?" -> pick existing client's matter
          Matter Type, Accident Type, incident date/location
          "Incident Description (goes on the contract)" sentence, built from those fields
  -> retainer sent through DocuSign (at the time; Dochly since DDT-1070)
  -> on signed: HiredMatterConvertController converts the Lead and creates the Matter
```

## 5. Root cause

### DDT-364

Not a defect. The send-and-convert path was built only for Leads.

### DDT-365

The companion search (`HiredMatterConvertController`) returned only the parent matter's Id, name, Matter Type, client name and accident date. The LWC set the companion selection but did not copy any of those values into the Step 2 fields, so Matter Type and Accident Type stayed blank and the contract sentence was built without them. The ticket's guess was that the wrong preview came from those blank inputs; the fix fills them and rebuilds the sentence, which is consistent with that guess.

## 6. Solution

### DDT-364: client-page Send for Signature

The closing comment on the ticket records what was delivered:

> We added a "Send for Signature" button on the Client page that emails/texts the retainer for e-signature through DocuSign. Once the client signs, the system automatically creates their case (Matter + Incident) and saves the signed PDF to the record. There's also an "Already signed on paper — create matter" option for clients who signed in person.

No component for this appears in the ddt-salesforce repo on any branch: the `convertToHiredMatter` LWC targets the Lead page only. How it was built, and whether it is still in use after the move to Dochly (DDT-1070), is not recorded in the ticket.

### DDT-365: inherit the companion's accident details

The companion search now also selects `Causes_of_Action__c` (Accident Type), `Description__c` and the parent Incident's street, city and state, and returns them on the result wrapper:

```apex
r.accidentType = m.Causes_of_Action__c;
r.description = m.Description__c;
if (m.Incident__r != null) {
  r.locationStreet = m.Incident__r.Location__Street__s;
  r.locationCity = m.Incident__r.Location__City__s;
  r.locationState = m.Incident__r.State__c;
}
```

In the LWC, picking a companion copies them into the form and rebuilds the contract sentence:

```js
this.matterType = ds.type || "";
this.accidentType = ds.accidenttype || "";
this.incidentDate = ds.date || null;
this.locCity = ds.loccity || "";
if (ds.locstate) this.incidentState = ds.locstate;
this.locationType = "address";
this.locAddress = ds.locstreet || "";
this._descDirty = false;
this.refreshAccidentTypeOptions();
this.syncSentence();
```

Removing the companion clears the same fields and rebuilds the sentence, so the form resets to blank. A note under the fields reads "Pre-filled from `<client>`'s matter — edit if needed."

Why it is done this way:

- **Pre-filled, not hidden.** The ticket suggested skipping the two prompts. The fix fills them in and leaves them editable, so staff can see and correct what was inherited.
- **The sentence is left unlocked** (`_descDirty = false`), so changing the Matter Type or Accident Type afterwards rebuilds the contract sentence live.
- **The incident date and location are seeded too**, because the contract sentence is built from them.

The commits describe these files as matching "the deployed prod versions": the change was made in production and the repo was brought in line the same day.

## 7. Testing

- **DDT-364:** not recorded in the ticket.
- **DDT-365:** the ticket asked for Apex and Jest regression tests for companion inheritance. The commits change only the controller and the LWC and add no tests. How the change was checked is not recorded in the ticket.

**Not covered:** an automated test that picking a companion fills Matter Type, Accident Type and the contract sentence, or that answering "No" fills nothing.

## 8. Step by step

For the DDT-365 change:

1. In `HiredMatterConvertController`'s companion search, add `Causes_of_Action__c`, `Description__c` and the Incident's `Location__Street__s`, `Location__City__s` and `State__c` to the query, and new `@AuraEnabled` fields on the result wrapper.
2. In `convertToHiredMatter.html`, pass those values on the companion result row as `data-` attributes.
3. In `handleCompanionPick`, copy them into the form fields, reset `_descDirty`, refresh the accident-type options and call `syncSentence()`.
4. In `handleCompanionRemove`, clear the same fields and call `syncSentence()`.
5. Validate against production, get approval, deploy, and commit to the repo the same day.
6. Check: on a PI Lead, answer "Yes" on the companion question, pick an existing client, and confirm Matter Type, Accident Type and the contract sentence fill in; remove the companion and confirm they clear.

For DDT-364: not recorded in the ticket.

## 9. Rollback

- **DDT-365:** redeploy the previous `HiredMatterConvertController` and `convertToHiredMatter`. Staff go back to typing the fields by hand. No data changes, so it is safe.
- **DDT-364:** not recorded in the ticket.

## 10. Known limits

- **Inherited values can be edited**, so two companions can still end up with different Accident Types if someone changes one.
- **Only the fields listed above are inherited.** The ticket's suggested extras (at-fault party, coverage limits, insurance carriers) were not added.
- **The client-page send is not in git.** Anyone changing the Send for Signature workflow should check production for it first rather than trusting the repo.
- **The workflow has since moved from DocuSign to Dochly** (DDT-1070). The companion pre-fill lives in `convertToHiredMatter` and `HiredMatterConvertController`, which that move also changed, so read the current versions before relying on the snippets above.

## 11. Reuse for similar issues

**How to recognise one:** a form asks the user to choose a related record, then asks again for values that record already holds.

- When a pick implies known values, return them with the search result and copy them in on pick; clear them again on un-pick.
- Pre-fill and leave editable, with a note saying where the values came from, rather than hiding the fields.
- If a preview is built from form fields, rebuild it every time those fields are filled programmatically.
- Anything changed in production gets a PR or commit the same day; DDT-364's component is the example of what happens when it does not.
