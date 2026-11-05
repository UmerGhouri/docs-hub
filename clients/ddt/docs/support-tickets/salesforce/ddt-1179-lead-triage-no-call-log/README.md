# Lead Triage Existing Client fails: no call log found

| | |
|---|---|
| **Ticket** | DDT-1179 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-02 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-10-01 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC, Apex), RingCentral |
| **Area** | Intake: Lead Triage, linking a caller lead to an existing client |
| **Related** | None |
| **Code** | PR #599 and PR #603 in the `ddt-salesforce` repo |

---

## 1. Summary

On a "RingCentral Caller" lead, Lead Triage → Existing Client → "Temporary / Do Not Save" refused with **"No call log found for this lead"**. The component treated "this lead has no call log" as a fatal error. But call log rows are only created when a rep logs a call from the softphone; the webhook that creates RingCentral Caller leads never writes one. So the option failed on most of the leads it exists for. Now "no call log" means "nothing to relink", and the lead is linked to the client and removed as normal.

## 2. What was reported

On 2026-10-01 a staff member ran Lead Triage → Existing Client on a RingCentral Caller lead whose phone number already belonged to a client. "Temporary / Do Not Save" showed the error toast and did nothing. Only "Replace Primary Phone" went through, and it replaced the client's number with the identical number.

How general it was, measured in production on 2026-10-02:

| Measure | Count |
|---|---|
| Leads created by the RingCentral webhook, last 90 days | 3,476 |
| ...of which have no `RingCentral_Call_Log__c` row | 2,101 |
| Same, last 7 days / with no call log | 283 / 222 |
| Converted webhook leads, last 90 days / with no call log | 1,027 / 386 |

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `existingClient` | LWC | `updateRingCentral()` treats `NO_LOG` like `SUCCESS` |
| `existingClient/__tests__/existingClient.test.js` | Jest test | New; the bundle's first test file |
| `docs/deploys/ddt-1179-existing-client-no-call-log.md` | Deploy doc | Review and deploy record (PR #603 recorded the production deploy) |

**Out of scope**

- **Apex.** `ExistingClientController.updateRingCentralClient` still returns `NO_LOG`, and its test still asserts it.
- **`personalContactToClient` and `caseRelatedParty`.** They carry the same `NO_LOG` refusal. Left for a separate scope decision.
- **Converted leads.** Nothing in Lead Triage checks whether a lead is already converted. Whether it should is a product decision.
- **"Add as Secondary" when the number is already on the client.** It still copies it into `Phone_Secondary__c`. Harmless, not in the acceptance criteria.

## 4. How it works (background)

```
Inbound call → RingCentral webhook creates a "RingCentral Caller" lead
               (the call itself is recorded on Call_Session__c, not RingCentral_Call_Log__c)

Lead Triage → Existing Client → pick the client → choose an option → Save
  Replace Primary Phone   → writes the phone, then links the lead and deletes it
  Add as Secondary        → writes the secondary phone, then links and deletes
  Temporary / Do Not Save → calls updateRingCentralClient to move call logs to the client
                            → on SUCCESS: linkLeadToClientAndDelete
                            → on NO_LOG:  error toast, stop          (the bug)
```

A `RingCentral_Call_Log__c` row exists only when a rep logs a call from the RingCentral softphone.

## 5. Root cause

`existingClient.js` treated the controller's `NO_LOG` reply as fatal and never reached `linkLeadToClientAndDelete`. For webhook-created leads, `NO_LOG` is the normal case, because the webhook never writes a call log row.

The pre-check was also redundant: `LeadTriageUtil.linkLeadToClientAndDelete` already moves every call log on the lead to the client's Contact before deleting the lead.

**Two ticket claims the code did not support:**

- "Add as Secondary" cannot produce this toast. Only the Temporary branch calls the call-log method.
- The reported lead and client were not two independent records. The lead had been **converted** about 20 minutes earlier, and the client account was created by that conversion. Lead Triage ran on an already-converted lead because nothing checks for it. No further action was needed on that record.

## 6. Solution

In `existingClient.js` `updateRingCentral()`:

- `'NO_LOG'` is handled the same as `'SUCCESS'`: proceed to `linkLeadToClientAndDelete`, show the success toast, navigate to the client.
- Any other reply is still refused, with a toast naming the reply.
- The inner promise is now returned, so the spinner stays up until the delete finishes.

Why fix the component and not the Apex: the Apex reply is accurate ("there were no logs to move"). The component was wrong to treat it as failure. Changing only the browser side kept the fix to one hunk, with no Apex deploy.

The bundle was first reformatted with Prettier in its own commit (no logic change), so the fix commit shows only the real change. Three existing `setTimeout` calls gained the repo's standard eslint-disable marker because the lint check now covered the file.

## 7. Testing

`existingClient.test.js`, five cases driving the real markup (type a name, pick the client, pick a radio, Save):

1. Temporary + `NO_LOG` → link-and-delete is called, one success toast, no "No call log found" toast. **This is the ticket.**
2. Temporary + `SUCCESS` → unchanged behaviour.
3. Temporary + unknown reply → not deleted, one error toast naming the reply.
4. Temporary + the call rejects → not deleted, "Something went wrong".
5. Replace Primary Phone → never calls the call-log method, writes the phone, deletes.

5 of 5 pass. Against the pre-fix component, cases 1 and 3 fail and 2, 4 and 5 pass, so the suite tells the fix from the bug.

**Sandbox screen check:** a test client and a RingCentral Caller lead with the same phone and no call logs were created in a sandbox. Temporary / Do Not Save gave a success toast and moved to the client; the lead was deleted and the client's phones were unchanged. On the pre-fix bundle the same step gave the error toast.

**Production:** validated with `ExistingClientControllerTest` and `LeadTriageUtilTest` (20 of 20), then deployed on 2026-10-02.

**Not covered:** CI does not render the component in a browser, and the sandbox has no real webhook leads. A production screen check on a live, unconverted RingCentral Caller lead was left with the requester.

## 8. Step by step

1. Reproduce: find a webhook lead with no call log.

    ```sql
    SELECT Id, CreatedDate FROM Lead
    WHERE CreatedBy.Name = '<webhook user>'
      AND IsConverted = false
      AND Id NOT IN (SELECT Lead__c FROM RingCentral_Call_Log__c)
    ORDER BY CreatedDate DESC LIMIT 3
    ```

2. Read `updateRingCentral()` in `existingClient.js` and confirm the `NO_LOG` branch stops before the link.
3. Retrieve `existingClient` from production and confirm it matches `main`.
4. Reformat the bundle in its own commit if it is not Prettier-clean.
5. Change the `NO_LOG` branch to proceed exactly as `SUCCESS`; return the inner promise.
6. Add the five jest cases; run them against the old component to confirm cases 1 and 3 fail.
7. Deploy to a sandbox and run the Temporary path in the browser on a fixture lead.
8. Validate against production, get approval, deploy (LWC only).
9. Check with the lead's owner, then run the Temporary path on a real unconverted lead.

## 9. Rollback

Redeploy `existingClient` from the version before PR #599. Safe; no Apex, metadata or data changed. It brings the refusal back.

## 10. Known limits

- The same refusal is still in `personalContactToClient` and `caseRelatedParty`.
- Lead Triage still runs on converted leads.
- Every option on this tab deletes the lead, so check with the lead's owner before testing on a real one.

## 11. Reuse for similar issues

**How to recognise one:** a UI action fails with "no X found" on records that are never expected to have X.

- **Ask what creates the record being checked for.** Here only manual softphone logging created call logs, so webhook leads never had one.
- **"Nothing to do" is not "failed".** Treat an empty result as success when the next step already handles it.
- **Measure how general it is** with a count in production before deciding it is a one-off.
- **Check the ticket's record history.** The reported lead had already been converted, which changed the story.
- **Search for the same pattern in sibling components**, and ticket them separately.
