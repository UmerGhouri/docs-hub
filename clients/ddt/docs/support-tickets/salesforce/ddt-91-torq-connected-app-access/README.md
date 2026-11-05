# Torq integration connected app and read-only access

| | |
|---|---|
| **Ticket** | DDT-91, DDT-179 |
| **Project** | Support tickets |
| **Status** | Done 2026-06-17 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-06-02 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (External Client App, permission set, integration user) |
| **Area** | Integrations: giving Torq Legal Labs server-to-server API access to the org |
| **Related** | DDT-93 (client account field cleanup, which changes fields an integration may read) |
| **Code** | Not applicable: configured in Setup, no pull request |

---

## 1. Summary

Torq Legal Labs, a third-party legal-tech provider, needed to read from the Salesforce org without logging in as a real person. We set up a Salesforce **External Client App** (the modern replacement for a Connected App) for programmatic, server-to-server sign-in, tied to a **dedicated integration user** whose access comes from a single permission set.

DDT-91 asked for the app and its policies. DDT-179 settled the open question DDT-91 left, which sign-in pattern to use, by choosing one that needs no interactive login and no person in the loop.

## 2. What was reported

- **DDT-91 (2026-06-02):** set up an External Client App so Torq can call Salesforce securely, instead of relying on one person's own login.
- **DDT-179 (2026-06-17):** written as the concrete implementation of DDT-91's open question: a read-only integration user and a non-interactive sign-in, so Torq can read schema and data on a schedule. It was closed the same day it was opened, because the work had already been done under DDT-91.

Nothing was broken. This was new setup work.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| External Client App for Torq | Setup configuration | Created, with OAuth enabled and sign-in settings supplied by Torq |
| Integration user for Torq | User | Dedicated user, not tied to a real person |
| Pre-authorization permission set | Permission set | Assigned to the integration user; only users holding it may use the app |
| Credentials handoff document | Internal document | Holds what Torq needs to connect (shared privately, not in this page) |

**Out of scope**

- **Exactly which objects and fields Torq may read.** Both tickets list this as an open question to settle with Torq. The completion note confirms only that Lead object metadata was readable. Not recorded in the ticket beyond that.
- **Audit logging of Torq's API calls** and **key rotation cadence.** Raised as open questions in DDT-179; no decision is recorded.

## 4. How it works (background)

```
Torq's service
  -> connects through the External Client App, with no person logging in
  -> Salesforce runs the session as the dedicated integration user
  -> Torq reads data, limited to what that user's permission set grants
```

Two Salesforce rules make this safe:

- **The app only admits pre-authorized users.** Permitted Users is set to "Admin approved users are pre-authorized", so a user must hold the assigned permission set to use the app at all.
- **What Torq can see is what the integration user can see.** Object and field access come from that user's permission set, so the permission set is where access is widened or narrowed.

## 5. Root cause

Not applicable: this was a request for new setup, not a defect.

The reasoning in the tickets for the design choices:

- **A dedicated integration user**, so the integration does not depend on, or break with, one employee's account.
- **A non-interactive sign-in**, because no person is present when Torq's service connects.
- **Least privilege**, because the tickets name over-broad access as the most common cause of integration security incidents. Start with the minimum and add on request.

## 6. Solution

Done in Setup, as recorded in the completion note on DDT-91 (2026-06-04):

1. Created the External Client App and entered the sign-in settings Torq supplied.
2. Configured the OAuth settings the app needs.
3. Set pre-authorization through a dedicated permission set, and assigned it to the integration user.
4. Wrote a handoff document holding what Torq needs to connect, shared privately.

Why it was done this way:

- **Pre-authorization through a permission set**, not "all users may self-authorize". It keeps the list of who can use the app to exactly the integration user.
- **Credentials shared privately**, never in Slack or email in plain text, as both tickets require.

**Repo note.** Both tickets asked for the app, the permission set and a runbook to be committed to the `ddt-salesforce` repo. The repo holds one Torq External Client App, `Torq_Legal_Labs_Integration`, which was retrieved from production on 2026-05-22, before these tickets, and whose description says it inserts Lead records. Whether it is the same app configured here is not recorded in the ticket. No `Torq_Schema_Read_Access` permission set and no `docs/integrations/torq.md` runbook exist in the repo.

## 7. Testing

From the completion note on DDT-91:

- The sign-in was tested end to end with Postman.
- An access token was returned.
- Lead object metadata was readable as the integration user.

**Not covered:** DDT-179's acceptance criteria also asked to prove that write, update and delete attempts fail. That check is not recorded in the ticket.

## 8. Step by step

1. **Agree the access list with the vendor first:** which objects, which fields, read-only or read-write. The permission set is built from this list.
2. **Create the integration user.** Pick the cheapest licence that covers the objects needed, and give it a minimum-access profile.
3. **Build the permission set.** Object Read only where needed, field Read only where needed, API Enabled, nothing else.
4. **Create the External Client App** in Setup → External Client App Manager. Enable OAuth and enter the sign-in settings the vendor supplies, keeping scopes to the minimum.
5. **Set the policies:** Permitted Users to "Admin approved users are pre-authorized", then add the permission set from step 3 as the pre-authorization.
6. **Assign the permission set** to the integration user.
7. **Test the sign-in end to end** and confirm a token comes back.
8. **Confirm read-only:** attempt a write as the integration user and confirm it is refused.
9. **Hand off the connection details privately**, and keep the firm's own copy in its password vault.

## 9. Rollback

- **To cut Torq off immediately:** remove the pre-authorization permission set from the integration user, or deactivate the user. Either stops new sessions.
- **To remove the app entirely:** delete the External Client App in Setup. This is not reversible; the app and its settings would have to be rebuilt with the vendor.

## 10. Known limits

- **The access list was never fully written down.** Until it is, the permission set is the only record of what Torq can see. Review it before widening anything.
- **Anyone holding Torq's credentials can act as the integration user.** The tickets suggest watching that user's API activity for anomalies; no monitoring is recorded as set up.
- **Field reshuffles affect the integration.** DDT-93 moved fields between the client Account and the Matter. Any field Torq reads that moves or is deprecated breaks Torq silently.

## 11. Reuse for similar issues

**How to recognise one:** a vendor or internal service needs to read or write Salesforce data with nobody logged in.

- **Never reuse a person's login** for an integration. Create a dedicated user.
- **Gate the app with pre-authorization**, so only users holding a named permission set can use it.
- **Grant access through one permission set** you can read in one place, starting from nothing.
- **Write down the access list** with the vendor before building, and keep it with the permission set.
- **Test the negative case:** prove a write is refused, not only that a read succeeds.
