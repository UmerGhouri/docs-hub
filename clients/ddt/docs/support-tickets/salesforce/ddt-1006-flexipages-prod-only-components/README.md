# Flexipages carrying production-only components

| | |
|---|---|
| **Ticket** | DDT-1006 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-14 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-09-14 (found during review of DDT-1004) |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Lightning record pages / FlexiPage metadata), git |
| **Area** | Release process: keeping git's copy of Lightning pages in step with production |
| **Related** | DDT-1004 (where the near-miss happened) |
| **Code** | PR #376 in the `ddt-salesforce` repo |

---

## 1. Summary

Deploying a Lightning record page (FlexiPage) **replaces the whole page**. Six pages had components in production that were missing from git's copy, so the next person to deploy any of them for an unrelated reason would have silently deleted those components. We added only the missing components to git's six files. Nothing was deployed: production was already correct, and the change stops git from deleting the components later.

## 2. What was reported

During the consult-scoring work (DDT-1004), a new component was added to the repo copy of a Court Case page that was four months behind production. Deploying it would have **removed a live quick action** (`Build_DUI_Gameplan`) from that page. The check-only validation passed **while carrying the deletion**, because removing an item from a page is valid metadata. Only a diff against a fresh production retrieve caught it.

An audit of all **131** pages present in both git and production then found no other production-only quick actions, but six pages still carried production-only **components**:

| Page | Component only in production |
|---|---|
| `Account_Record_Page` | `linkRcCallsAuto`, `inboundContactReferrerPicker` |
| `Account_Record_Page2` | `linkRcCallsAuto` |
| `Client_Record_Page7` | `linkRcCallsAuto` |
| `Insurance_Carrier_Record_Page` | `linkRcCallsAuto` |
| `Opportunity_Hired_BB_Mobile_Only` | `dochly:dhSignatureRelatedList` |
| `Opportunity_Record_Page` | `dochly:dhSignatureRelatedList` |

The two LWC bundles were already in git; only their **placements** were missing. `dochly:dhSignatureRelatedList` comes from the Dochly managed package.

## 3. Scope

**In scope:** the six FlexiPage files above, each gaining only its missing component block (42 lines added, none removed).

**Out of scope**

- **Deploying.** Production already had all six components.
- **Drift in the other direction.** The same pages also hold things in git that production lacks (for example an extra related-list container on `Opportunity_Record_Page`, and differing component properties and visibility rules). Deploying git would *add* those to production. That needs a per-item decision about which side is right, and was left for a separate ticket.

## 4. How it works (background)

```
Someone edits a page in App Builder in production       -> production changes, git doesn't
Later, someone edits git's copy of the page for a new feature
  -> deploys the whole page
  -> production gets git's version: anything only production had is gone
  -> the deploy reports success; no error, no warning
```

Check-only validation can't catch this, because a page with fewer items is still valid metadata.

## 5. Root cause

Components had been placed on these pages directly in production (in App Builder) without the page being recorded in git. Git's copies were therefore missing them, and any deploy of those files would have removed them.

**The ticket's first proposed fix was wrong.** It proposed rebuilding the six files from a fresh production retrieve. That was tried first, and the diff showed it **removed 58 lines of real git-side content**, including a merge-candidates preview card, a Drive panel visibility rule, and live user-facing text on the Court Case page warning staff that a client is past due on payments. These pages drift **in both directions**, so copying production over git would have erased merged work. That is the shape of an earlier incident in which a bulk production-to-git copy erased ten weeks of merged work.

## 6. Solution

Add only the missing component blocks to each file. Six blocks of six lines; everything else byte-identical:

```xml
<itemInstances>
    <componentInstance>
        <componentName>linkRcCallsAuto</componentName>
        <identifier>c_linkRcCallsAuto</identifier>
    </componentInstance>
</itemInstances>
```

**Where it goes matters.** Each block is inserted **after the region's last `</itemInstances>`**, before `<mode>`, `<name>` and `<type>`. The FlexiPage schema orders `itemInstances` ahead of those elements. The first attempt put the block after `<type>Region</type>`, which is invalid.

## 7. Testing

- **Validated check-only against production**: 6 of 6 components, so the placements are known to resolve.
- **After merge**, all six pages were checked against `origin/main`: none has a component that exists in production but not in git.

**Not covered:** the git-ahead differences in section 3.

## 8. Step by step

1. **Retrieve the page from production** into a scratch location (don't overwrite your working copy).
2. **Diff it against git's copy, both ways.** List what production has that git lacks, and what git has that production lacks.
3. **If git has anything production lacks, don't copy production over git.** That would erase merged work.
4. **Add only the missing items** to git's file, each `itemInstances` block after the region's last existing `itemInstances`.
5. **Confirm the diff is additions only** (here, 42 insertions, zero deletions).
6. **Validate check-only** against production to confirm the placements resolve.
7. **Merge.** Don't deploy: production is already in the desired state.
8. **Re-diff after the merge** to confirm nothing is production-only any more.

## 9. Rollback

Revert the commit. That only changes git, and it puts back the risk of a later deploy deleting the six components. Production is unaffected either way.

## 10. Known limits

- **Git-ahead drift remains** on the same pages. Deploying one of them as-is would add items to production that production doesn't have today.
- **It will happen again** whenever someone changes a page in App Builder in production without recording it. No automated gate catches a page deploy that removes items.

## 11. Reuse for similar issues

**How to recognise one:** any metadata type that deploys by **replacing the whole thing**: Lightning pages, page layouts, permission sets. A deploy of a stale copy silently removes whatever production has that git doesn't.

- **Retrieve from production and diff before you edit** any replace-on-deploy file. That is the only thing that catches it; check-only validation passes.
- **Diff both ways.** Drift can run in both directions on the same file.
- **Never bulk-copy production over git.** Add the missing items one at a time and read every diff.
- **Mind the element order** when hand-editing metadata XML. Salesforce rejects elements in the wrong position.
- **A PR that deploys nothing can still be the fix**: its job is to stop git from being the thing that deletes production's work.
