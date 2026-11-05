# Apex controller for the Red Flag Panel


| | |
|---|---|
| **Ticket** | DDT-793 |
| **Project** | Red Flag Panel |
| **Status** | Live since 2026-10-06 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-08-21 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex) |
| **Area** | Matter red flags |
| **Related** | DDT-791, DDT-794, DDT-795 |
| **Code** | PR #639 in the ddt-salesforce repo (later adjusted in PR #663, see DDT-794) |


---


## 1. Summary

`RedFlagPanelController` is the Apex behind the Red Flag Panel. It reads the codes the formula emits, joins them to the active rule records, drops any flag whose bypass checkbox is ticked, and returns them sorted by severity. It also saves one field for an inline fix, and only a field that an active rule declares as its target.

## 2. What was reported

- 2026-08-21: build the controller with two methods, one to read flags and one to save a single field for inline fix, plus a test class covering both through the real entry point and as a non-admin.

## 3. Scope

**In scope**

- `RedFlagPanelController` and `RedFlagPanelControllerTest`.

**Out of scope**

- The component (DDT-794) and permission sets (DDT-795). No page used the controller yet.

## 4. How it works (background)

The formula `Opportunity.Red_Flag_Codes__c` holds codes like `;NO_DOB;`. The catalog `Red_Flag_Rule__mdt` says what each code means. Some rules fix a field on the matter, some on the client account, so the controller works across both objects.

## 5. Root cause

Not applicable: new build.

## 6. Solution

**`getFlags(recordId)`**, cacheable:

1. Read active rules ordered by sort order.
2. Read the matter's codes, its account, and every bypass and target field the active rules declare. The field list is built from the rules.
3. Split the codes on `;`, drop blanks, join to rules by code, drop bypassed rules.
4. Return the flags grouped and sorted Blocker, Warning, Info, then sort order.

**`saveTargetField(recordId, objectApi, fieldApi, value)`**: writes one field on the matter or its account, and rejects any object/field pair that no active Inline Edit rule declares. The rule metadata is the allowlist, not the caller.

Choices a reviewer checked:

- The record read and the save respect the running user's own field access, because the field being written is chosen by configuration. Fields the user cannot read are left out of the query instead of failing it.
- Every API name from metadata is checked against the object describe before use; a rule pointing at a missing field falls back to message only.
- The result is returned as a JSON string, the same pattern as an existing panel controller, because lists of inner classes from a cacheable method come back empty for non-admins.
- A user who cannot read the codes field gets an error naming the permission set, not an empty panel.

## 7. Testing

- Check-only validation against production: `RedFlagPanelControllerTest` 18 of 18; the deploy itself ran 20 of 20.
- Tests cover both methods through the entry point, the bypass path, a code with no rule, a rule whose code never appears, and an inactive rule.
- Not covered: Apex class access for non-admins (DDT-795) and the screen (DDT-794 onward).

## 8. Step by step

1. Read the active rules and build the field list from what they declare.
2. Validate each field name against the describe before querying.
3. Query the record respecting user field access.
4. Join codes to rules, drop bypassed ones, sort.
5. For saves, accept only object/field pairs an active rule names.
6. Test through the public methods, including a non-admin case.

## 9. Rollback

Delete both classes with a destructive deploy. Nothing referenced them until DDT-794 shipped.

## 10. Known limits

- Non-admins need class access through `Red_Flag_Panel_Access` (DDT-795), or the panel shows an access error.

## 11. Reuse for similar issues

- When configuration chooses which field to write, make the configuration the allowlist and let the user's own field access decide.
- Validate every configured API name against the describe before it reaches a dynamic query.
- Return JSON strings from cacheable methods if inner-class lists come back empty for non-admins.
