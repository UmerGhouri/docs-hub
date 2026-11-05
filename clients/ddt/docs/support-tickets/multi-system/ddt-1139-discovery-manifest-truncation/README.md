# Discovery Puller manifest truncates on large pulls

| | |
|---|---|
| **Ticket** | DDT-1139 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-28 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-09-28 |
| **Resolved by** | Support team |
| **Systems** | Discovery Puller (TypeScript worker), Salesforce (Apex trigger, Chatter) |
| **Area** | Discovery pulls: the file list the puller sends back to Salesforce for the Chatter audit post |
| **Related** | DDT-1137 (retry and timeouts), DDT-1138 and DDT-1140 (sandbox window) |
| **Code** | PR #16 in the `ddt-discovery-puller` repo |

---

## 1. Summary

When a pull finishes, the worker sends Salesforce a JSON list of every file it uploaded (the manifest). To fit a Long Text Area field it cut the JSON string at 130,000 characters, which chops the last entry in half and leaves invalid JSON. The Salesforce trigger that builds the Chatter audit post could not read it.

The fix drops **whole entries** until the list fits, so the JSON always parses, and logs how many were left out. A manifest that already fits is sent exactly as before.

## 2. What was reported

At about 245 characters per entry, any pull of more than about 530 files produced an unparseable manifest. A 1,400-file manifest is about 343 KB.

**Six production pulls** were found carrying manifests of exactly 130,000 characters that would not parse. Pulls of 1,057 and 1,126 files had already been delivered, well past the threshold.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `lib/manifest.ts` | New module | `serializeManifest()` keeps as many whole entries as fit and returns the omitted count |
| `worker/discovery.ts` | Worker | Both places that send the manifest (final status, and the partial-failure path) use it and log the omitted count |
| `scripts/validate-manifest.ts` | New validation script | Proves the behaviour |

**Out of scope**

- **A sentinel row** such as "… 867 more omitted". Not added on purpose: the Apex consumer reads `driveFileId` off every row, and a synthetic row with an empty id could break the very Chatter post this protects. Adding one would need `DiscoveryPullTriggerHandler` read and changed first.
- **The six manifests already stored.** They are saved values. Their posts need a re-pull or a hand edit of the field if they matter.
- **The files themselves.** Every file is still in Drive; this only affects what the audit post can list.

## 4. How it works (background)

```
Worker finishes uploading
  -> builds manifest: [{ filename, driveFileId, sizeBytes, webViewLink }, ...]
  -> sends it to Salesforce in a Long Text Area field (limit 131,072 characters)
Salesforce
  -> DiscoveryPullTriggerHandler parses the JSON
  -> posts the file list to Chatter as the pull's audit record
```

## 5. Root cause

```ts
const manifestJson = JSON.stringify(manifest).slice(0, 130000); // safe under 131072 LongTextArea limit
```

The comment is right about the field's size and wrong about the result. `slice` cuts at a character position, not at an entry boundary, so once the list passes 130,000 characters the output ends mid-object, with no closing brackets. `JSON.parse` in the trigger throws on it.

The same line appeared in the partial-failure path, so a pull that failed part-way through had the same problem.

## 6. Solution

```ts
export const MANIFEST_LIMIT = 130000;

export function serializeManifest(entries, limit = MANIFEST_LIMIT) {
  const full = JSON.stringify(entries);
  if (full.length <= limit) return { json: full, omitted: 0 };

  let used = 2; // the enclosing []
  let kept = 0;
  for (const entry of entries) {
    const piece = JSON.stringify(entry);
    const cost = piece.length + (kept > 0 ? 1 : 0); // comma
    if (used + cost > limit) break;
    used += cost;
    kept++;
  }
  return { json: JSON.stringify(entries.slice(0, kept)), omitted: entries.length - kept };
}
```

And at each call site:

```ts
const serialized = serializeManifest(manifest);
if (serialized.omitted > 0) {
  await log('warn', `Manifest too large for Salesforce — reporting ${manifest.length - serialized.omitted} of ${manifest.length} files; ${serialized.omitted} omitted from the audit post (all files are in Drive).`);
}
```

Why it's done this way:

- **Whole entries keep the JSON valid**, so the trigger can always parse it.
- **A list that fits is byte-identical to before.** Nothing changes for the pulls that were already fine.
- **Entries are kept in order**, so what survives is the start of the pull, which matches the order the Chatter post lists them.
- **Each entry is measured once** instead of re-serialising the whole list per candidate, so a 1,400-entry manifest stays cheap.
- **The omitted count goes to the log**, so a shortened manifest is never silent.

## 7. Testing

`npx tsx scripts/validate-manifest.ts` checks:

- a manifest that fits comes out byte-identical;
- an oversized one parses, and the kept entries are whole, in order, and none has an empty `driveFileId`;
- the boundary: a manifest of exactly n characters fits at limit n and n+1 and is cut at n−1;
- empty input and a single oversized entry still produce valid JSON;
- **the old byte slice produces invalid JSON**, so the script cannot pass against the behaviour it replaces.

**Mutation-checked:** four deliberate breakages (byte slice restored, comma ignored, which ran 431 characters over the limit, off-by-one keeping one entry too many, omitted count always 0) each made the script fail.

Type check clean, worker bundle builds, all six offline validation suites pass.

**Not covered:** a live pull of more than about 530 files producing a Chatter post that renders, with the log line naming the omitted count. The deploy doc leaves that as a check after deploy.

## 8. Step by step

1. **Confirm the symptom in Salesforce:** find pulls whose manifest field is exactly 130,000 characters, and check that it does not parse.
2. **Find every place** the worker truncates the manifest (there were two).
3. **Add `lib/manifest.ts`** with `serializeManifest()` as in section 6.
4. **Replace both `.slice(0, 130000)` calls** with it, and log the omitted count when it is above zero.
5. **Write `scripts/validate-manifest.ts`**, including a must-fail case for the old slice, and break the function on purpose to confirm the script notices.
6. **Merge and deploy.**
7. **On the next large pull**, check that the Chatter post renders and that the log names how many files were left off.

## 9. Rollback

Revert the PR. No schema change, no Salesforce metadata, no environment variable. Reverting brings back invalid JSON on large pulls.

## 10. Known limits

- **The audit post lists only the files that fit.** On a very large pull the rest are named only in the log, not on the record.
- **The six manifests stored before the fix are still unparseable.**
- **If the field limit or the entry shape changes,** the 130,000 margin should be re-checked.

## 11. Reuse for similar issues

**How to recognise one:** a parse error on a stored JSON field, and the stored value's length is exactly some round number.

- **Never truncate structured data with a string slice.** Drop whole items and re-serialise.
- **Report what was dropped** somewhere a person will see it.
- **Don't add placeholder rows** a downstream consumer was never written to expect; read the consumer first.
- **Keep output byte-identical** for inputs that were already fine, so the fix changes nothing it doesn't need to.
