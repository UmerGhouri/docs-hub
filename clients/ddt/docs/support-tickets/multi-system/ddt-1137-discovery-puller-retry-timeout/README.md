# Discovery Puller has no retry or timeout on network calls

| | |
|---|---|
| **Ticket** | DDT-1137 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-28 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-09-28 |
| **Resolved by** | Support team |
| **Systems** | Discovery Puller (TypeScript worker), TechShare and Axon evidence portals, Google Drive |
| **Area** | Discovery pulls: downloading evidence from a portal and uploading it to the client's Drive folder |
| **Related** | DDT-1138 and DDT-1140 (sandbox window), DDT-1139 (manifest truncation) |
| **Code** | PR #14 in the `ddt-discovery-puller` repo |

---

## 1. Summary

No network call in the Discovery Puller had a retry, a backoff or a timeout. One dropped connection on one chunk of one file failed the whole pull, and a portal that accepted a connection and then went quiet left the worker waiting until the watchdog killed it an hour later.

The fix adds a small shared helper with a hard timeout and a bounded, backed-off retry, and applies it at the three places that had actually failed: every TechShare call, the Google Drive resumable upload, and the Axon segment download. Auth refusals (401, 403, 404) are deliberately never retried.

## 2. What was reported

Three pulls had failed this way:

| When | What happened |
|---|---|
| 2026-08-27 | A single `fetch failed` on one Drive chunk, after 97 of 98 files (36.4 GB) had already uploaded. The whole pull was marked Failed. |
| 2026-09-15 | `fetch failed` on the TechShare login, 12 seconds in, no files. |
| 2026-09-15 | TechShare opened the connection and never answered. With no timeout, the worker sat for 61 minutes until the watchdog killed it. |

The exposure scales the wrong way: a 98-file pull has roughly ten times the chance of hitting a blip as a 10-file pull, so the largest packages, which cost the most to redo, were the likeliest to fail.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `lib/retry.ts` | New module | `withRetry`, `fetchRetry` (timeout plus retry), and the two rules that decide what is worth another attempt |
| `lib/adapters/techshare.ts` | Portal adapter | `jfetch`, which every TechShare call goes through, uses `fetchRetry` with a 60-second timeout |
| `lib/adapters/axon.ts` | Portal adapter | `streamFileToDisk` (the segment download) uses `fetchRetry` with a 15-minute timeout |
| `worker/discovery.ts` | Worker | `uploadToDrive`: the resumable-session start and each 32 MB chunk upload are retried with a timeout; the loop advances from what Drive says it holds |
| `scripts/validate-retry.ts` | New validation script | Proves the behaviour (the repo has no test framework) |

**Out of scope**

- **The wall-clock limit on large packages** (DDT-1138) and **the sandbox-stop timing bug** (DDT-1140). A retry helps a transient failure; it does nothing for a pull that simply runs out of time.
- **A body that dies mid-stream on the Axon download.** Only the request itself is retried; a broken stream is still caught downstream by the existing zip checks.

## 4. How it works (background)

```
Salesforce Discovery Pull record
  -> a worker starts in a cloud sandbox
  -> portal adapter (TechShare or Axon) logs in and downloads the package
  -> worker uploads each file to Google Drive in 32 MB chunks (resumable upload)
  -> worker reports progress and the final status back to Salesforce
```

Before the fix, every arrow that crossed the network was a plain `fetch` with no deadline and no second attempt.

## 5. Root cause

The adapters and the worker were written with bare `fetch` calls. Nothing set a timeout and nothing retried:

```
$ grep -rnE "AbortSignal|AbortController|signal:|timeout" lib/adapters/*.ts worker/discovery.ts
  (nothing)
```

Two consequences:

1. **Any transient network error was final.** The upload loop threw on the first failed chunk, the file was marked failed, and one failed file turned the whole pull red.
2. **A silent server hung forever.** Without a timeout, a connection that opened and never answered blocked the worker until the 60-minute watchdog ended it, holding a sandbox the whole time.

A third, smaller bug sat in the same upload loop: the result of the file read was never checked, so a short read would have uploaded an uninitialised buffer.

## 6. Solution

### 6.1 One helper, used at the three sites that failed

```ts
const RETRYABLE_STATUS = new Set([408, 425, 429, 500, 502, 503, 504]);

export async function fetchRetry(url, init, opts) {
  const timeoutMs = opts.timeoutMs ?? 60_000;
  return withRetry(
    async () => {
      const res = await fetch(url, { ...init, signal: AbortSignal.timeout(timeoutMs) });
      if (isTransientStatus(res.status)) {
        throw new RetryableHttpError(res.status, `${opts.what}: HTTP ${res.status}`);
      }
      return res;
    },
    (err) => err instanceof RetryableHttpError || isTransientError(err),
    opts,
  );
}
```

- **Four attempts by default, 500 ms backoff doubling each time.** Enough to absorb the failures above.
- **The timeout matters most.** It turns a 60-minute silent hang into an immediate, honest error.
- **Transient errors are read from `err.cause.code`.** Node reports a dropped socket as an opaque `fetch failed`; the real code (`ECONNRESET`, `ETIMEDOUT` and so on) hides underneath.
- **The last error is re-thrown unchanged**, so the log shows the real cause, not a wrapper.

### 6.2 What is deliberately not retried

**401, 403 and 404.** An auth refusal is not transient. Retrying it burns the sandbox's clock to reach the same answer. A non-retryable status comes back as an ordinary response for the caller to handle.

### 6.3 The Drive chunk loop

- **The retry resends the same chunk with the same byte range.** A resumable upload is idempotent at a given range: if Drive already has those bytes, it answers 308 with an advanced range instead of storing them twice.
- **The loop advances from what Drive says it holds, not from what was sent:**

```ts
const committed = /bytes=0-(\d+)/.exec(res.headers.get('range') ?? '');
offset = committed ? Number(committed[1]) + 1 : offset + len;
```

After a retry the two can differ. Trusting the local count would leave a gap that shows up later as a corrupt file in the client's folder.

- **The file read is checked.** A short read now throws instead of uploading garbage.

## 7. Testing

`npx tsx scripts/validate-retry.ts` checks:

- what counts as transient (network codes and 5xx/429 do; 401/403/404 and ordinary errors do not);
- recovery after a transient failure;
- a non-transient failure stops on attempt 1 with its original error;
- the attempt limit holds and the backoff actually waits;
- against a local test server: a 503 is retried to success, **a 401 is requested exactly once**, and a server that never answers times out.

**Mutation-checked.** Five deliberate breakages (401 made retryable, `err.cause.code` ignored, timeout removed, attempt limit ignored, backoff removed) each made the script fail. Two of them only failed after the script was strengthened: removing the backoff at first passed unnoticed because nothing measured the delay, and ignoring the attempt limit made the script hang instead of fail.

Type check clean and all six existing validation suites pass.

**Not covered:** a real transient failure in production recovering silently. The deploy doc leaves that as an open check to watch for.

## 8. Step by step

1. **Find every network call** in the adapters and the worker, and confirm none has a timeout (the `grep` in section 5).
2. **Add `lib/retry.ts`** with `withRetry`, `fetchRetry`, and the two predicates from 6.1.
3. **Route TechShare's single request helper** (`jfetch`) through `fetchRetry` with a 60-second timeout. This covers login, CSRF, proxy and download-prep in one place.
4. **Route the Axon segment download** through `fetchRetry` with a 15-minute timeout.
5. **Change `uploadToDrive`:** `fetchRetry` for the session start; `withRetry` plus `AbortSignal.timeout` (10 minutes) for each chunk; check the read length; advance from Drive's `Range` header.
6. **Write `scripts/validate-retry.ts`** and run it. Break each rule on purpose and confirm the script fails.
7. **Merge and deploy** the puller as usual.
8. **Watch production** for a `fetch failed` that is followed by a retry log line and a successful pull.

## 9. Rollback

Revert the PR. There is no schema change, no Salesforce metadata and no environment variable, so reverting is safe. It brings back the original failures.

## 10. Known limits

- **Four attempts is a ceiling.** A real outage still fails the pull, just faster and with an honest error.
- **Only three call sites are wrapped.** Any new network call added to an adapter needs the same treatment.
- **A stream that dies mid-body** on the Axon download is not retried at the request level.

## 11. Reuse for similar issues

**How to recognise one:** a long-running job fails on a single "fetch failed" or "socket hang up", or hangs until a watchdog kills it.

- **Put a timeout on every outbound request.** A hang is worse than an error: it hides the cause and holds resources.
- **Retry only what can succeed on a second try.** Network errors and 5xx/429 yes; auth refusals and 404 never.
- **Read the real error code** from the error's cause, not just the message.
- **For resumable uploads, trust the server's committed offset**, not your own counter.
- **Prove each guard can fail.** Break it on purpose and confirm the check notices; a guard that passes either way is not a guard.
