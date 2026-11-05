# Lead conversion fails: unable to find default record type

| | |
|---|---|
| **Ticket** | DDT-471, DDT-1003 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-15 |
| **Priority** | High |
| **Reported by** | Intake team, 2026-07-20 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow, Lead page layouts) |
| **Area** | Intake: converting a Lead into a potential client |
| **Related** | None recorded |
| **Code** | PR #375 and PR #383 in the `ddt-salesforce` repo |

---

## 1. Summary

Intake staff clicking **Convert** on a Lead got the error **"Unable to find default record type."** Both tickets guessed a missing default record type on the user's profile. That was wrong: the real cause was a hidden `Lead.Company` value written by the website integration, which makes Salesforce try to build a **business** Account that the intake profiles have no default record type for.

The fix is a before-save flow that blanks `Company` on new Leads when it holds the person's own name or a known placeholder, plus the removal of the standard **Convert** button from the three Lead layouts so intake always uses the triage-panel conversion.

## 2. What was reported

- **DDT-471 (2026-07-20).** An intake staff member could not convert an active Lead and got the error. The ticket was closed on 2026-07-21 by an admin converting the Lead by hand. No fix was made, so the cause survived.
- **DDT-1003 (2026-09-14).** A second intake staff member hit the identical error on another Lead. The Lead type was already set to Defense, so that was not the cause.

The deploy doc for PR #375 records that the problem affected **everyone on the intake profiles** (DDT Intake, DDT Manager User, DDT Case Management, DDT Intake Director), not one user. Admins were immune. At the time, **818 open Leads** carried a `Company` value and **none was a genuine business**: 410 held the client's own name and 404 held placeholders.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Lead_Clear_Person_Company` | Record-triggered flow (before-save, create only) | New. Blanks `Company` when it is the person's own name or a known placeholder (PR #375) |
| `scripts/ddt-1003-clear-person-company.py` | Python script | New. Read-only backlog report that writes apply and rollback CSVs (PR #375) |
| `Lead-Defense Layout`, `Lead-PI Layout`, `Lead-SOS Layout` | Page layouts | Standard `Convert` button removed from the action bar (PR #383) |
| `Lead_Clear_Person_Company` | Flow definition | Records the activation already live in production, at version 1 (PR #383) |

**Out of scope**

- **Giving the intake profiles a default business Account record type.** It was tested in a sandbox and rejected (see 6.3).
- **The website integration itself.** It writes `Company` through the API from outside this repo. The flow is the permanent safety net either way.

## 4. How it works (background)

```
Intake opens a Lead
  -> clicks the standard "Convert" button          (bypasses NewPotentialClientController)
  -> Salesforce checks Lead.Company
       blank      -> builds a Person Account        (works)
       non-blank  -> builds a business Account
                     -> needs the user's default business Account record type
                     -> intake profiles have none -> UNAVAILABLE_RECORDTYPE_EXCEPTION
```

The intended conversion paths are **New Potential Client** (Defense and SOS) and **Convert to Hired Matter** (PI) in the triage panel. The standard Convert button bypasses `NewPotentialClientController`, so even a successful conversion through it loses the referral attribution linking, Hire Status and the UTM mapping onto the Opportunity.

## 5. Root cause

1. The website integration writes the **client's own name**, or a placeholder such as `Company Placeholder`, `[none]` or `New Lead Company`, into `Lead.Company`. Nobody sees it, because `Company` is not on `Lead_3_sections`, the Dynamic Forms page that renders the Lead. Every Lead a person creates has `Company` blank.
2. Salesforce treats **any** non-blank `Company` as a business and converts the Lead into a business Account.
3. The intake profiles have **no Account record type on the profile at all**. Their business Account access comes only from the `Business_Account_Record_Type_Access` permission set, and a permission set has no `default` element. It can grant access to a record type but can never supply a default.
4. So conversion fails with `UNAVAILABLE_RECORDTYPE_EXCEPTION`. The Admin profile does carry the Account record types, which is why "ask an admin to convert it" always worked and why DDT-471 closed without a fix.

**The ticket's first guess was wrong.** Both tickets said the user's profile was missing a default record type. The reporter's defaults were all correct (Person Account, Contact, Opportunity, Lead, Task), and colleagues on the same profile converted Leads every day. The acceptance criterion "assign the correct default record type" would have changed nothing.

**Evidence.** Reproduced in a sandbox with a throwaway test that ran a real conversion under `System.runAs` an intake user:

| Case | Result |
|---|---|
| Intake user, `Company` populated | `UNAVAILABLE_RECORDTYPE_EXCEPTION: Unable to find default record type` |
| Intake user, `Company` blank | Success, Person Account |
| Admin, `Company` populated | Success, business Account |

## 6. Solution

### 6.1 Blank `Company` on new person Leads (PR #375)

`Lead_Clear_Person_Company` is a **before-save** flow on Lead, **create only**. Its entry formula:

```
AND(
  NOT(ISBLANK({!$Record.Company})),
  OR(
    LOWER(TRIM({!$Record.Company})) = LOWER(TRIM({!$Record.FirstName} & " " & {!$Record.LastName})),
    LOWER(TRIM({!$Record.Company})) = LOWER(TRIM({!$Record.LastName})),
    AND(
      NOT(ISBLANK({!$Record.FirstName})),
      LOWER(TRIM({!$Record.Company})) = LOWER(TRIM({!$Record.FirstName}))
    ),
    CONTAINS("|company placeholder|new lead company|[none]|none|n/a|na|test|testtest|this is a test|acme (sample)|-|.|",
             "|" & LOWER(TRIM({!$Record.Company})) & "|")
  )
)
```

One assignment sets `$Record.Company` to blank. The Lead then converts to a Person Account, which is what these records are.

Why it's done this way:

- **Before-save, and create only, is load-bearing.** `NewPotentialClientController` warns that any record-triggered Lead automation doing DML in the conversion transaction locks the record and makes `Database.convertLead` fail with `RECORD_IN_USE_BY_WORKFLOW`. A before-save flow does no DML, and limiting it to create means it can never fire during conversion.
- **Genuine company names are left alone.** A real business Lead must keep its `Company`, so it still converts to a business Account (by an admin).
- **The placeholder list is duplicated on purpose.** The flow holds it as a pipe-delimited string; the backlog script holds it as `PLACEHOLDERS`. Flow metadata cannot share a list with Python, so the two must be edited together, in the same order.
- **It shipped as Draft** and was activated in Setup after the deploy, as a deliberate step.

### 6.2 Remove the standard Convert button (PR #383)

Requested by firm management. `Convert` sat at `sortOrder 2` in the Record action list of all three Lead layouts. It was removed and the remaining actions renumbered; nothing else changed. This takes the button away from **admins too**, because Admin, DDT Intake and DDT Manager User all share the same three layouts. That was accepted deliberately rather than cloning three layouts for admins.

### 6.3 Rejected: a default business Account record type on the profiles

Tested in a sandbox and rejected for two reasons:

1. **The obvious edit silently does nothing.** Salesforce allows one default record type per object, and the Person Account record type already holds it. Adding a business default is discarded on deploy with no error; the deploy reports success and the conversion still fails.
2. **The version that does work produces bad data.** Moving the object default to the business record type clears the error by filing a person as a business Account. That swaps a loud failure for silent data corruption.

### 6.4 The backlog

`scripts/ddt-1003-clear-person-company.py` reports the open Leads with `Company` set and writes an update CSV and a backup CSV. It does not write to Salesforce. The load is a manual `sf data update bulk` step. Four odd values were left for a person to decide.

## 7. Testing

- **No Apex test.** The change is a flow and layouts only. PR #375 was validated against production with `RunLocalTests`, because the flow runs on every Lead insert.
- **Sandbox behaviour checks, as an intake user:** a full name and a first-name-only value were cleared; `Company Placeholder` and `[none]` saved blank and the Lead converted to a Person Account; a genuine company name was preserved and still raised the exception, as intended.
- **PR #383** was validated check-only against production (3 of 3 components changed, 1 test, 0 failures). The sandbox could not validate the layouts at all because it is missing a Lead field production has.

**Live check (2026-11-05):**

| Query | Result |
|---|---|
| `SELECT ApiName, IsActive FROM FlowDefinitionView WHERE ApiName = 'Lead_Clear_Person_Company'` | Active |
| `SELECT COUNT(Id) FROM Lead WHERE IsConverted = false AND Company != null AND CreatedDate > 2026-09-15T00:00:00Z` | **0**: no new Lead has arrived with `Company` set since the flow went live |
| `SELECT COUNT(Id) FROM Lead WHERE IsConverted = false AND Company != null` | **810**. The largest groups are still `Company Placeholder` (239), `[None]` (131) and `New Lead Company` (23), the same counts the deploy doc measured, so the backlog load does not appear to have been run |

**Not covered:** the backlog. Older open Leads that still carry a `Company` value will still fail the standard conversion; with the Convert button removed, they go through the triage panel instead.

## 8. Step by step

1. **Confirm the cause on the stuck Lead.** Check whether `Company` is populated (it is not on the page; query it).
2. **Unblock the user now.** Clear `Company` on that Lead, then convert it through **New Potential Client** (Defense/SOS) or **Convert to Hired Matter** (PI) in the triage panel.
3. **Create `Lead_Clear_Person_Company`** as in 6.1: before-save, create only, Lead, with the entry formula above and one assignment that blanks `Company`.
4. **Validate against production** with `RunLocalTests`, get the PR approved, deploy the flow as Draft.
5. **Activate the flow** in Setup, Flows.
6. **Report the backlog:** `python3 scripts/ddt-1003-clear-person-company.py --org <prod-alias>`. Review the update CSV.
7. **Load it** with `sf data update bulk --target-org <prod-alias> --sobject Lead --file <update-csv> --wait 10`. Keep the backup CSV.
8. **Remove `Convert`** from the Record action list of the three Lead layouts and renumber the rest.
9. **Check** with the live-check queries in section 7.

## 9. Rollback

- **Flow:** deactivate `Lead_Clear_Person_Company` in Setup.
- **Backlog data:** reload the backup CSV the script wrote next to the update CSV.
- **Layouts:** redeploy the previous three layout files, or re-add the `Convert` standard button at `sortOrder 2`. Nothing is deleted and no data moves, so this is a clean redeploy.

## 10. Known limits

- **The flow catches two known shapes only:** the person's own name and the listed placeholders. A new placeholder still saves with `Company` set and still fails the standard conversion. Add it to the flow's list **and** to `PLACEHOLDERS` in the script.
- **Admins lost the standard Convert button** as well, which was their way to rescue a stuck Lead. Admins can still convert through the triage panel or the API.
- **Non-admins still cannot convert a genuine business Lead** with the standard path. Revisit separately if that is ever needed.
- **The website integration still sends `Company`.** Stopping it there needs whoever owns the website.

## 11. Reuse for similar issues

**How to recognise one:** a conversion or create error that admins never see and every non-admin on a profile does, where the profile's own settings look correct.

- **Don't trust the error text's suggested fix.** "Unable to find default record type" pointed at profile defaults; the real trigger was a data value.
- **Reproduce under `System.runAs`** an affected user, varying one field at a time, before changing any profile.
- **Remember permission sets cannot set defaults.** A record type granted only by a permission set is available but never the default.
- **Look for hidden fields written by integrations.** A field that is not on the page can still change platform behaviour.
- **Clean data at the door** with a before-save, create-only flow when later steps (such as conversion) must not see the record locked.
- **Backlog scripts:** report first, write an update CSV and a backup CSV, and leave odd values for a person.
