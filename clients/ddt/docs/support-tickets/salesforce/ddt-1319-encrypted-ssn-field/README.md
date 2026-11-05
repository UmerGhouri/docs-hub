# Encrypted SSN field and full-SSN access

| | |
|---|---|
| **Ticket** | DDT-1319, DDT-1364 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-13 (DDT-1319); access change live since 2026-10-23 (DDT-1364) |
| **Priority** | Medium |
| **Reported by** | Firm operations, 2026-10-12 (DDT-1319); Case Manager, 2026-10-16 (DDT-1364) |
| **Resolved by** | Support team |
| **Systems** | Salesforce (field metadata, Lightning record page, permission set) |
| **Area** | Client Social Security Number on PI Person Accounts |
| **Related** | None recorded |
| **Code** | PR #698 in the `ddt-salesforce` repo (a record of the Setup change; nothing deployed) |

---

## 1. Summary

The client SSN field on Account was rebuilt as Salesforce's built-in encrypted text type, masked so users only see the last four digits. The full value is visible only to users holding the `View Encrypted Fields` permission set, which was limited to a small, approved group.

A week later a case manager who needs the full SSN for CMS work asked for access. With approval, they were added to `View Encrypted Fields`, and confirmed they could see the full number.

## 2. What was reported

- **DDT-1319, 2026-10-12.** Firm operations asked for `Account.Social_Security_Number__c` to become an encrypted field so staff only ever see the last four digits. The change was made in Setup the same day, and PR #698 recorded it in git on 2026-10-13.
- **DDT-1364, 2026-10-16.** A case manager reported that the SSN showed as `***-**-` followed by four digits. They need the full number for CMS tasks on PI matters. The ticket guessed the field was on Contact; it is the Account field rebuilt in DDT-1319, shown on Person Accounts.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Account.Social_Security_Number__c` | Custom field | Rebuilt as Text (Encrypted), length 11, SSN mask, asterisk mask character |
| `Person_Account_Lightning_Page` | Lightning record page | Field placed back on the page, shown only for PI clients |
| `View_Encrypted_Fields` | Permission set | Assigned to an approved group; one more user added in DDT-1364 |

**Out of scope**

- **Migrating existing SSN values.** The ticket's owner backed up the old values and re-entered them personally.
- **Any other field.** The ticket said not to change other fields.
- **Opening full-SSN access firm-wide.** DDT-1364 was scoped to one user.

## 4. How it works (background)

```
Account.Social_Security_Number__c  (Text (Encrypted), mask type SSN)
  user WITHOUT View Encrypted Data  -> sees ***-**-1234
  user WITH View Encrypted Data     -> sees the full number

View Encrypted Data is a user permission.
In this org it is granted through the View_Encrypted_Fields permission set only.
```

Field-level security still decides whether a user can see or edit the field at all. The encryption permission only decides whether they see it masked or in full.

## 5. Root cause

### DDT-1319

Not a defect. The field was plain text, so anyone with field access saw the full SSN. The firm wanted it masked.

**Why it had to be rebuilt.** Salesforce cannot convert an existing field to encrypted text in place. The ticket's plan was delete, erase from Deleted Fields (until it is erased the API name stays reserved), and recreate under the same API name.

### DDT-1364

Not a defect either. The mask was working as designed. The case manager did not hold `View Encrypted Fields`, and DDT-1319 had deliberately limited it to a small group pending further approval. They already had edit access to the field through their profile, so only the unmasking permission was missing.

## 6. Solution

### 6.1 DDT-1319: the encrypted field

Done in Setup on 2026-10-12 by the ticket's owner, so the planned build was not needed. The field as production has it, recorded in git by PR #698:

```xml
<CustomField>
    <fullName>Social_Security_Number__c</fullName>
    <label>Social Security Number</label>
    <length>11</length>
    <maskChar>asterisk</maskChar>
    <maskType>ssn</maskType>
    <required>false</required>
    <trackHistory>false</trackHistory>
    <type>EncryptedText</type>
</CustomField>
```

**The receipt was corrected once before merge.** The first copy of the Person Account page in the PR was retrieved before the Setup work was finished. It had the field in a different section with no visibility rule. Production shows the field only when `Client_Type__c` is `PI Matter`. A fresh retrieve was committed unchanged, and a check-only deploy reported `changed: False` for both the field and the page, which proves git now matches production.

Why that mattered even though nothing deploys from a receipt PR: a Lightning record page deploys as a full replace. Merging the stale copy would have meant the next deploy of that page quietly removed the PI-only rule.

### 6.2 DDT-1364: access for one more user

1. The support team asked the field's owner, in the ticket, whether to add the case manager to `View Encrypted Fields`. The owner approved on 2026-10-20, scoped to that one user.
2. The case manager was assigned the `View Encrypted Fields` permission set.
3. They were asked to open a PI client with an SSN, refresh, and confirm the full number shows.

## 7. Testing

- **DDT-1319:** check-only deploy of the field and page against production reported no changes, so git matches production. No Apex test applies.
- **DDT-1364:** the case manager confirmed on 2026-10-23: "I can now see it."
- **Not covered:** the ticket asked for a check while logged in as a PI user without the permission, to confirm the mask. Whether that check was done is not recorded in the ticket.

## 8. Step by step

**Rebuilding a field as encrypted**

1. Back up every existing value of the field.
2. Delete the field and remove it from layouts and pages.
3. Erase it in Setup → Object Manager → the object → Deleted Fields, so the API name is free again.
4. Create a Text (Encrypted) field with the same API name, the length and mask you need.
5. Grant field-level security through a permission set.
6. Put it back on the record page, with the same visibility rule it had.
7. Re-enter the data.
8. Retrieve the field and page from production and open a PR with them the same day.

**Giving one user the full value**

1. Get approval from the field's owner for that named user, recorded in the ticket.
2. Assign the user the `View_Encrypted_Fields` permission set.
3. Ask the user to refresh a record with a value and confirm they see it in full.

## 9. Rollback

- **DDT-1364:** remove the user's `View_Encrypted_Fields` assignment. The field goes back to masked for them.
- **DDT-1319:** not practical. Converting back means deleting the field again and re-entering every value. There is no reason to.

## 10. Known limits

- **Erasing a deleted field is permanent.** Back up first; the old values cannot be recovered from Salesforce afterwards.
- **The page rule is part of the control.** Anyone deploying `Person_Account_Lightning_Page` must start from production's current copy, or the PI-only rule can be lost.

## 11. Reuse for similar issues

**How to recognise one:** a user sees `***-**-` and four digits and needs the full value.

- **Masked is not the same as hidden.** If the user sees a mask, field-level security is fine; what is missing is the View Encrypted Data permission.
- **Ask the field's owner before granting**, and record the approval in the ticket.
- **Grant through a permission set**, scoped to the named user, never through a profile.
- **To make an existing field encrypted**, it has to be deleted, erased and recreated. Back up first.
- **A receipt PR must match production exactly.** Retrieve after the Setup work is finished, and prove it with a check-only deploy that reports no changes.
