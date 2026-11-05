# Change Set Fails: "Cannot compare savepoints from two different transactions"

| | |
|---|---|
| **Status** | Workaround delivered: Workbench zip deploy, proven on a Developer Edition org. Production outcome: TBD — not found in records |
| **First seen** | 10 Sep 2026 (Production validation) |
| **Workaround delivered** | 15 Sep 2026 (zip + Word step-by-step guide) |
| **Affects** | Change set `Investment Task Mobile` from [05](../05-new-investment-task-mobile/README.md): LWC `investmentReminder`, Flow `New_Investment_Task`, Action `Investments__c.New_Investment_Task_Flow` |
| **Type** | Deployment problem (no code change) |
| **Route that worked** | Metadata API zip deployed through Workbench |
| **Files** | `files/InvestmentTaskMobile.zip`, `files/InvestmentTaskMobile-Prereqs.zip`, `files/New Investment Task - Deployment Guide.docx` |

---

## 1. Summary

The client admin validated the change set *Investment Task Mobile* in Production, and it failed on the LWC bundle with an internal Salesforce error. Retrying didn't help, and neither did splitting the LWC into its own change set. The same metadata deploys cleanly through the Metadata API, so we packaged it as a zip and sent it with a short Workbench guide. The zip and the Workbench steps were proven end to end on a Developer Edition org first.

---

## 2. The error (as received)

Deployment Details, 10 Sep 2026: **Validation Failed**, *Deploy Components 3/3 (2 errors)*, *Run Apex Tests: Not Required*.

| API Name | Type | Error Message |
|---|---|---|
| *(blank)* | | `Programmer error: Cannot compare savepoints from two different transactions Field info: entity name = LightningComponentResource, field name = Source.` |
| `Investments__c.New_Investment_Task_Flow` | Action | `invalid cross reference id` |
| `New_Investment_Task` | Flow Version | `invalid cross reference id` |
| *(blank)* | | `Package Installation Failure` |

Read these in dependency order. Only the **first row** is a real failure: the `investmentReminder` bundle didn't save. The flow references `c:investmentReminder` and the action references the flow, so both fail as a knock-on. The last row is the generic "everything rolled back" wrapper.

_Screenshot in the private copy: 01-change-set-error.png (Validation failed – component errors)._

---

## 3. Theory

- **It's a platform error, not a metadata error.** The signs: the words *"Programmer error"*, a blank API Name, and an internal entity name (`LightningComponentResource.Source`). A real problem (for example, a missing label) produces a clean message that names the component, such as *cannot resolve module*.
- The client's own research pointed to a known Salesforce issue with LWC bundles in change sets, most often **LWCs exposed as quick actions**. `investmentReminder` declares `lightning__RecordAction`, so it matches. The Stack Exchange thread found is from about 2020. We treated this as a working theory, not a confirmed cause.
- The bundle files were checked: pure ASCII, no BOM, LF line endings, about 8 KB. Nothing unusual.
- The same 4 files deployed to the sandbox through the Metadata API more than once.

**Conclusion:** the change set channel can't write this bundle into that org. No change to the metadata will fix it, so use a different deployment channel.

---

## 4. What was tried

| # | Date | Attempt | Result |
|---|---|---|---|
| 1 | 10 Sep | Validate change set *Investment Task Mobile* (LWC + Flow + Action) | Failed, error above |
| 2 | 10 Sep | Considered adding Custom Label `InvestmentRecordTypeId` | **Not done.** A missing label gives a clean "cannot resolve" error, not this one. Deploying the label could also overwrite Production's value. |
| 3 | 10 Sep | Asked the client to **re-validate as-is** | Recommended as a free first step. The outcome isn't recorded separately; the split was built the same day. |
| 4 | 10 Sep | **Split** into `Investment Task Mobile 1 - Component` (LWC only) and `Investment Task Mobile 2 - Flow & Action`, to deploy in that order | Change set 1 failed with the same error (reported 15 Sep). This time *Deploy Components* showed *Fatal Error*. |
| 5 | 15 Sep | **Metadata API zip through Workbench** | Worked on a Developer Edition org (3/3 components). Sent to the client for Production. |

_Screenshot in the private copy: 02-split-change-set-error.png (Split change set – same error)._

The split proved the point. With only the LWC in the set there are no knock-on errors, no mixed component types and no dependencies, and it still fails the same way.

---

## 5. The zip packages

| File | Contents | Use |
|---|---|---|
| `InvestmentTaskMobile.zip` (6,033 bytes) | `package.xml`, `flows/New_Investment_Task.flow`, `lwc/investmentReminder/*` (4 files), `quickActions/Investments__c.New_Investment_Task_Flow.quickAction` | **The deliverable.** Production and any test org. |
| `InvestmentTaskMobile-Prereqs.zip` (1,170 bytes) | `package.xml`, `objects/Investments__c.object` (label *Financial Account*, with checkbox `GVA_Asset_Management_Accounts__c`), `labels/CustomLabels.labels` (`InvestmentRecordTypeId`) | **Only for a blank test org** (for example, Developer Edition) that doesn't have the GVA schema. **Never deploy to Production or the GVA sandbox.** |

The sources are in `deploy/mdapi/` and `deploy/prereq/`, and the manifest is `deploy/package.xml`. The Lightning page is left out on purpose so Production's page isn't overwritten.

> **Build zips with forward-slash paths and `package.xml` at the root.** The first zip was built with Windows PowerShell 5.1 `Compress-Archive`, which writes backslash path separators. That breaks the ZIP spec, and Salesforce can't find `package.xml`. We rebuilt it. Don't unzip and re-zip from Explorer either, because that can nest everything inside an extra folder.

---

## 6. Testing on a Developer Edition org first

The GVA sandbox already had all three components, so a re-deploy there would show nothing new. To watch the zip land, we used the team's Developer Edition org.

1. **First attempt:** *Status Succeeded* but *Number Components Total: 0*, *Number Files: 0*, and nothing appeared in the org. Two causes:
   - the backslash zip (see above)
   - **Single Package** was not ticked
2. **Prerequisites.** The developer asked for an anonymous Apex script to create the missing object, field and label. **Anonymous Apex can't do this.** Custom objects, fields and labels are metadata, not records, and the Apex `Metadata` namespace doesn't support these types. We built `InvestmentTaskMobile-Prereqs.zip` instead and dry-ran it against the Developer Edition org.
3. **Prereqs deploy, first try:** errors `labels/package.xml → No package.xml found`, `objects/package.xml → No package.xml found`. Again, **Single Package** was not ticked, so Workbench treated each folder as its own package. Ticking it fixed the problem.
4. **Main zip:** *Number Components Deployed: 3* (`investmentReminder`, `New_Investment_Task`, `New_Investment_Task_Flow`). We checked by query that the flow came in **Active**, version 1.

_Screenshot in the private copy: 03-zero-components.png (Workbench – 0 components (Single Package not ticked))._
_Screenshot in the private copy: 04-no-package-xml.png (Workbench – No package.xml found)._
_Screenshot in the private copy: 05-dev-org-success.png (Workbench – 3 components deployed)._

This proves the zip and the process. It doesn't prove the button **works** there, because a Developer Edition org has no Task *Investment Task* record type to open.

---

## 7. Workbench steps (Production)

These are the same steps as the Word guide sent to the client (`files/New Investment Task - Deployment Guide.docx`).

1. Go to **https://workbench.developerforce.com**. Environment: **Production** (use **Sandbox** for a sandbox).
2. Agree to the terms and click **Login with Salesforce**.
3. In the **jump to** menu, select **Deploy** and click Select.
4. Click **Choose File** and select **`InvestmentTaskMobile.zip`**.
5. Tick **Rollback On Error** and **Single Package**. Both must be ticked. Leave **Check Only** unticked for a real deploy, or tick it to validate without writing anything.
6. Click **Next**, then **Deploy**.
7. Confirm **Status: Succeeded** and **3 components deployed**. The 4th entry under *componentSuccesses* is `package.xml`, which is the manifest, not a component.
8. Go to Setup → Flows, open **New Investment Task**, and click **Activate** if the status is Draft.
9. Go to Setup → Object Manager → Financial Account → Lightning Record Pages, edit the page, click the Highlights Panel, and **add** the **New Investment Task** action. Save.

_Screenshot in the private copy: 06-wb-home.png (Step 1 – Workbench home)._
_Screenshot in the private copy: 07-wb-login.png (Step 2 – Login with Salesforce)._
_Screenshot in the private copy: 08-wb-jump-to-deploy.png (Step 3 – jump to Deploy)._
_Screenshot in the private copy: 09-wb-choose-file.png (Step 4 – Choose File)._
_Screenshot in the private copy: 10-wb-options.png (Step 5 – Rollback On Error + Single Package)._
_Screenshot in the private copy: 11-wb-deploy.png (Step 6 – Deploy)._
_Screenshot in the private copy: 12-wb-result.png (Step 7 – Result)._

Step 9 **adds** the action. It doesn't replace anything. If a button with a similar name is already there, leave it and report back.

Workbench is a community-hosted tool, so it means logging Production in through a third-party site. Some orgs don't allow that. If someone has Salesforce CLI access to Production, this is the alternative:
```
sf project deploy start --target-org <prod> --manifest deploy/package.xml --dry-run   # validate
sf project deploy start --target-org <prod> --manifest deploy/package.xml             # deploy
```

Message sent to the client with the zip and the guide (15 Sep):
> Hi [client admin], attached are the deployment file and a short step-by-step guide. Could you please deploy **InvestmentTaskMobile.zip** to production by following the guide? Please make sure **Single Package** and **Rollback On Error** are both ticked — and note steps 8 and 9, which need to be done after the deployment for the button to appear. Thanks!

---

## 8. Post-deployment

The steps are the same as in [05 §10](../05-new-investment-task-mobile/README.md#10-post-deployment): make sure the flow is **Active** and **add the action** to the Financial Account page.

**Why did the flow arrive active?** The flow file has `<status>Active</status>`. Whether that takes effect depends on the target org's **Setup → Process Automation Settings → "Deploy processes and flows as active"** setting. Production may differ, so always check.

---

## 9. Lessons and gotchas

- **"Programmer error" + blank API Name = Salesforce's own failure.** Don't go hunting for a missing dependency. Read the first row and treat the rest as knock-on errors.
- **When a change set fails the same way twice, change the channel**, not the contents. The Metadata API (CLI or Workbench) gives real error messages and avoids this bug.
- **Workbench: always tick Single Package.** Without it you get either a false "Succeeded" with 0 components or *"No package.xml found"* for each folder.
- **Zip on Windows:** don't use PowerShell 5.1 `Compress-Archive` for Salesforce packages, because it writes backslash paths. Check that `package.xml` is at the root.
- **Anonymous Apex can't create metadata** (objects, fields, labels). Use a small prerequisite package instead.
- **Check Only writes nothing.** It is a safe way to test a zip against a real org.
- **A dev-org deploy proves the packaging, not the feature.**

---

## 10. Seeing a similar issue?

- *"Cannot compare savepoints from two different transactions"* on `LightningComponentResource` → skip change sets and deploy the bundle with the Metadata API (Workbench steps in §7).
- Workbench says *Succeeded* but 0 components → tick **Single Package**, and check the zip has `package.xml` at the root with forward slashes.
- Need to rehearse a deploy on an empty org → build a prerequisite zip like `InvestmentTaskMobile-Prereqs.zip`.

## Files

> ⚠️ **The two zips below are reference copies with placeholders, not ready to deploy.** Before deploying, either use the originals in `deploy/` or replace the placeholders:
> - `InvestmentTaskMobile.zip` → `lwc/investmentReminder/investmentReminder.html`: `trading-team@example.com` → the client's trading team mailbox (2 places)
> - `InvestmentTaskMobile-Prereqs.zip` → `labels/CustomLabels.labels`: `REPLACE_WITH_INVESTMENT_RECORD_TYPE_ID` → the target org's Investment record type Id
>
> If you re-zip, keep forward-slash paths. Backslash paths make Workbench deploy 0 components (see above).

- `files/InvestmentTaskMobile.zip`: a copy of `deploy/InvestmentTaskMobile.zip`, with the trading mailbox replaced by a placeholder
- `files/InvestmentTaskMobile-Prereqs.zip`: a copy of `deploy/InvestmentTaskMobile-Prereqs.zip`, with the record type Id replaced by a placeholder, **test orgs only**
- `files/New Investment Task - Deployment Guide.docx`: client guide (screenshot slots not filled)
- Sources: `deploy/package.xml`, `deploy/mdapi/`, `deploy/prereq/`
