# Discovery Puller long pulls exceed the sandbox window

| | |
|---|---|
| **Ticket** | DDT-1138, DDT-1140 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-28 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-09-28 |
| **Resolved by** | Support team |
| **Systems** | Discovery Puller (TypeScript worker), Vercel Sandbox, Upstash Redis |
| **Area** | Discovery pulls: how long a pull may run, and stopping its sandbox when it finishes |
| **Related** | DDT-1137 (retry and timeouts), DDT-1139 (manifest truncation) |
| **Code** | PR #15 in the `ddt-discovery-puller` repo |

---

## 1. Summary

Each discovery pull runs in one cloud sandbox with a hard time cap. Large packages were running out of time mid-upload (DDT-1138). Separately, the record that lets the puller stop a finished sandbox expired ten minutes **before** the sandbox's own cap, so every pull longer than 35 minutes kept billing until the cap (DDT-1140).

The two had to be fixed together. Both numbers now live in one file, the stop record's lifetime is **derived** from the sandbox cap, and the cap was raised from 45 to 90 minutes.

## 2. What was reported

- **DDT-1138, 2026-09-28.** Eight pulls since July had been killed mid-upload, every one a large package that had already reported real progress ("Uploading 955: …", "Downloading package 3 of 3"). One of them recovered on a retry. A measured end-to-end run of a 55.3 GB, 58-file package took 34.6 minutes: 13.4 minutes to download and extract at about 70.7 MB/s, then about 6.9 minutes per 10.7 GB segment. That put the practical ceiling of the 45-minute cap at about six segments, roughly 65 GB, and packages were already arriving at that size. File count alone was not the predictor: one 1,057-file pull finished in 28.7 minutes. Total bytes and upload speed were.
- **DDT-1140, 2026-09-28.** The sandbox-stop record was kept for 2,100 seconds (35 minutes) while the sandbox could live for 2,700 seconds (45 minutes). On any pull longer than 35 minutes the record had gone by the time the pull finished, so nothing stopped the sandbox and it billed to the full cap.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `lib/sandbox-window.ts` | New module | Holds `SANDBOX_TIMEOUT_MS` (90 minutes) and `SANDBOX_ID_TTL_SECONDS`, derived from it plus a 15-minute margin |
| `lib/sandbox.ts` | Module | `createSandbox` defaults to `SANDBOX_TIMEOUT_MS` instead of a literal |
| `lib/upstash.ts` | Module | `storeSandboxId` defaults to `SANDBOX_ID_TTL_SECONDS` instead of a literal `2100` |
| `scripts/validate-sandbox-window.ts` | New validation script | Asserts the relationship between the two values |

**Out of scope**

- **Per-segment jobs** (one sandbox per segment, each retryable on its own) and **parallel Drive uploads**. Both were offered as the structural fix. Parallel uploads in particular would break the rule that the worker holds about one file on disk at a time, so they need an explicit disk budget, not just a concurrency number. Not built here.
- **Transient network failures** are DDT-1137.

## 4. How it works (background)

```
Spawn a pull
  -> create a sandbox with a time cap
  -> store jobId -> sandboxId in Redis with an expiry (TTL)
Worker runs, then reports a terminal status (Done / Failed)
  -> the callback looks up sandboxId by jobId
  -> stops the sandbox, so the pull pays only for the time it used
If the lookup finds nothing
  -> the sandbox keeps running, and billing, until its cap
```

The design only works while **the stored record outlives the sandbox**.

## 5. Root cause

### 5.1 The stop record expired first (DDT-1140)

```
lib/upstash.ts   storeSandboxId(..., ttlSeconds = 2100)    // 35 min
lib/sandbox.ts   createSandbox(timeoutMs = 45 * 60 * 1000) // 45 min
```

History, from the repo:

```
2026-06-16  "Stop sandbox on terminal callback to cap Vercel cost"
            TTL 2100s written while the cap was 1800s (30 min). Correct.
2026-07-16  "sandbox: raise timeout 30min -> 45min for huge (30GB+) packages"
            Cap raised to 2700s. TTL left at 2100s. Relationship inverted.
```

The comment above `storeSandboxId` still said the TTL "is set above the sandbox timeout". Nothing tied the two numbers together and they lived in different files, so the inversion went unnoticed for about two months. The pulls that run past 35 minutes are the 30 GB+ packages, exactly the ones the cap was raised for and the most expensive to leave running.

### 5.2 Big packages outgrew the cap (DDT-1138)

At roughly 6.9 minutes per 10.7 GB segment, a 45-minute cap allowed about six segments. Packages larger than that were killed mid-upload.

### 5.3 Why they had to ship together

Raising the cap without fixing the stop record would have made things worse: a 90-minute sandbox that never gets stopped doubles the bill on exactly the pulls the raise is meant to help.

## 6. Solution

```ts
// lib/sandbox-window.ts
export const SANDBOX_TIMEOUT_MS = 90 * 60 * 1000;

/** Slack beyond the sandbox's own lifetime, so a missed stop still self-cleans. */
const TTL_MARGIN_SECONDS = 15 * 60;

/** Derived, never hardcoded: it must outlive the sandbox or the callback cannot stop it. */
export const SANDBOX_ID_TTL_SECONDS = Math.ceil(SANDBOX_TIMEOUT_MS / 1000) + TTL_MARGIN_SECONDS;
```

`lib/sandbox.ts` and `lib/upstash.ts` both import from this file.

Why this way:

- **The coupling is the fix.** The drift only happened because two related numbers sat in different files with nothing connecting them. Now changing the cap moves the TTL with it.
- **Raising the cap is only safe because the sandbox is stopped on the terminal callback.** An ordinary pull pays for the minutes it uses, not for the window.
- **90 minutes, not "unlimited".** Raising the cap is the simplest option but has unbounded cost, and it only moves the ceiling.

## 7. Testing

`npx tsx scripts/validate-sandbox-window.ts` checks:

- the TTL outlives the sandbox cap by at least 5 minutes;
- **the old 2,100 s / 2,700 s pair fails that rule**, so the check cannot pass against the bug it exists for;
- neither default is hardcoded in `storeSandboxId` or `createSandbox`.

**Mutation-checked:** three deliberate breakages (TTL back to a literal 2100, margin set to zero, TTL re-hardcoded in `upstash.ts`) each made the script fail. An earlier version of the "not hardcoded" check also matched an unrelated `ttlSeconds = 600` elsewhere in the file; it was narrowed to the two specific signatures so it does not raise false alarms.

Type check clean and all six offline validation suites pass.

**Not covered by the script:** a real pull over 35 minutes stopping on its terminal callback. The deploy doc leaves that as a check to make after deploy, since only a long pull exercises it.

## 8. Step by step

1. **Find every time limit** that applies to one pull: the sandbox cap, the stop-record TTL, and anything else that must outlive the worker.
2. **Confirm the relationship is broken**: the TTL (2,100 s) is below the cap (2,700 s).
3. **Create `lib/sandbox-window.ts`** with the cap and a derived TTL, as in section 6.
4. **Import both values** in `lib/sandbox.ts` and `lib/upstash.ts`; remove the literals.
5. **Raise the cap** to 90 minutes in that one file.
6. **Write `scripts/validate-sandbox-window.ts`**, include the old pair as a must-fail case, and break each rule on purpose to confirm the script notices.
7. **Merge and deploy.**
8. **After deploy, watch a pull longer than 35 minutes** and confirm its sandbox stops as soon as the pull reports its final status.

## 9. Rollback

Revert the PR. No schema change, no Salesforce metadata, no environment variable. Reverting brings back both the 45-minute ceiling and the stop record that expires too early.

## 10. Known limits

- **90 minutes raises the ceiling; it does not remove it.** At the measured speed that is roughly twelve segments. Per-segment jobs or parallel uploads with a disk budget remain the structural answer.
- **Any other time limit tied to the pull must follow the cap too.** After this change, a third limit elsewhere in the puller was found still sized for the old 45-minute cap; pulls running past 60 minutes stopped reporting progress. A follow-up change dated 2026-10-15 derives that one from the same file. Whether it has reached production is not recorded in the sources.
- **The puller still uploads one file at a time.** That keeps disk use bounded but caps throughput.

## 11. Reuse for similar issues

**How to recognise one:** a cost control or cleanup step "works" on short runs and silently stops working on long ones.

- **When two numbers must keep a relationship, put them in one file and derive one from the other.** A literal in one file next to a literal in another is how this drifted.
- **Write a check that the old, broken pair fails.** Otherwise the check may pass for the wrong reason.
- **When you raise one limit, list every other limit that must stay above it**: record TTLs, token lifetimes, watchdogs. This change itself missed one, which is why the follow-up exists.
- **Fix the cleanup before raising the cap.** A longer window with a broken stop multiplies the cost of the bug.
