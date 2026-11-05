# Error handling and failure alerts for Flowyte routing callouts

| | |
|---|---|
| **Ticket** | DDT-99 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-03 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-06-03 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex Queueable, custom object), Flowyte call-routing service |
| **Area** | Phone routing: pushing each caller's number and owner extension to the call router |
| **Related** | DDT-498 (converted leads excluded from the push), DDT-1490 (log only failed pushes) |
| **Code** | No pull request. The change was deployed straight to production on 2026-06-03 and reached git later, in the bulk backfill commit dated 2026-07-03 |

---

## 1. Summary

`FlowyteRoutingService` pushes routing rules (a caller's phone number and the extension that should answer it) to Flowyte from a Queueable job. When the callout failed, for example on a timeout, the exception was not caught, so the job crashed and nobody found out until a call went to the wrong place.

The fix catches every callout failure, records it in `Flowyte_Error_Log__c`, and sends one summary email per job run listing the failures.

## 2. What was reported

The ticket described the routing callouts as failing silently: on a timeout, an HTTP error or a malformed response, records were not routed and the gap was found only downstream, as a missing assignment or a missed callback. It asked for defensive error handling and proactive alerts so the right people know within minutes.

How many routings had been lost before the fix is not recorded in the ticket.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `FlowyteRoutingService.postUpsert` | Apex method | `http.send` wrapped in try/catch; a failure returns an `Error` log row instead of throwing |
| `FlowyteRoutingService.execute` | Apex method (Queueable) | After saving the log rows, collects the failures and calls `sendFailureAlert` |
| `FlowyteRoutingService.sendFailureAlert` | Apex method | New. One plain-text email per run listing each failure |
| `FlowyteRoutingServiceTest.testCalloutTimeoutIsLogged` | Apex test | New. A timed-out callout produces one `Error` log and does not fail the job |
| `Flowyte_Error_Log__c` | Custom object | Used to record each push: status, response code, response message, payload, record Ids, object name |

**Out of scope** (asked for in the ticket, not in the delivered code)

- **Retries with backoff.** `postUpsert` makes one attempt. A failed push is logged, not retried.
- **A new `Integration_Error__c` object.** The existing `Flowyte_Error_Log__c` was used instead.
- **Slack alerts, a distribution list, and throttling of repeated errors.** The alert is an email to one fixed recipient.
- **A "Routing Failed" banner with a Retry button on the record.** Not recorded as delivered.
- **Scrubbing personal data from the stored payload.** Not recorded as delivered (see 10).

## 4. How it works (background)

```
LeadService / ContactService / AccountService, or a scheduled routing batch
  queues the records' routing mappings to FlowyteRoutingService (Queueable, allows callouts)
  execute() groups the mappings by routing extension
  postUpsert() POSTs each group to every active Flowyte_Credentials__c target
  one Flowyte_Error_Log__c row per target is saved
```

Before the fix, `http.send(req)` sat outside any try/catch. A `CalloutException` such as "Read timed out" was unhandled, so the whole Queueable failed: no log row was saved for any group in that run, and no one was told.

## 5. Root cause

An uncaught exception on the callout. A timeout or network failure throws a `CalloutException` rather than returning a response with an error status, and the code only handled responses. The job died, taking its logging with it, so the failure left no trace in Salesforce.

What the alert recipient and investigation found beyond that is not recorded in the ticket.

## 6. Solution

### 6.1 Catch the callout and log it

```apex
HttpResponse res;
try {
    res = http.send(req);
} catch (Exception ex) {
    // Callout failed (e.g. "Read timed out"). Log it instead of letting the
    // Queueable throw an unhandled exception, so we keep a record of the attempt.
    fel.Status__c = 'Error';
    fel.Response_Code__c = 0;
    fel.Response_Message__c = 'Callout failed: ' + ex.getMessage();
    return fel;
}
```

- **Response code 0** marks "no response at all", which is different from an HTTP error code returned by Flowyte.
- **Returning a log row, not throwing,** keeps the rest of the run going: other groups and other targets are still pushed, and every row is saved.

### 6.2 Email a summary of the failures

After the log rows are inserted, `execute` collects the `Error` rows and calls `sendFailureAlert`, which sends one email per run listing response code, object, record Ids, message and payload for each failure. The send itself is wrapped in try/catch so a failing alert can never break the job.

### 6.3 Later changes to the same code

- **2026-07-19:** rate-limit responses (HTTP 429) stopped triggering the email. A full resync bursts past Flowyte's per-minute limit by design and the rule is re-pushed, so a 429 is expected noise. The rule is now `isAlertable`:

  ```apex
  private static Boolean isAlertable(Flowyte_Error_Log__c logRec) {
      return logRec.Status__c == 'Error' && logRec.Response_Code__c != 429;
  }
  ```

- **DDT-1490 (2026-11-02, PR #869):** `Flowyte_Error_Log__c` rows are now saved only for failed pushes; about 5,000 a week had been successes.

## 7. Testing

- **`testCalloutTimeoutIsLogged`.** A mock throws `CalloutException('Read timed out')`. The test runs the Queueable and asserts exactly one log row with `Status__c = 'Error'`, `Response_Code__c = 0`, and a message containing "Callout failed". Without the try/catch the exception would escape the Queueable and the test would fail.
- **`testRateLimit429NotAlerted`** (added with the 429 change). Asserts that a 429 and a success are not alertable, and that a 500 and a timeout (code 0) are.

Live check after the deploy: not recorded in the ticket.

**Not covered:** whether the alert email is delivered; retries (there are none).

## 8. Step by step

1. **Find every callout** in the service and check whether `http.send` is inside a try/catch.
2. **Catch the exception and return a failure record** instead of throwing, so the Queueable finishes and saves what it has.
3. **Save the log rows first, then alert,** so a failed alert never loses the log.
4. **Wrap the alert in its own try/catch.**
5. **Decide which failures are worth an alert.** Expected, self-healing errors (here, 429) should be logged but not emailed.
6. **Add a test with a mock that throws**, and assert the log row, not just that nothing crashed.
7. **Validate against production with that test class, get approval, deploy,** and open a pull request the same day so the change is in git.

## 9. Rollback

Redeploy the previous `FlowyteRoutingService`. That brings back the crashing job and the silence, so there is no reason to unless the new code itself fails. The log object and its rows are unaffected.

## 10. Known limits

- **No retry.** A failed push stays failed until the record is pushed again, by a later change or by the scheduled routing batch (`FlowyteRoutingBatchScheduler`).
- **One hardcoded email recipient.** If that person leaves or the address changes, alerts go nowhere. A custom setting or a public group would be sturdier.
- **The stored payload and the alert email contain caller phone numbers.** The ticket listed scrubbing personal data as a risk; it was not done.
- **No pull request at the time.** The change existed only in production for a month until the 2026-07-03 backfill. That gap is what the repo's "a PR the same day for anything put in production" rule now prevents.

## 11. Reuse for similar issues

**How to recognise one:** an integration "sometimes doesn't happen", with no error anywhere in Salesforce.

- **Look for `http.send` outside a try/catch** in Queueables, batches and future methods. A thrown callout exception kills the whole job and its logging.
- **Distinguish no response (code 0) from an error response.** They need different handling.
- **Log, then alert, and make the alert unable to break the job.**
- **Filter expected noise** (rate limits during a resync) out of the alert, but keep logging it.
- **Test with a mock that throws**, not only with mocks that return error codes.
