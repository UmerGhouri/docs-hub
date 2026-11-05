# Discovery Puller handoff: merges, TechShare and sandbox cost fixes

| | |
|---|---|
| **Ticket** | DDT-180, DDT-181, DDT-182, DDT-183, DDT-184 |
| **Project** | Support tickets |
| **Status** | Done 2026-06-26 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-06-17 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC, custom metadata), Discovery Puller cloud service (Vercel app and sandbox worker), Google Drive |
| **Area** | Pulling law-enforcement discovery from Axon, NICE and TechShare into the matter's Drive folder |
| **Related** | DDT-596 (cleared the org's failing Apex tests, which changed the deploy rule below), DDT-701, DDT-1144 (later Discovery Puller changes) |
| **Code** | PR #32 in the ddt-salesforce repo; PR #2 and PR #4 in the ddt-discovery-puller repo |

---

## 1. Summary

The Discovery Puller was handed over from its original developer to the support team while it was already live in production. Because it had been deployed straight to production, the `main` branch of both repos was behind what production was running.

DDT-180 was the umbrella ticket. Its four sub-tickets merged the three pull requests that brought `main` back in line with production (DDT-181, DDT-182, DDT-183) and asked for a live check of a heavy TechShare case (DDT-184). In Salesforce the only change was one custom metadata record, `Discovery_Portal.TechShare`, plus a copy of the handoff document.

## 2. What was reported

All five tickets were opened on 2026-06-17 as a handoff checklist, not as a user-reported fault.

- **DDT-180 (2026-06-17, closed 2026-06-26).** Umbrella ticket: take over the Discovery Puller, set up the puller repo locally, get access to the services it uses, merge three sync PRs, and verify one heavy TechShare pull.
- **DDT-181 (2026-06-17, closed the same day).** Review and squash-merge PR #4 in the puller repo: the NICE and TechShare adapters, streaming downloads, the credential manager, Axon Disclosure support, and the sandbox spawn fix. It was already running in production; the merge only synced `main`.
- **DDT-182 (2026-06-17, closed the same day).** Review and squash-merge PR #2 in the puller repo: a fix to cut Vercel Sandbox cost. Also already in production.
- **DDT-183 (2026-06-17, closed the same day).** Review and merge PR #32 in the ddt-salesforce repo: sync the TechShare portal configuration to its live production values and add the handoff document to the Salesforce repo.
- **DDT-184 (2026-06-17, closed the same day).** Confirm that a TechShare case with 228 pieces of digital media evidence (DME) pulls all 228 files into the matter's Drive folder. The fix for heavy cases was live but had never been confirmed on a real case.

## 3. Scope

**In scope**

| Component | Repo | Type | Change |
|---|---|---|---|
| `Discovery_Portal.TechShare` | ddt-salesforce | Custom metadata record | `Adapter_Available__c` set to `true`, `URL_Pattern__c` cleared |
| `DISCOVERY-PULLER-HANDOFF.md` | ddt-salesforce | Document | New: a mirror of the puller repo's handoff document |
| PR #4 | ddt-discovery-puller | Vercel app and worker | NICE and TechShare adapters, streaming, credential manager, Axon Disclosure, spawn fix |
| PR #2 | ddt-discovery-puller | Vercel app | Sandbox stops when the job finishes (cost fix) |

**Out of scope**

- **Redeploying anything.** Every ticket says the code was already live and the merges were only to bring `main` in line with production. No redeploy was asked for.
- **New portals.** Axon standard share, Watchguard, Lake County and Sharepoint stay as stubs with `Adapter_Available__c = false`.
- **The Apex and LWC side of the puller.** The handoff document records that the Salesforce Apex and LWC changes had already been merged before the handoff; only the TechShare metadata record was out of sync.

## 4. How it works (background)

```
Staff member opens a discovery email in the Salesforce shared inbox
  -> clicks "Pull Discovery" (pullDiscovery LWC)
  -> DiscoveryPullController works out the portal and the target
     (a link in the email, or for TechShare a case number found by a subject/body marker)
  -> a Discovery_Pull__c record is created and the cloud service is called
  -> the cloud service starts a sandbox worker
  -> the worker downloads each file from the portal and uploads it to the matter's Drive folder
  -> the worker reports progress back to the Discovery_Pull__c record
  -> on Done: Chatter post, Slack message and email
```

`Discovery_Portal__mdt` holds one record per portal. Two fields matter here:

- `Adapter_Available__c` turns the portal on or off for the "Pull Discovery" button.
- `URL_Pattern__c` is the pattern the controller uses to spot a portal link in an email. TechShare emails carry no portal link, so TechShare is detected by a marker in the subject or body instead.

## 5. Root cause

Not a defect. The puller had been deployed directly to production while it was being built, so both repos fell behind production:

1. **ddt-salesforce.** `main` still had TechShare switched off (`Adapter_Available__c = false`) and an old TechShare URL pattern. Production had TechShare switched on since go-live, with no URL pattern.
2. **ddt-discovery-puller.** `main` was missing the work in PR #2 and PR #4, which production was already running.

**Evidence.** Production today matches what PR #32 put on `main` (queried 2026-11-05):

```sql
SELECT DeveloperName, Adapter_Available__c, URL_Pattern__c
FROM Discovery_Portal__mdt
WHERE DeveloperName = 'TechShare'
```

Result: `Adapter_Available__c = true`, `URL_Pattern__c = null`.

## 6. Solution

### 6.1 Sync the TechShare record (DDT-183, PR #32)

Two commits dated 2026-06-15 are on `main` of the ddt-salesforce repo: one changes `Discovery_Portal.TechShare.md-meta.xml`, the other adds `DISCOVERY-PULLER-HANDOFF.md`.

```xml
<values>
    <field>URL_Pattern__c</field>
    <value xsi:nil="true"/>
</values>
...
<values>
    <field>Adapter_Available__c</field>
    <value xsi:type="xsd:boolean">true</value>
</values>
```

Why it's done this way:

- **`main` was changed to match production, not the other way round.** Production was the working version and TechShare had been approved to go live.
- **The URL pattern was cleared, not fixed.** TechShare is found by a marker in the email, so the pattern was never used. Leaving it in place meant a stray TechShare link in an email could be matched by the link check by mistake.
- **To switch TechShare off**, set `Adapter_Available__c` back to `false`. No code change is needed.

### 6.2 Merge the puller PRs (DDT-181, DDT-182)

PR #4 and PR #2 were reviewed against what production runs and squash-merged. One part of PR #4 must not be undone: the worker bundle (about 480 KB) is passed into the sandbox as a file, not as a base64 shell argument. As a shell argument it exceeds the command-size limit, the sandbox fails to start, and every portal stops working. That happened once.

### 6.3 Heavy TechShare cases (DDT-184)

For a heavy case, TechShare's default DME list comes back empty and the portal shows a "View DME" button instead. The adapter falls back to the portal's fuller DME listing when the default list is empty. The check asked for was to pull the 228-piece case and count 228 files in Drive.

## 7. Testing

**Salesforce.** The only Salesforce change was a custom metadata record. No Apex test is tied to it.

**Puller repo.** The handoff asked for `pnpm typecheck` to pass locally. The result is not recorded in the ticket.

**The 228-piece check (DDT-184).** The ticket was closed 29 seconds after it was opened, with no comment, so the result of the live check is not recorded in the ticket. Production data shows (queried 2026-11-05):

| Query | Result |
|---|---|
| `SELECT Status__c, COUNT(Id) FROM Discovery_Pull__c WHERE Portal__c = 'TechShare' GROUP BY Status__c` | Done 58, Failed 11, TimedOut 5 |
| `SELECT MAX(Files_Added__c) FROM Discovery_Pull__c WHERE Portal__c = 'TechShare' AND Status__c = 'Done'` | 100 |

No completed TechShare pull has recorded more than 100 files added. That does not prove the 228-piece pull failed (it may have been done in parts, or the field may count differently), but it does mean the production data doesn't show a single 228-file pull.

**Not covered:** a recorded end-to-end check of a heavy TechShare case.

## 8. Step by step

For a future handoff or sync of the same kind:

1. **Set up the puller repo locally:** clone it, `pnpm install`, link it to the Vercel project, pull the environment variables, then run `pnpm typecheck`.
2. **Get access** to the Vercel project, the Upstash Redis store, the Google service account that uploads to Drive, the shared portal logins (through the password manager) and the Salesforce org. Never copy portal passwords into a repo or a plain-text file.
3. **Compare each PR with production before merging.** For Salesforce, retrieve the production version of every component it touches. For the puller, compare with the deployed version. A sync PR should show no unexpected differences.
4. **Squash-merge** each PR into `main`, then confirm the merged commit is on `main`.
5. **Do not redeploy** unless something is actually out of sync. If you must, the puller deploys with `pnpm vercel deploy --prod`.
6. **For a Salesforce redeploy of `DiscoveryPullController`**, `RunLocalTests` is now the default (see 10). If you use `RunSpecifiedTests` instead, name both `DiscoveryPullControllerTest` and `PortalLoginManagerTest`, or the class falls under 75% coverage and the deploy rolls back.
7. **Check the TechShare record in production** with the query in section 5.
8. **Verify a heavy TechShare case:** run "Pull Discovery" on it, wait for `Done`, count the files in the matter's Drive folder, and record the count on the ticket. If it's short, keep the worker logs and the Chatter post and compare the portal's DME listing with what was downloaded.

## 9. Rollback

- **TechShare record:** set `Discovery_Portal.TechShare.Adapter_Available__c` to `false`. That hides TechShare from the "Pull Discovery" button without touching code. Putting the old URL pattern back is not needed and brings back the false-match risk.
- **Handoff document:** it's documentation only; nothing depends on it.
- **Puller PRs:** they were already live before the merge, so undoing the merge would only put `main` behind production again. Do not revert the spawn fix in PR #4 (see 6.2).

## 10. Known limits

- **The deploy rule in the tickets is out of date.** The tickets say Salesforce production deploys must use `RunSpecifiedTests`. Since DDT-596 cleared the org's failing tests (2026-08-12), `RunLocalTests` is the default. The two-test-class rule still applies if you choose `RunSpecifiedTests`.
- **The heavy-case fix has no recorded confirmation** (see 7).
- **Large single files.** TechShare videos can be 1 to 2 GB. The worker holds one file on disk at a time, with a 5 GB per-file limit; anything larger is flagged for a manual pull.
- **NICE needs a person to sign in.** The portal blocks automated sign-in, so a staff member signs in and the session is reused by the worker.
- **TechShare logins are per attorney.** A pull for an attorney with no stored login fails with a message asking for one to be added.

## 11. Reuse for similar issues

**How to recognise one:** a system was deployed straight to production while it was built, and `main` no longer describes what's running.

- **Sync `main` to production, not production to `main`,** and do it one component at a time, reading each difference.
- **Retrieve or query the production version first.** A query on the custom metadata record settles which side is right faster than reading the history.
- **Keep on/off switches in custom metadata** (like `Adapter_Available__c`), so a feature can be turned off without a deploy.
- **Clear settings that are no longer used** rather than leave stale values that something could still match.
- **Record the result of a verification on the ticket.** Closing a "verify" ticket without the count leaves the next person unable to tell whether it was checked.
- **Write down the gotchas that broke production once**, next to the code, so the next owner doesn't undo them.
