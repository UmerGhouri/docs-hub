# Creating a vendor from the expense lookup

| | |
|---|---|
| **Ticket** | DDT-672, DDT-989 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-10 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-08-07 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC, Lightning page, lookup filter) |
| **Area** | Vendors: creating vendor Accounts, and picking them on expenses and liens |
| **Related** | DDT-690 (Subrogation Vendor field on Lien) |
| **Code** | PR #126 (DDT-672) and PR #345 (DDT-989) in the ddt-salesforce repo |

---

## 1. Summary

Two related vendor problems, fixed separately.

- **DDT-672:** the Vendor field on the Expense form could only search existing Accounts, so staff were stuck when a vendor was not in Salesforce yet. A **+ New vendor** action now creates the vendor Account inside the form and selects it.
- **DDT-989:** the standard "New Client: Vendor" create screen had no email box, and the Lien's Subrogation Vendor lookup searched every Account. The email box was missing because the page showed the person-account email field, which never renders on a business account; a vendor email field was added for the Vendor record type. The lookup now only returns Vendor accounts.

## 2. What was reported

- **DDT-672 (2026-08-07):** the PI team, while recording an expense, found the Vendor field "only letting us search", with no way to add a new vendor.
- **DDT-989 (2026-09-10):** found while working on the Lien Subrogation Vendor field. The "New Client: Vendor" screen had a **No Email** checkbox but no email input, and the Subrogation Vendor lookup returned clients, carriers, providers and vendors together.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `expenseManager` | LWC | DDT-672: **+ New vendor** action and New Vendor modal on the create-expense form |
| `Person_Account_Lightning_Page` | Lightning page | DDT-989: `Client_Email__c` added after `PersonEmail` in "Account Information", visible only for the Vendor record type |
| `Lien__c.Subrogation_Vendor__c` | Custom field | DDT-989: required lookup filter, Account record type must be `Vendor` |

**Out of scope**

- **The edit-expense form.** Its Vendor lookup is unchanged; preselecting a new vendor there would have needed more server data.
- **An inline "+ New Vendor" on the Subrogation Vendor lookup.** Considered for DDT-989 and held back because of the risk of duplicate vendor accounts. It is a scoping decision, not a platform limit; the DDT-672 pattern could be reused.
- **Email on other business record types** (for example Repair Facility). Only Vendor was asked for.

## 4. How it works (background)

- Vendors are **Accounts**. `Expense__c.Vendor__c` and `Lien__c.Subrogation_Vendor__c` are lookups to Account.
- `Expense__c.Vendor__c` has a lookup filter allowing the `Vendor` or `Medical_Providers` record types. Existing expense vendors split roughly half and half between them.
- The Expense form is a custom LWC, `expenseManager`. It renders Vendor with `lightning-input-field`, which only searches; it does not offer the "+ New Account" option that standard record-page lookups have.
- Accounts mix **person accounts** (clients) and **business accounts** (vendors, carriers, providers). `PersonEmail` exists only on person accounts. Salesforce silently hides a person-account field on a business account.

## 5. Root cause

### DDT-672

Not a fault: `lightning-input-field` lookups have no create option, so this needed a change to the component rather than a setting.

### DDT-989: no email box

Three plausible causes were checked against production and ruled out first:

- **Not field access.** `Client_Email__c` is readable and editable on every profile and permission set (40 field-permission rows checked).
- **Not the page layout.** `Account-Vendor Layout` already carried `Client_Email__c`; the create screen does not use it.
- **Not a missing field.** `Client_Email__c` exists and is populated on 21,682 of 34,940 accounts.

**The real cause:** the create screen is driven by Dynamic Forms on `Person_Account_Lightning_Page`. Its "Account Information" section held fields in exactly the order on the reported screen, which is how the page was identified. The email field there was `PersonEmail`, and every Vendor account has `IsPersonAccount = false`, so the field never rendered and left **No Email** beside nothing.

### DDT-989: unfiltered lookup

`Lien__c.Subrogation_Vendor__c` had no lookup filter, so it searched every Account.

## 6. Solution

### 6.1 DDT-672: + New vendor on the Expense form

- A **+ New vendor** action opens a **New Vendor** modal: a **Vendor type** choice (Medical Providers or Vendor, default Medical Providers), **Name** (required) and **Phone**.
- It saves through `lightning-record-edit-form`, so Account duplicate rules and field access still apply.
- On success, the new Account is selected into the Vendor field.
- Record type Ids are looked up at run time with `getObjectInfo` by record type name, never hardcoded, because Ids differ between orgs.
- No Apex.
- The first commit only reformatted the component to the repo's formatter, so the functional change could be reviewed on its own.

### 6.2 DDT-989: vendor email on the create screen

`Client_Email__c` added straight after `PersonEmail`, with component visibility `{!Record.RecordType.DeveloperName}` equal to `Vendor`. Without the rule, every client record would show two email boxes with different values.

The rule keys on record type, not `IsPersonAccount`: no page in the repo used `IsPersonAccount` in a visibility rule, while record-type rules were common, and the record type is chosen before the create screen opens, so it is known on a record that does not exist yet.

### 6.3 DDT-989: Vendor-only Subrogation Vendor lookup

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

A required filter blocks saving any record that breaks it, so existing data was checked: exactly one Lien had the field filled in, and its account was already a Vendor.

## 7. Testing

- **DDT-672:** no automated test is recorded for the modal. The ticket's closing comment describes the live feature on Financials → Expenses → New Expense.
- **DDT-989:** the page could not be tested in the sandbox, because it holds a component the sandbox lacks. The deploy doc therefore lists screen checks in production after deploy:
  1. New Client → **Vendor**: the email box shows above No Email and saves to `Client_Email__c`. This gates sign-off; if the box is missing, nothing else will have gone red.
  2. Any client record shows **one** email field; an existing Vendor record shows the vendor email.
  3. Searching a client name in Subrogation Vendor returns nothing; a known vendor is returned.
  4. The one existing Lien with a subrogation vendor saves unchanged.

  The results of these checks are not recorded in the ticket.

**Not covered:** neither change has a test that renders the screen. A green validation proves the metadata deploys, not that the field appears.

## 8. Step by step

**Add inline create to a custom LWC lookup**

1. Add a "+ New" button next to the `lightning-input-field`.
2. Open a modal with a `lightning-record-edit-form` on the target object and only the fields staff need.
3. If the lookup allows several record types, offer a choice, resolved by name with `getObjectInfo`.
4. In `onsuccess`, put `event.detail.id` into the value the lookup is bound to, and close the modal.

**Fix a field that "isn't on the create screen"**

1. Rule out field access, the page layout and a missing field, against production.
2. Find which Lightning page drives the screen by matching the field order you see.
3. Check whether the field shown is a person-account field on a business record type.
4. Retrieve production's current page, add the right field with a visibility rule, deploy, then open the create screen to confirm.

**Restrict a lookup to one record type**

1. Count existing records whose value would break the filter.
2. Add a required lookup filter on `Account.RecordType.DeveloperName`.
3. Search a non-matching name in the lookup and confirm it returns nothing.

## 9. Rollback

- **DDT-672:** redeploy the previous `expenseManager`. Vendors already created stay as ordinary Accounts.
- **DDT-989:** remove the field block from the page (from production's current copy) and remove the lookup filter. Neither touches data.

## 10. Known limits

- **Duplicate vendors.** Inline create makes it easy to add a vendor that already exists under a slightly different name. Duplicate rules are the only guard.
- **Other business record types still have no email box** on the create screen. Add their record types to the same visibility rule if asked; do not switch to `IsPersonAccount` without opening the screen first.
- **The Subrogation Vendor filter is narrower than the Expense one** (Vendor only, not Medical Providers). Backfilling older lien values may hit the filter where the vendor account does not exist or has another record type; DDT-690 tracks that.
- **Page changes must start from production's copy.** During DDT-989 a retrieve pinned to an older API version silently dropped a page property, which made it look as though production lacked it. Retrieve at the project's API version or higher.

## 11. Reuse for similar issues

**How to recognise one:** "it only lets me search", or "the field is missing from the New screen".

- **`lightning-input-field` lookups cannot create records.** Inline create needs a small modal in the component.
- **When a field is missing from a screen, rule out the three usual causes against production** (access, layout, field) before changing anything, and identify the actual page by field order.
- **Person-account fields vanish silently on business accounts.** Use a business-account field and gate it by record type.
- **Before adding a required lookup filter, count existing records that would break it.**
- **Gate a create-screen visibility rule on something the new record already has**, such as its record type.
