# Lien holder converted from text to an Account lookup

| | |
|---|---|
| **Ticket** | DDT-690 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-14 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-08-10 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (custom field, LWC, Lightning page, permission sets) |
| **Area** | PI liens: recording who holds a lien and who is collecting it |
| **Related** | DDT-989 (vendor-only filter and email on the vendor create form) |
| **Code** | PR #309 in the ddt-salesforce repo (follow-up: PR #345) |

---

## 1. Summary

Staff asked for the free-text **Lien Holder (Other)** box on the Lien record (`Lien_Holder_Other__c`) to become a lookup, so details are picked from existing records instead of retyped. That exact change was declined: Salesforce cannot convert a Text field into a Lookup, and the Lien already has an Account lookup for the lien holder (`Lienholder__c`).

Looking at the data showed the real problem. Every value in the text box was a **subrogation recovery vendor** (the company collecting for the carrier), not a lien holder. So a new lookup, **Subrogation Vendor** (`Subrogation_Vendor__c`), was added to hold the vendor as a real Account, next to the existing Lienholder lookup.

## 2. What was reported

- **2026-08-10:** staff working liens reported retyping provider and lien holder details on every Lien. They asked for `Lien_Holder_Other__c` to become a lookup, and for the existing lookups to search more record types. The goal was cleaner data and better merge data for letter generation.
- The request was first scoped down to one change: convert `Lien_Holder_Other__c` from Text(255) to Lookup(Account).
- **2026-08-27:** the scoped change was declined after checking production. The text field was populated on 10 of 155 liens at that time, and 9 of those 10 also had `Lienholder__c` set. The text value on those 9 was the vendor, not the holder.
- **2026-09-04:** recounted for the build: 11 of 168 liens had the text box filled, and `Lienholder__c` was populated on 161 of 168.

## 3. Scope

**In scope (PR #309)**

| Component | Type | Change |
|---|---|---|
| `Lien__c.Subrogation_Vendor__c` | Custom field | New: Lookup(Account), delete constraint SetNull, field history tracked |
| `lienManager` | LWC | The field added to both the New and Edit modals |
| `Lien_Record_Page` | Lightning record page | The field, directly after Lienholder |
| `Lien__c-Lien Layout` | Page layout | The field, directly under Lienholder |
| `Damages_Records_Mode` | Permission set | Read and edit on the new field |
| `PI_Matter_Access` | Permission set | Read on the new field |

**Follow-up (PR #345, DDT-989):** a lookup filter limiting `Subrogation_Vendor__c` to Accounts of the `Vendor` record type, and an email field on the "New Client: Vendor" create form.

**Out of scope**

- **Converting `Lien_Holder_Other__c`.** Not possible in Salesforce, and it would have duplicated `Lienholder__c`.
- **Expanding the search scope of `Lienholder__c` and `Damages__c`.** The original request included this. It was not built here.
- **Migrating the existing text values and deleting `Lien_Holder_Other__c`.** Planned as part 2. It was blocked on data questions about a few liens (see section 10). As of this writing the old field is still in the repo and still referenced by Apex and the `lienManager` LWC.
- **An inline "+ New" option on the vendor lookup.** Left out on purpose so staff don't create duplicate vendor Accounts. New vendors are created from the Client tab.

## 4. How it works (background)

A Lien (`Lien__c`) had two places to name who it was owed to:

```
Lien__c
  Lienholder__c          Lookup(Account)  -> the carrier or plan that holds the lien (Medicaid, Aetna ...)
  Lien_Holder_Other__c   Text(255)        -> meant for non-Account lien holders
```

The settlement statement and the lien manager both show the lien holder by reading `Lienholder__r.Name` first. They only fall through to `Lien_Holder_Other__c` when the lookup is empty.

Staff edit liens in the `lienManager` component (New and Edit modals). The Lien record itself opens on the `Lien_Record_Page` Lightning page, which lists its fields explicitly and does not render the classic page layout.

## 5. Root cause

The request was a symptom of **one box being used for two different things**.

1. The text field was labelled and described as a lien holder field.
2. In practice, every value staff typed into it was the **subrogation vendor**: the company collecting on behalf of the holder. Examples from the deploy doc: Katch collecting for Medicaid and Medica, Machinify for Aetna, MedSpeed for a third-party administrator.
3. Because the display logic reads `Lienholder__c` first, the vendor name was never shown on most liens. On 9 of the 11, the text was data that nothing displayed. On the 2 liens with an empty `Lienholder__c`, the vendor name was the only name there was.

**Where the first diagnosis was wrong.** An early plan said `Lienholder__c` had a record-type lookup filter, and that this was why vendors could not be picked in it. Its metadata has **no lookup filter at all**, so vendors were always selectable there. The problem was never a filter. Two different roles were sharing one field.

**Why "convert the field" was never an option.** Salesforce allows Text to change to picklist, number, email, phone, URL and similar types. Lookup is not one of them.

**Evidence.** Counts in this page come from the ticket comments and the deploy doc (`docs/deploys/ddt-690-subrogation-vendor.md`). A query of this shape reproduces them:

```sql
SELECT Lienholder__r.Name, Lien_Holder_Other__c
FROM Lien__c
WHERE Lien_Holder_Other__c != null
```

## 6. Solution

### 6.1 A new field for the vendor

```xml
<fullName>Subrogation_Vendor__c</fullName>
<label>Subrogation Vendor</label>
<type>Lookup</type>
<referenceTo>Account</referenceTo>
<deleteConstraint>SetNull</deleteConstraint>
<trackHistory>true</trackHistory>
<relationshipName>Liens_As_Subrogation_Vendor</relationshipName>
```

The field description says plainly that this is the vendor and **not** the lien holder. Record counts and Apex details are left out of the description so it does not go stale.

Why a new field instead of reusing `Lienholder__c`: the holder and the vendor are different parties on the same lien, and both matter. Keeping the vendor as a real Account means you can see every lien a vendor is collecting on, which helps when negotiating with them.

### 6.2 Put it where liens are actually edited

The first version added the field to the classic page layout only. A review found that this reaches nobody:

- `Lien__c` opens on the `Lien_Record_Page` Lightning page on desktop and mobile. That page lists 14 fields explicitly and has no record-detail component, so it never shows the page layout.
- The old text field was on neither the layout nor the Lightning page. Staff typed vendor names in the `lienManager` modals.

So the field went to all three places: both `lienManager` modals, the Lightning page (right after Lienholder) and the layout. The Lightning page was retrieved from production and edited on that copy, not taken from a sandbox.

### 6.3 Field-level security in permission sets

Read and edit through `Damages_Records_Mode`, read through `PI_Matter_Access`. This matches how `Lienholder__c` is granted in those two sets. No profile FLS was added.

### 6.4 Vendor-only filter (follow-up, DDT-989)

The field first shipped with no lookup filter. In PR #345 it was restricted to Accounts whose record type is `Vendor`:

```xml
<lookupFilter>
    <active>true</active>
    <filterItems>
        <field>Account.RecordType.DeveloperName</field>
        <operation>equals</operation>
        <value>Vendor</value>
    </filterItems>
    <isOptional>false</isOptional>
</lookupFilter>
```

It was safe to add because only one lien had the field filled at that point, and that Account was already a Vendor. The same PR added an email field to the vendor create form, which staff had asked for.

### 6.5 Vendor Accounts

None of the vendor Accounts existed when the field went live. Staff created vendor Accounts from the **Client** tab with the **Vendor** record type. Two of the vendors had no Account yet when the follow-up was announced on 2026-09-10.

## 7. Testing

- **Jest:** a new `lienManager` test asserts that the vendor lookup renders in the New Lien modal, after the lien holder lookup. It was run against the template before the change and fails there. `lienManager` had 6 passing tests after the change.
- **Validation against production:** the six components validated with 0 errors. No Apex was in the deploy, so no coverage bar applied.
- **Live check:** the field was confirmed deployed and working in production on 2026-09-08. A staff user found the field on the Lien and added the first two vendor Accounts.

**Not covered:** the data migration of existing text values and the removal of the old field. Neither was done in this ticket.

## 8. Step by step

1. Check what the free-text field really holds before changing it. Query the populated rows next to `Lienholder__c`.
2. Create `Lien__c.Subrogation_Vendor__c` as Lookup(Account), SetNull, with history tracking.
3. Add read and edit FLS in `Damages_Records_Mode`, and read in `PI_Matter_Access`. Deploy the permission sets with the field so it is not invisible.
4. Retrieve `Lien_Record_Page` from production. Add the field after `Lienholder__c` on that copy.
5. Add the field to both modals in `lwc/lienManager/lienManager.html`, after the lien holder input. Add the jest test.
6. Add the field to `Lien__c-Lien Layout` under Lienholder.
7. Validate against production, deploy, and confirm as a non-admin staff user that the field shows and can be edited.
8. Create vendor Accounts from the Client tab with the **Vendor** record type.
9. (DDT-989) Add the Vendor-only lookup filter once every populated value points at a Vendor Account.

## 9. Rollback

Remove the field from both permission sets, the layout, the Lightning page and the `lienManager` modals, then delete the field. Any vendor values entered after go-live would be lost on delete. Export them first.

## 10. Known limits

- **The old field is still there.** `Lien_Holder_Other__c` still exists and is still read as a fallback by the lien display logic. Removing it touches several Apex classes, the `lienManager` LWC, an object translation and about 50 profiles, and the settlement statement output, so it needs its own PR and review.
- **Part 2 was blocked on three data questions:** two liens with an empty `Lienholder__c` where the text box is the only information, and one where it is unclear whether a managed-care plan is the vendor or the real holder. These need an answer from the case team, not a guess.
- **Profile FLS gap.** `Lienholder__c` is also readable on one profile that does not grant the new field. A user on that profile who holds neither permission set sees Lienholder but not Subrogation Vendor.
- **Edit depends on the profile.** `Damages_Records_Mode` has no `Lien__c` object permission, so edit on the field relies on object access from the profile.
- **The filter only allows `Vendor` Accounts.** A vendor saved under another record type will not show up in the lookup.

## 11. Reuse for similar issues

- **When someone asks to convert a field, look at what it holds first.** Group the populated values next to the related fields. Here, the data showed a new field was needed, not a converted one.
- **Text cannot become Lookup.** Plan a new field, a migration and a later delete.
- **Check every surface a field is edited from.** A record page built from explicit field sections ignores the page layout. A custom LWC modal needs its own change. A field on the layout only can deploy green and still be invisible.
- **Add a test that fails without the change**, so a field that lands on no usable surface cannot pass silently.
- **Ship FLS with the field**, in permission sets, in the same deploy.
- **Add restrictive lookup filters only once the existing data fits them.**
