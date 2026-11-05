# Recreating a Marketing Cloud audience as an Account report

| | |
|---|---|
| **Ticket** | DDT-136 |
| **Project** | Support tickets |
| **Status** | Done 2026-06-16 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-06-09 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Reports), MassMailer |
| **Area** | Client email audiences moving from Marketing Cloud to MassMailer |
| **Related** | None recorded |
| **Code** | Not applicable: report built in the Salesforce report builder, setting changed in MassMailer |

---

## 1. Summary

A client-services staff member needed an email audience they had used in Marketing Cloud recreated inside Salesforce, so it could be used from MassMailer. The support team built an Account report with the same columns and filters, and saved it where the requester asked. When the requester then asked for duplicate email addresses to be removed, the team turned on MassMailer's duplicate-email suppression rather than changing the report.

## 2. What was reported

- **2026-06-09:** the requester supplied the original audience as a SQL-style query: Person Accounts with both `PersonContactId` and `PersonEmail` populated, with a list of 20 columns. They asked for it in a folder named **MassMailer Audiences**, titled **Master Client List**, with public sharing.
- **2026-06-10:** the support team shared the finished report for review.
- **2026-06-11:** the requester asked for duplicate records to be removed, because MassMailer would otherwise send the same email twice to one address. The support team confirmed duplicates meant the same `PersonEmail`, and the requester agreed.
- **2026-06-15:** the support team enabled duplicate-email suppression in MassMailer.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Master Client List | Salesforce report on Account (Person Accounts) | New, in the MassMailer Audiences folder |
| Duplicate-email suppression | MassMailer setting | Enabled |

Report columns, as requested:

`Case_Manager__c`, `Commercial_License__c`, `CreatedById`, `fw1__Entity__c`, `Id`, `Last_Outreach_User__c`, `LastModifiedById`, `MasterRecordId`, `Next_Court_Appearance__c`, `OwnerId`, `ParentId`, `PersonContactId`, `PersonIndividualId`, `RecordTypeId`, `Referring_Attorney__c`, `typeform__Typeform_Form_Mapping__c`, `typeform__Typeform_Form_Mapping__pc`, `PersonBirthdate`, `PersonEmail`, `FirstName`

Filters: `PersonContactId` not null **and** `PersonEmail` not null.

**Out of scope**

- **Building the audience inside MassMailer.** The plan was for the audience to be recreated in MassMailer once the report was approved. Who did that, and when, is not recorded in the ticket.
- **De-duplicating the data.** Duplicate Person Accounts sharing one email were left as they are; only the send is de-duplicated.

## 4. How it works (background)

```
Marketing Cloud (old): SQL query over Account -> audience -> email send
Salesforce (new):      Account report       -> MassMailer audience -> email send
```

A Salesforce report cannot remove rows that share a value in one column the way a SQL `DISTINCT` can. A Person Account is one row per person, so two Person Accounts with the same email address give two rows.

## 5. Root cause

Not applicable as a defect: this was a request to rebuild an existing audience in Salesforce.

The one problem that came up, duplicate sends, comes from the data: some Person Accounts share an email address. The report reproduces the audience faithfully, duplicates included.

## 6. Solution

### 6.1 The report

Built in the Salesforce report builder on Accounts, with the 20 columns and two filters in section 3, saved as **Master Client List** in a **MassMailer Audiences** folder with public sharing, as the requester asked. Whether a custom report type was needed for any of the columns is not recorded in the ticket.

### 6.2 Duplicate emails

MassMailer's option to **suppress duplicate emails** was enabled. When a send goes to a list view, any repeated email address is suppressed so each recipient gets one email.

Why it was done there and not in the report:

- **The report can't de-duplicate on one column.** Suppressing at send time handles it without changing the data.
- **It covers every list view**, not just this audience.

## 7. Testing

- **Live check:** the requester reviewed the report on 2026-06-11 and said it was what they needed, apart from the duplicates.
- **Not covered:** a recorded comparison of the report's row count against the original Marketing Cloud audience, and a recorded test send after duplicate suppression was enabled.

## 8. Step by step

1. **Get the original query** from the requester and list its columns and filters.
2. **Reports → New Report → Accounts** (or the Person Account report type). Add the columns.
3. **Add filters** `PersonContactId` not equal to blank and `PersonEmail` not equal to blank.
4. **Check every column is available.** If one is not, a custom report type is needed.
5. **Save it** in the folder and under the name the requester gives, with the sharing they ask for.
6. **Send it for review**, and ask specifically whether duplicate email addresses matter for their send.
7. **If they do, enable duplicate-email suppression in MassMailer's settings**, so each address gets one email.

## 9. Rollback

- **Report:** delete it, or leave it; it changes no data.
- **MassMailer setting:** turn duplicate suppression off. That affects every MassMailer send, not just this audience.

## 10. Known limits

- **Duplicate suppression is org-wide in MassMailer.** Any send that deliberately goes to the same address twice will now send once.
- **The report still contains duplicates.** Anyone exporting it for another tool will get them.
- **Columns from managed packages** (`fw1__`, `typeform__`) disappear from the report if those packages are removed.

## 11. Reuse for similar issues

- **Ask about duplicates at the start** when an audience is for email. It is the most common difference between a SQL audience and a report.
- **De-duplicate at send time** in the sending tool when the data legitimately has more than one record per address.
- **Copy the requester's filters exactly**, then have them compare the result against the old audience before the first send.
