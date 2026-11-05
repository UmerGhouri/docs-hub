# Could not create provider in New Medical Damages

| | |
|---|---|
| **Ticket** | DDT-777 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-20 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-08-19 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (permission set, LWC `damagesTable`, Apex `DamagesTableController`) |
| **Area** | PI damages: adding a medical provider from the New Medical Damages dialog |
| **Related** | DDT-654, DDT-662 (earlier access problems on the same component) |
| **Code** | PR #211 in the `ddt-salesforce` repo |

---

## 1. Summary

On a PI Matter, the "+ New Provider" form inside New Medical Damages failed with "Could not create provider". The ticket blamed the greyed-out Billing State picklist, but that was not the cause. The user could not see the `Medical Providers` Account record type, so the form fell back to the Person Account record type and the save was rejected. A new permission set grants that record type, and it was assigned to the reporting user.

## 2. What was reported

- **2026-08-19:** a staff member filling in New Provider and clicking **Create Provider** got:

  > Could not create provider. An error occurred while trying to update the record. Please try again.

- The Billing State/Province Code dropdown was greyed out, so the ticket assumed State was required and blocked the save.
- Users on the Matter Management profile created providers routinely. The reporter, on a different profile, could not.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Medical_Provider_Record_Type_Access` | Permission set (new) | Record type visibility for `Account.Medical_Providers`, nothing else |
| Permission set assignment | Data | Assigned to the reporting user after deploy |

**Out of scope**

- **The greyed-out State picklist.** `BillingStateCode` is a dependent picklist controlled by `BillingCountryCode`, and the form never shows `BillingCountryCode`, so State stays disabled for everyone, admins included. Needs an LWC change.
- **The generic error message.** `handleProviderError` in `damagesTable.js` shows the top-level message, which is always the generic text. The real reason is in `output.fieldErrors`. Needs an LWC change.
- **Other affected users.** Most users with the Damages panel lack the record type too. Assigning only the reporter was the requester's decision. The alternative, adding the record type to the `Damages_Records_Mode` permission set every Damages user already holds, was noted for when the next user reports it.

## 4. How it works (background)

```
+ New Provider (damagesTable LWC)
  -> DamagesTableController.getMedicalProviderRecordTypeId()
       returns the Medical Providers record type Id only if the running user can see it
  -> lightning-record-edit-form object-api-name="Account" record-type-id=(that Id)
  -> user fills Name, phone, address and clicks Create Provider
```

When `record-type-id` is null, `lightning-record-edit-form` uses the object's default record type for the user. In this org that is **Person Account**.

## 5. Root cause

**The ticket's guess was wrong.** There are no validation rules on Account in production, and of 421 Medical Provider accounts only 264 have a state code. Providers save fine with a blank state.

The real chain:

1. `getMedicalProviderRecordTypeId()` returns the Id only when `isAvailable()` is true for the running user.
2. The reporter's profile had no visible Account record types (four Account entries, all `visible=false`).
3. So the method returned `null`, and the form fell back to Person Account.
4. A Person Account has no `Name` field (it uses first and last name). Submitting `Name` was rejected with `INVALID_FIELD_FOR_INSERT_UPDATE`, wrapped in the generic UI API message.

**Evidence.** The failure was reproduced in a sandbox by posting the same payload to the UI API records endpoint with the Person Account record type:

```json
{
  "message": "An error occurred while trying to update the record. Please try again.",
  "output": {
    "errors": [],
    "fieldErrors": {
      "Name": [ { "errorCode": "INVALID_FIELD_FOR_INSERT_UPDATE",
                  "message": "Unable to create/update fields: Name. ..." } ]
    }
  }
}
```

The same call with the Medical Providers record type succeeded, including with a blank `BillingStateCode`.

**Supporting data:** every Medical Provider account created since 2026-07-01 had been created by users on the Matter Management profile, the only non-admin profile with the record type.

## 6. Solution

A new, additive permission set with record type visibility only, and no object, field or system permissions:

```xml
<PermissionSet>
    <label>Medical Provider Record Type Access</label>
    <recordTypeVisibilities>
        <recordType>Account.Medical_Providers</recordType>
        <visible>true</visible>
    </recordTypeVisibilities>
</PermissionSet>
```

Why this way:

- **A permission set, not a profile change.** It is additive, lives in git, and can be assigned to more users later.
- **Record type only.** The reporter already had the object, class and field access the component needs in production.
- **Keep the description short.** `PermissionSet.description` is capped at 255 characters; the first validation failed on that.

## 7. Testing

- **Validated against production** on 2026-08-20, check-only, with `DamagesTableControllerTest`: 54 of 54 passed, 0 coverage warnings. Production does not accept `NoTestRun` even for a metadata-only deploy.
- **Sandbox end to end:** after the grant, the reporting user created the same provider herself. The record got the Medical Providers record type, and phone, street, city and zip saved. State came through blank, as expected.
- **Live:** the ticket was closed on 2026-08-20 with confirmation that a new provider can be created.

**Not covered:** no automated test exercises the LWC with a user who lacks the record type.

## 8. Step by step

1. **Reproduce the error** and note the user's profile.
2. **Check record type visibility:** does the profile or any permission set make `Account.Medical_Providers` visible?
3. **Read the real error.** Look in `output.fieldErrors` of the UI API response (browser network tab), not the toast.
4. **Create the permission set** above and validate against production with `DamagesTableControllerTest`.
5. **Get the PR approved and deploy.**
6. **Assign the permission set** to the user. An assignment is data, not metadata, so it is not part of the deploy.
7. **Have the user log out and back in.** `getMedicalProviderRecordTypeId` is cacheable, so the session keeps the old null until then.
8. **Ask the user to create a provider.** The State dropdown will still be greyed out; that is expected.

## 9. Rollback

Remove the assignment or delete the permission set. That restores the error for the user. Nothing else depends on it.

## 10. Known limits

- **Other users without the record type still hit the error** until they are given it.
- **State stays greyed out** until the form shows `BillingCountryCode`.
- **The toast still hides the real error** until `handleProviderError` reads `output.fieldErrors`.
- **Side effect:** with two visible Account record types, the user's standard New Account button now shows a record-type choice step, with Person Account pre-selected. That is expected and is not a new bug.
- **The component should refuse to show the form when the record type Id is null.** There are at least two ways to get that null (no record type visibility, no Apex class access) and both look the same to the user.

## 11. Reuse for similar issues

**How to recognise one:** a create form works for some profiles and fails for others with "An error occurred while trying to update the record".

- **Read `output.fieldErrors`.** The top-level UI API message is always the generic text.
- **Check record type visibility** when a form passes a record type Id that the code only returns if the user can see it. A null quietly becomes the user's default record type.
- **Test the ticket's guess.** Here a quick count showed hundreds of providers saved without a state, which ruled the picklist out.
- **This was the third pass at the same component** (after DDT-654 and DDT-662). When the same form fails again, check class access, object access and record type visibility together.
