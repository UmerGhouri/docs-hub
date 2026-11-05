# Calendly scheduling host and meeting options

| | |
|---|---|
| **Ticket** | DDT-735, DDT-1309 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-17 (DDT-735); DDT-1309 live since 2026-10-14 |
| **Priority** | Medium |
| **Reported by** | Firm operations, 2026-08-13 (DDT-735) and 2026-10-12 (DDT-1309) |
| **Resolved by** | Support team |
| **Systems** | Salesforce (screen flow), Calendly |
| **Area** | The Schedule Calendly Meeting action on a hired Court Case (Opportunity) |
| **Related** | None recorded |
| **Code** | PR #184 (DDT-735) and PR #725 (DDT-1309) in the ddt-salesforce repo |

---

## 1. Summary

Both tickets change the same screen flow, `Calendly_Court_Case_Schedule_Calendly_Meeting_v3`, which shows the Calendly booking buttons on a Court Case. DDT-735 was a bug: for about 111 hired cases in six months, no onboarding call (Welcome or Introduction) appeared, because a flow formula compared the case type against picklist labels instead of the stored API names. DDT-1309 was a small feature: a fourth Host option, Lead of Collections, with its own payment-support booking button.

## 2. What was reported

- **DDT-735, 2026-08-13.** On a hired Traffic Misdemeanor case with Host set to Case Manager, the widget showed only "Case Discussion (Phone Call)" and "Case Conference". There was no Welcome Call option. The investigation found the report understated it: the **Introduction Call** tile was missing too, and the same thing affected about 111 hired cases over the previous 180 days.
- **DDT-1309, 2026-10-12.** The Host dropdown offered Case Manager, Primary Attorney and Other. Collections calls with the firm's Lead of Collections went through Other, with the booking link pasted by hand. The request was a first-class Host option for them. Asked where it should show, the requester said ideally on cases with a balance due, but showing it on all cases was the simplest.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Calendly_Court_Case_Schedule_Calendly_Meeting_v3` | Screen flow | DDT-735: formulas `CaseTypeCallType` and `CourthouseCallType` corrected |
| `Calendly_Court_Case_Schedule_Calendly_Meeting_v3` | Screen flow | DDT-1309: `Lead_of_Collections` choice added to `Host`, a payment-support URL formula, and a payment-support screen button |
| `docs/deploys/ddt-735.md` and the DDT-1309 review doc in `docs/deploys/` | Review docs | |

**Out of scope**

- **The `Case_Type__c` picklist.** Renaming the two API values to match their labels was considered and rejected: it would have rewritten 4,757 Opportunity records and broken 15 other components (42 references) that correctly match on the API names today.
- **Which call Traffic Misdemeanor gets.** It maps to Introduction Call, not Welcome Call; DDT-735 did not change that (see 10).
- **A pre-existing bug on the Host = Other path** where both onboarding tiles can show at once. Found during DDT-735's audit and filed as a follow-up.
- **DDT-1309:** showing the option only on cases with a balance due. It shows on all hired cases.

## 4. How it works (background)

```
Court Case -> Schedule Calendly Meeting (quick action)
  -> screen flow Calendly_Court_Case_Schedule_Calendly_Meeting_v3
  -> user picks a Host: Case Manager / Primary Attorney / Other (+ Lead of Collections after DDT-1309)
  -> formulas decide which booking buttons show:
       CaseTypeCallType    CASE(Case_Type__c, ...)            -> "Welcome", "Intro" or ""
       CourthouseCallType  CASE(courthouse, ...)              -> used for Aggravated Speeding
  -> DisplayWelcomeCall shows when the result is "Welcome"; DisplayIntroductionCall when "Intro"
  -> each button opens the host's Calendly event, prefilled with the case Id, client name, email and phone
```

The onboarding tiles share this visibility logic:

```
(1 AND 4 AND 5) OR (2 AND 3 AND 4) OR (1 AND 6) OR (2 AND 3 AND 6)

1 = Host = Case Manager
2 = a user was picked on the Host = Other path
3 = that user is a case manager
4 = CourthouseCallType = Intro / Welcome
5 = Case_Type__c = "Aggravated Speeding"
6 = CaseTypeCallType = Intro / Welcome
```

## 5. Root cause

### DDT-735

`CaseTypeCallType` compared `Case_Type__c` against picklist **labels**, but flow formulas evaluate the stored **API name**:

| API name (stored) | Label (shown in Setup) | What the formula matched |
|---|---|---|
| `Misdemeanor` | Traffic Misdemeanor | `"Traffic Misdemeanor"` |
| `Felony` | Traffic Felony | `"Traffic Felony"` |

These are the only two `Case_Type__c` values whose label differs from the API name, and they were the only two formula entries that matched nothing. With no match, `CASE()` returned `""`, so neither onboarding tile rendered and no error was shown. That is why it went unnoticed for about 17 months.

The same two formulas had three more defects:

- `"Other,"` had a stray trailing comma, so the value `Other` never matched.
- Seven active case types were missing entirely (Investigation, SOS Formal Hearing, SOS Informal Hearing, SOS Out of State Hearing, Code Violation, Order of Protection, DWLS/R - Misd), most added to the picklist in March 2026 without the flow being updated.
- `CourthouseCallType` matched one courthouse by its label instead of its API name. This only affects Aggravated Speeding, the one case type routed by courthouse.

**Not a regression.** The two values' label/API split predates the flow; the formula was wrong the day it was written.

**Blast radius:** 124 Closed Won cases in the last 180 days had a case type the formula could not match, of which about 111 were hired (the rest were named DNH). About 5 Aggravated Speeding cases were also affected through `CourthouseCallType`. Counted with a Salesforce report and a SOQL aggregate, which agreed case type by case type.

### DDT-1309

Not applicable: a new option, not a defect.

## 6. Solution

### DDT-735 (PR #184)

`CaseTypeCallType`, with every existing row kept:

- `"Traffic Felony"` changed to `"Felony"`, `"Traffic Misdemeanor"` to `"Misdemeanor"`, and `"Other,"` to `"Other"`.
- The seven missing case types added, each mapped to `"Intro"`.
- `"Aggravated Speeding", ""` added as a deliberate pin, so that case type still falls through to courthouse routing. Without it, the new default would override courthouse routing for about 184 cases per 180 days.
- The default changed from `""` to `"Intro"`, so a future picklist value shows an onboarding call instead of nothing. This is safe here because case-type routing is meant to apply on both Host paths.

`CourthouseCallType`: one line, the courthouse label replaced with its API name. Its `""` default was **kept**. An earlier revision widened it to `"Intro"`, and the AI reviewer caught that this made condition 4 true for about 30 courthouses, firing the `(2 AND 3 AND 4)` group, which has no Aggravated Speeding guard. On the Host = Other path that would have shown Welcome and Introduction side by side for, for example, a DUI case. Keeping the default removed the cause, so no visibility rule was edited.

**A constraint for the next editor:** `Case_Type__c` is a picklist, and picklists are only legal in some formula functions. A first attempt using `ISBLANK()` and `=` was rejected with "Field recordId is a picklist field. Picklist fields are only supported in certain functions." Use `CASE()` or `ISPICKVAL()`.

**Deployed Active, on purpose.** The PR checklist says flows deploy as Draft and are activated deliberately. The reviewer asked for Active instead, so the deploy itself fixed production and no one could forget to activate it. This was recorded in the review doc as a deliberate exception.

### DDT-1309 (PR #725)

Additive only (37 lines added, 0 removed), built on production's active version 7, which was retrieved first and matched `main`:

- A `Lead_of_Collections` choice added to `Host`, between Primary Attorney and Other.
- A payment-support formula holding the full Calendly event URL plus the same prefill parameters the other buttons send.
- A payment-support screen button, shown only when Host is Lead of Collections.

Two choices worth knowing:

- **The URL is hardcoded in the formula,** not built from the host's Calendly link on their User record, because the request was for one specific event. If that event's slug changes, the formula must change.
- **The button is HTML, not an image.** The six existing buttons are PNG files that each needed a file upload shared in the org. An HTML button needs no upload and renders the same in every org.

## 7. Testing

**DDT-735.**

- Production's active version 3 was retrieved and compared with `main` before any edit: identical on both formulas and all visibility rules.
- Validated in the Partial sandbox as a new flow version, debugged per record, on both the Host = Case Manager and Host = Other paths. The key pair: the same case type at two courthouses gave opposite results, proving the Aggravated Speeding pin; and a DUI case at a courthouse outside the list showed Welcome alone on Host = Other, proving the courthouse default was safe.
- Production check-only validation of the single flow: succeeded, 1 of 1 components, no errors. No Apex, so no tests selected.
- **Not run:** Host = Primary Attorney.
- Live: "Fixed and live in production" was recorded on the ticket on 2026-08-18.

**DDT-1309.**

- Deployed to the sandbox and seen on screen: Lead of Collections appears in the Host dropdown and shows only the new button. The restyled button had not been looked at on screen when the review doc was written.
- Live: confirmed in production on 2026-10-14. At that point the Calendly page showed "This calendar is currently unavailable", because the host's calendar was disconnected from Calendly. The host reconnected it the same day.

## 8. Step by step

For a missing button (DDT-735 pattern):

1. **Retrieve the active flow version from production** and compare with `main` before editing.
2. **List the picklist's API names and labels** (Setup → Object Manager → Opportunity → Case Type, or the field metadata) and compare each formula entry against the **API name**.
3. **Fix the formula in place**, keeping every existing row, and add any case types that are missing.
4. **Check the visibility rules on both Host paths** before changing any formula default; a default that looks harmless on one path can fire an unguarded condition group on the other.
5. **Validate in the sandbox per record,** then validate check-only against production.
6. **Decide Draft or Active** and record the decision. If deployed as Draft, activate the new version in Flow Builder the same day.

To add a Host option (DDT-1309 pattern):

1. Retrieve production's active version first.
2. Add a choice to `Host`, a formula for the booking URL with the same prefill as the other buttons, and a button gated on the new Host value.
3. Deploy, then in Flow Builder activate the new latest version.
4. Open a hired Court Case → Schedule Calendly Meeting, choose the new host, and confirm the Calendly page opens with the client details prefilled and shows open slots.

## 9. Rollback

Setup → Flows → the Schedule Calendly Meeting flow → activate the previous version (version 3 for DDT-735, version 7 for DDT-1309). The old versions stay in the org, so this is immediate. No data is written by either change.

## 10. Known limits

- **Traffic Misdemeanor gets Introduction Call, not Welcome Call.** The ticket expected Welcome on the reported case. The original flow, the client onboarding tiers and a year of bookings all map it to Intro, so the review doc recommended accepting "an onboarding call appears" as the deliverable; changing it would be a one-line formula change with business sign-off.
- **Host = Other can show both onboarding tiles at once** when the courthouse says Welcome and the case type says Intro. Pre-existing, filed as a follow-up. Fixing it also unblocks mapping one unmapped courthouse for Aggravated Speeding.
- **A case manager whose User title is not exactly "Case Manager"** cannot be picked on the Host = Other path, because the flow filters on that title.
- **The flow hardcodes 11 case-manager and 11 primary-attorney slots;** a twelfth would silently fall through to "User not found".
- **The Lead of Collections button only works while that host's calendar is connected in Calendly.** No Salesforce change fixes a disconnected calendar; only the host can reconnect it (Calendly → Availability → Calendar settings → Reconnect).
- The Lead of Collections URL is hardcoded, and the shared phone prefill carries a pre-existing doubled country code that all buttons share.

## 11. Reuse for similar issues

**How to recognise one:** a flow screen element that silently never shows for certain values, with no error.

- In flow formulas and decisions, picklists compare by **API name**. This org is heavily relabelled, so the label in Setup is often not the stored value.
- `CASE()` with no match and a blank default fails silently; prefer a sensible default plus explicit pins for values that must fall through.
- When a picklist gets a new value, search flows for `CASE()` lists on that field.
- Before changing any default in a formula used by visibility rules, audit every condition group that reads it.
- For a booking link that fails after a correct deploy, check the vendor side (here, a disconnected Calendly calendar) before changing Salesforce.
