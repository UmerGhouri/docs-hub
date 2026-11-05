# Assign Lead chat button in case conversion

| | |
|---|---|
| **Ticket** | DDT-317 |
| **Project** | Support tickets |
| **Status** | Done 2026-07-13 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-07-13 |
| **Resolved by** | Support team |
| **Systems** | Google Chat (Workspace add-on), Google Cloud Run service, Salesforce (Apex, Flow) |
| **Area** | Case conversion: assigning a lead attorney from the Google Chat card |
| **Related** | None recorded |
| **Code** | Not recorded in the ticket. No commit for DDT-317 exists in the ddt-salesforce repo. |

---

## 1. Summary

The **Assign Lead** button on the case-conversion card in Google Chat did nothing for every user. The button was configured with a function name, but the Chat app is an HTTP Workspace add-on, which needs the full address of the service. So the click never reached the service. We pointed the button at the service's own address, taught the service to read the add-on event format, and added a clear message for users who are not allowed to assign.

## 2. What was reported

Clicking **Assign Lead** on the case-conversion card did nothing, so intake staff could not assign a lead to the right attorney during conversion. It failed for all users.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Case-conversion service (`cloud-run/index.js`) | Node service on Google Cloud Run | Button action uses a new `SELF_URL` setting; handles the Workspace add-on event and response format (classic format kept as a fallback); more robust attorney and matter extraction; "not authorized" message for non-deciders |
| `SELF_URL` | Service environment variable | New. Holds the service's own address |

Deployed to a sandbox to test the full chain (not changed by this ticket, but needed for the test): `CaseConversionRest`, `CaseConversionNotifier`, a Named Credential, the `CC_Escalation_Base__c` field, and the `Case_Conversion_Notify` flow.

**Out of scope:** the fault-path work across other flows that the ticket mentions as an org-wide gap.

## 4. How it works (background)

```
Matter stage changes → Case_Conversion_Notify flow fires
  → Salesforce posts a card to Google Chat (via the Cloud Run service)
  → a decider clicks "Assign Lead" on the card
  → Google calls the Cloud Run service with the click event
  → the service writes Lead_Attorney__c and advances Onboarding_Stage__c in Salesforce
```

## 5. Root cause

The card button's action held a **function name**. That works for a classic Chat app, but an **HTTP Workspace add-on** app requires the button to carry the **full endpoint address** of the service. With only a function name, Google had nowhere to send the click, so nothing happened and no error was shown.

## 6. Solution

1. Added a `SELF_URL` setting to the service and pointed the card button at it, so every click carries the full address.
2. Added handling for the Workspace add-on event and response format. The classic format is kept as a fallback.
3. Made the extraction of the attorney and the matter from the event more robust.
4. Added a clear "not authorized" reply for users who are not allowed to make the assignment, instead of a silent failure.
5. Removed test-only code and test files from `index.js` before committing.

**Why both pieces are needed:** the new code reads `SELF_URL`. Deploy the code without setting the variable and the button stays broken.

## 7. Testing

Tested in a sandbox, per the ticket:

- Clicking **Assign Lead** set `Lead_Attorney__c` and advanced `Onboarding_Stage__c` to "Legal – Pending Tasks", confirmed by SOQL.
- Tested with multiple users; a non-authorized user was blocked with the new message.
- The automatic card posting on stage change still fired.

**Not covered:** a production check after deploy is not recorded. A query of production Apex classes on 2026-11-05 returned no class named `CaseConversionRest` or `CaseConversionNotifier`.

## 8. Step by step

1. Reproduce: click **Assign Lead** on a case-conversion card and confirm nothing happens.
2. Check how the Chat app is registered. If it is an HTTP Workspace add-on, button actions need the full service address.
3. Add the `SELF_URL` setting in the service code and use it for the button action.
4. Add parsing for the add-on event format, keeping the classic format as a fallback.
5. Test in a sandbox with an authorized and a non-authorized user, and confirm the Salesforce fields change.
6. Deploy the updated `cloud-run/index.js` and set the `SELF_URL` environment variable on the Cloud Run service. Both are required. The ticket says full details are in `cloud-run/PROD_DEPLOY.md`.

## 9. Rollback

Redeploy the previous service revision in Cloud Run. That brings back the broken button, so only do it if the new revision fails.

## 10. Known limits

- The button depends on `SELF_URL` being right. If the service moves to a new address, update the variable or the button breaks again in the same silent way.
- The service code is not in the ddt-salesforce repo, so its current version can only be read from Cloud Run.

## 11. Reuse for similar issues

- **A Chat button that does nothing for everyone** points at the app's configuration, not at Salesforce. Check whether the app is a classic Chat app or a Workspace add-on: the two use different event formats and different button actions.
- When a service needs to know its own address, keep it in an environment variable, not in code.
- Give users who lack permission a clear message; a silent no-op looks exactly like a bug.
