# Medical provider contact info in department sections

| | |
|---|---|
| **Ticket** | DDT-993 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-30 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-09-10 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Account fields, Lightning page, layout, permission set, list view, anonymous Apex), Dochly |
| **Area** | Medical providers: where records, billing and reductions letters get their fax, email and address |
| **Related** | DDT-1151 (separate ZIP-code bug in Dochly damages letters), DDT-1413 (Dochly custom input fields) |
| **Code** | PR #532, PR #558 and PR #567 in the `ddt-salesforce` repo (PR #528 was an earlier attempt, superseded by #532) |

---

## 1. Summary

Medical providers often use different fax numbers, emails and mailing addresses for their Records, Billing and Reductions departments, but the provider record only had one generic set. Staff asked for department sections so Dochly letters could reach the right desk. We added Records, Billing and Reductions sections (fax, email, phone, mailing address each) to medical provider accounts, copied existing generic values into Records, and gave the team a worklist of providers with no records contact at all.

Measuring first changed the priority: most providers had **no fax on file at all**, so the bigger job was collecting data, not adding fields. Repointing the Dochly letters was left for a later ticket, once the new fields hold data.

## 2. What was reported

Dochly letters (record requests, billing requests, reduction letters) often went to the wrong department, and staff had to fix the destination on letters by hand. The request was to split the provider's contact details by department and have each letter type pull from its own department.

After the first deploy (2026-09-29), the requester opened a provider and **saw none of the new sections**. That led to the second PR. On 2026-09-30 the requester also asked for four client-only fields to be taken off the provider page, which became the third PR.

## 3. Scope

**In scope**

| Component | Type | PR | Change |
|---|---|---|---|
| `Account.Records_Fax__c`, `Records_Email__c`, `Records_Phone__c`, `Records_Mailing_Address__c` | Custom fields | #532 | New |
| `Account.Billing_Phone__c`, `Billing_Mailing_Address__c` | Custom fields | #532 | New |
| `Account.Reductions_Fax__c`, `Reductions_Email__c`, `Reductions_Phone__c`, `Reductions_Mailing_Address__c` | Custom fields | #532 | New |
| `Account.Billing_Fax__c`, `Billing_Email__c` | Custom fields | #532 | Existing; reused as the Billing department's fax and email |
| `Provider_Department_Contacts` | Permission set | #532 | New: read and edit on all twelve department fields |
| `Providers_Missing_Records_Contact` | Account list view | #532 | New: the team's worklist |
| `Account-Treatment Providers Layout` | Page layout | #532 | Three department sections |
| Permission-set assignment and Records back-fill | Anonymous Apex scripts | #532 | One-off, run after deploy |
| `Client_Record_Page7` | Lightning record page | #558, #567 | Department sections; four client-only fields hidden for providers |

**Out of scope**

- **Repointing the Dochly templates.** Deliberately left until the new fields hold data. Pointing a letter at an empty field is worse than pointing it at a generic one: the letter goes out blank instead of to the wrong department. The requester opened a separate ticket for it.
- **The proof letters** (three letters for a fully filled provider, three for one with only generic fields) belong with the template work, because until then no letter reads the new fields.
- **The "Client Layout"** assigned to medical providers on 20 profiles was not updated.
- **A separate Dochly bug**, where damages letters merge the state code where the ZIP belongs: DDT-1151.

## 4. How it works (background)

- **Medical providers are `Account` records with the `Medical_Providers` record type**, not a custom object. There were 466 active on 2026-09-25.
- **Dochly letters did not choose a department, right or wrong.** Reading the merge tokens in the 13 active templates on 2026-09-29:

| Templates | Address merged from | Fax / email |
|---|---|---|
| 6 on `Damages__c` | Provider's Billing address | Typed by the user at generation |
| 5 on `Record_Request__c` | Provider's Shipping address | 2 merge the provider's fax and email; 3 use a typed prompt |
| `Reduction` on `Lien__c` | Provider's Billing address | Typed by the user |
| `PI Release of Lien` on `Lien__c` | None | None |

- **Billing and Shipping were the same address on every provider** (checked across all 468 on 2026-09-29). So every letter type went to one address, and the fax was mostly whatever someone typed.
- **The provider Account is the only source in use.** Of 875 medical damages rows, none used the alternative recipient lookups, so the fields went on Account rather than on a Contact or another object.
- **Which page providers see.** In the DDT Law Group app, medical providers open on `Client_Record_Page7`, a Dynamic Forms page that places its own field sections. It does not read the page layout. The same page also serves Business Accounts.

## 5. Root cause

The ticket framed this as Dochly "pulling the wrong destination". Measured before building, the main problem was missing data. Populated across 466 medical provider accounts on 2026-09-25:

| Field | Populated | % |
|---|---|---|
| `BillingStreet` | 318 | 68% |
| `Phone` | 269 | 58% |
| `Fax` | **64** | **14%** |
| `Website` | 48 | 10% |
| `Billing_Email__c` | 11 | 2% |
| `Billing_Fax__c` | 1 | 0.2% |

For 86% of providers there was no fax to pick, right or wrong. Splitting one empty field into three would produce three blanks. So the order of work became: small schema, back-fill what can be derived, hand the gaps to the team as a counted list, and change the letters last.

**Why the sections did not show after the first deploy.** PR #532 edited the page layout, on the assumption (taken from a page's name, never read from the app) that providers saw the layout. In production the app assigns them `Client_Record_Page7`, a Dynamic Forms page on 14 profiles, which ignores the layout. PR #558 added the sections to that page.

## 6. Solution

### 6.1 Fields and permission set (PR #532)

- **Ten new fields**: fax, email, phone and mailing address per department, minus the two Billing fields that already existed. Named to match the existing `Billing_*` pattern. The address fields are labelled "… Mailing Address" because Account already uses the label "Billing Address".
- **`Provider_Department_Contacts`** grants read and edit on all twelve, including the two existing Billing fields, so the Billing section is never half-visible. It is additive: new fields are invisible until granted, so FLS ships in a permission set assigned after deploy.
- The assignment script also covers the admin profile clones, so no admin sees three empty sections.
- **Account custom field count:** 234 of 800 before. Custom address fields are compound, so the after-count is measured, not added up.

### 6.2 Back-fill into Records (PR #532)

The single fax or email on file today is overwhelmingly the records fax, so existing generic values (`Fax`, `Client_Email__c`, `Phone`, Billing address) were copied into the **Records** department only. The script:

- touches only Records fields, only where blank, only on providers with something to copy;
- treats the Records address as blank only when **every** part is blank, so a partly hand-entered address is never overwritten;
- names any failed row from its input list, because a failed save result carries no Id;
- stops if the provider query hits its limit, instead of back-filling a subset;
- has a `PREVIEW` mode (no writes) and a `COMMIT` mode. There is no rollback-style dry run, because after-save automation runs before a rollback would.

Side effect checked beforehand: saving a provider makes the `Client_Address_Update` flow rewrite the address on each of its treatment records from the provider's Shipping address. 525 treatment records were affected, none with a different address, so no value changed.

### 6.3 The worklist (PR #532)

List view **"Providers Missing Records Contact"**: medical providers with neither a records fax nor a records email, so no way to receive a records request. A count somebody can finish is a task; "update provider info going forward" is not.

### 6.4 Sections on the page providers actually use (PR #558)

Three field sections (Records, Billing, Reductions Department) after Address Information on `Client_Record_Page7`, visible only when `Record.RecordType.Name = Medical Providers`.

`Billing_Fax__c` and `Billing_Email__c` appear **twice**: in Billing Department for providers, and still in Contact Information with a "not Medical Providers" rule. The page also serves Business Accounts, and moving the fields outright would have hidden them there (the PR review caught that). Salesforce accepts the same field twice when each copy has its own identifier.

The layout change from PR #532 stays: it is what any context without this page falls back to.

### 6.5 Four client-only fields hidden for providers (PR #567)

Country of Origin, Deceased, Client Representative and "How did you hear about us? Historial" sit in a section that also renders for Business Accounts. So they were **hidden, not removed**: each got a "not Medical Providers" visibility rule. No provider held a value in any of the four; 2 Business Accounts held one, which is why removing them would have been wrong.

## 7. Testing

- **Validated against production (check-only)** for each PR: 13 of 13 components and 69 of 69 tests for #532; 1 of 1 component and 69 of 69 tests for #558 and #567 (`DamagesTableControllerTest`). This proves the metadata deploys. It does not render the page, exercise FLS or run the scripts.
- **No sandbox test.** The sandbox rejects custom Address fields and lacks a field production's layout already uses. The back-fill was proven by its `PREVIEW` run in production, which writes nothing.
- **Fill rate after the back-fill** (2026-09-29, of 471 providers): Records fax 64, email 66, phone 274, address 318. Billing fax 2, email 12. Reductions 0. The list view held **368** providers with no records fax and no records email.
- **Live check:** the requester opened a provider after PR #558 and confirmed the three sections, then confirmed the hidden fields on 2026-09-30 ("All good for this one").

**Not covered:** no Dochly letter reads the new fields yet, so letter output was not tested.

## 8. Step by step

1. **Measure first.** Count how many providers hold each contact field (section 5). That decides whether the job is schema or data.
2. **Confirm what's already there.** `Billing_Fax__c` and `Billing_Email__c` existed, so reuse them and match their naming.
3. **Check the Account field count** before adding fields, and record it.
4. **Find the page providers actually see.** Read the app's record-page assignments for the `Medical_Providers` record type and profile. Don't infer it from a page's name. If it is a Dynamic Forms page, edit the page, not the layout.
5. **Retrieve the current production copy** of every page and layout you edit and make the edit on top of it.
6. **Create the fields, the permission set and the list view**, edit the page and layout, and validate against production.
7. **Get the PR approved and deploy.**
8. **Post-deploy, in order:** record the field count; run the permission-set assignment script; run the back-fill with `MODE = 'PREVIEW'` and read its output; run it with `MODE = 'COMMIT'`; re-run `PREVIEW` to get the after fill rate and post it on the ticket.
9. **Hand the list view to the team** with its count.
10. **Open a provider as a non-admin** holding the permission set and confirm the sections render and save. Open a Business Account and confirm nothing changed there.
11. **Later, in a separate ticket:** repoint the Dochly templates to the department fields once the fill rate is high enough. Templates are records edited in the Dochly screen, not metadata, so list each one changed.

## 9. Rollback

- **Page changes:** redeploy the previous `Client_Record_Page7` (retrieved from production just before each deploy). Safe; no data depends on it.
- **Fields:** they can be hidden by removing them from the page and layout. Deleting them would discard the data the team has entered since.
- **Back-fill:** it only filled blanks with copies of existing generic values, and the generic fields were not changed, so nothing was lost.

## 10. Known limits

- **Letters don't use the new fields yet.** Until the Dochly templates are repointed, letters still use the old fields or a typed fax.
- **Most providers still need data.** The worklist started at 368 providers with no records fax or email.
- **The visibility rules compare the record type label** (`Medical Providers`), as the page's other rules do. Renaming that record type in Setup would silently hide the department sections and un-hide the four client fields.
- **The duplicated Billing fields** may trip App Builder the next time someone edits the page there.
- **20 profiles use "Client Layout"** for providers and were not updated.

## 11. Reuse for similar issues

**How to recognise one:** "the letter goes to the wrong place" or "the wrong value is pulled", with a request to add more fields.

- **Measure fill rates before adding fields.** If the existing field is mostly empty, more fields won't help; the work is data collection.
- **Read the templates' merge tokens** to see what they actually pull. Here they didn't pick a department at all.
- **Give the team a counted list**, not a vague instruction.
- **Find the real record page** before editing a layout. A Dynamic Forms Lightning page ignores the layout entirely.
- **Check who else shares the page.** Use visibility rules to change one record type without hiding fields from another.
- **Hide, don't delete, when another record type still holds values.**
- **Back-fill scripts:** fill blanks only, preview first, stop rather than process a partial set, and report every failure.
