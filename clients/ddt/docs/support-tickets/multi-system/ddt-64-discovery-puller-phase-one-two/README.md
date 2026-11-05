# Discovery Puller phase 1 and 2 build and UX improvements

| | |
|---|---|
| **Ticket** | DDT-64, DDT-65 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-02 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-05-26 (both tickets) |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, trigger, LWC, custom metadata, permission sets), external download worker hosted on Vercel, Google Drive, Slack |
| **Area** | Court Cases: pulling discovery files from a prosecutor's evidence portal into the case's Drive folder |
| **Related** | DDT-1144 (later change to the puller's Slack post) |
| **Code** | PR #18 and PR #19 in the `ddt-salesforce` repo; deploy doc `docs/deploys/discovery-puller-phase-1-2.md` |

---

## 1. Summary

The **Discovery Puller** is a button on a Court Case that takes an evidence-portal link from a prosecutor's email, has an external worker download the package, and files the contents in the case's Google Drive folder. These two tickets tracked getting its next two pull requests reviewed, merged and shipped:

- **PR #18 (DDT-64)** finished phases 1 and 2: the right portal login is chosen automatically, a Chatter post records every pull, failed pulls can be retried, and new portals can be added through configuration.
- **PR #19 (DDT-65)** made it usable without watching it: a live progress panel, Slack and email notices when a pull finishes, a check for missing portal logins before starting, and a warning before pulling the same package twice.

Both were shipped to production by 2026-06-02.

## 2. What was reported

- **DDT-64 (2026-05-26).** PR #18 was open with merge conflicts in three files (`Portal__c` and the two Discovery permission sets) and needed resolving, review, merge, the production post-deploy steps, and a smoke test by the two pilot users.
- **DDT-65 (2026-05-26).** PR #19 was **stacked on PR #18's branch**, not on `main`, so it could not land on `main` until PR #18 did. It needed review, merge, one post-deploy setting, and its test plan run in production.

The user-facing problem PR #19 solved, from its commit message: users stared at "Discovery: Pending (0)" for the whole download and extract phase, often 5 to 10 minutes or more on body-cam packages, with no progress, no estimate and no way to leave.

## 3. Scope

**In scope: PR #18 (phases 1 and 2)**

| Component | Type | Change |
|---|---|---|
| `DiscoveryPullController`, `DiscoveryPullService` | Apex | Portal login chosen from the email's original recipient; retry; portal detection from metadata |
| `DiscoveryPullTrigger`, `DiscoveryPullTriggerHandler` | Trigger and handler | Chatter post to the Court Case when a pull finishes with files |
| `pullDiscovery` | LWC | Login chooser, default "Discovery Received" folder, Retry button |
| `Discovery_Portal__mdt` | New custom metadata type | Six records: Axon active, five placeholders (TechShare, NICE, Watchguard, Lake County Portal, SharePoint) |
| `Discovery_Pull__c` | Fields | New: `Started_By__c`, `Credential_Identity__c`, `Credential_Source__c`, `Retry_Of__c`, `Retried_By__c`, `Files_Manifest_JSON__c`; `Portal__c` picklist expanded to six values |
| `Discovery_Puller`, `Discovery_Integration` | Permission sets | Field access for every field the LWC reads |
| `assign-discovery-puller-permset.apex` | Script | Assigns the permission set to the pilot users |
| Matching test classes | Apex tests | Extended |

**In scope: PR #19 (UX improvements)**

| Component | Type | Change |
|---|---|---|
| `pullDiscovery` | LWC | Status panel with current step, elapsed time and inline error; missing-login and duplicate-pull dialogs |
| `DiscoveryPullTriggerHandler` | Apex | Slack and email notice on Done, Failed or TimedOut |
| `DiscoveryPullController` | Apex | Duplicate-pull check before starting |
| `Discovery_Pull__c.Progress__c`, `Discovery_Pull__c.Slack_Posted__c` | Fields | Latest progress line; flag that stops duplicate notices |
| `Discovery_Puller_Config__mdt.Slack_Webhook_URL__c` | Field | Where the Slack notice is sent; blank disables it |
| `Slack_Webhooks` | Remote site setting | Allows the Slack callout |

**Out of scope**

- **The five other portals' download adapters.** PR #18 only lays the foundation; each needs its own worker adapter.
- **The worker's own code.** It lives in a separate repository and was deployed separately.

## 4. How it works (background)

```
Court Case with a prosecutor's evidence-portal email
  -> user clicks Pull Discovery (pullDiscovery LWC)
  -> Discovery_Portal__mdt decides whether the link is a supported portal
  -> controller picks whose portal login to use and creates a Discovery_Pull__c
  -> the external worker downloads the package, unpacks nested zips,
     and uploads the files to the case's Drive folder
  -> the worker writes progress and the final status back to Discovery_Pull__c
  -> DiscoveryPullTrigger on the final status:
       Done with files -> Chatter post listing the files    (PR #18)
       Done / Failed / TimedOut -> Slack and email notice   (PR #19)
```

**Adding a portal is configuration plus one adapter**: one `Portal__c` picklist value, one `Discovery_Portal__mdt` record, and one adapter file in the worker. No change to the LWC, controller, service or trigger.

## 5. Root cause

Not applicable: these tickets tracked feature work and its release, not a fault.

What is worth knowing is **why DDT-65 was blocked**: PR #19 was based on PR #18's branch. A pull request stacked on another branch can't reach `main` until the one beneath it does, and it gets none of `main`'s checks or review rules in the meantime.

## 6. Solution

### 6.1 PR #18: phases 1 and 2

From its deploy doc:

1. **The portal login is chosen from who the email was originally sent to**, so case managers can use the button without holding their own portal logins. When it can't be decided, the user picks.
2. **The folder picker defaults to "Discovery Received"**, creating it if needed.
3. **Nested zips are unpacked up to five levels**, so Drive only receives the actual files.
4. **A Chatter post on the Court Case** lists every file uploaded (the first 20, then "and N more"), the folder, and who pulled it.
5. **Failed and timed-out pulls show a Retry button**, which clones the original pull and links the two through `Retry_Of__c` and `Retried_By__c`.
6. **Portals are defined in `Discovery_Portal__mdt`**, which drives link detection and which adapter the worker uses.

### 6.2 PR #19: UX improvements

From its commit message:

1. **A status panel** shows the current step ("Downloading package 1 of 2", "Uploading 14 of 47"), elapsed time, and a line telling the user they can close the tab.
2. **Slack and email notices** go out on Done, Failed or TimedOut. `Slack_Posted__c` stops a second notice on later updates. The Slack call runs asynchronously because a trigger can't make a callout directly.
3. **Missing logins are caught before the pull starts.** If no login is stored for the chosen identity and portal, the user can switch to another before the pull fails silently.
4. **Duplicate pulls are caught.** If this Court Case already has a finished pull of the same link from the same portal, the user is asked "Pull anyway?".
5. **The worker reports progress** at each checkpoint, no more than once every 1.5 seconds, and deletes each file after uploading it, which fixed a disk-full failure on multi-gigabyte packages.

### 6.3 How the code reached `main`

Neither PR's own merge put its commits on `main`. PR #18 has no merge commit in the repo. PR #19 shows as merged on 2026-06-02, but **into PR #18's branch**, and that merge commit is not on `main`. The commits of both arrived on `main` on 2026-06-02 because PR #20, merged the same day, was built on top of them.

## 7. Testing

**Apex tests** (from the PR descriptions), run with `RunSpecifiedTests` against the sandbox and as a production validation:

| PR | Result |
|---|---|
| PR #18 | 45 of 45 passed |
| PR #19 | 56 of 56 passed |

What they cover, per the deploy doc: login resolution, retry chain, in-flight guard, signing and callout mocks (success and failure), the Chatter post on Done (and not on other statuses or zero files), and a malformed file list degrading gracefully.

**Live checks:** both tickets were closed with a note that the change had shipped to production. The manual smoke tests listed (a real pull as each pilot user, a failed pull and Retry, the duplicate guard, the missing-login dialog) are **not recorded as run.**

**Not covered:** the LWC has no automated test in these PRs, so the progress panel and dialogs were only checked by hand, if at all.

## 8. Step by step

The release, in order:

1. **Resolve PR #18's conflicts** in `Portal__c` and the two permission sets against current `main`.
2. **Validate against production** with the three Discovery test classes, get approval, merge, deploy.
3. **Set the shared secrets** on the `Discovery_Puller_Config.Default` record in Setup and in the worker's environment. Each pair must hold the same value on both sides.
4. **Store the portal logins** for the office mailbox and each opted-in user, with the worker's credential tool.
5. **Assign the permission set**: `sf apex run --file scripts/apex/assign-discovery-puller-permset.apex --target-org <prod-alias>`.
6. **Deploy the worker** to production.
7. **Retarget PR #19 to `main`** once PR #18 has landed, validate (56 tests), get approval, merge, deploy.
8. **Set `Slack_Webhook_URL__c`** on the config record in Setup. Leave it blank to switch Slack notices off.
9. **Run the smoke tests**: a real pull, a failed pull plus Retry, the same pull twice, and a pull from an email whose recipient has no stored login.

## 9. Rollback

From the deploy doc:

- **Turn the button off without a deploy:** with no enabled records in `Discovery_Portal__mdt`, link detection returns nothing and the button hides for everyone.
- **Full revert:** redeploy the previous versions. New fields stay in the schema unused; existing pull records keep their state and do not re-trigger.
- **Worker:** roll back to its previous deployment.
- **Slack notices only:** blank `Slack_Webhook_URL__c`.

## 10. Known limits

- **Only Axon has an adapter.** The other five portals are placeholders until their adapters are built.
- **Never put the secrets in the config record's source file.** On 2026-05-26 the `Discovery_Puller_Config.Default` file in git still held placeholder values. Deploying it with the UX change overwrote the real values in production, and the next pull failed with a "signing key not configured" error. The secrets were then removed from the file so they are managed only in Setup. Any deploy that includes that record must be checked for this.
- **The puller originally ran as one named person's user.** PR #360 later moved it to an integration user.

## 11. Reuse for similar issues

**How to recognise the git trap:** a pull request shows "Merged" on GitHub but its changes aren't on `main`, or its check list is empty.

- **Base every pull request on `main`.** A stacked PR gets no CI, no required review and no AI review, and its merge goes into the branch below it.
- **When a PR depends on unmerged work, merge that first** or combine them. Here the code only reached `main` because an unrelated PR happened to be built on top of it.
- **Keep secrets out of deployable metadata.** If a custom metadata record holds both stable settings and secrets, every deploy of the file resets the secrets to whatever is in git.
- **Long-running work needs visible progress and a notice at the end.** Users who can't see progress assume it's broken and click again, which is why the duplicate-pull guard exists.
- **Make a feature switchable by configuration** (here, the portal metadata records) so it can be turned off without a deploy.
