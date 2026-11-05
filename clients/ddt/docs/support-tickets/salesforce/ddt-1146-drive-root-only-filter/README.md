# Root-only filter in the Drive files panel

| | |
|---|---|
| **Ticket** | DDT-1146 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-01 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-09-28 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC), Google Drive |
| **Area** | The Google Drive files panel on Court Case and Matter pages |
| **Related** | DDT-1145 (SOS cases get no Drive subfolders) |
| **Code** | PR #548 and PR #573 in the `ddt-salesforce` repo |

---

## 1. Summary

The firm asked for a new 🚫📁 chip on the Google Drive files panel. The ticket described it as a "root only" filter, and that is what was built first. The requester then explained he wanted the opposite: one flat list of **every** file on the case, wherever it lives, with no folder rows. The second version shipped that, with an emoji-only label.

## 2. What was reported

- **2026-09-28, ticket opened:** add a 🚫📁 chip between **All** and **PDF** that shows only files sitting in the root of the case's Drive folder, hiding anything inside a subfolder.
- **2026-09-30, first version live (PR #548):** "🚫📁 Root only" showed only the files directly in the case folder.
- **2026-09-30, staff report:** "It's not working" on one case. This turned out to be browser cache; the button appeared after a refresh.
- **2026-10-01, the requester:** the chip does the opposite of what he wanted. "Root only" came from the ticket's wording, not his request. He asked for an emoji-only label and a flat list of every file on the case, top level and nested, with no folders.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `driveFilePanel` | LWC | New chip in the `typeChips` row; `.js`, `.html` and jest tests edited |

**Out of scope**

- **Apex.** No change to `DriveFolderController`. The chip reuses its existing case-wide search.
- **Combining the chip with PDF or other type chips.** One chip at a time, like the others.

## 4. How it works (background)

```
driveFilePanel (on the record page)
  All chip     → browse view: folders and files, one folder at a time
  Type chips   → DriveFolderController.searchCaseFiles walks the case folder
                 and returns matching files, each with an inSubfolder flag
```

The case-wide search returns **files only**; it never adds folders to its results. It stops at three levels below the case folder, 40 folders and 300 files (`MAX_SEARCH_DEPTH`, `MAX_SEARCH_LISTINGS`, `MAX_SEARCH_RESULTS`), and reports when it stops early.

## 5. Root cause

Not applicable as a fault. The first version was built correctly to the ticket. The ticket's description ("root only") was a rewrite of the request that inverted its meaning. That is the lesson: the requester's own words were "no subfolders", meaning "no folder structure", which the ticket read as "no files from subfolders".

## 6. Solution

### 6.1 First version (PR #548), replaced

The chip ran the case search with no category and dropped every hit whose `inSubfolder` flag was true. Browser-side only.

### 6.2 Final version (PR #573)

- The chip runs the same case-wide search with no category and **keeps every hit**, top level and every subfolder down to the search's depth limit.
- **No folder rows** needs no extra code, because the search never returns folders.
- The Folder column still shows where each file lives, so staff can click through to its folder.
- **All** goes back to the normal folder view.
- The chip shows only the emoji. Hovering shows "Every file in this case, in one list with no folders", and the same text is the button's `aria-label` so screen readers still announce it.
- The root-only filter and its two status notes from PR #548 were removed.

Why browser-side only: the search already returned everything needed, so an Apex change would only have added deploy risk.

## 7. Testing

Jest, `driveFilePanel.test.js`, 8 of 8 passing, 4 of them for this chip:

- chip order, emoji-only label, tooltip and `aria-label`;
- every file listed (top level, one and two levels down) with no folder row, and **All** going back to the folder view;
- the server's message on a case with no files;
- the server's truncation message on a large case.

**Not covered:** none of this renders the component in an org, and no sandbox can, because the sandbox lacks the Drive controller. The first real render was in production, checked right after deploy. For the first version, production checks on one Matter and one Court Case showed only root files; for the final version, the requester was told both changes were live and how the chip behaves.

## 8. Step by step

1. Confirm what the requester means in their own words before building to a ticket's paraphrase.
2. In `driveFilePanel`, add the chip to the `typeChips` getter between **All** and **PDF**.
3. When the chip is on, call `DriveFolderController.searchCaseFiles` with no category and show every hit.
4. Set the chip's label to the emoji, with a tooltip and a matching `aria-label`.
5. Add jest tests for the order, the label, nested files, the empty case and truncation.
6. Retrieve `driveFilePanel` from production and confirm it matches `main`.
7. Validate against production, get approval, deploy.
8. Open a case with files nested two levels down, on both a Matter and a Court Case page, and confirm the flat list with no folders.

## 9. Rollback

Redeploy `driveFilePanel` from the version before PR #573 (the root-only chip) or before PR #548 (no chip). LWC only, no data change, safe either way.

## 10. Known limits

- The list is partial on a case deeper than three levels or larger than 40 folders or 300 files. The panel says so, using the server's own message. The type chips have the same limit.
- On mobile there is no hover, so a touch user cannot see the tooltip on an emoji-only chip.
- With a search term typed and **All** on, the panel already shows every match on the case, so the chip only changes the highlight.

## 11. Reuse for similar issues

- **Read the requester's original words, not only the ticket.** A paraphrase can invert a request.
- **Reuse an existing server call when it already returns what you need**; a browser-side filter keeps the change to one bundle.
- **An emoji-only control needs a tooltip and an `aria-label`.**
- **"It's not working" right after an LWC deploy:** ask for a hard refresh first. Here it was browser cache.
- **No sandbox can render this panel**, so plan to open the screen in production straight after deploy.
