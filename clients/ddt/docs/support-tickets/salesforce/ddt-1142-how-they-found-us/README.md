# How They Found Us panel, referral sources and referrers

| | |
|---|---|
| **Ticket** | DDT-1142, DDT-1255, DDT-1274, DDT-1285, DDT-1379, DDT-1384, DDT-1462 |
| **Project** | Support tickets |
| **Status** | Live; tickets closed between 2026-09-28 and 2026-11-05 (dates per ticket in section 2) |
| **Priority** | High (DDT-1142, DDT-1379); Medium (the rest) |
| **Reported by** | Intake team and firm staff, 2026-09-28 to 2026-10-27 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC, Lightning pages, global value set, record types, custom metadata) |
| **Area** | Intake attribution: recording how a client heard about the firm, and who referred them |
| **Related** | DDT-1296 (Apex class access for the panel), DDT-1178, DDT-1154, DDT-902, DDT-1372 |
| **Code** | PRs #526, #533, #643, #645, #669, #676, #679, #682, #728, #780, #791, #841 and #874 in the `ddt-salesforce` repo |

---

## 1. Summary

Seven tickets about the same piece of intake: the "How did you hear about us?" answer and the referrer behind it. They start with a broken "+ New outside attorney" button, then a new single **How They Found Us** panel that replaced two disconnected pieces of the client page, then a run of follow-ups as staff used it: new referral-source values, a save that an employee referral could never pass, an attorney the search could not find, and a Referring Attorney field that stayed blank.

Most fixes come back to one fact: the referral is stored on a referral row (`Inbound_Contact__c`), the how-heard answer is stored on the client and copied elsewhere, and every screen, search and record type that touches either has to agree on what an outside attorney or a referral looks like.

## 2. What was reported

### DDT-1142, 2026-09-28 (closed 2026-09-28): "+ New outside attorney" fails

On the Referral Source panel of a client record, choosing Outside Attorney, searching for the attorney (no results), then using "+ New outside attorney" with a first and last name returned "Could not create the referrer." Nothing was saved. An intake staffer could not record the referral for one client. Attorneys typed into the record page's **Referring Attorney** field could not be found by the panel's search either.

### DDT-1255, 2026-10-06 (closed 2026-10-07): add a referral source

Intake asked for a named outside law office to be added as a "How did you hear about us?" value on Lead, and everywhere else the field appears.

### DDT-1274, 2026-10-08 (closed 2026-11-02): require the Referring Attorney

Clients were being saved as "Referral: Outside Attorney" with the **Referring Attorney** field blank. The request was a validation rule to block that save.

### DDT-1285, 2026-10-09 (closed 2026-10-20): one How They Found Us panel

On the client record, "How did you hear about us?" was a loose field and the Referral Source panel sat on its own; neither read the other. The panel never locked, so one stray click could move referral credit, and it asked "who referred them?" of every client, although over the previous 12 months only 185 of 1,755 hired cases (10.5%) were referrals. On Leads from the Legal Growth Engine vendor (881 since 2026-03-02) it showed an empty "who referred them?" box for a paid lead.

### DDT-1379, 2026-10-20 (closed 2026-10-20): "Referral: Employee" cannot be saved

Choosing "Referral: Employee" on the new panel showed "An internal referral is credited to the firm. No referring party is needed." and then refused the save with "Pick who referred them, or choose 'They couldn't name anyone' and write what they said." No field was shown to satisfy it. No employee referral could be saved from the panel.

### DDT-1384, 2026-10-21 (closed 2026-10-21): an existing attorney is not found

On a Lead, "Who referred them?" returned nothing for an outside law office that already existed in Salesforce and was already shown as the Lead's Referring Attorney. The only way forward was "+ New outside attorney", which would have created a duplicate and split referral credit.

### DDT-1462, 2026-10-27 (closed 2026-11-05): add another referral source

Intake asked for a second named outside law office to be added as a "Referral: Outside Attorney" choice on the Lead's "How did you hear about us?".

## 3. Scope

**In scope**

| Component | Type | Ticket | Change |
|---|---|---|---|
| `referralPicker` | LWC | DDT-1142, DDT-1285 | Attorney search also finds the Referring Attorney Contact record type; two-way "They named someone / They couldn't name anyone" choice |
| `ReferralSourceController` | Apex | DDT-1142, DDT-1285 | Duplicate check widened; duplicate-rule refusal turned into a readable message; `clearReferral` added |
| `ReferralSourceService` | Apex | DDT-1142, DDT-1274 | `ATTORNEY_RECORD_TYPE` constant; `buildUpdates` fills a blank `Account.Referring_Attorney__c` |
| `Person_Account_Lightning_Page` | Lightning page | DDT-1142, DDT-1285 | Panel added to the page the DDT Law Group app opens |
| `howTheyFoundUs` | New LWC | DDT-1285, DDT-1379 | The single panel |
| `HowTheyFoundUsController` | New Apex | DDT-1285, DDT-1379 | Reads and saves the answer and referral; reads `Automated_Source__c` |
| `howHeardCapture` | LWC | DDT-1285 | Vendor two-step choice; pre-fill from the automated source |
| `newPotentialClient`, `convertToHiredMatter`, `disqualifiedLead` | LWC | DDT-1285 | Lead Triage screens pre-fill and lock how-heard from the automated source |
| `IntakeHowHeardService.stampRecordedBy` and five conversion controllers | Apex | DDT-1285 | Record who gave the answer, and when, at conversion |
| `How_Heard_Recorded_By__c`, `How_Heard_Recorded_On__c`, `How_Heard_Changed_From__c` | Account fields | DDT-1285 | Who answered, when, and the previous answer |
| `How_Heard_Attribution` | Permission set | DDT-1285, DDT-1296 | Read and edit on the new fields; access to the panel's Apex class |
| `Client_Advertising_Source` | Global value set | DDT-1255, DDT-1285, DDT-1462 | New values, relabels, one value activated |
| How-heard picklist on 16 active record types | Record types | DDT-1255, DDT-1285, DDT-1462 | New values enabled per record type |
| `Channel_Mapping__mdt` | Custom metadata | DDT-1285, DDT-1462 | New rows; Workforce Medical Network moved to paid vendor |
| `Account_Record_Page`, `Opportunity_Record_BB_Default_Hired`, `Lead_3_sections` | Lightning pages | DDT-1285 | Old Referral Source panel swapped for the new one |
| Backfill and verify scripts | Anonymous Apex | DDT-1274 | Fill the 29 blank Referring Attorney lookups |

**Out of scope**

- **A validation rule requiring Referring Attorney** (DDT-1274's original ask). Built as PR #674 and never deployed; see 5.3.
- **Removing the "They couldn't name anyone" escape for outside attorneys.** Requested on DDT-1274, raised as a separate scope decision.
- **Re-scoring past Workforce Medical Network Matters for sales compensation.** A runbook was prepared (PR #682); the firm decided on 2026-10-13 not to run it.
- **Splitting How-did-you-hear into two fields**, and any new scheduled job (DDT-1285 out of scope list).
- **`Attributed_Channel__c` on Inbound Contact**, found blank on all 26,597 rows; noted for its own ticket.

## 4. How it works (background)

```
How did you hear about us?   (picklist, values from Client_Advertising_Source)
  -> stored on the Client (Account); flows copy it to Court Cases and Matters
  -> Channel_Mapping__mdt row for that value decides: referral or not, origin type, paid or organic
If it is a referral:
  -> referrer saved on a referral row, Inbound_Contact__c (Referrer__c, Referrer_Type__c)
  -> ReferralSourceService.buildUpdates stamps Referral_Source__c on the parent from the latest row
Outside attorneys are Contacts:
  -> under the "Referring Attorney" Account, and/or with the Referring_Attorney record type
Each record type that uses the picklist carries its own list of allowed values
```

Before DDT-1285, the client page showed the how-heard field and a separate Referral Source panel (`inboundContactReferrerPicker`) that did not talk to each other.

## 5. Root cause

### 5.1 DDT-1142: two ways to create an attorney, one way to find them

1. The record page's **Referring Attorney** lookup has a "New Contact" action. It creates a Contact with the `Referring_Attorney` record type and **no account**.
2. The panel searched attorneys only under the "Referring Attorney" account, so it could not find that Contact.
3. `createReferrer`'s duplicate check also looked only under that account, so it tried to insert a second copy.
4. The org's active Contact duplicate rule (first plus last name, or email, or first name plus phone) refused the insert with `DUPLICATES_DETECTED`. The insert was not caught, so the panel showed its generic "Could not create the referrer."

Eleven other attorney Contacts in production were invisible to the panel for the same reason: 9 with no account and 2 under a different account.

Separately, the DDT Law Group app opens Person Accounts on `Person_Account_Lightning_Page`, which did not have the panel at all. It existed only on `Account_Record_Page`, which the Sales and Lead Generation apps use.

### 5.2 DDT-1255: a global value set does not reach filtered record types

The value already existed in `Client_Advertising_Source`, inactive. Activating it reached only the Master record types. Every other record type using the picklist keeps its own list of allowed values, and each held exactly the 23 previously active values, which made them look unfiltered. So after the first change (PR #643) Lead staff still could not pick it. Git's copies of several record types had also drifted from production, so they could not be deployed as they were.

### 5.3 DDT-1274: the attorney was saved, just not in that field

Checked on 2026-11-02. Nobody had skipped the question. Both the How They Found Us panel and the conversion screen refuse a referral answer without a referrer, and they store it on the client's referral row. **Nothing wrote it to `Account.Referring_Attorney__c`**, the field shown on the client page. A read-only script against production found 132 clients whose latest referral is an Outside Attorney naming a Contact: 29 had the lookup blank, 16 had the same attorney, 0 had a different one.

That is also why the requested validation rule (PR #674) was held: the panel and conversion screens never write the lookup, so the rule would have blocked their saves.

### 5.4 DDT-1285: two pieces that never talked

The how-heard field and the Referral Source panel were separate components. The panel never read the answer, so it asked everyone, and never locked. On the Lead page it never read `Automated_Source__c`, so vendor leads with no how-heard answer looked like unanswered referrals.

### 5.5 DDT-1379: the panel's own check had no exemption

Verified against production on 2026-10-20:

- "Referral: Employee" maps (in `Channel_Mapping__mdt`) to referrer type `Internal` with no referrer name, so the capture rule says "ask".
- `Internal` is the one referrer type with no referring party (`ReferralSourceService.hasReferringParty`). The picker knows this, shows the note and draws no search. `ReferralSourceService.validateCapture` exempts it too.
- `HowTheyFoundUsController.save` had its **own** extra check for an "ask" answer that never asked whether the type has a party, so it refused every employee referral. The same check was copied into the Lead save.

Every other active value was run through the capture rule in production; none had the same mismatch.

### 5.6 DDT-1384: the record was the wrong kind

The law office had been saved in Salesforce as a person rather than as an outside attorney, so the outside-attorney search (which looks for the attorney account or record type, see 5.1) did not return it. The exact fields that were wrong are not recorded in the ticket.

### 5.7 DDT-1462: same pattern as DDT-1255

A new value needs the global value set, every filtered record type, and a channel mapping row. One extra trap: the Person Account record type carries two how-heard blocks, `How_did_you_hear_about_us__c` and `How_did_you_hear_about_us__pc`. Lead convert copies the first onto the second, so a value missing from `__pc` fails the convert with `INVALID_OR_NULL_FOR_RESTRICTED_PICKLIST` (seen before on DDT-1372).

## 6. Solution

### 6.1 DDT-1142: find and name existing attorneys (PRs #526, #533)

Search and the duplicate check now cover both groups:

```js
// referralPicker.js: Outside Attorney search
return {
    criteria: [
        { fieldPath: 'Account.Name', operator: 'eq', value: ATTORNEY_PARENT_ACCOUNT_NAME },
        { fieldPath: 'RecordType.DeveloperName', operator: 'eq', value: ATTORNEY_RECORD_TYPE }
    ],
    filterLogic: '1 OR 2'
};
```

```apex
// ReferralSourceController: the insert, with duplicate rules turned into a message staff can act on
@TestVisible
private static void insertReferrer(SObject rec, String displayName) {
    try {
        insert rec;
    } catch (DmlException e) {
        String msg = e.getDmlType(0) == StatusCode.DUPLICATES_DETECTED
            ? displayName + ' already exists in Salesforce. Search for them instead of creating a new one.'
            : 'Could not create ' + displayName + ': ' + e.getDmlMessage(0);
        AuraHandledException ae = new AuraHandledException(msg);
        ae.setMessage(msg);
        throw ae;
    }
}
```

- **Any other failure keeps its own reason** instead of a guess.
- **`setMessage`** so the text reaches tests and logs, not "Script-thrown exception".
- **PR #533** added the panel to `Person_Account_Lightning_Page`, right after the duplicates card, so intake in the DDT Law Group app can use it.
- The attorney the staffer needed was then linked on the client by hand, and staff were told to use "+ New outside attorney" rather than the Referring Attorney field for new attorneys.

### 6.2 DDT-1255: activate the value and add it to every record type (PRs #643, #645)

- **PR #643:** activated the value in `Client_Advertising_Source`, using a fresh production retrieve of the value set (git's value order had drifted; deploying git's copy would have reordered intake's picklist). Added the value to the 4 business Account and 5 Contact record types, again starting from fresh production retrieves, because git's copies were missing production values and carried a block for a field that is not in production.
- **PR #645:** added it to Lead (Defense, PI, SOS), Person Account, Opportunity (Hired, Potential, on `How_did_you_hear_on_Creation__c`) and Matter. These git copies matched production, so 4 lines were added per file.

### 6.3 DDT-1285: the How They Found Us panel (PRs #669, #676, #679, #728, #780)

**One panel replaces both** on the Client, Business Account and Hired Court Case pages (#669), then the Lead page (#728). The loose how-heard field came off the client page.

- **One question first.** Only a "Referral: …" answer asks who referred them; the answer already gives the type, so the Referral Type dropdown is gone.
- **Locks once answered:** one line, "Recorded [date] by [user]", and a Change link. A change that moves referral credit asks for confirmation in the panel ("This moves referral credit from X to Y"), and the locked line then shows "was [old answer]".
- **"Didn't ask"** (new value) saves who and when but does **not** lock, with a red "Nobody asked yet" nudge.
- **"Other"** cannot be saved without text.
- **Vendors** collapse to one "Vendor…" choice plus "Which vendor?", but the saved value is the vendor's own existing value, because `Channel_Mapping__mdt` looks up that exact text.
- **Automated source** (#728): a vendor source with no how-heard answer shows one read-only line and no referral box; a staffer can still record a real referral. The three Lead Triage screens pre-fill and lock how-heard from it too. Attribution is unchanged.
- **Who answered** (#669, #780): `How_Heard_Recorded_By__c` and `How_Heard_Recorded_On__c` are stamped by the panel and by every conversion path through `IntakeHowHeardService.stampRecordedBy`. On a signed-contract hire the conversion runs as an automated user, so it credits the contract's sender instead. Linking a Lead to an existing client never claims the copied answer.

**Value-set and mapping changes**

| Change | Detail |
|---|---|
| New value "Referral: Medical Provider" | Provider referrals used to land in "Other"; new `Channel_Mapping__mdt` row with a referral origin |
| New value "Didn't ask" | Mapping row credits nothing, same as `Unknown` |
| "Unknown" relabelled "Asked, they don't remember" | Label only, API value unchanged |
| Workforce Medical Network (#679) | Relabelled "Vendor: Workforce Medical Network" (label only) and its mapping moved to Channel / Lead Vendors / Directories / Paid, matching Orange Nebula. 46 clients carry the value. |

**Why the record types were edited:** without the new values on all 16 active record types, Apex cannot save them, and the Matter flow that copies the client's answer would fail on Matter save. All 16 files matched production before the edit.

**Why the pages were rebuilt from production:** git's copy of the Account page was missing configuration that would have reset its related lists and action bar. The three pages were deployed from production retrieves and mirrored back to git in #676, along with a fix for a title the panel rendered twice.

**Follow-up (DDT-1296):** the first deploy did not grant access to `HowTheyFoundUsController`, so non-admins saw an Apex access error instead of the panel. The class was added to `How_Heard_Attribution`.

### 6.4 DDT-1379: an internal referral is complete on its own (PR #791)

```apex
private static Boolean referrerMissing(ReferralSourceService.Capture cap) {
    if (!ReferralSourceService.hasReferringParty(cap.referrerType)) {
        return false;
    }
    return String.isBlank(cap.referrerId)
        && !(cap.referrerNotKnown == true && String.isNotBlank(cap.referrerDescribedAs));
}
```

Used by both save paths (client and Lead), replacing the inline check. Every other type is refused exactly as before. The panel's locked view of a saved internal referral now reads "An internal referral, credited to the firm." instead of "Not a referral"; that view had never been reachable before, because the save always failed.

### 6.5 DDT-1384: corrected the record in place

The law office's record was changed to an outside attorney. Because it is the **same record**, every referral already credited to it stayed credited. Staff were asked not to use "+ New outside attorney" for it. No code change.

### 6.6 DDT-1274: fill the lookup from the referral (PR #874)

`ReferralSourceService.buildUpdates`, which already stamps `Referral_Source__c` from the latest referral row, now also fills `Account.Referring_Attorney__c`, **on Account only**, when:

- **the latest referral is Outside Attorney naming a Contact.** An internal referral can name a staff Contact, and a Court Case flow emails whoever is in this lookup a thank-you on hire;
- **the lookup is blank.** It is never replaced or cleared, because staff edit it by hand and this code runs again on every logged touch with the client.

A blank lookup with an attorney available now counts as a change; otherwise the 29 would be skipped, since their source and type were already correct. A dry-run-first backfill script then filled the 29.

**Knock-on:** the thank-you email to the referring attorney, sent 60 minutes after hire, reads this lookup. It comes back for future hires. The backfill does not send missed ones late.

### 6.7 DDT-1462: one value, deployed from production's own retrieve (PR #841)

Added to `Client_Advertising_Source`, to all 16 record types that offer the DDT-1255 value (including **both** Person Account blocks, `__c` and `__pc`), plus a `Channel_Mapping__mdt` row: Origin Type Outside Attorney, Paid/Organic Referral, Fee Bearing true, matching the DDT-1255 value.

**Why not deploy git's record types:** each file holds every picklist on the record type, and deploying it would overwrite the other picklists with git's copy. The package was built from a production retrieve of just the value set and the record types' how-heard fields, plus the one value. `__pc` is the exception, because the Metadata API never returns that block, so git's full list was carried.

After deploy the requester could not find the value: it sits at the bottom of the list, below "Didn't ask". The team also asked whether the requester meant a picklist value (what was built) or a searchable attorney under "Referral: Outside Attorney" → "They named someone"; the answer is not recorded on the ticket.

## 7. Testing

| Ticket | Tests | Notes |
|---|---|---|
| DDT-1142 | 3 new methods in `ReferralSourceControllerTest`: a no-account attorney is refused as existing by name; a real duplicate-rule refusal becomes a readable message; any other failure keeps its reason | 47 of 47 across `ReferralSourceControllerTest` and `ReferralSourceServiceTest` in the Partial sandbox |
| DDT-1255 | None; metadata only | Confirmed on the ticket as live on Lead, Account, Contact, Opportunity and Matter, across record types |
| DDT-1285 | New `HowTheyFoundUsControllerTest`; Jest for `howTheyFoundUs`, `howHeardCapture`, `referralPicker`; #728 added 5 Apex and 7 Jest tests (controller coverage 67.6% → 92.3%); #780 added tests in each conversion controller | Each production deploy was checked by retrieving the components back. Render check in production on 2026-10-15: a real Legal Growth Engine Lead showed one line and no referral box |
| DDT-1379 | 3 new Apex tests (client save with `Internal`, client save with no type sent, Lead save); 2 new Jest tests | Production check-only: 20 of 20. The new locked-view Jest test fails against the old component |
| DDT-1274 | 6 new tests in `ReferralSourceServiceTest`: fills a blank lookup; fills on recompute; does not replace a hand edit; ignores an internal referral; leaves a typed attorney in place; never writes a Lead | Verify script before and after the backfill; `blank` must reach 0. Not done: running the new tests against the old code |
| DDT-1384 | Not applicable: data change | Requester asked to confirm the search finds the record |
| DDT-1462 | Not applicable: metadata only | Planned check in both sandboxes: retrieve before and after, save a Lead, convert it and confirm `__c` and `__pc` both hold the value |

**A test trap worth knowing:** `HowTheyFoundUsControllerTest` passes only when the running user already holds `How_Heard_Attribution`. Its setup assigns the permission set, but the methods do not run inside `System.runAs`, so the assignment does not reach the field checks.

**Not covered:** no Apex test or validation renders an LWC. Panel behaviour on screen was checked by people.

## 8. Step by step

**Attorney not found, or "Could not create the referrer"**

1. Find the attorney Contact. Check its account and record type. The search finds Contacts under the "Referring Attorney" account **or** with the `Referring_Attorney` record type.
2. If it is neither, correct the record in place (do not create a new one), so existing credit stays.
3. If create fails, read the message: "already exists in Salesforce" means search for them.

**Add a new "How did you hear about us?" value**

1. Retrieve `Client_Advertising_Source` and every record type that uses the picklist from production. Do not trust git's copies.
2. Add or activate the value in the value set.
3. Add it to each filtered record type: Lead ×3, Account ×4, Contact ×5, Matter, Opportunity ×2 (on `How_did_you_hear_on_Creation__c`), Person Account (**both** `__c` and `__pc`).
4. Add a `Channel_Mapping__mdt` row so attribution knows what the value means.
5. Deploy a package built from the production retrieve with only the how-heard blocks changed.
6. Retrieve again afterwards and confirm only the how-heard blocks changed. Save a Lead with the value and convert it.
7. Tell the requester where the value sits in the list.

**A referral answer refuses to save**

1. Run the value through `ReferralSourceService.captureRule` in anonymous Apex and read the referrer type.
2. If the type has no referring party (`hasReferringParty` is false), the save must not demand one.

**Referring Attorney blank on Outside Attorney clients**

1. Run `scripts/ddt-1274-verify-referring-attorney.apex` (read-only) and read the `blank` count.
2. Run `scripts/ddt-1274-backfill-referring-attorney.apex` with `DRY_RUN = true`, check the list, then with `DRY_RUN = false`.
3. Run the verify script again. `blank` must be 0.

## 9. Rollback

- **Code changes** (DDT-1142, DDT-1285, DDT-1379, DDT-1274): redeploy the previous versions of the named classes and components. For DDT-1379 and DDT-1274 the rows saved in the meantime are valid and stay.
- **New picklist values** (DDT-1255, DDT-1462): deactivate the value in `Client_Advertising_Source` in Setup and delete its mapping row. Records already saved keep the value.
- **Lightning pages:** redeploy from a production retrieve, never from git's copy.
- **The DDT-1274 backfill** only filled blank lookups and the script listed them before writing; it does not record a separate undo list.
- **DDT-1384:** not recorded in the ticket.

## 10. Known limits

- **Five Outside Attorney clients stay blank** after the DDT-1274 backfill: staff chose "They couldn't name anyone", so there is no attorney to copy. They need the attorney added by hand.
- **The record page's Referring Attorney lookup still creates attorneys with no account.** The panel now finds them, but filing them under the account would need a page or quick-action change.
- **A user without `How_Heard_Attribution`** (and not a System Administrator) cannot load the panel.
- **The `HowTheyFoundUsControllerTest` permission-set dependency** is pre-existing and not fixed.
- **Already-attributed records do not self-heal** when a mapping changes; the attribution sweeper only picks up unstamped records. One Court Case had to be re-scored by hand after the Workforce Medical Network change.
- **New picklist values appear at the end of the list** unless they are reordered.

## 11. Reuse for similar issues

**How to recognise one:** a referral or how-heard answer that "won't save", "can't be found" or "is blank" on one page but present somewhere else.

- **Find where the data actually lives** before building what the ticket asks for. DDT-1274 asked for a rule to force a field that nothing was meant to fill; the real answer was to fill it.
- **One search, one duplicate check, one definition.** If records can be created two ways, the search and the duplicate check must find both.
- **Catch duplicate-rule refusals** and turn them into a message that tells staff to search.
- **A global value set is not enough.** Every record type that filters the picklist needs the value, and Person Account needs it on `__pc` too.
- **Build metadata deploys from a production retrieve**, especially record types and Lightning pages; git's copies drift.
- **Fix a bad record in place** rather than creating a new one, so existing credit stays attached.
- **When two layers validate the same thing,** give them the same exemptions, or call one from the other.
