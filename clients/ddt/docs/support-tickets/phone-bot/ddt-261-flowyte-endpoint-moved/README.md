# Flowyte phone routing failing after the API moved

| | |
|---|---|
| **Ticket** | DDT-261 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-02 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-07-02 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (custom setting, Apex callout), Flowyte call-routing service |
| **Area** | Phone routing: pushing each caller's number and owner extension to the call router |
| **Related** | DDT-99 (error handling and failure alerts on the same callout), DDT-498 (converted leads excluded from the push) |
| **Code** | No pull request. The endpoint is a data value in a custom setting record, not code |

---

## 1. Summary

Salesforce pushes routing rules to Flowyte so incoming calls reach the right person. Flowyte moved its API from the bare domain to the `www` subdomain, and the old address started answering with **HTTP 301 (moved permanently)**. Salesforce callouts do not follow redirects, so every push failed. The fix was to point the endpoint at the new host. The endpoint lives in a custom setting record, so this was a data change in Setup, not a deploy.

## 2. What was reported

On 2026-07-02 the routing callouts were returning HTTP 301 instead of a success code. The impact named in the ticket: when routing rules don't reach Flowyte, client calls may not reach the right attorney or queue, and the failure is silent from the caller's side.

How many calls were affected, and for how long, is **not recorded in the ticket**. It can no longer be measured: the oldest row in `Flowyte_Error_Log__c` today is from 2026-10-05, so the July failure logs no longer exist.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Flowyte_Credentials__c`, record `Production`, field `End_Point__c` | Custom setting record | Host changed from the bare domain to the `www` subdomain; path unchanged |

**Out of scope**

- **`FlowyteRoutingService` code.** It reads the endpoint from the custom setting, so no code change was needed. The ticket deliberately rejected adding redirect-following in code: chasing a 301 adds complexity and leaves the integration pointing at the wrong address.
- **Failure alerting.** That is DDT-99, which already existed for this service.

## 4. How it works (background)

```
A Lead, Contact or Account in the routing feed changes in Salesforce
  -> FlowyteRoutingService builds a routing rule (number -> extension)
  -> reads every active Flowyte_Credentials__c record
       (Name, End_Point__c, API key, customer id)
  -> POSTs the rule to each record's End_Point__c
  -> any non-2xx response is logged to Flowyte_Error_Log__c as an Error
```

Since 2026-07-19 the same service also pushes to a second target, a replacement call router, through a second `Flowyte_Credentials__c` record. That change came later and is not part of this ticket.

**The Salesforce rule behind this:** `HttpRequest` does **not** follow redirects. A 301 comes back to Apex as a plain 301 response, which the service treats as a failure.

## 5. Root cause

Flowyte moved its API host. The `Production` record's `End_Point__c` still named the old host, so every POST was answered with a 301, which Apex does not follow.

The ticket listed four places the endpoint could live (Named Credential, custom metadata, custom setting, hardcoded Apex). In this org it is the `Flowyte_Credentials__c` custom setting: `FlowyteRoutingService` passes `End_Point__c` straight to `req.setEndpoint(...)`.

**Evidence today (2026-11-05):**

```sql
SELECT Name, End_Point__c, LastModifiedDate FROM Flowyte_Credentials__c
```

The `Production` row now points at the `www` subdomain. Its `LastModifiedDate` is 2026-07-29, a later edit, so this query cannot show the exact date of the host change.

## 6. Solution

Edit the `Production` record of the `Flowyte_Credentials__c` custom setting and change the host in `End_Point__c` to the `www` subdomain, keeping the same path.

Why this way:

- **Data, not code.** The endpoint was already configurable, so a Setup edit takes effect on the next push with no deploy and no tests.
- **Fix the address rather than follow the redirect.** Following a 301 in Apex means a second callout on every push and keeps the integration dependent on Flowyte maintaining the redirect. If Flowyte drops the redirect, a 301 would turn into a 404 or a connection failure.

## 7. Testing

The ticket's plan: trigger a routing push, confirm no 301, and confirm the call actually routes, not just that the HTTP call succeeds. Whether each step was carried out, and the result, is **not recorded in the ticket**. It was opened and closed within a minute, with no comments.

**What can be shown today:** the `Production` record points at the `www` subdomain (query in section 5).

**Not covered:** no automated test checks the endpoint value. The Apex tests insert their own fake `Flowyte_Credentials__c` record and mock the callout, so they pass whatever production's record says.

## 8. Step by step

1. **Confirm the symptom.** Check `Flowyte_Error_Log__c` for recent rows with `Response_Code__c = 301`.
2. **Find the endpoint.** Setup → Custom Settings → `Flowyte_Credentials__c` → Manage. Open the record the service uses (`Production` for Flowyte).
3. **Edit `End_Point__c`.** Change only the host to the vendor's new host. Keep the path.
4. **Check the Remote Site Setting.** A callout to a new host needs a Remote Site Setting (or Named Credential) that covers that exact host. The `flowyte` remote site in the repo still names the bare domain. Whether production has one for the `www` subdomain is not recorded in git, so check Setup → Remote Site Settings before relying on the change.
5. **Trigger a routing push** on a test record and confirm the new `Flowyte_Error_Log__c` row (if any) shows a 2xx, not a 301.
6. **Place a test call** to confirm it routes to the expected extension.

## 9. Rollback

Set `End_Point__c` back to the previous value. That only makes sense if the vendor reverts its move. Otherwise it brings the 301s back.

## 10. Known limits

- **Remote site drift.** The repo's `flowyte` remote site names the bare domain. If production is ever rebuilt from git, callouts to the `www` subdomain would be blocked as an unauthorised endpoint.
- **The outage window is unknown** and can no longer be measured (section 2).
- **Flowyte is being replaced.** The second push target added on 2026-07-19 is a replacement call router, so this record matters only while Flowyte is still active.

## 11. Reuse for similar issues

**How to recognise one:** an integration that worked fails with a 3xx (301, 302, 307, 308) status, with no change on our side.

- **Read the `Location` header** of the 3xx response. It names the vendor's new address.
- **Find where the endpoint lives** before touching code: custom setting, custom metadata, Named Credential, or a literal in Apex. Search the repo for the old host.
- **Update the address. Don't add redirect-following.**
- **Update the Remote Site Setting or Named Credential** in the same change, or the callout fails with an unauthorised-endpoint error instead.
- **Alert on any non-2xx**, not only on exceptions (DDT-99), so the next vendor move is noticed before users notice it.
