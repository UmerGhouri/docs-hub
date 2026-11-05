# Mass Mailer (Mass Emails tab, Send Email / Send Weekly Article)

> Based on the local source in the GVA project (retrieved from the sandbox); the org may have changed since.
> Almost every file below comes from the **17 Jun 2026** retrieve. Nothing in the mass mailer has been changed by us since then.

| | |
|---|---|
| **What it is** | A custom tool that sends one personalised email per selected client (Person Account), in the background |
| **Objects** | Account (Person Accounts / Individuals), Contact (person contact), `Investments__c` (for the advisor website), `AccountListViewConfig__c` (saved list views) |
| **Type** | LWC composer + Apex controller + Batch Apex (`SendEmailBatchJob`, batch size 10) |
| **Entry points** | **Mass Emails** tab (buttons *Send Email* and *Send Weekly Article*); standard Account list view buttons *Send Email* and *Send Weekly Article* |
| **Main classes** | `SendEmailButtonForAdInvController`, `SendEmailBatchJob`, `AccountListViewController`, `AccountListViewConfigController` |
| **Test classes** | `SendEmailButtonForAdInvControllerTest`, `AccountListViewControllerTest`, `AccountListViewConfigControllerTest` |
| **Owner of changes** | None by us so far. Treat this doc as a map, not a change log. |

---

## 1. What it does for users

A user picks a set of clients, writes (or loads) an email, and clicks **Send**. Salesforce then sends **one separate email to each selected client**, with merge fields (name, owner, etc.) filled in for that client. The email is logged as an activity on the client's record.

There are two modes:

- **Send Email**: a normal email. The user writes the subject and body, can pick a template, add CC/BCC and attachments, and chooses the From address.
- **Send Weekly Article**: the weekly market-update email. Subject and body come from the template **GVAAM - Articles** (`GVAAM_Articles`), the subject is locked, each email is sent **from the client's owner** (advisor) via an Org-Wide Email Address, and a link to the article on the **advisor's own website** is added.

The send runs as a background job. The "Email sent" / "Email batch job has been initiated" message only means the job was **queued**, not that emails were delivered.

---

## 2. What is the mass mailer, and what is not

Several email features in this org look alike. Only the first group is the mass mailer.

### The mass mailer

| Component | Type | Role |
|---|---|---|
| `accountListView` | LWC (tab **Mass Emails**) | Filterable client list with saved views, *Select all across pages*, and the **Send Email** / **Send Weekly Article** buttons. Opens the composer in a modal. |
| `AccountListViewController` | Apex | List data, filters (incl. a virtual "Bounced?" filter), `getAllAccountIds` (up to 50,000), `canSendWeeklyArticle` |
| `AccountListViewConfigController` + `AccountListViewConfig__c` | Apex + custom object | Per-user saved list views (columns, filters, sort, page size) |
| `sendEmailButtonForAdInv` | LWC | **The composer.** From, To/CC/BCC, subject, rich-text body, templates, merge-field picker, attachments, Send |
| `emailTemplateDropdown` | LWC | Small "recent templates" menu used inside the composer |
| `SendEmailButtonForAdInvController` | Apex | Composer back end: From options, recipient list, templates, merge-field resolution, `sendPersonalizedEmails1` (starts the batch) |
| `SendEmailBatchJob` | Batch Apex | Builds and sends the emails, 10 Accounts per chunk, sends an error report at the end |
| `SendEmailButton.page` / `SendEmailWeeklyButton.page` | Visualforce | Host the composer for the standard list view buttons (via Lightning Out) |
| `LWCComp` (Aura app) + `SendEmailLauncher` (Aura cmp) | Aura | Lightning Out wrappers. `SendEmailLauncher` passes `mode = WEEKLY_ARTICLE` |
| Account list buttons `Send_Email`, `Send_Weekly_Article` | WebLink (list button) | On the standard Account list view (`listViewButtons` in `Account.object-meta.xml`) |
| Email template `GVAAM_Articles` (folder GVAAM) | Classic HTML template | Weekly article subject and body |
| `GVA_Signature_Field_Mapping__mdt` (8 records) | Custom metadata | Which department signature fields appear in the merge-field picker |
| Custom labels `Batch_Error_Email`, `Profile_Names_for_Weekly_Button_Access` | Labels | Error report recipient; profiles that see *Send Weekly Article* on the Mass Emails tab |
| Permission set `Account_List_Button` | Permission set | Grants the `SendEmailButton` VF page only |
| Flows `Weekly_Articles_Individual_Clients` (on `Investments__c`) and `Weekly_Articles_Individual_Clients_RepID` (on `Rep_ID__c`) | Record-triggered flows | Keep `Account.Has_Open_GVA_Investment__c` up to date, which users filter on in the Mass Emails tab |

### Related but separate (not the mass mailer)

| Component | What it actually is |
|---|---|
| `emailComposer` + `EmailController` | **Single household email.** Quick action **Email Household** on the Household / Business / Professional Relationship record pages, tab **Email Composer**, and the screen flow `Send_List_Emails` (quick action `Send_List_Emails1`). Sends **synchronously**, one email per To address, from the running user. Recipients are the household's ACR contacts with an email, not opted out, and `Account.Type != 'Former'`. |
| `sendListEmailAccount` (LWC + Apex) | Older weekly-article sender. Uses the `GVAAM_Articles` template with `setTemplateId`, honours `PersonHasOptedOutOfEmail`, Person Accounts only. Exposed as quick action **Send Email** (`Account.Send_Email`) on the *Account-Household Layout (GVA)*. See gotcha G9. |
| `advisorListView` + `WeeklyArticlesController` | Read-only table (tab **Advisor List**): clients of advisors subscribed to weekly articles whose Rep ID is GVA Asset Management, minus opted-out clients. It sends nothing. |
| `weeklyArticlesIndividualClients` | Tab **Weekly Articles – Individual Clients**. Imports `SendEmailButtonForAdInvController.getFilteredAccountsDynamic` and `.getAccountFields`, which **do not exist** in the local controller. It looks like an earlier version of the Mass Emails tab. Not confirmed whether it works in the org. |
| `Email_Logger` | Generic "email this error to these addresses" helper. Used by other batches (`BatchSyncAcrFromFinancialAccount`, `BatchUpdateAccountOwnerFromInv`, `BatchUpdateTaskFromInvestment`, `TaskRecurringProcessor`). **Not used by the mass mailer.** |
| `CSUtils` | General utility class (string/describe helpers, `sendEmail`, `canSendEmails` via `reserveSingleEmailCapacity`). **Not used by the mass mailer.** |
| `lwc/testing` | A copy of a composer used for experiments. Not placed anywhere we could find. |
| Quick action `Account.Test_EMail`, WebLink `test_vf_button` | Test hooks for the composer. Not on any layout or list view in the local source. |

> There is **no custom logging object** for the mass mailer. "Logging" is (a) the email activity saved on each Account and (b) one error-report email at the end of the job.

---

## 3. Entry points

| Where | Button | Opens | Mode | Who sees it (local source) |
|---|---|---|---|---|
| **Mass Emails** tab (`Account_List_View`, LWC `accountListView`) | **Send Email** | Composer in a modal | Normal | Tab is visible to Admin, *Salesforce - GVA Asset Management*, *Salesforce - GVA Home Office* |
| **Mass Emails** tab | **Send Weekly Article** | Composer in a modal | `WEEKLY_ARTICLE` | Only profiles listed in label `Profile_Names_for_Weekly_Button_Access` (currently *Salesforce - GVA Asset Management*) |
| Standard Account list view | **Send Email** (`Send_Email`) | VF page `SendEmailButton` → composer | Normal | Anyone with access to the VF page (perm set `Account_List_Button`) |
| Standard Account list view | **Send Weekly Article** (`Send_Weekly_Article`) | VF page `SendEmailWeeklyButton` → `SendEmailLauncher` → composer | `WEEKLY_ARTICLE` | Anyone with access to that VF page (only the *Home Office* profile in the local source). **Not** limited by the profile label. |

Notes:
- On the Mass Emails tab you can tick **Select all across pages**. The tab then calls `getAllAccountIds` and sends up to **50,000** Account Ids to the composer.
- The standard list view buttons need at least one row selected (`requireRowSelection`). After sending, the VF page redirects to the Account *Recent* list view.
- Apex class access: `SendEmailButtonForAdInvController`, `AccountListViewController`, `sendListEmailAccount` and `EmailController` are enabled on the *GVA Asset Management* and *GVA Home Office* profiles in the local source. Profile files from a retrieve can be partial, so check the org before relying on this.

---

## 4. End-to-end flow

```
User selects clients
  ├─ Mass Emails tab (accountListView)      ─┐
  └─ Account list view → VF page (Lightning Out)┘
        └─ sendEmailButtonForAdInv (composer LWC)
             • normalizeRecordIds(): pulls 15/18-char Ids out of whatever it was given
             • getRecipientEmails(): fills the To box with PersonEmails (display only, see §6)
             • getFromAddresses(): "me" + Org-Wide addresses this profile may use
             • WEEKLY_ARTICLE: getTemplateByDeveloperName('GVAAM_Articles') → subject + body, subject locked
             • Send → sendPersonalizedEmails1(...)
                  └─ SendEmailButtonForAdInvController.sendPersonalizedEmails1
                       • returns silently if no Account Ids
                       • Database.executeBatch(new SendEmailBatchJob(...), 10)
                            ├─ start():   loads running user, Organization, the Accounts,
                            │             advisor websites (Investments__c.Office_Website__c),
                            │             all-fields copy of each Account, person Contacts,
                            │             matching Org-Wide addresses, owners' signatures
                            ├─ execute(): per Account (10 at a time)
                            │     skip + log if PersonEmail blank
                            │     WEEKLY: skip + log if owner email has no Org-Wide address
                            │     build SingleEmailMessage (To = PersonEmail, CC/BCC, From/Reply-To)
                            │     resolve merge fields, {{SENDER_SIGNATURE}}, {{ARTICLE_URL}}
                            │     WhatId = Account, save as activity
                            │     Messaging.sendEmail(all 10)
                            └─ finish():  if anything was logged → one plain-text error report
                                          to label Batch_Error_Email + the running user
  UI: "Email batch job has been initiated." (Mass Emails tab) / "Email sent" (VF)
```

---

## 5. Modes

| | Normal (*Send Email*) | Weekly (*Send Weekly Article*, `mode = 'WEEKLY_ARTICLE'`) |
|---|---|---|
| Subject / body | Typed by the user, or from any active template | From template `GVAAM_Articles`. Subject field is **disabled**. Body is editable. |
| Template picker | Shown | Hidden |
| From shown in UI | Combobox: "you (Me)" + allowed Org-Wide addresses | Disabled text box showing the first option (you). **This is not the real sender**, see §7. |
| Real From | You, or the chosen Org-Wide address | Org-Wide address whose address equals the **Account owner's** email |
| Reply-To | Your email (if you chose "Me"); the Org-Wide address otherwise | Account owner's email |
| Extra placeholder | none | `{{ARTICLE_URL}}` → link to the article on the advisor's website |
| Skipped when | PersonEmail blank | PersonEmail blank, or no Org-Wide address for the owner's email |

### How the weekly article link is built
1. The **article title** is the part of the template **subject after the first `:`**, trimmed, with spaces replaced by `-`.
   Example: subject `GVA Economic & Market Update: Mr. Brightside` → `Mr.-Brightside`.
2. The **website** is `Investments__c.Office_Website__c` (formula: Rep ID → Primary Advisor → Office → `Website__c`) from a Financial Account whose **Client Name** (`Account_Name__c`) is the client (or the client's `ParentId`, if set). If the client has several Financial Accounts, the last one read wins.
3. `https://` is stripped from the website, `/` is added if missing, and the link becomes `https://<website>/<title>`.
4. If no website is found, `{{ARTICLE_URL}}` is **left in the email as literal text** (gotcha G6).

So to send a new weekly article, someone edits the **subject** (and body) of the `GVAAM_Articles` template. The *Article URL* slug input in the composer is commented out, and the `articleSlug` value passed to Apex is not used.

---

## 6. Recipient rules

| Rule | What the code does |
|---|---|
| Who receives an email | **Each selected Account's `PersonEmail`**, one email per Account. |
| The To box in the composer | **Display only.** It is filled with the selected Accounts' PersonEmails, and you must have at least one To/CC/BCC entry to press Send, but `SendEmailBatchJob` **ignores the To list**. Adding or removing addresses there changes nothing. |
| Person Accounts vs Households | Only Accounts with a PersonEmail get an email. Household / Business Accounts are skipped and **each one is listed in the error report** ("Skipped — PersonEmail is blank"). |
| Contacts | The person Contact is looked up by email. If found and **not** opted out, it is set as `TargetObjectId` (so the activity lands on that Contact too). Other household contacts are **not** emailed. |
| **Email Opt Out** | **Not enforced.** If the person Contact has opted out, the code only skips `setTargetObjectId`. The email is still sent to `PersonEmail`. `PersonHasOptedOutOfEmail` and `Email_Opt_Out_Weekly_Articles__c` are not checked anywhere in the batch (gotcha G1). |
| Filtering in practice | Users must filter in the Mass Emails tab. The default columns include `PersonEmail`, `Has_Open_GVA_Investment__c`, `Email_Opt_Out_Weekly_Articles__c`, `PersonHasOptedOutOfEmail` and `RecordType.Name`, and there is a virtual **Bounced?** filter (latest `EmailMessage.IsBounced` related to the Account). The standard list view buttons have **no** filtering beyond what the list view shows. |
| Duplicates | Account Ids are de-duplicated by the batch query. Two Accounts with the same PersonEmail get two emails. |

---

## 7. From address / Org-Wide Email Address logic

**Options in the From box** (`SendEmailButtonForAdInvController.getFromAddresses`):
1. The running user's email, labelled "(Me)".
2. Every `OrgWideEmailAddress` with *Allow All Profiles*, or granted to the user's profile (`OrgEmailAddressSecurity`).

**What the batch does with it**

| Mode | Value chosen | Result |
|---|---|---|
| Normal | Org-Wide address (Id starts with `0D2`) | `setOrgWideEmailAddressId(...)`, so the email comes from that address |
| Normal | "Me" (an email address) | `setReplyTo(yourEmail)`; From is the running user (default sender) |
| Weekly | ignored | `setOrgWideEmailAddressId(OWEA matching the Account owner's email)` and `setReplyTo(owner email)` |

For weekly sends, **every advisor who owns clients needs their own email registered and verified as an Org-Wide Email Address**. Otherwise all their clients are skipped with "No OrgWideEmailAddress found for owner email" in the error report.

> Change sets can't carry Org-Wide Email Addresses. They are set up by hand in each org (Setup → Organization-Wide Addresses) and must be verified.

**Department signatures.** `GVA_Signature_Field_Mapping__mdt` maps department Org-Wide addresses (accounting, asset management/investments, commissions, compliance, info, insurance, operations, salesforce) to User fields such as `GVA_Accounting_Email_Signature__c`. A field shows up under **Sender** in the merge-field picker only if the user can use that department's address. This only controls the picker. Nothing stops a user typing the merge field by hand.

---

## 8. CC, BCC and attachments

- **CC/BCC are added to every email.** Selecting 300 clients with one CC sends 300 copies to that CC address. Each copy counts toward the daily email limit (§11).
- Attachments are uploaded in the composer (`lightning-file-upload`; the UI rejects files over **25 MB**). The batch receives the `ContentDocument` Ids and **reloads the file bodies in every chunk** (`createEmailAttachments`), then attaches them to each email.
- Large attachments are a heap risk in the batch (12 MB async heap). Not tested with large files.

---

## 9. Templates and merge fields

**Templates.** The template picker lists every active `EmailTemplate`, filtered by Lightning vs Classic (`TemplateType`). The HTML is copied into the editor; Salesforce's own template merge engine is **not** used (no `setTemplateId`). Recently used templates are remembered in the browser's `localStorage` (`recentTemplates`), per user per browser.

**Merge syntax.** Both `{!Object.Field}` and `{{{Object.Field}}}` work (the second is converted to the first). Anything that can't be resolved becomes **blank**.

| Merge object | Resolves to |
|---|---|
| `Account.*` | The selected Account, all fields the running user can see (dynamic SOQL). Special case: `{!Account.OwnerFirstName}` → owner's first name. |
| `Recipient.*` | The person Contact found by email (falls back to the first ACR contact, then the Account) |
| `Sender.*` | The **running user** (including the `GVA_*_Email_Signature__c` fields) |
| `Organization.*`, `SenderBrand.*` | The Organization record |
| `Today`, `Now`, `User.Name`, `User.Email`, `User.FirstName`, `User.LastName`, `Organization.Name` | Hard-coded specials |
| `SomeObject__c.Link` | Lightning URL of the related custom-object record |
| Any other object | The code guesses a relationship field (`AccountId`, `Account__c`, `Parent_Account__c`, `Related_Account__c`) and runs a dynamic query **per merge object per client** |

**Placeholders handled by the batch**

| Placeholder | Replaced with |
|---|---|
| `{{SENDER_SIGNATURE}}` | Three line breaks + the **Account owner's** `User_Signature__c` (not the sender's), in both modes |
| `{{ARTICLE_URL}}` | Weekly mode only, see §5 |

The `GVAAM_Articles` body uses `{!Account.FirstName}`, `{!Account.OwnerFirstName}`, `{!Account.Owner}`, `{{ARTICLE_URL}}` and `{{SENDER_SIGNATURE}}`. `{!Account.Owner}` has no special case, so it most likely renders blank. Not confirmed in the org.

The preview in the composer resolves merge fields against the **first** selected Account only.

---

## 10. Error handling and logging

| Situation | What happens | Where you see it |
|---|---|---|
| Account has no PersonEmail | Skipped, logged | Error report email |
| Weekly: owner email is not an Org-Wide address | Skipped, logged | Error report email |
| Exception while building one email (e.g. owner missing) | That Account skipped, logged with stack trace | Error report email |
| `Messaging.sendEmail` fails (bad address, daily limit reached, etc.) | **Not caught.** Default all-or-none, so the whole chunk of 10 fails. | **Setup → Apex Jobs** only. Not in the error report. |
| The whole job can't be queued | Apex error | Red error text in the composer |
| Success | `setSaveAsActivity(true)` with `WhatId = Account` | Activity History / Email on the Account (and the person Contact when `TargetObjectId` was set) |

**Error report.** In `finish()`, if anything was logged, one plain-text email (subject `SendEmailBatchJob — N Failed Records | Job: <Id>`) goes to the address in custom label **`Batch_Error_Email`** and to the running user.

> ⚠️ In the local source, `Batch_Error_Email` holds a **developer's address at the implementation vendor**, not a GVA mailbox. Error reports for client email jobs go outside GVA. Point it at a GVA admin mailbox.

---

## 11. Governor limits and sizing

| Limit | Value / behaviour | Impact here |
|---|---|---|
| Batch size | **10** Accounts per `execute` (hard-coded in `sendPersonalizedEmails1`) | One `Messaging.sendEmail` call per chunk |
| Daily single-email limit | Salesforce caps Apex single emails to external addresses per org per 24 h (5,000 in production per Salesforce docs; sandboxes can be lower). **Every To, CC and BCC address counts.** | 1,000 clients + 1 CC = 2,000 sends. The batch doesn't check remaining capacity; once the cap is hit, the remaining chunks fail (only visible in Apex Jobs). |
| Recipients per message | To ≤ 100, CC ≤ 25, BCC ≤ 25 (Salesforce limit) | Matters only for CC/BCC here |
| Heap (async 12 MB) | `start()` keeps an **all-fields copy of every selected Account** plus Contacts, in a `Database.Stateful` job | Very large selections (thousands to tens of thousands) may run out of heap in `start()` or between chunks. Not tested. |
| SOQL rows in `start()` | Several non-locator queries over all selected Accounts, their Contacts and Financial Accounts (50,000-row limit per transaction) | Same: large selections are a risk |
| SOQL per chunk | Merge resolution can run dynamic queries per client (e.g. Recipient fallback, related objects) | 10 clients per chunk keeps this well under 100 |
| Concurrent batch jobs | 5 active, 100 in the flex queue (org-wide) | Many users sending at once queue behind each other |

---

## 12. Security

| Class | Sharing | Notes |
|---|---|---|
| `SendEmailButtonForAdInvController` | `with sharing` | The composer only sees records the user can see. Dynamic "all fields" queries keep only fields the user can read (`isAccessible`). |
| `AccountListViewController`, `AccountListViewConfigController` | `with sharing` | Saved views are filtered to `OwnerId = current user`. Dynamic filters only accept real Account fields (`FIELD_MAP`, from the Account describe) or a short list of relationship fields, and values are escaped with `String.escapeSingleQuotes`. |
| `SendEmailBatchJob` | **no sharing keyword** | The batch reads the selected Accounts, Contacts and Financial Accounts without the user's sharing (system mode). It only ever processes Ids the user selected. |
| `EmailController`, `sendListEmailAccount`, `WeeklyArticlesController` | `with sharing` | Separate features |

- Permission set `Account_List_Button` only grants the `SendEmailButton` page. Class access and the weekly page come from profiles (§3).
- The weekly button on the Mass Emails tab is gated by profile **name** (label `Profile_Names_for_Weekly_Button_Access`, comma-separated). The weekly **list view** button is not.

---

## 13. Test classes and coverage approach

| Test class | Methods | Covers | Approach |
|---|---|---|---|
| `SendEmailButtonForAdInvControllerTest` | 85 | `SendEmailButtonForAdInvController`, `SendEmailBatchJob` | Mostly coverage-driven. Merge-field tests assert `!result.contains('{!')`. The 15 batch tests at the end (`test_batch_*`) run the job and end with `System.assert(true)`: **they prove it doesn't crash, not that the right emails are built**. Includes a test with `HasOptedOutOfEmail = true` that does not check whether the email was suppressed. |
| `AccountListViewControllerTest` | 44 | `AccountListViewController` | Filters, sort, pagination, bounced filter, weekly-button check |
| `AccountListViewConfigControllerTest` | 12 | `AccountListViewConfigController` | Save / rename / delete / last-used |
| `WeeklyArticlesControllerTest` | 6 | `WeeklyArticlesController` (separate feature) | |
| `EmailControllerTest` | 59 | `EmailController` (separate feature) | |
| `Email_LoggerTest` | 1 | `Email_Logger` | No assertions |
| `CSUtilsTest` | 25 | `CSUtils` | |
| none | | `sendListEmailAccount` | **No test class found.** 0% unless something else calls it. |

Coverage percentages were not measured for this doc (not run). Org-wide coverage in the sandbox was about **35%** (Sept/Oct 2026 sessions), so any deploy that includes Apex must use **Run specified tests**.

---

## 14. Known risks and gotchas

| # | Gotcha | Why it matters |
|---|---|---|
| G1 | **Opt-out is not enforced** by the batch (§6) | Compliance risk. Opted-out clients get the email unless the user filtered them out. |
| G2 | **The To box is ignored** | Users think they removed someone; they didn't. Only the selected Accounts decide who gets it. |
| G3 | Success toast = job queued | Failures appear later in the error report or in Apex Jobs |
| G4 | Send failures (`Messaging.sendEmail`) are not in the error report and fail **10 clients at a time** | One bad address can block 9 good ones |
| G5 | Weekly From is the **owner's** Org-Wide address, not what the UI shows | Owners without a verified Org-Wide address: all their clients are skipped |
| G6 | No advisor website → `{{ARTICLE_URL}}` sent as literal text | Clients see `{{ARTICLE_URL}}`. The code also does `.replace('www/','')`, which looks like a typo for `www.`; not confirmed. The title is not URL-encoded. |
| G7 | `{{SENDER_SIGNATURE}}` is the **Account owner's** signature in both modes | If a Home Office user sends a normal email, the advisor's signature appears |
| G8 | Error reports go to a vendor developer's address (`Batch_Error_Email`) | Change before relying on it in production |
| G9 | `sendListEmailAccount` as a **record** quick action receives no `recordIds`, returns without sending, and still shows "Weekly articles sent successfully." | Looks like a no-op. From code reading only; not confirmed in the org. |
| G10 | The `Test_EMail` quick action on a record page passes no selected Ids → `sendPersonalizedEmails1` returns silently | Same "success with nothing sent" pattern. Not on any layout. |
| G11 | `weeklyArticlesIndividualClients` imports Apex methods that don't exist locally | Deploying the `lwc` folder as-is would fail on that component |
| G12 | Large "select all" sends (up to 50,000) | Heap / row-limit risk in `start()` (§11), plus the daily email cap |
| G13 | Custom label `Default_Email_For_Button` holds a specific Account Id | Only used in commented-out code. Ignore it, but don't delete the label without checking the org. |

---

## 15. Troubleshooting "email not received"

Work through these in order.

1. **Did the job run?** Setup → **Apex Jobs**, class `SendEmailBatchJob`. Check *Status*, *Batches Processed*, *Failures* and the *Status Detail* message. A failure here means a whole chunk of 10 failed to send.
2. **Was there an error report?** Look for `SendEmailBatchJob — N Failed Records` in the sender's inbox (and the `Batch_Error_Email` mailbox). Common lines: *PersonEmail is blank* (Household selected) and *No OrgWideEmailAddress found for owner email* (weekly).
3. **Is it on the record?** Open the client → Activity / Emails. If the email is logged there, Salesforce sent it and the problem is downstream (spam, bounce, client's mail server).
4. **Bounced?** Use the **Bounced?** filter on the Mass Emails tab, or check the EmailMessage's bounce flag. Fix the address and resend.
5. **Weekly sends:** is the owner's email in Setup → Organization-Wide Addresses, **verified**? Does the client have a Financial Account (Client Name = client) whose Rep ID → Primary Advisor → Office has a website?
6. **Deliverability:** Setup → **Deliverability** → *Access level* must be **All email** (sandboxes default to *System email only*, which silently drops these emails).
7. **Daily limit:** if many sends failed at once late in the day, the org may have hit the daily single-email cap. Check *Status Detail* in Apex Jobs.
8. **Spam:** Org-Wide addresses with poor SPF/DKIM setup have gone to Gmail spam before in this org (seen with the notification sender in Sept 2026). Ask the recipient to check spam.

---

## 16. How to test safely in a sandbox

> Sandboxes contain copies of real client email addresses. **Never** send to selected real clients from a sandbox.

1. **Deliverability.** Setup → Deliverability. Leave it on *System email only* until you're ready, then set **All email** only for your test window, and set it back afterwards.
2. **Make your own test client.** Create a Person Account (Individual) with **your own** email as PersonEmail. For weekly tests, make yourself (or a test user) the owner and give that user's email an Org-Wide address, verified.
3. **Select only your test record(s).** On the Mass Emails tab, filter on your record's name; don't use *Select all across pages*.
4. **Leave CC/BCC empty**, or use your own address.
5. **Weekly article test:** link the test client to a Financial Account (Client Name = test client) whose Rep ID's Primary Advisor has an Office with a website, so `{{ARTICLE_URL}}` resolves.
6. **Check** Apex Jobs, your inbox, the error report, and the Activity on the record.
7. If you change `Batch_Error_Email` for testing, change it back.

---

## 17. Seeing a similar issue?

- *"Someone got the email after opting out"* → expected with the current code (G1). The fix belongs in `SendEmailBatchJob.execute` (skip when `PersonHasOptedOutOfEmail`, or `Email_Opt_Out_Weekly_Articles__c` in weekly mode) plus a real assertion in the test.
- *"I removed an address from To but they still got it"* → G2. Deselect the Account instead.
- *"The weekly email has `{{ARTICLE_URL}}` in it"* → the client's advisor has no Office website reachable through a Financial Account (§5).
- *"Nobody in Advisor X's book got the weekly article"* → that advisor's email isn't a verified Org-Wide address (§7).
- *"The UI said sent but nothing arrived"* → §15, starting with Apex Jobs.
- You need to send a different weekly article → edit the subject/body of template `GVAAM_Articles`; the link slug comes from the subject text after `:`.

---

## Components

| Type | Name | File |
|---|---|---|
| LWC | `accountListView` (tab *Mass Emails*) | `lwc/accountListView` |
| LWC | `sendEmailButtonForAdInv` (composer) | `lwc/sendEmailButtonForAdInv` |
| LWC | `emailTemplateDropdown` | `lwc/emailTemplateDropdown` |
| Apex | `SendEmailButtonForAdInvController` | `classes/SendEmailButtonForAdInvController.cls` |
| Apex (batch) | `SendEmailBatchJob` | `classes/SendEmailBatchJob.cls` |
| Apex | `AccountListViewController` | `classes/AccountListViewController.cls` |
| Apex | `AccountListViewConfigController` | `classes/AccountListViewConfigController.cls` |
| Custom object | `AccountListViewConfig__c` | `objects/AccountListViewConfig__c` |
| Visualforce | `SendEmailButton` | `pages/SendEmailButton.page` |
| Visualforce | `SendEmailWeeklyButton` | `pages/SendEmailWeeklyButton.page` |
| Aura | `LWCComp` (Lightning Out app) | `aura/LWCComp` |
| Aura | `SendEmailLauncher` | `aura/SendEmailLauncher/SendEmailLauncher.cmp` |
| List button | `Send_Email` | `objects/Account/webLinks/Send_Email.webLink-meta.xml` |
| List button | `Send_Weekly_Article` | `objects/Account/webLinks/Send_Weekly_Article.webLink-meta.xml` |
| Email template | `GVAAM_Articles` | `email/GVAAM/GVAAM_Articles.email` |
| Custom metadata | `GVA_Signature_Field_Mapping__mdt` | `customMetadata/` (`GVA_Signature_Field_Mapping.*`) |
| Custom labels | `Batch_Error_Email`, `Profile_Names_for_Weekly_Button_Access`, `Default_Email_For_Button` | `labels/CustomLabels.labels-meta.xml` |
| Permission set | `Account_List_Button` | `permissionsets/Account_List_Button.permissionset-meta.xml` |
| Tab | `Account_List_View` (*Mass Emails*) | `tabs/Account_List_View.tab-meta.xml` |
| Flow | `Weekly_Articles_Individual_Clients` | `flows/Weekly_Articles_Individual_Clients.flow-meta.xml` |
| Flow | `Weekly_Articles_Individual_Clients_RepID` | `flows/Weekly_Articles_Individual_Clients_RepID.flow-meta.xml` |
| Test | `SendEmailButtonForAdInvControllerTest` | `classes/SendEmailButtonForAdInvControllerTest.cls` |
| Test | `AccountListViewControllerTest` | `classes/AccountListViewControllerTest.cls` |
| Test | `AccountListViewConfigControllerTest` | `classes/AccountListViewConfigControllerTest.cls` |

**Related, not part of the mass mailer:** `lwc/emailComposer`, `classes/EmailController.cls`, `flows/Send_List_Emails.flow-meta.xml`, `quickActions/Account.Email_Household.quickAction-meta.xml`, `lwc/sendListEmailAccount`, `classes/sendListEmailAccount.cls`, `quickActions/Account.Send_Email.quickAction-meta.xml`, `lwc/advisorListView`, `classes/WeeklyArticlesController.cls`, `lwc/weeklyArticlesIndividualClients`, `classes/Email_Logger.cls`, `classes/CSUtils.cls`.
