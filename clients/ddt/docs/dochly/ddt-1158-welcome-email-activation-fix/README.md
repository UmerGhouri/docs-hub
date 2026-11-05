# Attorney-stage welcome email switched on (LV, HV, SOS)

| | |
|---|---|
| **Ticket** | DDT-1158 |
| **Project** | Dochly (no project set on the ticket) |
| **Status** | Live since 2026-09-30 |
| **Priority** | None set |
| **Reported by** | Support team, 2026-09-30 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow), Dochly templates and email |
| **Area** | Client communication at the Case Manager to Attorney handover |
| **Related** | DDT-1057 (the build), DDT-1058 (Attorney Welcome Call event) |
| **Code** | PRs #555, #559 and #607 in the ddt-salesforce repo (the build was PR #416) |

---

## 1. Summary

The welcome email built under DDT-1057 sat in production as two Draft flows. When the time came to switch it on, Flow Builder refused to activate the trigger flow, because its entry condition was a formula and a flow with scheduled paths needs a filter that fires only when a record *changes to* meet it. PR #555 swapped the formula for an equivalent filter, the trigger was activated on 2026-09-30, and PR #559 recorded it as Active in git. Two days later PR #607 stopped the email for Investigation cases.

## 2. What was reported

- Production held `Welcome_Email_on_Attorney_Stage` as an invalid draft. Flow Builder refused activation with: "This flow has a scheduled path with a time source based on when a record is created or updated. Configure the flow to run only when a record is updated to meet the condition requirements."
- On 2026-10-02 Firm operations asked for Investigation cases to be left out. They do not go through the Case Manager to Attorney handover the email is written for, and **one Investigation case had already received the LV email** on 2026-10-01.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Welcome_Email_on_Attorney_Stage` | Record-triggered flow | Entry condition changed from a formula to a filter (PR #555); status recorded as Active (PR #559) |
| `Welcome_Email_Send` | Subflow | Status recorded as Active (PR #555); new first outcome that stops for Investigation cases (PR #607) |
| `welcome-email-on-attorney-stage.md` | Deploy review doc | Activation record, gates, and what the first live emails must prove |

**Out of scope**

- **The templates.** The HV and SOS templates were edited in Dochly on 2026-09-28; nothing here changed them.
- **The fee-band line.** The roughly $3,000 HV/LV split from DDT-1057 was kept. The Support team decided it did not need the firm's sign-off before go-live; that is recorded as a decision, not an approval.

## 4. How it works (background)

```
Opportunity.Blueback_Stage__c becomes 'Attorney'
   |
   +-- Mon-Thu ................ send now
   +-- Fri / Sat / Sun ........ +1, +2, +3 day paths, each sends only if today is Monday
   |
   v  Welcome_Email_Send
      1. no client email                   -> stop
      2. welcome document already exists   -> stop
      3. Case_Type__c = Investigation       -> stop        (from 2026-10-02)
      4. SOS -> HV -> else LV; blank type   -> stop
      5. Dochly sends the band's template from the firm's office address
```

## 5. Root cause

**Why activation failed.** The trigger's entry condition was a formula:

```
ISCHANGED(Blueback_Stage__c) && TEXT(Blueback_Stage__c) = "Attorney"
```

A formula condition runs on every update that satisfies it. Salesforce only allows scheduled paths on a flow set to run **only when a record is updated to meet the condition requirements**, and that option is not available with this kind of formula. The three weekend paths made the setting mandatory, so the flow could not be activated as built.

**Why an Investigation case got the email.** The routing sent every case type that was not SOS or HV to the LV template, including Investigation.

## 6. Solution

### 6.1 Replace the formula with a filter (PR #555)

The entry condition became the filter `Blueback_Stage__c = Attorney` with `doesRequireRecordChangedToMeetCriteria = true`. That fires when a record goes from not meeting the condition to meeting it, which is exactly a change into Attorney, so who gets an email is unchanged.

### 6.2 Activate, then record it in git (PR #559)

The deploy did not activate the flow: production overwrote version 1 in place and left it Draft. It was activated by hand in Flow Builder at 11:20 UTC on 2026-09-30. PR #559, the same day, changed only its `status` element from Draft to Active, so git matches production.

### 6.3 Stop for Investigation cases (PR #607)

A new first outcome, "Investigation - Stop", in the subflow's band routing. It has no next step, so the flow ends there. Every other case type routes as before.

**Why in the subflow and not the trigger's entry condition:** the trigger fires when a record *newly* meets its condition. With an Investigation filter there, a case already at Attorney that changed type away from Investigation would newly meet the condition and get the email.

## 7. Testing

| Check | Result |
|---|---|
| Production copies of both flows against the repo | Identical apart from status |
| Fixed trigger deployed to the sandbox as Active | Activated cleanly |
| Check-only validation against production | Succeeded, 26 of 26 named tests |
| Real deploy to production | Succeeded, 26 of 26 named tests |
| Trigger status after the deploy | Draft (production did not activate it) |
| Trigger activated in Flow Builder | Active, 2026-09-30 |

**What those validations did not cover.** The named test class does not exercise either flow. No validation ran the flow logic, rendered a template or sent an email.

**Sandbox test.** The three welcome templates and the three templates they embed (video block, welcome-call footer and table) were copied into the sandbox, and the subflow was debugged on test Court Cases. The email arrived. This proves the routing and the Dochly send in the sandbox. It does **not** prove the production HV and SOS templates, because the sandbox copies' attorney conditions were rewritten for sandbox usernames.

**Gates skipped.** The deploy doc set two production gates before activation: one send per band to a staff address with the attorney section checked, and a re-run to prove no second email. **Neither was run.** The Support team chose to activate without them. So:

- the first real HV and SOS emails were the first proof of those templates since their 2026-09-28 edits,
- the duplicate guard had not been seen working in production,
- the weekend paths were untested until the first Monday after activation, 2026-10-05.

The closing comment on the ticket says the team would watch the first HV and SOS emails and the 2026-10-05 weekend send. The result of that watch is not recorded in the ticket.

**Live check (2026-11-05):** both flows have an active version in production, and in each case it is the latest version.

## 8. Step by step

1. Try to activate the trigger in Flow Builder and read the error.
2. Replace the formula entry condition with a filter on `Blueback_Stage__c = Attorney`, and set the flow to run only when a record is updated to meet the condition.
3. Retrieve both flows from production and confirm they match the repo apart from status.
4. Deploy the fixed trigger to the sandbox as Active and confirm it activates.
5. Validate check-only against production, then deploy.
6. Check the status in production. The deploy leaves it Draft.
7. Before activating, send one email per band to a staff address and re-run one to confirm no duplicate. (Skipped this time; see section 7.)
8. Activate the trigger in Flow Builder.
9. The same day, open a PR setting the flow's status to Active in git.
10. Open the first real email of each band and check the attorney name, bio link and video block render.

For the Investigation exclusion: add a first outcome to the subflow's routing that checks `Case_Type__c = Investigation` and ends the flow.

## 9. Rollback

Deactivate `Welcome_Email_on_Attorney_Stage` in Flow Builder. The flows change nothing on the Court Case; the only side effects are the Dochly document and the email, and **an email cannot be recalled**. If any first email is wrong, deactivate before the next case moves to Attorney.

## 10. Known limits

- **Cases with no client email get nothing,** and nothing records that they were skipped.
- **The request-key duplicate guard is still unproven.** The document lookup is what stops a resend.
- **The weekday check uses the running user's timezone.** All users were Central at the time.
- **A deploy does not activate a record-triggered flow in production.** Every redeploy needs the same check of status afterwards.
- **New case types route to LV by default.** Any new type that should not get the email needs its own stop, like Investigation.

## 11. Reuse for similar issues

- **A flow with scheduled paths needs a "changed to meet the condition" entry.** Use a filter with that option, not an `ISCHANGED` formula.
- **Check status after deploying a flow to production.** It can arrive as Draft even when git says Active; record the real state in git the same day it changes.
- **Put exclusions in the logic, not the entry condition,** when the entry fires on records newly meeting it. Otherwise a change of the excluded field can itself trigger the action.
- **Say what a validation did not run.** A green validation of flow metadata sends no email and renders no template.
- **If the pre-go-live gates are skipped, write down what the first real runs must prove,** and who is watching for them.
