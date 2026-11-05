# Representation Agreement status never updated for Dochly signatures

| | |
|---|---|
| **Ticket** | DDT-717 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-12 |
| **Priority** | High |
| **Reported by** | Support team, 2026-08-12 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow), Dochly e-signature, S-Sign (legacy e-signature) |
| **Area** | Court cases: tracking whether the client has signed the representation agreement |
| **Related** | None recorded on the ticket |
| **Code** | PR #160 in the `ddt-salesforce` repo |

---

## 1. Summary

The Court Case field `Representation Agreement` (`Opportunity.Representation_Agreement__c`) was only ever updated by a flow that watches **S-Sign** envelopes. Once the firm moved its e-signatures to **Dochly**, nothing watched Dochly's signature requests, so the field stayed blank or `N/A` even after the client had signed.

We added a new flow on Dochly's signature request object that mirrors the S-Sign flow, step for step. The same day, the old S-Sign flow was switched off on purpose.

## 2. What was reported

Measured on 2026-08-12: **4** court cases whose Dochly signature request had reached `Completed` (the client had signed) showed the field as blank (3) or `N/A` (1), and none had a signed date. None of them had an S-Sign envelope.

At that point Dochly held **94** signature requests, **30** of them `Completed`.

Why it matters: the report `Open_Cases_without_Signed_Representation` counted these cases as unsigned, so staff were chasing signatures that already existed. It also explained why the field was being corrected by hand: one case was set to `Signed` manually while its S-Sign envelope still read `Sent`, because no automation could do it.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Dochly_Signature_Request_After_Trigger` | Record-triggered flow (new) | On `dochly__DH_Signature_Request__c`, create and update, after save. Sets `Opportunity.Representation_Agreement__c` from the Dochly status |
| `Dochly_Signature_Request_After_Trigger` | FlowDefinition | Activates version 1 |
| `S_Sign_Envelope_After_Trigger` | FlowDefinition | Deactivated (`activeVersionNumber` 0) the same day |

**Out of scope**

- **The signed date.** `Representation_Agreement_Signed__c` is still never filled in. The S-Sign flow never set it either, and filling it from `dochly__Completed_Date__c` would be a change in behaviour rather than parity.
- **Protecting `N/A`.** A case deliberately marked `N/A` is still overwritten, because the guards only protect `Sent` and `Signed`. Same as the S-Sign flow.
- **Backfilling** the cases already showing the wrong value. It is a data change and needs sign-off.

## 4. How it works (background)

Before the change:

```
S-Sign envelope created or updated  (SSign__SSEnvelope__c)
  -> S_Sign_Envelope_After_Trigger  (active, v4, unchanged since 2022-11-23)
  -> document name contains "Representation Agreement"?
  -> sets Opportunity.Representation_Agreement__c to Not Sent / Sent / Signed

Dochly signature request created or updated  (dochly__DH_Signature_Request__c)
  -> nothing
```

The field is a restricted picklist with four values: `Not Sent`, `Sent`, `Signed`, `N/A`.

## 5. Root cause

The only automation that wrote the field was `S_Sign_Envelope_After_Trigger`, which is triggered by S-Sign envelopes. Dochly writes to a completely different object, `dochly__DH_Signature_Request__c`, and no automation was listening to it. So for any case signed through Dochly, the field never moved.

The ticket's diagnosis held up: no other writer of the field was found.

## 6. Solution

### 6.1 A new flow that mirrors the S-Sign flow

`Dochly_Signature_Request_After_Trigger` is a deliberate element-for-element copy of the S-Sign flow, with the same `Representation Agreement` name filter and the same three outcomes. Nothing existing was modified.

| Dochly status | Field becomes |
|---|---|
| `Draft` | `Not Sent` |
| `Sent`, `Viewed`, `In_Progress`, `Partially_Signed`, `Declined`, `Voided`, `Expired` | `Sent` |
| `Completed` | `Signed` |

All nine values of `dochly__Status__c` are covered. They were read from the live picklist, not assumed.

### 6.2 Two differences forced by Dochly's data model

- **No Opportunity lookup.** S-Sign has a direct Opportunity lookup. Dochly links to its source through a text field, `dochly__Source_Record_Id__c`, and `dochly__Source_Object__c` is not populated in this org. So the flow runs a Get Records on `Opportunity` by that Id and stops when nothing is found. That is also what makes a Lead- or Matter-sourced request harmless: it finds no Opportunity and the flow ends. No Id-prefix parsing.
- **The name filter reads the template.** S-Sign filters on the envelope document's name. The Dochly equivalent is the template name, `dochly__DH_Template__c.Name`. The live templates were:

```
Representation Agreement + Affidavit (ESIGN)   matches
Representation Agreement (ESIGN)               matches
Affidavit (ESIGN) - 2                          correctly excluded
```

### 6.3 Shipped inactive, then activated through metadata

The flow file in git says `Draft`, so git records what was actually deployed. Activation was a separate deploy of a `FlowDefinition` pointing at version 1, which keeps the live state in git instead of only in Setup.

### 6.4 The S-Sign flow was switched off, knowingly

Running both flows meant two automations writing one field, with the last writer winning. On request, `S_Sign_Envelope_After_Trigger` was deactivated the same day. The decision rested on S-Sign volume: 9 to 22 new envelopes a day up to 2026-08-11, and **0** on 2026-08-12.

The cost was recorded at the time: **53** S-Sign envelopes created in the previous 14 days were still in flight (28 `Sent`, 18 `Viewed`, 6 `Created`, 1 `Partially Signed`). If any of those clients sign, the field will not move to `Signed`, with no error and no log.

## 7. Testing

**Validation.** The sandbox had no Dochly package in August, so validation was check-only against production: 1 component, 5 of 5 tests passing. A flow has no test coverage of its own, so an unrelated passing test class was named only because production refuses a deploy with no tests.

**Live check (2026-08-12),** on a test court case, using real signature requests:

| Request | Template | Status | Field |
|---|---|---|---|
| Request 1 | `Representation Agreement (ESIGN)` | `Completed` | moved from `Sent` to `Signed` |
| Requests 2 and 3 | `Affidavit (ESIGN) - 2` | `Sent` | untouched, as intended |
| Request 4 | Motion filing template | `Viewed` | untouched, as intended |

The affidavit and motion-filing rows are the name filter proving itself on live data.

**Still active (2026-11-05):** `Dochly_Signature_Request_After_Trigger` reads as active in production.

**Not covered:** no automated test. A Matter-sourced request changing nothing is listed as a check in the deploy doc; its result is not recorded.

## 8. Step by step

1. **Confirm the gap.** Find cases whose Dochly request is `Completed` and whose `Representation_Agreement__c` is not `Signed`.
2. **List the live Dochly template names** and confirm which ones the `Representation Agreement` filter should match.
3. **Build the flow** on `dochly__DH_Signature_Request__c`, create and update, after save, mirroring the S-Sign flow, with a Get Records on `Opportunity` by `dochly__Source_Record_Id__c`.
4. **Validate check-only against production.**
5. **Deploy the flow as `Draft`.** Query that it is deployed and inactive.
6. **Activate** by deploying a `FlowDefinition` with `activeVersionNumber` 1. Query that it is active.
7. **Test on a court case:** send a representation agreement (field should become `Sent`), sign it (`Signed`), then send a standalone affidavit and confirm the field does **not** move.
8. **Decide about the old S-Sign flow.** If you switch it off, first count the envelopes still in flight:

```sql
SELECT SSign__Status__c, COUNT(Id) FROM SSign__SSEnvelope__c
WHERE SSign__Status__c NOT IN ('Completed','Voided','Expired','Declined')
  AND CreatedDate = LAST_N_DAYS:14
GROUP BY SSign__Status__c
```

## 9. Rollback

- **Dochly flow:** deploy its `FlowDefinition` with `activeVersionNumber` 0. Safe; the field simply stops updating for Dochly signatures again.
- **S-Sign flow:** deploy its `FlowDefinition` with `activeVersionNumber` 4. One line, no code. If both are active, two automations write the same field and the last one wins.

## 10. Known limits

- **S-Sign envelopes in flight are orphaned.** 53 were in flight at the switch-off. A follow-up sweep of any that reach `Completed` was suggested; whether it ran is not recorded in the ticket.
- **The signed date stays blank,** and **`N/A` is overwritten** (see section 3).
- **The filter depends on template names.** A new representation agreement template that does not contain `Representation Agreement` in its name will be ignored. Renaming an existing one has the same effect.
- **Existing wrong values were not backfilled** in this ticket.

## 11. Reuse for similar issues

**How to recognise one:** a field that used to update by itself stops updating after the firm changes vendors (e-signature, telephony, document generation). The automation is still there, but it watches the old vendor's objects.

- **Find every writer of the field** before building a new one. Here there was exactly one, a flow on S-Sign's object.
- **Mirror rather than redesign** when the goal is parity. Copying the old flow's filter and outcomes means both systems behave the same, and the differences are listed explicitly.
- **Read the picklist values from the org** rather than guessing which statuses the vendor uses.
- **Resolve a text Id with a Get Records** on the object you expect, and stop when nothing comes back. It is safer than parsing Id prefixes.
- **Ship a flow inactive, activate it through a `FlowDefinition`,** and check the result by query, not by the deploy succeeding.
- **When switching off the old automation, count what is still in flight** and write the number down.
