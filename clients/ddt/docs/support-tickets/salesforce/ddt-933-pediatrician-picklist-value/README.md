# Adding Pediatrician to provider picklists

| | |
|---|---|
| **Ticket** | DDT-933, DDT-1051 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-07 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-09-03 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (picklist field metadata, Contact record types), PI Damages panel |
| **Area** | PI medical damages: recording the treating physician for a minor client |
| **Related** | None recorded |
| **Code** | PR #306 in the `ddt-salesforce` repo |

---

## 1. Summary

The PI team needed a "Pediatrician" option when recording the treating doctor for a minor client. The ticket pointed at a "Treating Physician object" that does not exist: the Treating Physician on a medical damage is a lookup to a **Contact**, so the dropdown in the New Treating Physician window is `Contact.Specialty__c`. "Pediatrician" was added to that field and to the Contact record types. Two weeks later a second ticket reported Pediatrician "missing" from **Provider Type**, which is a different field on the damage itself. The value was still on Specialty; the requester was asked whether they also wanted it on Provider Type, and the ticket closed without that change.

## 2. What was reported

- **DDT-933, 2026-09-03:** in the New Medical Damages screen, the **New Treating Physician** window had no Specialty value for a pediatrician. Needed for accident cases where the client is a minor. The ticket's automated triage could not find the object, guessing names like `Treating_Physician__c`, `Physician__c` and `Medical_Provider__c`.
- **DDT-1051, 2026-09-18:** the same requester reported Pediatrician missing from the **Provider Type** dropdown on the New Medical Damages screen, recalled the earlier ticket, and suspected a record type or value-set problem.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Contact.Specialty__c` | Picklist field | New value `Pediatrician` (PR #306) |
| The five Contact record types | Record type picklist assignment | `Pediatrician` made available, granted at deploy time from a fresh production copy (see 6.2) |

**Out of scope**

- **`Damages__c.Provider_Type__c`.** DDT-1051 asked for it there; the requester was asked to confirm and the ticket closed without that change. As of 2026-11-05 the field has 15 values and no Pediatrician.
- **The dead Marketing Cloud picklist block in the repo's Contact record type files.** It makes those files undeployable to production (see 6.2) and was left for its own ticket.
- **The Damages panel LWC.** No code change was needed; the window renders the field with a standard `lightning-input-field`, so its options come from metadata.

## 4. How it works (background)

```
PI matter -> Damages panel -> New Medical Damages
  -> Provider Type        = Damages__c.Provider_Type__c        (the kind of treatment at this facility)
  -> Treating Physician   = Damages__c.Treating_Physician__c   (a Lookup to Contact)
       -> "+ New Physician" opens the New Treating Physician window
            -> Specialty  = Contact.Specialty__c
```

Two dropdowns on the same screen, on two different objects. A picklist value also has to be **assigned to each record type** before a record-typed form shows it; adding it to the field is not enough.

## 5. Root cause

### DDT-933: the value did not exist, and the ticket named the wrong object

There is no Treating Physician object. `Damages__c.Treating_Physician__c` is a Lookup to Contact, so the Specialty dropdown is `Contact.Specialty__c`. That is why the triage could not resolve any of the object names it tried.

### DDT-1051: a different field, not a lost value

Pediatrician had been added to **Specialty** (on the treating physician), not to **Provider Type** (on the damage). The support team checked on 2026-09-21 and confirmed it was still on Specialty, with screenshots. The report was the same word on the neighbouring field, not a regression.

Checked on 2026-11-05 for this page:

```sql
SELECT COUNT(Id) FROM Contact WHERE Specialty__c = 'Pediatrician'
```

Returns 1, so the value is in use on Specialty.

```sql
SELECT Provider_Type__c, COUNT(Id) FROM Damages__c GROUP BY Provider_Type__c
```

No Pediatrician row, and the field's describe lists 15 values without it.

## 6. Solution

### 6.1 Add the value to the field

PR #306 adds one value to `Contact.Specialty__c`, in alphabetical position:

```xml
<value>
    <fullName>Pediatrician</fullName>
    <default>false</default>
    <label>Pediatrician</label>
</value>
```

### 6.2 Make it available on the record types

**The field change alone does not satisfy the ticket.** This was verified in the partial-copy sandbox rather than assumed: after deploying only the field, every Contact record type still offered 23 values, and the UI API (the same source `lightning-record-edit-form` uses) confirmed Pediatrician was absent.

The obvious way to grant it, deploying the repo's five Contact record type files with Pediatrician added, does not work. Production rejects the repo's copies outright:

```
Global value set 'et4ae5__Mobile_Country_Code__c' in record type 'DDT Employee'
on the entity 'Contact' cannot be resolved
```

The `et4ae5` namespace is Marketing Cloud, which is not installed in production. Each repo file carries a large block assigning values of a picklist that no longer exists. The repo copies also restricted another Contact picklist to fewer values than production has, so deploying them would have hidden values from users.

So PR #306 ships only the field, and the record type grant is a deploy-time step: retrieve the five record types fresh from production, add only Pediatrician, and deploy that. That bundle validated 6 of 6 components against production. The repo's record type files were left untouched, per review direction that the PR should only add, never remove.

**Worth knowing when reading the diff:** an omitted `picklistValues` block for a field in a record type file means *all values assigned*, not none. After the grant, a fresh retrieve of the record types shows no Specialty block at all, which is the unrestricted state.

### 6.3 DDT-1051

No change. The requester was shown that Pediatrician was on Specialty and asked, on 2026-09-21 and again on 2026-09-24, whether they also wanted it on Provider Type. Not recorded in the ticket whether they answered. The ticket was closed on 2026-10-02.

## 7. Testing

From PR #306, in the partial-copy sandbox, read through the UI API:

| Contact record type | State | Specialty values | Pediatrician |
|---|---|---|---|
| PI Participant | changed | 24 | present |
| Participant | changed | 24 | present |
| DDT Employee | untouched (control) | 23 | absent |
| Referring Attorney | untouched (control) | 23 | absent |
| Emergency Contact | untouched (control) | 23 | absent |

The untouched rows are the control: they show what shipping the field alone would have left behind.

**Save path:** a Contact inserted with the PI Participant record type and `Specialty__c = 'Pediatrician'` read the value back. Run under a savepoint and rolled back; nothing persisted.

**In production:** the support team confirmed on 2026-09-07 that the value had been added, and on 2026-09-21 shared screenshots showing it still on the Specialty picklist. One Contact carries the value as of 2026-11-05.

**Not covered:** the PR's own testing did not click through the New Treating Physician window in a browser; the dropdown was verified through the UI API, not by eye.

## 8. Step by step

1. **Find the real field.** Open the screen, identify the dropdown, and follow it to its object. Here, Treating Physician is a Lookup to Contact, so the field is `Contact.Specialty__c`.
2. **Add the value** to the field's metadata (or in Setup) and validate against production.
3. **Retrieve every record type on that object fresh from production.** Do not use the repo's copies.
4. **Add only the new value** to each record type's block for that field, and nothing else.
5. **Validate** the field plus the record types against production; it should pass all components.
6. Deploy, after approval.
7. **Check as a user**, or through the UI API per record type, that the value is offered. Do not stop at "the field has the value".
8. If a later request names the same value on a different field, show the requester which field has it and ask which one they meant before changing anything.

## 9. Rollback

Deactivate the `Pediatrician` value on `Contact.Specialty__c` in Setup. Deactivating stops it being offered while leaving the existing record that uses it as it is, so prefer it over deleting the value. Not recorded in the ticket whether a rollback was ever needed.

## 10. Known limits

- **The repo's Contact record type files still cannot be deployed to production** while they carry the dead Marketing Cloud block. Any future picklist value on Contact needs the same fresh-retrieve step.
- **Provider Type has no Pediatrician.** If the PI team wants to classify a facility as pediatric, that is a separate change to `Damages__c.Provider_Type__c`.
- **A new Contact record type** would need Pediatrician assigned to it as well.

## 11. Reuse for similar issues

**How to recognise one:** "add a value to the X dropdown" where the ticket's object name does not match anything, or "the value we added is missing".

- **Trace the dropdown to its field before building.** The label on the screen and the object in the ticket are often not where the field lives.
- **A picklist value is two grants:** the field, and each record type. Check the record types every time.
- **Never deploy stale record type files to production.** Retrieve fresh, change only your value, and deploy that.
- **"It's missing again" may be a neighbouring field.** Check where the earlier change went before assuming it was lost.
