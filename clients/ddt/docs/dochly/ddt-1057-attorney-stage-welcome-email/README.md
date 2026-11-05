# Automatic welcome email when a case reaches the Attorney stage

| | |
|---|---|
| **Ticket** | DDT-1057 |
| **Project** | Dochly Template Migration |
| **Status** | Done 2026-09-21 (flows built as Draft; switched on 2026-09-30 under DDT-1158) |
| **Priority** | High |
| **Reported by** | Support team, 2026-09-21 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow), Dochly templates and email |
| **Area** | Client communication at the Case Manager to Attorney handover |
| **Related** | DDT-1058 (Attorney Welcome Call event in Calendly), DDT-1158 (activation) |
| **Code** | PR #416 in the ddt-salesforce repo |

---

## 1. Summary

The firm asked for a welcome email to go to the client automatically when a Court Case moves from Case Manager to Attorney. PR #416 built it as two flows that send a Dochly email template, one of three by fee band (LV, HV or SOS), and shipped both as **Draft**, so nothing fired. This ticket recorded what was built and what was still needed before it could go live. The switch-on happened under DDT-1158.

## 2. What was reported

A feature request, not a fault. The email was to carry:

- an embedded video,
- content that varies by payment status,
- a Calendly link for booking the Attorney Welcome Call (DDT-1058).

The original request also said "within 24 hours of hire" and named the `Blueback__c` object as the trigger. Both were changed on evidence (section 5).

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Welcome_Email_on_Attorney_Stage` | Record-triggered flow on `Opportunity` (Court Case) | New. Fires on the stage change and applies the weekday rule |
| `Welcome_Email_Send` | Subflow | New. Recipient check, duplicate guard, band routing and the Dochly send. A subflow so the four entry paths share one copy |
| `welcome-email-on-attorney-stage.md` | Deploy review doc | New. Lists six open gaps |

**Out of scope**

- **The Dochly templates themselves.** The three welcome templates are edited in Dochly's template editor, not in git.
- **Activation.** Both flows shipped as Draft on purpose. Switching on is a human step; it was done under DDT-1158.

## 4. How it works (background)

```
Opportunity.Blueback_Stage__c changes into 'Attorney'
   |
   +-- Mon-Thu ................ send now
   +-- Fri / Sat / Sun ........ scheduled paths land on the following Monday
   |
   v  Welcome_Email_Send subflow
      1. Get the Court Case
      2. no Case_Client_Email__c          -> stop
      3. welcome document already exists  -> stop
      4. route: SOS -> HV -> else LV; blank Case_Type__c -> stop
      5. Dochly "generate and email" action sends the band's template
```

| Band | `Case_Type__c` | Template |
|---|---|---|
| SOS | any value beginning `SOS` (6 values) | SOS welcome |
| HV | Felony DUI, Criminal Felony, Felony, DUI, Civil Asset Forfeiture, Order of Protection, Criminal Misdemeanor | HV welcome |
| LV | everything else | LV welcome |

SOS is tested first because it straddles both fee bands. The HV/LV line sits at about $3,000 average fee, at the largest gap between case types. The deploy doc records that this line was chosen during the build, not set by the firm.

## 5. Root cause

Not applicable as a fault. Two parts of the original request were changed after checking production data, and they are the useful lesson here.

**The trigger is not the `Blueback__c` object.** The request named it, but production showed it was dead:

```
Blueback__c records             85, all created Sep-Nov 2022
most recent modification        2022-11-30
case types                      all DUI or Felony
```

A trigger there would never fire. The live equivalent is the `Blueback_Stage__c` picklist on the Court Case: 151 cases at Attorney and 245 moves into Attorney in the 30 days before the build.

**It fires on any move into Attorney, not only from Case Manager.** 344 cases sat at Intake and some jump straight to Attorney. The duplicate guard covers the overlap.

**The 24-hour rule was dropped.** Over 1,350 cases that reached Attorney in the previous 180 days:

| Hire to Attorney stage | Share |
|---|---|
| Same calendar day | 12.0% |
| Next day | 24.1% |
| 2-3 days | 25.0% |
| 4-7 days | 23.0% |
| 8-30 days | 13.9% |
| 31+ days | 1.9% |

The median is 3 days. As a deadline the rule would miss two cases in three; as a delay it changes nothing for 88%. So the flow sends on the stage change, and the Friday rule keys off the stage change rather than the hire date. The ticket left this as an open yes-or-no for the firm before activation.

## 6. Solution

**Two flows, both Draft.** The trigger handles only "when"; the subflow handles "who" and "what", so the immediate path and the three weekend paths share one copy of the logic.

**Weekend handling.** Three scheduled paths (+1, +2 and +3 days) each check "is today Monday" before sending. A Friday change sends on Monday through the +3 path; the other two find it is not Monday and do nothing.

**Duplicate guard, deliberately doubled:**

1. A request key of `welcome-` plus the Court Case Id on the Dochly action, whose input is labelled "Idempotency Key".
2. A lookup for an existing Dochly document on the same Court Case from any of the three welcome templates.

Layer 1 is inferred from the input's label and was never tested, so layer 2 stays until someone proves layer 1 works. It matters because cases do move backwards: of 3,179 cases that had reached Attorney, 17 sat at an earlier stage.

**Recipient.** The email goes to `Case_Client_Email__c`, sent from the firm's office address (an org-wide email address).

## 7. Testing

**Verified at build, 2026-09-18, against production:**

- Both flows passed a check-only deploy (dry run) with no component failures.
- The org-wide sender address exists and allows all profiles.
- The Dochly action exists, and every input the flow uses is a real input.
- `Case_Client_Email__c` was filled on 21,299 of 21,970 hired Court Cases.

**Not verified:** the flows had never run. A dry run checks that the metadata deploys; it runs no flow logic, renders no template and sends no email. No email had been sent by this path in any org.

**Open before go-live, as listed on the ticket:**

- [ ] Fix the HV and SOS templates, which still carried attorney conditions on `Primary_Attorney__r.Username` that render nothing. This was the hard blocker: activating first would send a blank body.
- [ ] Confirm the payment-status variation renders the right text for each state.
- [ ] Confirm the embedded video shows in clients' real mail clients.
- [ ] Point the Calendly link at the Attorney Welcome Call event (DDT-1058).
- [ ] Test the request-key duplicate guard.
- [ ] Activate both flows.

How these were settled is recorded on DDT-1158. Several were not run before activation.

## 8. Step by step

1. Confirm which field really moves when a case reaches Attorney. Count recent changes on the candidate fields; do not trust the field named in the request.
2. Build the trigger flow on `Blueback_Stage__c` with an immediate path and three weekend paths.
3. Build the send logic as a subflow: recipient check, duplicate lookup, band routing, then the Dochly send action with a request key.
4. Dry-run both flows against production.
5. Deploy both as **Draft**.
6. Clear the open items in section 7 before anyone activates them (DDT-1158).

## 9. Rollback

Deactivate the trigger flow. The flows create and update nothing on the Court Case; the only side effects are the Dochly document and the email. **An email cannot be recalled**, which is why the template checks were meant to come before activation.

## 10. Known limits

- **No email for cases without a client email.** 671 hired cases had a blank `Case_Client_Email__c`. The flow stops silently: no Task, no Chatter post.
- **Weekday and timezone.** The weekday check uses the running user's timezone. All users were Central, but that is an assumption about who triggers the flow.
- **A blank `Case_Type__c` sends nothing.**
- **The band line is a build-time choice.** If the firm's view of HV and LV changes, the routing in the subflow must change with it.

## 11. Reuse for similar issues

- **Check the field the request names before building on it.** Count recent changes; a dead object is easy to name in a request because it still exists.
- **Measure a timing rule before encoding it.** The 24-hour rule looked reasonable and was wrong for most cases.
- **Ship automated outbound email as Draft,** and list exactly what must be proven before someone switches it on.
- **Keep send logic in a subflow** when several paths (immediate and scheduled) must do the same thing.
- **Do not rely on an untested idempotency setting alone.** Keep a lookup guard until the setting is proven.
