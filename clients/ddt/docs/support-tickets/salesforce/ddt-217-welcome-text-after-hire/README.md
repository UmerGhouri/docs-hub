# Welcome text after hire

| | |
|---|---|
| **Ticket** | DDT-217, DDT-772 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-24 (DDT-217) and 2026-09-01 (DDT-772) |
| **Priority** | Medium (DDT-217), High (DDT-772) |
| **Reported by** | Firm staff, 2026-06-22 (DDT-217) and 2026-08-19 (DDT-772) |
| **Resolved by** | Support team |
| **Systems** | Salesforce (record-triggered flows with scheduled paths), Heymarket SMS |
| **Area** | Client onboarding: the automated text sent the day after a client hires the firm |
| **Related** | DDT-1073 (close the Heymarket chat after the welcome text), DDT-1159 (PI contract texts sent from the PI line) |
| **Code** | DDT-217: built in production Flow Builder, no PR. DDT-772: PR #213 in the `ddt-salesforce` repo |

---

## 1. Summary

DDT-217 added two flows that send a client a one-time welcome SMS through Heymarket **24 hours after they are hired**: one for defense clients on the Court Case (Opportunity), one for PI clients on the Matter. Both used the same text, with the firm's defense line as the number to save.

DDT-772 found that PI clients were therefore being told to save the **defense** line, so their callbacks went to the wrong team. The PI flow's message was changed to the **DDT Injury Team** branding with the **PI line**.

## 2. What was reported

- **DDT-217 (2026-06-22).** Client services asked for an automated text 24 hours after a client becomes hired, reminding them the firm is there for a ticket, arrest or injury and asking them to save the firm's number. In the thread the requester named `Hire_Status__c` as the hired marker, asked for it to go to **both** Matter clients and Court Case clients, and asked for it to come from the firm's main number.
- **DDT-772 (2026-08-19).** A PI client received the welcome text with the general defense number, then followed up asking when they would be called about their appointment. If PI clients save that number, their calls land on the wrong queue. The thread asked for defense clients to keep the main line, PI clients to get the PI line, and PI texts to say **DDT Injury Team** rather than Driver Defense Team.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Court_Case_Hired_Welcome_Text` ("[Court Case] Hired Welcome Text") | Record-triggered flow on `Opportunity` | New (DDT-217). Defense text, main defense line |
| `Matter_Retained_Welcome_Text` ("[Matter] Retained Welcome Text") | Record-triggered flow on `Matter__c` | New (DDT-217). Message changed to DDT Injury Team and the PI line (DDT-772) |

**Out of scope**

- **Which Heymarket inbox sends the text.** DDT-772 changed the message wording and the number in it. Both flows still look up the same Heymarket team mapping to choose the sender.
- **Closing the Heymarket conversation after the send.** Sending reopens the chat in the inbox. That is DDT-1073.
- **Other practice areas.** The ticket asked to confirm whether any need their own number. No other branch is recorded.

## 4. How it works (background)

Both flows have the same shape:

```
Record saved with Hire_Status__c = "Hired" (Opportunity) or "Retained Active" (Matter__c)
  (start condition requires the record to have changed to meet it)
  -> scheduled path: 24 hours after the trigger event
  -> Decision: is Hire_Status__c still the hired value?        no -> stop
  -> Get the client Account
  -> phoneToUse = PersonMobilePhone, or Phone if mobile is blank
  -> Decision: is phoneToUse present?                          no -> stop
  -> Get the Heymarket team mapping for the sending user
  -> heymarket__SendUpto5KMessagesAction
       inbox = that user's default inbox number, message = tmpl_WelcomeMessage,
       phone = phoneToUse, salesforceID = the Account
```

The re-check after 24 hours is what stops a text going to a client who was un-hired within the window.

## 5. Root cause

**DDT-772.** When DDT-217 extended the text to Matter clients as well as Court Case clients, the Matter flow was given the same message as the Court Case flow, including the defense line. Matters are the PI side of the firm, so every retained PI client was told to save the defense number.

The ticket's guess at the trigger ("after a new lead/client is created", on Lead or Contact) was not right: the text is sent from the **Matter** (PI) and **Opportunity** (defense) flows, 24 hours after hire. Splitting by practice area therefore needed no new branch. The two flows already were the split, and only the Matter flow's template had to change.

## 6. Solution

### 6.1 DDT-217: two scheduled-path flows

Built so the text goes to both client groups, as the requester asked.

| | Court Case flow | Matter flow |
|---|---|---|
| Object | `Opportunity` | `Matter__c` |
| Hired value | `Hired` | `Retained Active` |
| Client Account | `AccountId` | `Account__c` |
| Delay | 24 hours after the trigger event | 24 hours after the trigger event |

Why a scheduled path and not a scheduled flow: the delay is measured from the moment each record became hired, and the path re-reads the record when it runs, so the "still hired?" check sees the current value.

### 6.2 DDT-772: PI wording and number

The Matter flow's text template now reads (number replaced here with a placeholder):

```
You're part of the DDT Injury Team now. If you ever need us for a ticket, arrest,
or injury, we're in your corner. Save our number: <PI line>.
```

The Court Case flow keeps the original:

```
You're part of the Driver Defense Team family now. If you ever need us for a ticket,
arrest, or injury, we're in your corner. Save our number: <main defense line>.
```

Per the closing comment, the PI text goes out 24 hours after the Matter's status becomes Retained Active.

## 7. Testing

- **DDT-217.** Not recorded in the ticket beyond the requester's thanks on 2026-06-23.
- **DDT-772.** Not recorded in the ticket. The requester asked how to test it; the answer given was to set a Matter to Retained Active and expect the text 24 hours later.
- **State on 2026-11-05.** Both flows are Active in production. Their stored copies (production state as of 2026-09-24) show the Matter flow with the DDT Injury Team text and the Court Case flow with the defense text.

**Not covered:** no automated test. A scheduled path cannot be fast-forwarded in a sandbox without waiting the 24 hours, and the sandbox has no live Heymarket connection.

## 8. Step by step

**Build a delayed welcome text**

1. Confirm the hired marker with the requester. Here: `Hire_Status__c` = `Hired` on Opportunity, `Retained Active` on Matter.
2. Create a record-triggered flow, after save, on create and update, with the start condition on that value and "only when a record is updated to meet the condition".
3. Add a scheduled path: 24 hours after the trigger event.
4. On the path, re-check the hired value first. Stop if it changed.
5. Get the client Account, build the phone from mobile then phone, and stop if both are blank.
6. Get the Heymarket team mapping for the sending user and call `heymarket__SendUpto5KMessagesAction` with that user's default inbox, the message template, the phone and the Account Id.
7. Activate. Records that were already hired before activation are not picked up; only records saved into the hired value afterwards start the path.

**Change the number for one practice area (DDT-772)**

1. Identify which flow serves that practice area (Matter for PI, Opportunity for defense).
2. Open that flow in Flow Builder, save as a new version.
3. Edit `tmpl_WelcomeMessage` only.
4. Activate the new version and check the old one is inactive.

## 9. Rollback

- **Message text:** activate the previous flow version. Texts already sent cannot be recalled.
- **Turning the text off:** deactivate the flow. What happens to paths already waiting at that moment is not recorded in the ticket, so check the pending scheduled paths in Setup before relying on it.

## 10. Known limits

- **"Sends only once" is in the description, not the logic.** Both flow descriptions say they are guarded by `Welcome_Text_Done__c`, but no field by that name exists on either object in production (checked 2026-11-05), and the decision checks only `Hire_Status__c`. A record that leaves the hired value and returns to it will start the path again.
- **The sender is one hardcoded user.** Both flows find the Heymarket mapping by a single Salesforce user Id written into the flow. If that user leaves or loses Heymarket, the texts stop. Look it up by a setting rather than an Id if this is reworked.
- **The sending inbox is shared by both flows.** DDT-772 changed what the PI text says, not which inbox it is sent from.
- **Sending reopens the Heymarket chat.** Handled separately by DDT-1073.
- **No opt-out check is in the flow.** DDT-217 asked for opt-out suppression; the flows check hire status and phone only. Whether Heymarket itself suppresses opted-out numbers is not recorded in the ticket.

## 11. Reuse for similar issues

- **When one automation serves two audiences, check every template.** Copying a flow for a second group copies its wording, numbers and branding too.
- **Find the real trigger before designing a branch.** Here the practice-area split already existed as two flows; the fix was one template.
- **Delay from the event, then re-check.** A scheduled path 24 hours after the trigger, with the condition re-tested when it runs, avoids texting someone whose status changed in the meantime.
- **Read a flow's description as a claim to verify.** A guard mentioned in the description is not proof the guard exists.
- **Never hardcode a user Id as a sender.** Use a custom setting or metadata record so the sender can change without a flow edit.
