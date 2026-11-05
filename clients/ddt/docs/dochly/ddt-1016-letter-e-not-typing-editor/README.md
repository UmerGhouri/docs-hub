# Lowercase "e" would not type in the Dochly template editor

| | |
|---|---|
| **Ticket** | DDT-1016 |
| **Project** | Dochly |
| **Status** | Done 2026-09-16 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-09-15 |
| **Resolved by** | Support team |
| **Systems** | Dochly template editor |
| **Area** | Editing Dochly templates and documents |
| **Related** | None recorded |
| **Code** | None recorded |

---

## 1. Summary

Pressing the "e" key did nothing inside the Dochly template editor, while the same key worked everywhere else. Several staff hit it. Shift+E still typed a capital E. The support team reported it fixed later the same day; what was changed is not recorded.

## 2. What was reported

- The reporter could not type a lowercase "e" in the Dochly editor. A hard refresh did not help. The only workaround was copying and pasting the letter.
- Within hours, at least three more staff said the same happened to them, one of whom had assumed their keyboard was broken. Reports came from more than one kind of machine, including a desktop and a Windows PC.
- Shift+E worked, so only the unshifted key was affected.

The editor in the report was a rich text editor with a CKEditor-style toolbar.

## 3. Scope

**In scope**

- The Dochly template editor.

**Out of scope**

- Any other text field in Salesforce. The key worked normally everywhere outside the Dochly editor.

## 4. How it works (background)

```
User presses a key inside the Dochly editor
  -> the editor's keyboard handling decides whether it's text or a shortcut
  -> text is inserted into the document
```

A rich text editor can bind single keys to actions. A key bound that way is consumed by the action and never reaches the page as text, while the shifted key, which isn't bound, still types.

## 5. Root cause

Not recorded in the ticket. The ticket listed three candidates: a keyboard shortcut in the editor bound to "e", a browser extension, or a Dochly-side bug. That Shift+E worked, and that it affected several people on different machines, fits an editor shortcut better than a browser extension, but which one it was is not recorded.

## 6. Solution

The resolution on the ticket reads: "it's fixed now". What was changed is not recorded.

## 7. Testing

Not recorded in the ticket. The acceptance criteria asked the reporter to confirm they could type "e" normally; that confirmation is not recorded.

## 8. Step by step

Not recorded in the ticket. To investigate the same symptom:

1. Reproduce it in the Dochly editor, and note whether the shifted key works.
2. Try the same user in a private browser window with extensions off. If it still fails, it is not an extension.
3. Try a second user on a different machine. If both fail, it is the editor, not the machine.

## 9. Rollback

Not applicable: the change made is not recorded.

## 10. Known limits

- **The cause is not recorded**, so if it comes back there is nothing to compare against. Record the cause and the change the next time.

## 11. Reuse for similar issues

**How to recognise one:** one key, or one key combination, does nothing in one editor and works everywhere else.

- **Several users on different machines points at the application**, not a keyboard or a browser extension.
- **The shifted key working while the plain key fails** points at a single-key shortcut taking the keystroke.
- **Copy and paste is a usable stopgap** while it is fixed.
- **Write down what was changed** when you close a ticket like this. "It's fixed" leaves the next person starting from scratch.
