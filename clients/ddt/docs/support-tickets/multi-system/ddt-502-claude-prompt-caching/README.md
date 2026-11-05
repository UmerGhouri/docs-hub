# Prompt caching on Claude API calls

| | |
|---|---|
| **Ticket** | DDT-502 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-30 |
| **Priority** | Medium |
| **Reported by** | Firm operations, 2026-07-24 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex), Claude API |
| **Area** | AI features built in Salesforce: case briefs and mitigation letters |
| **Related** | None |
| **Code** | PR #556 in the `ddt-salesforce` repo |

---

## 1. Summary

Anthropic's console flagged a low prompt-cache hit rate on the firm's Claude API account and estimated that caching repeated content could cut up to 52% of direct API spend. The Salesforce features that call Claude resend a large, fixed system prompt on every request.

The shared client class already supported caching, but only one feature used it. The fix turned it on for brief generation and both mitigation-letter features, and corrected the token count stored on each record so it still includes the cached part. Measured at the firm's current volume, the expected saving is close to none; it pays off only when generations cluster together.

## 2. What was reported

- **2026-07-24:** Anthropic estimated up to 52% savings from caching. Claude Code was excluded from that estimate because it caches automatically, so the ticket targeted the Salesforce-built features that call the API directly. The ticket's first step was to confirm in the Claude Console that those features really are the bulk of the spend.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `BriefGenerationQueueable` | Apex class | Asks for the system prompt to be cached; stores all input tokens |
| `MitigationLetterQueueable` | Apex class | Same, on both the draft call and the editor call |
| `MitigationEditAnalysisQueueable` | Apex class | Asks for the system prompt to be cached |
| `ClaudeApiMock` | Apex test helper | Records the last request body; can return cache token counts |
| `BriefGeneratorControllerTest`, `MitigationLetterGeneratorTest` | Apex tests | New tests for caching and token counting |
| `docs/deploys/ddt-502-cache-system-prompts.md` | Review doc | Deploy notes and the savings estimate |

**Out of scope**

- **`ClaudeApiClient` itself.** It already had the caching overloads; it was not changed. (The ticket's proposed edit to `ClaudeApiClient.send` turned out to be unnecessary.)
- **Consult call scoring.** It already cached its system prompt before this ticket.
- **A second cache breakpoint on the user message.** The ticket suggested it as optional; it was not done.

## 4. How it works (background)

```
Brief / mitigation feature (queueable Apex)
  -> ClaudeApiClient.send(model, maxTokens, systemPrompt, userMessage [, cacheSystemPrompt])
       4-argument form:  "system" sent as a plain string        -> never cached
       5/6-argument form with true:
                         "system" sent as a content block with
                         cache_control: ephemeral               -> cached for 5 minutes
  -> Claude API
  <- usage: input_tokens, cache_creation_input_tokens, cache_read_input_tokens
  -> Input_Tokens__c on the brief or letter record
```

Prompt caching prices, as used in the review doc: a cache write costs 1.25 times the normal input price, and a cache read costs 0.1 times. The default cache lasts 5 minutes from the last hit.

## 5. Root cause

The ticket assumed caching needed to be built into `ClaudeApiClient`. Reading the code showed it already existed: the five- and six-argument `send` overloads wrap the system prompt as a cacheable block. Only `ConsultScoringService` called those overloads. The three features named in the ticket used the four-argument form, so their system prompts were always billed at full price.

A second issue only appears once caching is on: the API's `input_tokens` then counts only the **uncached** part of the input. The features stored `input_tokens` alone in `Input_Tokens__c`, so after the change that field would have under-reported every call by the size of the system prompt.

## 6. Solution

### 6.1 Ask for caching at each call site

```apex
ClaudeApiClient.Response r = ClaudeApiClient.send(
    /* model, max tokens, system prompt, */
    userMessage,
    true // DDT-502: cache the fixed system prompt
);
```

This was added to the brief call, both mitigation-letter calls (draft and editor) and the edit-analysis call.

Why at the call site and not in the client: the client already had the switch. Making caching the default for every caller would change the cost of features that call rarely, where a cache write is pure overhead.

### 6.2 Keep `Input_Tokens__c` meaning "everything the call sent"

```apex
@TestVisible
private static Integer allInputTokens(ClaudeApiClient.Response r) {
    return (r.inputTokens == null ? 0 : r.inputTokens)
        + (r.cacheCreationTokens == null ? 0 : r.cacheCreationTokens)
        + (r.cacheReadTokens == null ? 0 : r.cacheReadTokens);
}
```

`BriefGenerationQueueable` and `MitigationLetterQueueable` now store the sum of uncached, cache-write and cache-read tokens. Anything that reports on `Input_Tokens__c` keeps working without knowing about caching.

### 6.3 What it actually saves

From the review doc, measured in production on 2026-09-30:

| Measure | Value |
|---|---|
| Break-even share of calls within 5 minutes of the previous one on the same prompt | about 22% |
| Opportunity briefs in the last 90 days | 46, of which 17% within 5 minutes of the previous |
| Matter briefs | 0 |
| Mitigation letters ever | 19 |
| System prompt sizes | about 1.4K to 3.6K tokens, all above the 1,024-token caching minimum for the models used |

At 17%, caching costs about 5% more on the prompt tokens, which the doc puts at cents per quarter. It pays as soon as generation clusters. It was shipped on that basis.

## 7. Testing

| Test | What it proves |
|---|---|
| `BriefGeneratorControllerTest.generateSummary_cachesTheSystemPrompt` | Every Claude request from the brief path sends the system prompt as a block with `cache_control` |
| `MitigationLetterGeneratorTest.bothLetterCallsCacheTheirSystemPrompt` | Both letter calls do the same |
| `BriefGeneratorControllerTest.generateSummary_inputTokensIncludeCachedPrompt` | `Input_Tokens__c` includes cache tokens |
| `MitigationLetterGeneratorTest.inputTokensIncludeCachedPrompt` | Same, for letters |

The tests read the actual request body through `ClaudeApiMock.lastRequestBody`, so they prove what was sent, not just that a flag was passed. 27 of 27 passed in the partial-copy sandbox. That sandbox cannot compile the brief feature, so the brief path was proven by a check-only validation against production.

The three classes were confirmed in production with a last-modified date of 2026-09-30.

**Not covered:** the ticket's own steps 1 and 3 were left for after deploy and are not recorded as done: confirming in the Claude Console that the Salesforce key is the main direct-API spend, and reading a non-zero `cache_read_input_tokens` on a real repeat generation.

## 8. Step by step

1. **Check where the spend is.** In the Claude Console, break usage down by API key and confirm the Salesforce key is the main line.
2. **Find every Claude call site** and note which `ClaudeApiClient.send` overload it uses.
3. **Check the system prompt size** against the model's minimum cacheable size. A prompt below it is simply not cached; the request still succeeds.
4. **Pass `true` for caching** at the call sites that resend a large fixed prompt.
5. **Sum all three input-token counts** wherever usage is stored.
6. **Add a test that reads the request body** and asserts `cache_control` is present.
7. **Validate against production**, get approval, deploy.
8. **After a real repeat generation within 5 minutes**, check the response usage for non-zero cache-read tokens.

## 9. Rollback

Redeploy the three queueable classes from their previous version. Safe; it only stops caching and returns `Input_Tokens__c` to counting uncached tokens, which is correct again once caching is off.

## 10. Known limits

- **At current volume it barely pays.** The 52% estimate assumes repeated calls inside the cache window. Briefs and letters are generated a few at a time, often more than 5 minutes apart.
- **The draft and editor calls use different prompts**, so one does not warm the cache for the other.
- **A shorter future prompt may stop caching silently** if it drops below the model's minimum size.

## 11. Reuse for similar issues

**How to recognise one:** a feature resends the same long instructions to an LLM on every call.

- **Check whether the shared client already supports caching** before changing it. Here it did, and only the callers needed to opt in.
- **Work out the break-even before promising savings.** With a 1.25 times write and a 0.1 times read, roughly a fifth of calls must land inside the cache window.
- **Fix token accounting at the same time.** Once caching is on, `input_tokens` no longer means total input.
- **Test the request body,** not the flag. A mock that records the last body lets the test prove what actually went over the wire.
