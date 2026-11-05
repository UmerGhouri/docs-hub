# PI contract sending moved from DocuSign to Dochly

| | |
|---|---|
| **Ticket** | DDT-1070 |
| **Project** | DocuSign Template Migration (replace with Dochly) |
| **Status** | Live since 2026-09-24 |
| **Priority** | High |
| **Reported by** | Firm operations, 2026-09-21 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex / Flow / LWC / Custom Metadata), Dochly e-signature, DocuSign (retired) |
| **Area** | PI intake: sending the contingency fee agreement and converting the signed Lead into a Matter |
| **Related** | DDT-1072 (template shape), DDT-1159 (text sender and prompt wording), DDT-1160 (signing date), DDT-1071 (DocuSign retired), DDT-1377 and DDT-1287 (side effects of the new running user), DDT-1083 (referrer gate), DDT-721 (sender credit) |
| **Code** | PR #458 and PR #474 in the `ddt-salesforce` repo |

---

## 1. Summary

The PI contract (the contingency fee agreement) was sent through DocuSign, and a completed DocuSign signature is what turned a PI Lead into a hired Matter. This ticket moved the whole path to Dochly: sending, voiding, resending, status tracking, filing the signed PDF, and the Lead-to-Matter conversion.

It was built next to the DocuSign path rather than over it, so contracts already out on DocuSign could still finish. Real signing in the sandbox found eight defects that no automated check caught, most of which would have failed silently. All were fixed before or with the production deploy. Linear closed the ticket on 2026-09-23; the new Apex classes were created in production on 2026-09-24.

## 2. What was asked for

The ticket was a migration, not a bug report. Its central warning: **this is the PI signup path, not a document swap.** If the chain breaks, a client signs and nothing happens: no Matter, no signed PDF on file, and only a Lead sitting at `Signed`. At about 15 signings a month, that could go unnoticed for weeks.

Constraints set in the ticket:

- **No new scheduled job.** Production was at 87 of 100 scheduled Apex jobs. If Dochly updates its own status records, delete the DocuSign poller rather than port it.
- **Dochly licences:** 41 of 42 used when the ticket was written (42 of 42 by the time the PR was written).
- **Cut over in order:** build alongside, prove with one real contract, turn off DocuSign sending with the existing switch, leave the DocuSign poller running so in-flight envelopes still convert, then retire DocuSign.
- **Prove it end to end:** a real send, a real signature, a real Matter, both PDFs on file, plus the decline and void paths and one deliberate failure.

## 3. Scope

**In scope**

| Component | Type | What it does |
|---|---|---|
| `DochlyContractService` | Apex class (new) | Send, void and resend through Dochly's invocable actions. No HTTP callouts |
| `DochlyContractSignedInvocable` | Apex class (new) | Called by the flow. Maps Dochly status onto `Lead.Contract_Status__c` and queues the conversion once the client has signed |
| `DochlyContractConvertQueueable` | Apex class (new) | Runs `convertLeadToHiredMatter` from the saved payload, sets `Hire_Date_Time__c` to the client's signing time, and links the executed PDF and completion receipt to the Lead and the Matter |
| `Dochly_PI_Contract_Status` | Record-triggered flow (new) | After save, update only, on `dochly__DH_Signature_Request__c`. Runs only for requests this path sent |
| `Lead.Contract_Incident_Description__c` | Field (new, Long Text Area) | The "contract will read" incident sentence. Before this it existed only inside the DocuSign payload |
| `dochly__DH_Signature_Request__c.PI_Contract_Payload__c` | Field (new, on the managed object) | Holds the wizard payload the conversion needs. Replaces the DocuSign envelope's payload field |
| `dochly__DH_Signature_Request__c.PI_Contract_Sent_By__c` | Field (new, on the managed object) | Who pressed Send. The conversion runs as an automated user, so this is the only place the real sender is kept (DDT-721) |
| `Dochly_Template_Map__mdt` + 10 records | Custom metadata (new) | Language × fee schedule → Dochly template. 6 rows filled, 4 deliberately blank |
| `DocuSign_Envelope_Payload_Access` | Permission set (extended) | Class access and field access for the above. Extended because the senders already hold it, so there was no assignment step |
| `convertToHiredMatter`, `leadTriage` | LWC | Point at `DochlyContractService` instead of the DocuSign service; toast wording |

Tests: `DochlyContractServiceTest` and `DochlyContractSignedInvocableTest`.

**Out of scope**

- **The DocuSign Apex, poller and config.** Left untouched so envelopes already out could finish. Retired later under DDT-1071.
- **The template redesign** (2 templates instead of 6). That is DDT-1072. This PR built the six-template, Lead-based shape that existed in Dochly at the time.
- **Template content edits.** Dochly templates are records, not metadata, so changes to their bodies could not ride in the PR. They were listed as manual steps (section 8).

## 4. How it works (background)

Before, in production:

```
convertToHiredMatter LWC -> SendContractService.sendContract
    -> creates the DocuSign envelope (name, DOB, incident sentence pushed as values)
    -> saves the whole wizard payload on DocuSign_Envelope__c
    -> Lead.Contract_Status__c = 'Sent'

DocuSignStatusPoller   (a branch of GlobalMinuteScheduler)
    -> client signed  -> Lead 'Signed', convertLeadToHiredMatter runs from the payload
    -> both signed    -> DocuSignPDFDownloader files the signed PDF and certificate
```

After:

```
convertToHiredMatter LWC
    -> call 1: prepareLeadForContract   (validate, write the form values to the Lead, commit)
    -> call 2: sendContract             (pick template, generate in Dochly, send, mark Lead 'Sent')

Client signs on Dochly's public signing site
    -> Dochly updates dochly__DH_Signature_Request__c
    -> flow Dochly_PI_Contract_Status   (only for requests carrying PI_Contract_Sent_By__c)
    -> DochlyContractSignedInvocable    (one Lead status write; queue the rest)
    -> DochlyContractConvertQueueable   (Lead -> Matter, then file the PDF and receipt)
```

Status mapping:

| Dochly status | `Lead.Contract_Status__c` |
|---|---|
| `Sent` | Sent |
| `Viewed`, `In_Progress` | Viewed |
| `Partially_Signed`, `Completed` | Signed |
| `Declined` | Declined |
| `Voided`, `Expired` | Voided |

Dochly distinguishes Viewed from Sent, so the `Viewed` state survived the move (one of the ticket's open questions). A client who has signed outranks a request still sitting at `In_Progress`, so the Lead reads `Signed` during the wait for the countersignature.

## 5. Root cause

Not applicable as a single cause: this was a planned migration. The design problems that had to be solved, and the defects found on the way, are worth recording because each one changes how Dochly differs from DocuSign.

**Design differences that shaped the build**

1. **Dochly merges from the record; DocuSign merged what we sent.** DocuSign received the values in the send call. Dochly reads `Name`, date of birth and the incident sentence off the Lead at generation time. If the Lead is not updated first, the contract shows the old values, with no error.
2. **The signing transaction belongs to Dochly.** In production, Dochly writes `Viewed` as the Dochly Sign Site Guest User and `Completed` as the Automated Process user. Anything the flow runs inline runs inside the client's signing. An exception there rolls back the client's signature, and the Matter trigger chain was already known to hit `Too many queueable jobs added to the queue` when run this way.

**Defects found by signing real documents in the sandbox (2026-09-23)**

| # | Symptom | Cause |
|---|---|---|
| 1 | The whole send failed in the sandbox | The `createAsDraft` input does not exist in every Dochly version |
| 2 | Send refused: "This template has fields for signer 2, but only 1 recipient was added" | Passing signer 1 explicitly makes Dochly drop the template's default recipients, including the countersigner |
| 3, 4 | Lead status never updated; conversion reported "Lead not found" for a Lead that existed. Nothing was logged | The signing runs as the site guest user, which has no access to Leads. The failed write and its error log were both swallowed |
| 5 | Logged "Request is Completed but carries no final PDF" | Dochly writes the executed PDF a moment **after** flipping the status to `Completed` |
| 6 | Logos broke for the client but not for staff | Template images pointed at static resources, which need a Salesforce session. Staff have one; a client signing as a guest does not |

**Defects found after PR #458 merged (fixed in PR #474)**

| # | Symptom | Cause |
|---|---|---|
| 7 | Every merged field on the PDF was blank except the name | Dochly renders the PDF through a Visualforce page in a separate request, which cannot see uncommitted changes. PR #458 wrote the Lead and generated in one transaction, so only values already committed (the name, from when the Lead was created) appeared |
| 8 | The Matter was created only when the firm countersigned, not when the client signed | The conversion waited for `Partially_Signed`, a status Dochly never writes: **0 of 595** production requests had ever held it. One sandbox request never converted at all because nobody countersigned |

**Why defect 7 was missed three times:** the request's own `Rendered_Body_HTML__c` field always showed the right values, because it is written later, after commit. Checking it "confirmed" the bug was absent. The PDF is what the client sees, and it disagreed.

## 6. Solution

### 6.1 Write the Lead, commit, then generate

The send is split into two Apex calls from the LWC: `prepareLeadForContract` validates the form and writes the values to the Lead; `sendContract` runs in a second transaction, after the first has committed, and asks Dochly for the document. Validation still runs in the first call, so a bad form is refused before anything is written. Signer 1's name and email are also passed explicitly, so who receives the email never depends on the order.

### 6.2 Convert when the client signs, read from the signer

The conversion is triggered when **sign order 1 (the client) has a signed date**, not by the request's overall status. `Completed` is kept as a second route in, so a fully executed contract can never end up with no Matter. A request that is voided or declined stays dead even if a signature was collected before it died.

### 6.3 Queue the conversion, never run it inline

The invocable makes one Lead field write and queues everything else. Conversion failures are written to `API_Error_Log__c` and never thrown, so a failure cannot roll back the client's signature. If no queueable slot is left in the transaction, that is logged too. No new scheduled job was added.

### 6.4 Pick the template from custom metadata, per org

`Dochly_Template_Map__mdt` maps language and fee schedule to a Dochly template. A Dochly template is a record, so its Id is different in every org. Each row carries two columns, a production Id and a sandbox Id, chosen at runtime on `Organization.IsSandbox`. Without that, a sandbox deploy would pass a production Id and fail with the `templateId` error seen in DDT-718.

Four rows are deliberately blank: 40/45 and Other, in both languages. Those schedules **refuse to send** rather than send a contract with the wrong fee numbers.

### 6.5 Keep the countersigner on the template

Because passing signer 1 drops the template's default recipients, the service copies signers 2 and 3 back off the template onto the call. The template stays the single place the countersigner is defined. An unresolved `{{placeholder}}` is refused rather than emailed literally.

### 6.6 The other fixes

- **Older Dochly versions:** if `createAsDraft` is not available, the service falls back to a direct email-only send rather than refusing. Without this, no PI contract could be sent at all after a sandbox refresh.
- **Record access for the guest user:** the status update and the conversion were changed so they no longer depend on the running user's record access. This was flagged in the PR as a security widening that needed its own review.
- **Late PDF:** the flow's entry criteria became `1 AND (2 OR 3)`, so it also fires when the executed PDF arrives. Both passes are idempotent.
- **Images:** templates point at public Documents served through `servlet.ImageServer`, the pattern the org's working templates already used, instead of static resources.
- **Texts:** the service sets the client's mobile on signer 1 and turns on Dochly's SMS delivery. A failure there is logged and the send continues by email only, the same rule DocuSign's fallback followed.

### 6.7 Things carried over so nothing was lost

- **The DDT-1083 referrer gate.** It had landed on the DocuSign send path while this work was open. Since every PI send now went through the new service, it was copied to the same point (before template choice, before any Lead write). Leaving it out would have switched the gate off silently.
- **Void and resend safety (from the AI review).** Void & Edit could have reset a Lead whose contract was still live on DocuSign, leaving two retainers in front of one client. Void and resend now check for a live DocuSign envelope first and refuse with a message the LWC does not swallow.
- **Ownership filter.** Void and resend only ever touch requests this path sent (`PI_Contract_Sent_By__c` set), never a representation agreement or fee waiver on the same Lead.

## 7. Testing

**Apex tests.** `DochlyContractServiceTest` and `DochlyContractSignedInvocableTest` stub every Dochly action, so they never touch the managed package. They pin the order of writes, the refusals, the saved payload, every status mapping, the ownership filter and the void-message wording. A test that a signed contract produces a Matter was added after the AI review pointed out that both queueable tests only covered skip paths.

| Run | Components | Tests |
|---|---|---|
| PR #458, check-only against production, 2026-09-23 | 26 of 26 | 28 of 28 passing |
| PR #474, dry run against production, full Dochly component set | 27 of 27 | 36 of 36 passing |

Coverage after PR #474: 77.2%, 88.8% and 80.6% across the three new classes.

**What those runs did not cover:** they compiled the Apex, ran tests with Dochly stubbed and confirmed the metadata deploys. They generated no document, sent no email, signed nothing and ran no LWC code. Every one of defects 1 to 8 passed every automated check.

**Sandbox signing.** Real documents were sent and signed in the sandbox on 2026-09-23 and 2026-09-24. That is where all eight defects were found. Defect 8 was measured on three sandbox requests: client signed and converted 19 minutes later; client signed and converted 67 minutes later; client signed and never converted, because nobody countersigned.

**CI static analysis** flagged an empty `catch` block in the queueable. It was real: a malformed file Id would have meant the executed PDF silently never reached the Lead or the Matter. Fixed by logging and skipping.

**Live state (2026-11-05):**

```sql
SELECT dochly__Status__c, COUNT(Id) FROM dochly__DH_Signature_Request__c
WHERE PI_Contract_Sent_By__c != null GROUP BY dochly__Status__c
```

| Status | Requests |
|---|---|
| Completed | 14 |
| Voided | 12 |

**Not recorded in the ticket:** the results of the production end-to-end checks the ticket required (one real send per language, both PDFs opened, decline and void paths, one deliberate failure, and the DocuSign in-flight count at cutover).

## 8. Step by step

1. **Build the Dochly path next to DocuSign.** Change nothing live except the LWCs.
2. **Copy the templates to the sandbox** and fill both Id columns in `Dochly_Template_Map__mdt`.
3. **Sign real documents in the sandbox,** on a phone, as a client with no Salesforce session. Open the generated PDF, not the request's rendered HTML.
4. **Validate against production** with the two new test classes.
5. **Get the PR approved and deploy.** From this point new sends go through Dochly; there is no runtime toggle.
6. **Do the manual steps that are not metadata,** in production, after the deploy (the new field must exist first):
   1. Assign the Dochly package licence to the signing site's guest user. Without it the signing page shows a bare "Authorization Required".
   2. Create public Documents for the four template logos.
   3. In every template body, replace each static-resource image with the matching `servlet.ImageServer` link.
   4. Replace the `{{MVA_Injuries__c}}` token with `{{Contract_Incident_Description__c}}`. `MVA_Injuries__c` is the Injuries picklist, not the incident sentence.
   5. On the one Spanish template with no incident token, replace the empty spacer after the incident line with the token block its siblings use.
   6. Do not change the templates' default recipients; the countersigner stays as is.
   7. Open a generated PDF in a private browser window. The images only fail for someone without a session.
7. **Send one real contract end to end.** Confirm the Matter exists, `Hire_Date_Time__c` equals the client's signing time, and both the PDF and the receipt are on the Lead's and the Matter's Files.
8. **Turn off DocuSign sending** with `Sends_Enabled__c = false` on the DocuSign production config.
9. **Leave the DocuSign poller running** until envelopes in flight finish. There were 3 inside its window on 2026-09-22.
10. **Retire DocuSign** afterwards. Under DDT-1071, on 2026-10-19, the switch was set to false and the poller taken off the scheduler.

## 9. Rollback

- **Before DDT-1071:** redeploy `convertToHiredMatter` and `leadTriage` from before the change, and sends return to DocuSign. Everything else is inert without the LWCs, because the flow only acts on requests carrying `PI_Contract_Sent_By__c`.
- **Since 2026-10-19:** DocuSign sending is switched off and its poller removed from the scheduler, and the unused DocuSign managed package was uninstalled the same day (DDT-1071). There is no DocuSign path left to fall back to, so a problem has to be fixed forward.
- **Do not delete the new fields** on the managed object. It is a manual step with nothing to gain.

## 10. Known limits

- **The conversion now runs as Dochly's guest or automated user.** Anything that relied on the running user's defaults changed with it. DDT-1377 (Matters saved with no record type) and DDT-1287 (tasks assigned to the guest user) both came from this.
- **40/45 and Other fee schedules refuse to send** until those template rows are filled. DDT-1072 records 42 Matters on 40/45.
- **The countersigner is defined on each template.** There is no fallback if that person is unavailable; DocuSign envelopes had already waited weeks on a countersignature.
- **Non-US mobiles get email only, silently** (found under DDT-1159).
- **`Lead.Causes_of_Action__c` is a dependent picklist.** If the Matter Type and Accident Type chosen in the wizard disagree, the send fails with a raw `INVALID_OR_NULL_FOR_RESTRICTED_PICKLIST` message. Pre-existing on the DocuSign path too; not changed here.
- **The DocuSign service class cannot simply be deleted.** Its payload type is what the saved Dochly payloads are written against, so removing it would stop the conversion (recorded under DDT-1071).
- **The permission set name is a misnomer** (`DocuSign_Envelope_Payload_Access`). It is still the one that grants the Dochly contract fields.

## 11. Reuse for similar issues

**How to recognise the risk:** you are replacing the vendor behind a process where a vendor event (a signature, a payment, a call) creates a record downstream. The failure mode is silence: the vendor step succeeds and the downstream record never appears.

- **Build alongside, cut over in order,** and keep the old status checker running until everything already in flight has finished.
- **Know whether the new vendor merges from the record or from the call.** If it reads the record, write and commit the record first. If it renders through a separate request, uncommitted data is invisible to it.
- **Check the artifact the customer sees,** not a field that summarises it. Here the rendered HTML was right and the PDF was wrong.
- **Find out which user the vendor runs your automation as,** and design for a guest with no record access and no session: record access, images, record types, owners.
- **Never do heavy work inside a vendor's transaction.** Queue it, log failures, and never throw back into the vendor's save.
- **Read the vendor's real status history before gating on a status.** A picklist value can be active and never written.
- **Map external template Ids per org** in custom metadata, never as a single hardcoded value.
- **Treat a green deploy as proof that the code compiles,** and list in the same sentence what it did not exercise.
