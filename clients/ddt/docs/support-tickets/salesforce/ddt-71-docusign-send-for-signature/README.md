# Send for Signature through DocuSign: send, sign, convert, void

| | |
|---|---|
| **Ticket** | DDT-71, DDT-190, DDT-191, DDT-192, DDT-193, DDT-194 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-17. Retired 2026-10-19 (DDT-1071), replaced by Dochly (DDT-1070) |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-06-01 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC, custom metadata, scheduled Apex), DocuSign eSignature API |
| **Area** | PI intake: sending the contract from the Lead and turning a signed Lead into a client and Matter |
| **Related** | DDT-721 (credit the sender as Matter intake owner), DDT-768 (poller queueable limit), DDT-1070 (move to Dochly), DDT-1071 (DocuSign retired) |
| **Code** | Branch `feature/pi-contract-sender-backend`, merged to `main` on 2026-06-17 in the ddt-salesforce repo |

---

## 1. Summary

Intake staff can send a PI contract for signature from the Lead with one button. The contract is a DocuSign template chosen by the client's language and fee schedule, pre-filled from the Lead. A scheduled poller then reads the envelope's status from DocuSign and writes it back to the Lead. When the client signs, Salesforce creates the client (Person Account), the incident and the Matter automatically. When the firm countersigns, the signed PDF and the certificate of completion are attached to both the Lead and the Matter. A Void button cancels an envelope that has not been signed.

> **This integration is no longer in use.** The Lead screens moved to Dochly on 2026-09-23 (DDT-1070), and the DocuSign send switch and poller were turned off on 2026-10-19 (DDT-1071). The page is kept because the same pattern, and some of the same Apex, carries the Dochly flow.

## 2. What was reported

These were feature requests, not faults.

- **DDT-71 (2026-06-01):** umbrella request: let users send documents for signature through DocuSign from Salesforce, track status, and get the signed copy back on the record.
- **DDT-190 (2026-06-18):** map the template fields so a send needs no manual entry, from a "Send" action on the Lead.
- **DDT-191 (2026-06-18):** when the client signs, convert the Lead to a client and create the Matter.
- **DDT-192 (2026-06-18):** attach the signed document to both the Lead and the Matter.
- **DDT-193 (2026-06-18):** update Lead fields when the document is sent, and as its status changes.
- **DDT-194 (2026-06-18):** handle voided envelopes, whether voided by the firm or by DocuSign.

DDT-190 to DDT-194 were written as the detailed pieces of DDT-71 and closed the same day, after the feature was already live.

## 3. Scope

**In scope**

| Component | Type | Role |
|---|---|---|
| `SendContractService` | Apex class | `sendContract`, `voidEnvelope`, `resendEnvelope`, called from the Lead screens |
| `DocuSignStatusPoller` | Scheduled and batch Apex | Reads each open envelope's status from DocuSign and updates the Lead |
| `DocuSignPDFDownloader` | Queueable Apex | Downloads the signed PDF and certificate, attaches them to Lead and Matter |
| `HiredMatterConvertController` | Apex class (changed) | Creates Person Account, incident and Matter from the stored send payload |
| `DocuSignConfig`, `DocuSign_Config__mdt` | Apex and custom metadata | Account settings, anchors, email text, and the `Sends_Enabled__c` switch |
| `DocuSign_Template_Map__mdt` | Custom metadata, 10 records | Language × fee schedule → template |
| `DocuSign_Envelope__c` | Custom object | One row per envelope: status, Lead, sender, stored payload, attached file Ids, last error |
| `Lead.Contract_Status__c`, `Contract_Sent_Date__c`, `Wanted_Status__c` and others | Lead fields | Status and data captured on send |
| `leadTriage`, `convertToHiredMatter` | LWC | Send for Signature form, the awaiting-signature card, Void and Resend buttons |
| `DocuSign_Envelope_Payload_Access` | Permission set | FLS on the stored payload field |
| `SendContractServiceTest`, `DocuSignStatusPollerTest`, `DocuSignPDFDownloaderTest`, `DocuSignConfigTest`, `DocuSignTestMock` | Apex tests | See section 7 |

**Out of scope, or built differently from the tickets**

- **DocuSign Connect (webhook).** The tickets assumed one. A scheduled poller was built instead; its header says it "replaces the role of a DocuSign Connect webhook in environments where one isn't set up yet".
- **The DocuSign managed package.** The firm had it installed, but this integration does not use it. It calls DocuSign from its own Apex.
- **The field names in the tickets** (`Agreement_Status__c` and so on). The built field is `Lead.Contract_Status__c`.
- **Chatter posts and notifications** on send, sign or void. None of the three classes posts to Chatter or sends a notification.
- **A free-text void reason.** The Salesforce Void button uses a fixed reason (see 6.4).
- **Other objects.** Only PI Leads; nothing on Court Cases or existing Matters.

## 4. How it works (background)

```
Lead (PI) -> Send for Signature form (convertToHiredMatter)
  -> SendContractService.sendContract
       picks the template from DocuSign_Template_Map__mdt (language x fee schedule)
       creates the envelope in DocuSign with the merge fields filled
       creates DocuSign_Envelope__c (status 'sent'), stores the form as JSON on it
       updates the Lead: Contract_Status__c = Sent, Contract_Sent_Date__c = now, ...
  -> leadTriage parks the Lead on an "awaiting signature" card (Resend / Void / Refresh)

DocuSignStatusPoller (scheduled)
  for each envelope in 'sent', 'delivered' or 'signed_by_client', sent in the last 30 days:
    envelope voided or declined     -> Lead Voided / Declined (terminal)
    a signer opened it              -> envelope 'delivered', Lead Viewed
    the CLIENT has signed           -> envelope 'signed_by_client', Lead Signed,
                                       create Person Account + incident + Matter
                                       (Matter hire time = the client's sign time)
    every signer done (firm too)    -> envelope 'completed',
                                       queue DocuSignPDFDownloader

DocuSignPDFDownloader
  download the combined signed PDF + certificate of completion
  save both to the Lead's Files
  link the same files to the Matter (one file, two links)
```

Lead `Contract_Status__c` values: Not Sent, Sent, Viewed, Signed, Declined, Voided.

## 5. Root cause

Not applicable: these were feature requests, not faults.

Two design facts explain choices in section 6:

- **`Database.LeadConvert` could not be used.** Per the commit history, a workflow lock kept failing it. The conversion creates the Person Account, incident and Matter directly instead, links the Matter to the Lead through `Matter__c.Lead__c`, and then marks the Lead converted as a separate step.
- **The client and the firm both sign.** The client signs first; the firm countersigns later. The fully signed PDF only exists after the second signature, but the firm wanted the Matter created the moment the client signed.

## 6. Solution

### 6.1 Send (DDT-190, DDT-193)

- **Template choice is data, not code.** `DocuSign_Template_Map__mdt` has one row per language (English, Spanish) and fee schedule (30/35, 33/33, 33/40, 40/45, Other). `resolveTemplateId` looks the row up. If the row has no template, the send stops with "This template does not exist." and nothing is created. This replaced a hardcoded map that quietly fell back to the 33/40 template for a 40/45 selection, which would have put the wrong fees on a signed contract.
- **The form is stored on the envelope** (`Payload_Json__c`), so the poller can run the conversion later without asking anyone to re-enter it.
- **The Lead is updated on send:** `Contract_Status__c = 'Sent'`, `Contract_Sent_Date__c`, `Wanted_Status__c = 'Wanted'`, plus the address, incident details, language and other fields captured on the form. State and country names are normalised to codes ("Illinois" → `IL`) before writing, because a long name in a state-code field fails the whole Lead update.
- **SMS notification with a fallback.** The envelope asks DocuSign to text the signer too; if DocuSign rejects the SMS part, the send is retried email-only.

### 6.2 Client signs → client and Matter (DDT-191)

The poller reads each recipient's status. When the recipient with the **Client** role has signed, it sets the envelope to `signed_by_client`, flips the Lead to Signed, and runs the conversion with the client's actual sign time as the Matter's hire time. It keeps polling `signed_by_client` envelopes until the firm countersigns.

**Idempotent:** if the Lead already has a Matter, the conversion returns the existing one rather than creating a second.

### 6.3 Both signed → files on Lead and Matter (DDT-192)

When every signer is done, the poller queues `DocuSignPDFDownloader`, which:

1. downloads the combined signed PDF and the certificate of completion;
2. saves them to the Lead's Files (`Signed_Document.pdf`, `Certificate_Of_Completion.pdf`);
3. links the **same** files to the Matter with a second `ContentDocumentLink`, so there is one copy in storage and it shows on both records;
4. records the file Ids on the envelope, and skips the download if they are already set, so a repeat run does not attach twice.

### 6.4 Void (DDT-194)

There are two paths, and they leave the Lead in different states:

| Who voids | What happens |
|---|---|
| **Firm user**, with the Void button on the awaiting-signature card | `voidEnvelope` voids it in DocuSign with the reason "Edited by sender before signing", sets the envelope to `voided`, and **resets the Lead to Not Sent** with the sent date cleared, so the contract can be corrected and sent again |
| **DocuSign** (or someone voiding in DocuSign directly) | The poller sees the envelope is voided and sets the Lead to **Voided** |

The resolution comment on DDT-194 says the Lead "flips to Voided" in both cases. The code resets it to Not Sent when the firm voids from Salesforce.

**Stale events after a void:** a voided envelope drops out of the poller's query (it only reads `sent`, `delivered` and `signed_by_client`), so a later "signed" status for it is never read and cannot trigger a conversion.

### 6.5 The off switch

`DocuSign_Config__mdt.Sends_Enabled__c` gates sending. It was set to `false` on 2026-10-19 under DDT-1071.

## 7. Testing

Per the ticket, 21 of 21 Apex tests passed in the sandbox on 2026-06-02, and the full sign flow was tested end to end in production on 2026-06-17: Lead → envelope → client signs → Lead Signed → Account, incident and Matter created → firm countersigns → PDF and certificate attached.

All DocuSign callouts are mocked by `DocuSignTestMock`. Examples of what the tests prove (some were added later, under DDT-721 and DDT-768):

| Test | Proves |
|---|---|
| `sendContract_HappyPath_English3035_CreatesEnvelopeAndUpdatesLead` | A send creates the envelope row and updates the Lead |
| `sendContract_TemplateBlank_ThrowsThisTemplateDoesNotExist` | A map row with no template blocks the send |
| `voidEnvelope_WithActiveEnvelope_MarksVoidedAndResetsLead` | The Void button voids the envelope and resets the Lead |
| `poller_clientSignedOnly_flipsToSignedByClient` | Client-only signature moves the envelope to `signed_by_client` |
| `poller_anyDelivered_flipsLeadToViewed` | An opened envelope sets the Lead to Viewed |
| `poller_envelopeVoided_setsTerminalVoidedOnLead` | A DocuSign-side void sets the Lead to Voided |
| `downloads_and_attaches_signed_pdf_and_certificate` | Both files are saved |
| `skips_when_signed_document_already_set` | A second run does not attach again |
| `existing_matter_links_files_without_re_creating` | Files link to an existing Matter without a second conversion |

**Not covered:** a real DocuSign envelope in an automated test; nothing proves the template field placement except a real send.

## 8. Step by step

How the feature was put together, in order:

1. **Create the data model:** `DocuSign_Envelope__c`, the Lead contract fields, `DocuSign_Config__mdt` and `DocuSign_Template_Map__mdt`. Grant FLS on new fields through a permission set.
2. **Fill the template map:** one record per language × fee schedule. Leave the template blank for any combination that has no template, so the send is blocked rather than falling back.
3. **Build `SendContractService.sendContract`:** resolve the template, create the envelope, store the payload, update the Lead.
4. **Build the poller** with the state machine in section 4, and schedule it.
5. **Build the downloader** and make it check for existing file Ids first.
6. **Change the conversion** to create Account, incident and Matter directly, and return an existing Matter if there is one.
7. **Wire the LWCs:** the send form, the awaiting-signature card, Void, Resend and Refresh.
8. **Write the tests** with a callout mock, including replay and void cases.
9. **Send a real envelope in production** to a test signer and walk it through client signature and countersignature, checking the Lead status, the new Matter and the attached files at each step.

To check an envelope's progress on a Lead, read its `DocuSign_Envelope__c` row (`Status__c`, `Last_Error__c`) alongside `Lead.Contract_Status__c`.

## 9. Rollback

Done for real under DDT-1071, in this order: set `Sends_Enabled__c` to `false`, then take the poller off the scheduler. Nothing was deleted. The Lead screens had already moved to Dochly. Envelopes that were in progress stop being read once the poller is off.

## 10. Known limits

- **Retired.** Sending is switched off and the poller no longer runs. Seven old envelopes were still in open states when it was switched off (per DDT-1071), four of them test sends.
- **Polling, not push.** Status reaches Salesforce only when the poller runs. The interval changed over the feature's life; it last ran from the firm's shared minute scheduler.
- **The poller reads envelopes sent in the last 30 days only.** An envelope signed after that is never picked up.
- **No Chatter post or notification** on send, sign or void, though the tickets asked for them.
- **Two void outcomes.** A firm void leaves the Lead at Not Sent, a DocuSign-side void at Voided (6.4). Reports on Voided miss the first kind.
- **`SendContractService` still matters after retirement.** The Dochly path runs through it, so it must not be deleted as dead DocuSign code (DDT-1071).

## 11. Reuse for similar issues

The Dochly integration (DDT-1070) follows the same shape. For any e-signature integration:

- **Keep one tracking row per envelope**, with its own status, separate from the business record's status.
- **Store what the downstream step needs at send time**, so the step can run later with no user present.
- **Drive everything from a small state machine** and only read non-terminal states, so a stale event on a finished envelope is never acted on.
- **Make each step safe to repeat:** return the existing Matter, skip files already attached.
- **When two parties sign, decide which signature is the business event.** Here the client's signature creates the Matter; the second signature only brings the final PDF.
- **Make template choice data, and fail loudly on a gap.** A silent fallback sends the wrong contract.
- **Ship an off switch** so the integration can be stopped without deleting anything.
- **Watch who runs the code.** When the conversion later ran as a signing-site guest user under Dochly, records were created without a record type and tasks went to the wrong user (DDT-1377, DDT-1287).
