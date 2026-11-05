# Email to Legal POD flow: welcome email, tasks and sender

| | |
|---|---|
| **Ticket** | DDT-139, DDT-514 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-27 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-06-10 (DDT-139) and 2026-07-27 (DDT-514) |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow, Apex invocable action) |
| **Area** | Onboarding a newly hired client: notifying the legal POD, welcoming the client, creating tasks |
| **Related** | DDT-140 and DDT-141 (later onboarding stages, downstream of this flow) |
| **Code** | No PR or commit in the ddt-salesforce repo references either ticket |

---

## 1. Summary

The **Email To Legal POD** flow notified a legal POD when a newly hired matter reached it. DDT-139 extended it to also send the client a welcome email, give the Lead Attorney a to-do, and create the onboarding tasks that later stages use as gates. DDT-514 then fixed who the two emails come from: the client welcome email now comes from the Case Manager, and the internal "New Case Assignment" email from the CCS address, instead of from whichever user's edit triggered the flow. Because a flow's standard Send Email action cannot send on behalf of another user, a small Apex action, `SendWelcomeEmailAction`, does the sending.

## 2. What was reported

- **2026-06-10 (DDT-139):** the flow notified the POD when a matter landed, but the client welcome email and the onboarding tasks were not part of it. That left manual gaps at the moment a new client expects a response, and new staff could not rely on the system to show them the work. The change was built and closed the same day.
- **2026-07-27 (DDT-514):** both emails the flow sent came from the same sender, the user whose change triggered the flow. A welcome email from that user, or a generic address, felt impersonal to the client; and replies to the internal assignment email went to that individual instead of to CCS for triage. The change was built and closed the same day.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Email To Legal POD | Record-triggered flow | DDT-139: new trigger stage, welcome email, to-do, three onboarding tasks, User lookups. DDT-514: both email steps replaced by an Apex action |
| `SendWelcomeEmailAction` | Apex class (invocable action) | DDT-514: new. Sends an email as a chosen sender (Case Manager or CCS) |

**Out of scope**

- **Moving the matter to the next stage automatically.** The later stage changes (Post-Hire to Awaiting Welcome Call, Welcome Call to Onboarded) are DDT-140 and DDT-141.
- **A "meet your new Case Manager" email when the Case Manager is reassigned.** Raised as an open question in DDT-514; not built.
- **The internal email's recipients, CC, subject and body.** Unchanged by DDT-514.

## 4. How it works (background)

After both tickets:

```
Matter's stage changes to "Legal – Pending Tasks"
  -> Email To Legal POD flow runs
     -> looks up the Case Manager, Legal Assistant 1 and Legal Assistant 2 Users (for names in the email)
     -> "New Case Assignment" email to the POD          sent by SendWelcomeEmailAction, as CCS
     -> client welcome email                            sent by SendWelcomeEmailAction, as the Case Manager
          to the Primary Contact if there is one, otherwise the client
     -> To-Do "Post-hire tasks" for the Lead Attorney, due in 2 days
     -> onboarding tasks:
          Post-Hire – Case Manager   (Case Manager)
          Post-Hire – Attorney       (Lead Attorney)
          Welcome Call               (Lead Attorney)
```

The three onboarding tasks are the gates the later onboarding automation checks.

## 5. Root cause

### DDT-139

Not a defect: a request to extend the flow. Before the change, the flow's trigger used an older stage value, and the welcome email and onboarding tasks were done by hand.

### DDT-514

The flow's emails were sent with the standard Send Email action, which sends as the **running user**: whoever made the change that triggered the flow. That action cannot send on behalf of a different user, so it could not send as the Case Manager or as CCS.

## 6. Solution

### 6.1 DDT-139: extend the flow

Done in the flow itself, on 2026-06-10:

1. **Trigger changed** from the old stage value to **Legal – Pending Tasks**.
2. **Client welcome email added**, sent to the client's email address, introducing their Case Manager, Lead Attorney and Legal Assistants with their contact details.
3. **To-Do added:** "Post-hire tasks", assigned to the Lead Attorney, due in 2 days.
4. **Three onboarding activity tasks added:** Post-Hire – Case Manager (Case Manager), Post-Hire – Attorney (Lead Attorney), Welcome Call (Lead Attorney).
5. **User lookups added** for Case Manager, Legal Assistant 1 and Legal Assistant 2, so their names resolve correctly in the client email.

### 6.2 DDT-514: send each email from the right person

Done on 2026-07-27, deployed to production and the Partial sandbox:

- **`SendWelcomeEmailAction`**, a new Apex invocable action, sends an email as a chosen person (the Case Manager or CCS).
- **The flow calls it** in place of the native Send Email step, for both emails.
- **Client welcome email:** body rewritten; sent from the **Case Manager**, who is also the person who signs it; sent once, to the Primary Contact if there is one, otherwise the client.
- **"New Case Assignment" email:** sent from **CCS** instead of the current user. Recipients, CC, subject and body unchanged.

Why it was done this way:

- **The welcome email comes from a person** the client will actually work with, and replies reach that person.
- **The internal email comes from CCS** so replies route to CCS for triage, and attorneys can see it is a system notification.
- **Apex instead of the standard action**, because the standard action can only send as the running user.

The ticket does not record how `SendWelcomeEmailAction` sets the sender (for example, an organization-wide email address), or what it does when no Case Manager is assigned. DDT-514 asked for a fallback to CCS or intake in that case; whether it was built is not recorded.

## 7. Testing

- **DDT-139:** the ticket asked for an end-to-end test on a sample hired matter across several case types. The result is not recorded in the ticket.
- **DDT-514:** the ticket asked for four checks: Case Manager assigned, Case Manager missing, the internal email from CCS, and replies routing correctly. The results are not recorded in the ticket.
- **Apex test:** not recorded in the ticket.
- **Not covered:** idempotency (DDT-139 required that a re-run must not resend the welcome email or duplicate tasks). Whether it was handled is not recorded.

## 8. Step by step

To make a similar change:

1. **Open the flow** in Setup → Flows and note its trigger, decisions and actions before changing anything.
2. **Set the trigger** to the stage value the process now uses.
3. **Add Get Records steps for the Users** whose names or addresses go in the email.
4. **Add the email and task steps.** Give each task a clear subject, owner and due date relative to the trigger.
5. **If an email must come from someone other than the running user,** use an invocable Apex action to send it rather than the standard Send Email action.
6. **Guard against re-runs** so a stage that is re-entered does not resend emails or duplicate tasks.
7. **Test in a sandbox** with and without each optional person (Case Manager, Primary Contact) populated.
8. **Get approval, deploy, and activate the flow version in production.**

## 9. Rollback

- **Flow:** activate the previous flow version in Setup → Flows. Emails go back to the earlier sender, or the DDT-139 additions stop, depending on which version is restored.
- **Apex action:** leave it in place; once the flow stops calling it, it does nothing.

## 10. Known limits

- **What runs in production today is not recorded here.** A read-only query on 2026-11-05 found no flow whose label contains "POD" and no Apex class named `SendWelcomeEmailAction`, and neither is in the ddt-salesforce repo. When or why they were renamed, replaced or removed is not recorded in these tickets. Check the org before relying on this page:

  ```sql
  SELECT ApiName, Label, IsActive FROM FlowDefinitionView WHERE Label LIKE '%POD%'
  ```

  ```sql
  SELECT Name, LastModifiedDate FROM ApexClass WHERE Name = 'SendWelcomeEmailAction'
  ```

- **No record in git.** Neither ticket has a commit or PR, so the flow and class versions that went live are not reviewable from the repo.
- **Case Manager reassignment.** The welcome email has already gone from the first Case Manager; nothing tells the client about a new one.

## 11. Reuse for similar issues

- **An email from a flow comes from the running user** unless something else is configured. If it must come from a specific person or a shared address, decide that up front.
- **When the standard action can't do it, a small invocable Apex action can,** and the flow stays the place where the logic lives.
- **Tasks used as gates** for later automation are part of that automation's contract. Rename or remove one only together with the later steps that depend on it (DDT-140, DDT-141).
- **Commit flow and Apex changes to the repo through a PR** when they go live. These two tickets left no trace in git, which is why today's state can't be read from it.
