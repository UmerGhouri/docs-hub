# Torq Gameplan integration

| | |
|---|---|
| **Ticket** | DDT-974, DDT-975, DDT-976, DDT-977, DDT-978, DDT-979, DDT-980, DDT-981, DDT-1499 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-14 (DDT-1499 is sandbox only, done 2026-11-04) |
| **Priority** | High (DDT-978 and DDT-979 Medium) |
| **Reported by** | Firm staff, 2026-09-09 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex callout, LWC, quick action, Lightning record page, permission sets), Torq |
| **Area** | DUI intake: generating a personalised DUI Gameplan for a Potential Court Case |
| **Related** | None outside this group |
| **Code** | PR #308, PR #333, PR #340 and PR #371 in the `ddt-salesforce` repo |

---

## 1. Summary

A quick action on a Potential DUI Court Case asks Torq, an outside vendor, to build a personalised DUI Gameplan for the client. These tickets record the production cutover of that integration: deploying the recipient resolver that had only ever run in a sandbox, changing the button to match Torq's draft-and-review flow, sending the clicking user's email so Torq can route the draft, answering Torq's field questions, proving the button cannot message a client from Salesforce, and finally opening the button to intake users on 2026-09-14.

DDT-1499 is a later, sandbox-only follow-up that gave Torq's sandbox integration user read access to court appearances.

## 2. What was reported

- **DDT-974, 2026-09-09.** While preparing the cutover, the recipient resolver turned out to exist only in the partial sandbox. It had been verified there on 24 August but never merged, so `main` and production both still ran the older version. A deploy from `main` would have silently reverted it.
- **DDT-975, 2026-09-09.** Production was deliberately inert: no credential, the old callout class, and the button restricted to System Administrators. Torq issued the production credential on 9 September.
- **DDT-976, 2026-09-09.** Torq confirmed on 7 September that a successful call creates a **draft** that a person reviews and approves in Torq before anything reaches the client. The button still said it would "build and send" and that Torq would text the client.
- **DDT-977, 2026-09-09.** Torq routes each draft to the person who generated it, but Salesforce sent nothing that identified that person.
- **DDT-978, 2026-09-09.** Torq asked for the Salesforce field API names their landing page personalises from.
- **DDT-979, 2026-09-09.** Torq needed sandbox records that exercise the personalised blocks; the existing reference records could not.
- **DDT-980, 2026-09-09.** Before the first controlled run, firm leadership asked for proof from the code that clicking the button cannot send a client a message from Salesforce.
- **DDT-981, 2026-09-09, completed 2026-09-14.** Remove the System Administrator restriction so intake users can see the button.
- **DDT-1499, 2026-11-03.** Torq reported that the sandbox Court Case had no next-court-appearance lookup, and asked for test data with a court date.

## 3. Scope

**In scope**

| Component | Type | Change | Ticket |
|---|---|---|---|
| `GameplanTriggerCallout` | Apex class | Shared recipient resolver; result wording; `actorEmail` in the request body | DDT-974, DDT-976, DDT-977 |
| `GameplanTriggerCalloutTest` | Apex test | Tests for each change above | DDT-974, DDT-976, DDT-977 |
| `buildDuiGameplan` | LWC | Draft wording and result messages | DDT-974, DDT-976 |
| `Opportunity.Build_DUI_Gameplan` | Quick action | Label now **Generate DUI Gameplan Draft** | DDT-976 |
| `Opportunity.Torq_Gameplan_Sent_At__c` | DateTime field | Label now **Torq Gameplan Draft Created At**; API name unchanged | DDT-976 |
| `Opportunity_Record_Page_Potential` | Lightning record page | Visibility rule on the action loses the profile criterion | DDT-981 |
| `Torq_Gameplan` | Named Credential | Confirmed pointing at Torq's production endpoint | DDT-975 |
| Torq credential record | Custom setting data | Created by hand in production Setup | DDT-975 |
| `Torq_Court_Appearance_Read` | Permission set (sandbox only) | Read on `Court_Appearance__c` and four of its fields | DDT-1499 |

**Out of scope**

- **The callout itself in DDT-976.** Same endpoint, same signing; only wording and the meaning of the result changed.
- **The other three System Administrator gates** on the Potential page. They belong to other actions and were left alone.
- **The attorney's own `Gameplan__c` field** on Opportunity. Nothing here touches it, which is why every new field is prefixed `Torq_`.
- **DDT-1499 in production.** Torq asked for the sandbox only, and their production user already has the access.
- **Intake's use of Populate QWLR.** Found during DDT-981 and raised separately (see 10).

## 4. How it works (background)

```
Intake user on a Potential DUI Court Case
  -> quick action "Generate DUI Gameplan Draft"  (LWC buildDuiGameplan)
  -> Apex GameplanTriggerCallout.triggerBuild()
  -> one signed HTTP call to Torq through the Torq_Gameplan Named Credential
  -> Torq creates a draft and queues it for review by the person who clicked
  -> Salesforce saves Torq's run id (and the draft time) on the Court Case
  -> a person approves the draft in Torq; only then does Torq text the client
```

The call is server to server. The signing secret covers the whole firm, so it must never reach browser JavaScript, and the request body is part of what is signed. A request Torq cannot verify comes back as a bare `401` with no explanation.

## 5. Root cause

### DDT-974: the resolver failed to find the client

`Opportunity.ContactId` has no relationship name, so `Opportunity.Contact.*` cannot be traversed in SOQL. The Contact has to be queried separately. On top of that, about 15% of Potential DUI cases are created outside lead conversion and have no `ContactId` at all, and DDT intake writes the client's number to `Account.Phone`, not `PersonMobilePhone` (213 of 213 against 2 of 213). The fix existed in the sandbox only; `main` had zero occurrences of the resolver.

### DDT-976 and DDT-977: built for the old Torq contract

Salesforce was built when a successful call meant the client had **already** been texted. Torq changed to draft and review. That left the button promising a text that had not happened, a "Sent At" field stamped for something nobody sent, and an empty request body that gave Torq nothing to route the draft on.

### DDT-978: our own earlier answer was wrong

On 17 August DDT told Torq that court date was 0% filled before hire. There are two near-identical fields: `Next_Court_Date_Time__c` (DateTime), which is 0% filled and was the one measured, and `Next_Court_Date__c` (Date), which intake fills on 85 to 89% of cases. The correction was owned in the reply.

### DDT-1499: a missing object grant that looked like a missing field

The lookup `Next_Upcoming_Court_Appearance__c` existed in the sandbox and Torq's user already had field access to it. Their user is on the standard Read Only profile, which has no access to the `Court_Appearance__c` object, so any query that followed the lookup failed and read like a missing field.

### DDT-975, DDT-979, DDT-980, DDT-981

Not applicable: these were cutover steps, test data, an audit and a planned release, not defects.

## 6. Solution

### 6.1 One shared recipient resolver (DDT-974, PR #308)

The SMS destination and the transcript anchor now use the same lookup, in this order:

1. `Opportunity.ContactId`, then a separate `Contact` query for `MobilePhone`, falling back to `Phone`.
2. The primary `OpportunityContactRole`.
3. `Account.Phone`, where intake actually writes the number.

```apex
if (String.isBlank(r.phone) && String.isNotBlank(opp.Account.Phone)) {
    r.phone = opp.Account.Phone;
    r.source = 'Account.Phone';
}
```

A build that reaches nobody now records only Torq's run id and leaves the timestamp blank. Torq's own verdict wins: if it reports the recipient as unresolved or returns `no_contact_match`, the build counts as reaching nobody. Before, every successful call was stamped, so the 10-minute resend guard could block a retry that would actually have worked.

### 6.2 Production cutover (DDT-975)

1. Deployed `GameplanTriggerCallout`, its test class and the LWC (6.1).
2. Created the org-default credential record by hand in Setup. Credential values are data and are never committed to git.
3. Confirmed the Named Credential points at Torq's production endpoint.
4. Updated the sandbox's stored staging secret after Torq rotated it, and confirmed it with a signed call that returned `201`.

Two misunderstandings were corrected with Torq: the credential record is a DDT setting only DDT can create (production had zero rows until it was created), and the button does not ungate itself on deploy, because the System Administrator criterion is a page visibility rule someone has to remove.

The admin gate was deliberately left up until Torq's production consumer was live (6.6).

### 6.3 A successful call means a draft (DDT-976, PR #333)

- Quick action label: **Generate DUI Gameplan Draft** (was "Build & Send DUI Gameplan").
- Success message: "Draft created in Torq — review and approve it in Torq, the client is only texted once you do."
- When no recipient is found, the message says the draft cannot be sent **even after approval**.
- `Torq_Gameplan_Sent_At__c` keeps its API name, because the LWC, the `Torq_Gameplan_User` permission set and reports reference it. Its label is now **Torq Gameplan Draft Created At**, and its description says nothing is sent from Salesforce.
- The resend guard now says a draft was already generated, and that generating again puts a second draft in Torq's review queue.

### 6.4 Send the clicking user's email (DDT-977, PR #340)

Torq confirmed email is the join key: their users are keyed by email, so no lookup table is needed on either side. The body is now a single JSON field, built with `JSONGenerator`:

```apex
JSONGenerator gen = JSON.createGenerator(false);
gen.writeStartObject();
gen.writeStringField('actorEmail', UserInfo.getUserEmail());
gen.writeEndObject();
return gen.getAsString();
```

Why `JSONGenerator` and not string concatenation: it escapes the value. An unescaped quote would be signed on our side and rejected on Torq's, and that shows up only as a bare `401`.

Because the body is signed, this changed the signing contract. If Torq had expected the old empty body, every call would have failed until their side shipped. That risk was tested before deploying: the new signed body was sent to Torq's staging environment from the sandbox and returned `201`, proving Torq hashes what it receives. So deploying ahead of Torq's consumer was safe.

### 6.5 Field API names for the landing page (DDT-978)

Every field is on `Opportunity` directly, so no traversal is needed. Measured in both the sandbox and production:

| What the page needs | API name | Sandbox | Production |
|---|---|---|---|
| Court date | `Next_Court_Date__c` | 89% | 85% |
| County | `County__c` | 99% | 98% |
| Courthouse | `Courthouse_Simplified__c` | 97% | 96% |
| Notice date | Does not exist; proxy `Date_Time_of_Arrest__c` | 51% | 45% |
| SSS start | `SSS_Start_Date__c` | 51% | 45% |
| SSS length | `SSS_Length__c` | 30% | 37% |
| FST outcome | Does not exist | — | — |
| Variant selector | `QWLR_Concerns_Multi_Select_DUI__c` | 55% | 50% |

Traps flagged to Torq:

- **`SSS_Start_Date__c` is arrest date plus 46 days, filled whether or not a suspension is happening.** Of 1,062 Potential DUI cases with a start date, 388 had a blank length, 32 "I don't know" and 20 "No SSS". Gate the countdown on an **allow-list** of real lengths (3 months, 6 months, 12 months, 3 years), not an exclusion list, or the page shows a licence-loss countdown to someone who is not losing their licence.
- `QWLR_Days_Until_License_Suspension__c` is a formula evaluated when read and goes negative; compute from `SSS_Start_Date__c` instead.
- `Courthouse_Simplified__c` is blank for some courthouse values; fall back to `Courthouse__c`.
- `QWLR_Court_Date__c` is a stale text snapshot that drifts from the live field.
- Picklist apostrophes differ between fields (a curly quote in one, a straight quote in another).

Torq's user reads the org as a user on a profile, so access is profile field-level security. All the template fields were confirmed readable. Torq adopted the list and the allow-list gate as given, and dropped the FST and notice-date blocks.

### 6.6 Personalisation fixtures in the sandbox (DDT-979)

The three existing reference records only test contact resolution: all have a blank court date and SSS length and the same county and concerns. Rather than disturb records Torq's regression tests depend on, four new Person Account / Potential / DUI cases were created on reserved fictional phone numbers:

| Scenario | Court date | SSS length | Countdown |
|---|---|---|---|
| Full | set | 6 months | 36 days |
| No SSS | set | No SSS | 36 days (should not render) |
| Blank length | set | blank | 36 days (should not render) |
| Sparse | blank | blank | none; courthouse falls back |

The countdown is the same 36 days on all three that have one, so only `SSS_Length__c` differs. If "No SSS" or "Blank length" shows a countdown, the bug is in the length check, not the date arithmetic. Blank length has its own record because it is the 388-case group an exclusion-list gate lets through.

### 6.7 The button cannot message a client (DDT-980)

`GameplanTriggerCallout` and `buildDuiGameplan.js` contain no `Messaging.*`, `sendEmail`, Heymarket, Twilio, RingCentral, platform events, `@future` or `System.enqueueJob`. The path is the quick action, the LWC, `triggerBuild()`, one HTTP call, and one single-record `update`. Nothing asynchronous is queued.

That `update` does fire record-triggered flows on Court Case. All 45 active ones were checked. Seven contain a messaging action, and six cannot be reached from a Potential case (they require Hired status, a specific stage change, or are scheduled rather than record-triggered).

The seventh, `Court_Case_CMT_Email_Notifs`, emails and texts **the client** an evaluation video. Its entry criteria are re-checked on every save, so any save, including this one, could release it. It is held back by a guard that requires `ADES_Video_Sent__c = false`, and it sets that to true once sent. **Measured: 0 Potential DUI cases were in that state.** So the click cannot trigger it today, but the coupling is real.

### 6.8 Open the button to intake (DDT-981, PR #371)

Torq confirmed on 2026-09-14 that their production release and the DDT gameplan flag were live, including the consumer that reads `actorEmail`. The page was retrieved from production first and matched `main` exactly. The action's visibility rule went from `(1 OR 2) AND 3` to `1 OR 2`: the two `Case_Type__c` criteria (DUI, Felony DUI) stay and the profile criterion is removed.

### 6.9 Court appearance read access in the sandbox (DDT-1499)

`Torq_Court_Appearance_Read`: read on `Court_Appearance__c` and on `Court_Date__c`, `Courthouse__c`, `Courtroom__c` and `Court_Address__c`. Opportunity read is included only because Salesforce refuses the child grant without it ("depends on permission(s): Read Opportunity"). Deployed and assigned to Torq's user in the sandbox only, and not in git.

A test court appearance was created on a sandbox Court Case. The case's next-appearance lookup and next court date were filled by existing automation, not by hand. `Courthouse__c` and `Court_Address__c` on the appearance are formulas (the first is the Court Case courthouse as text), so they cannot be set directly.

## 7. Testing

**Apex tests.** `GameplanTriggerCalloutTest`, 12 of 12 passing for PR #333. One assertion changed from "already sent" to "already generated". For PR #340, `signatureMatchesContract` asserts the request body **literally** instead of calling `buildBody()`. Reusing the method would let a payload change move both sides at once and still pass, which is exactly what the test exists to catch.

For PR #340 the full local test suite also passed on the same commit (2,514 tests, 0 failures), but that run was never posted because the connection dropped. A second full run was cancelled by another validation, not failed. The posted check used the 12 specified tests.

**Live checks in the sandbox (DDT-974).** Torq's three reference records were re-run: two unchanged (resolved through the contact), and the regression case flipped from `no_contact_match` to resolved through `Account.Phone`.

**Live checks in production.** After each deploy on 2026-09-09 the production class was compared with the merged source and found byte-identical, with the expected changes present (the resolver; "already generated" present and "already sent" gone; the new field label; the `actorEmail` body and content type). After PR #371, the live page was retrieved again and showed `1 OR 2`, with the other three System Administrator gates untouched.

**Not covered.** No automated test runs the real call to Torq. The first real production Gameplan was planned as a deliberate controlled run on a chosen case.

## 8. Step by step

The order matters: the button stays restricted until everything behind it is live.

1. **Check `main` has what was tested.** Search `main` for the change (here, `resolveRecipient`). If it exists only in a sandbox, merge it first, or the next deploy from `main` reverts it.
2. **Deploy the class, its test and the LWC** to production after validating against production.
3. **Create the credential record** by hand in Setup (org default) with the production values from the vendor. Never commit them.
4. **Confirm the Named Credential endpoint** is production's.
5. **Change wording and labels** to match what the vendor actually does (draft, not sent). Keep API names; change labels and descriptions.
6. **Add the user identity to the body** with `JSONGenerator`, update the contract test with a literal expected body, and send one signed call to the vendor's staging from a sandbox before deploying.
7. **Answer field questions by measuring both orgs,** naming the exact API name and fill rate, and flag fields that look right but mean something else.
8. **Audit the click path** for any messaging, and check every record-triggered flow on the object the code updates.
9. **When the vendor confirms production is live,** retrieve the live page, remove the profile criterion from the action's visibility rule, deploy, and retrieve again to confirm.

## 9. Rollback

- **Restrict the button again:** add the profile criterion back to the action's visibility rule. This is the quickest safe stop and needs no code change.
- **Code changes:** redeploy `GameplanTriggerCallout` from an earlier commit. Rolling back PR #340 would remove `actorEmail` and leave Torq with nothing to route drafts on. Rolling back PR #308 would bring back the failed recipient lookups.
- **Credential:** deleting the credential record makes the integration inert again.
- **DDT-1499:** remove the permission set assignment in the sandbox.

## 10. Known limits

- **Gameplans are largely generic today.** Intake stopped running Populate QWLR around 27 July, so arrest date, SSS start, SSS length and concerns are missing on almost every Potential DUI since (arrest date filled on 38 of 74 in July, 1 of 54 in August, 0 of 24 in September). The SSS countdown rendered on zero live cases and concerns-based branching worked on about 1 in 8. `Court_Case_Populate_QWLR` is still active and unchanged; it simply is not being used. This is a process question, raised separately.
- **The video email flow coupling (6.7) is real.** Re-check it before any bulk update of Court Cases.
- **The rate limit is firm-wide.** One credential covers every user.
- **DDT-1499's permission set exists only in the sandbox.** If the production Torq user's access is ever narrowed, it has to be rebuilt there.
- **The fixtures' SSS start date** matches the arrest-plus-46-days rule, so they cannot show whether the stored value was set by hand or by automation.

## 11. Reuse for similar issues

**How to recognise one:** a vendor integration that works in a sandbox and is about to go to production, or a vendor saying a Salesforce field "does not exist".

- **Before a production cutover, confirm `main` has what was tested.** A sandbox-only fix is one deploy away from being silently reverted.
- **When a vendor changes what a success means, change the wording and labels too,** keeping API names so nothing that references them breaks.
- **Escape anything you sign.** Build JSON with `JSONGenerator`, and test the signed body against a literal string.
- **Test contract changes against the vendor's staging** before deploying ahead of them.
- **"Field does not exist" through a lookup often means no object access.** Check the parent object grant before adding fields. A child grant may also need read on its parent object.
- **Two fields with near-identical names can mean different things.** Measure the one the screen actually uses.
- **Gate on an allow-list** when a wrong positive is worse than showing nothing.
- **"Can this click message a client?"** Read the code path, then check every record-triggered flow on the record it saves.
