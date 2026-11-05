# Matter Type has no options on Send for Signature

| | |
|---|---|
| **Ticket** | DDT-711 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-12 |
| **Priority** | High |
| **Reported by** | Intake team, 2026-08-11 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (permission set, LWC) |
| **Area** | PI intake: sending the representation agreement for signature from a Lead |
| **Related** | None recorded |
| **Code** | PR #156 in the `ddt-salesforce` repo |

---

## 1. Summary

On the Send for Signature panel, the required **Matter Type** dropdown showed only `--None--`, so an intake user could not send the contract. The panel reads the picklist from `Matter__c.Matter_Type__c`, and users without read access to that field get no values back. The fix grants read-only access to the field through a permission set every affected user already holds.

## 2. What was reported

On 2026-08-11, an intake user opening Send for Signature for one Lead reached "Step 2 of 2: Matter & Incident" and found Matter Type with no selectable options. Fee Schedule could be chosen; Matter Type and Accident Type showed `--None--`. Matter Type is required, so the form could not be submitted. A colleague with different access sent that contract instead, which unblocked the client but left the question of why the intake user could not.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Matter_Wizard_Access` | Permission set | Adds read-only (not edit) field permission on `Matter__c.Matter_Type__c`; description updated to say why |

**Out of scope**

- **Picklist values, dependencies and record types.** Nothing was wrong with them (see section 5), so nothing was changed.
- **Edit access to the field.** Not needed: the Matter is written by Apex during conversion, so the user only has to be able to read the picklist values.
- **New permission set assignments.** Every affected user already held `Matter_Wizard_Access`.

## 4. How it works (background)

```
Lead record page
  -> Send for Signature panel (LWC convertToHiredMatter)
  -> getPicklistValuesByRecordType wired to the Matter__c object
  -> Matter Type dropdown shows the values of Matter__c.Matter_Type__c
  -> user submits; Apex builds the Matter and sends the contract
```

**The Salesforce rule behind this:** the UI API (which `getPicklistValuesByRecordType` uses) leaves out any field the running user cannot read. For a picklist that means no values at all, so the dropdown shows only `--None--`. Nothing errors.

## 5. Root cause

**The ticket's first guess was wrong.** It suggested a dependent picklist with no valid mapping, or a record type with no values assigned, on the **Lead** field. On checking:

- `Matter_Type__c` has **no controlling field**, and all 8 values are active.
- The panel does not read the Lead field at all. Although it sits on a Lead, it wires `getPicklistValuesByRecordType` to the **`Matter__c`** object, so it reads `Matter__c.Matter_Type__c`.

Three objects carry a `Matter_Type__c` picklist (`Lead`, `Matter__c` and `Intake__c`), which is why the Lead diagnosis looked plausible.

The real cause: the intake user had no field-level read access to `Matter__c.Matter_Type__c`. UI API therefore returned no picklist values for it, and the panel showed an empty dropdown.

**The lesson.** An empty picklist in an LWC is far more often missing field-level security than a broken value set. Check which object the component actually reads before you check the values.

## 6. Solution

Add one field permission to `Matter_Wizard_Access`:

```xml
<fieldPermissions>
    <editable>false</editable>
    <field>Matter__c.Matter_Type__c</field>
    <readable>true</readable>
</fieldPermissions>
```

Why it was done this way:

- **Read only.** Showing the values needs read; the write happens in Apex, so edit would grant more than the panel needs.
- **An existing permission set** that every affected user already holds, so the fix took effect on deploy with no assignment step.
- **The description records why** the field is there, so nobody removes it later as unrelated to the Matter Completion panel.

## 7. Testing

- **Apex tests:** not applicable; the change is a permission set only.
- **Live check:** Not recorded in the ticket beyond the ticket being closed as done on 2026-08-12.

**Not covered:** an automated check that a user without the field permission sees an empty dropdown. Nothing in CI renders the component.

## 8. Step by step

1. **Reproduce as the affected user** (or a user with the same permission sets): open the Lead, start Send for Signature, and confirm Matter Type shows only `--None--`.
2. **Find which object the component reads.** In the LWC's JavaScript, look at what `getPicklistValuesByRecordType` is wired to. Here it is `Matter__c`, not `Lead`.
3. **Rule out the value set.** In Setup, open the field on that object and confirm it has active values and no controlling field.
4. **Check field-level security** for the user on that object's field. If it is not readable, that is the cause.
5. **Add read-only field permission** to a permission set the affected users already hold, as in section 6.
6. **Validate against production**, get the pull request approved, then deploy.
7. **Ask the affected user to reopen the panel** and confirm Matter Type now lists its values.

## 9. Rollback

Remove the `fieldPermissions` entry from `Matter_Wizard_Access` and redeploy. That brings the empty dropdown back for every user who relies on this permission set, so there is no reason to unless the grant itself is wrong.

## 10. Known limits

- **Users who do not hold `Matter_Wizard_Access`** still see an empty dropdown. Anyone new who sends contracts needs this permission set.
- **The same trap applies to any other `Matter__c` picklist the panel reads.** A field added to the panel later needs read access granted the same way.

## 11. Reuse for similar issues

**How to recognise one:** a picklist in a custom component shows only `--None--` for some users but works for others.

- **Check field-level security first.** UI API silently drops unreadable fields; nothing errors.
- **Confirm which object the component reads.** A component on a Lead page can read another object's field with the same API name.
- **Verify the ticket's diagnosis against the org** before building the fix it proposes. Here the suggested dependency and record-type causes did not exist.
- **Grant the smallest access that works**, read only when Apex does the write, through a permission set the users already have.
