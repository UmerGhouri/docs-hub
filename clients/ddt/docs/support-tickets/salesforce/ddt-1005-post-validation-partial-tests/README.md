# Validation script ran only a fraction of the requested tests

| | |
|---|---|
| **Ticket** | DDT-1005 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-14 |
| **Priority** | No priority (tracking record) |
| **Reported by** | Firm staff, 2026-09-14 (tracking record created after the fix) |
| **Resolved by** | Support team |
| **Systems** | Deployment tooling: `scripts/post-validation.py` (Python), the required `salesforce/validation` merge check |
| **Area** | Release process: validating a pull request against production before merge |
| **Related** | DDT-1004 (the PR where it was caught) |
| **Code** | PR #358 in the `ddt-salesforce` repo |

---

## 1. Summary

`scripts/post-validation.py` runs a check-only validation against production and posts the required `salesforce/validation` status on a pull request. When someone passed several test classes with a repeated `--tests` flag, only the **last** one ran, and the status still posted green. The fix makes repeated flags add up, so `--tests A --tests B --tests C` runs all three.

This was a script change only. Nothing was deployed to Salesforce.

## 2. What was reported

On a consult-scoring PR (see DDT-1004), the validation status read:

```
PASS: RunSpecifiedTests 30 tests, 0 failed, 9/9 components
```

The three test classes requested contain **57** tests. One of them, `ConsultScoringServiceTest`, has 30 by itself. So the gate ran only that class and skipped `ClaudeApiClientTest`, which was the only test of that PR's new code, on a class shared by five features.

The arithmetic was the only clue, and it was only noticed because the same three classes had been run separately with the `sf` CLI and returned 57.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `scripts/post-validation.py` | Python script | `--tests` declared with `action="extend"` |
| `scripts/test_post_validation.py` | Python test | Four new checks |

**Out of scope**

- **Re-checking earlier PRs.** Any earlier PR validated with repeated `--tests` flags ran only the last class. That was flagged as worth a look, not re-run here.
- **Salesforce metadata.** None changed.

## 4. How it works (background)

```
Developer runs:  python scripts/post-validation.py --pr <n> --org <prod-alias> --tests A --tests B
  -> argparse reads the arguments
  -> check-only deploy against production with RunSpecifiedTests for the named classes
     (or RunLocalTests when --tests is omitted)
  -> posts the salesforce/validation status on the PR
```

`salesforce/validation` is a **required** status check: a PR can't merge without it.

## 5. Root cause

The argument was declared with a plain `nargs`:

```python
ap.add_argument("--tests", nargs="*", default=None, ...)
```

Python's `argparse` default action is `store`, so each repeated flag **replaces** the previous value. `--tests A --tests B --tests C` produces `['C']`. Nothing in the output said so.

**Why it went unnoticed.** The `sf project deploy start` CLI does the opposite: there, repeated `--tests` flags do add up. People used the same habit with both tools. The declaration also looked correct when read.

## 6. Solution

```python
ap.add_argument("--tests", nargs="+", action="extend", default=None,
                help="run only these test classes instead of RunLocalTests (fast path). "
                     "Repeatable, and accepts several names after one flag; both forms accumulate.")
```

| Invocation | Before | After |
|---|---|---|
| `--tests A B C` | `['A','B','C']` | `['A','B','C']` |
| `--tests A --tests B --tests C` | `['C']` | `['A','B','C']` |
| flag omitted | `None` | `None` |

Omitting the flag still gives `None`, so `RunLocalTests` remains the default.

## 7. Testing

`scripts/test_post_validation.py` gained four checks:

```python
_ap = argparse.ArgumentParser()
_ap.add_argument("--tests", nargs="+", action="extend", default=None)
check(_ap.parse_args(["--tests", "A", "B", "C"]).tests == ["A", "B", "C"], ...)
check(_ap.parse_args(["--tests", "A", "--tests", "B", "--tests", "C"]).tests == ["A", "B", "C"], ...)
check(_ap.parse_args([]).tests is None, ...)
check('action="extend"' in <the --tests declaration in post-validation.py>, ...)
```

The tests check **real `argparse` behaviour**, not only the source text, because the bug was that the declaration looked right. It was confirmed that the repeated-flag check fails under the old declaration (it returns `['C']`) and passes with the fix.

**Not covered:** a full end-to-end run that counts the tests reported by the org against the classes requested.

## 8. Step by step

1. Notice the mismatch: compare the test count the status posted with the sum of the test methods in the classes you named.
2. Find how the script parses `--tests`. A plain `nargs` without `action="extend"` keeps only the last repeated flag.
3. Change the declaration to `nargs="+", action="extend", default=None`.
4. Add tests that parse real argument lists for both forms, plus the omitted case.
5. Run `python scripts/test_post_validation.py` and confirm it passes, and that the repeated-flag check fails if you revert the declaration.
6. Merge the PR. Nothing to deploy to Salesforce.

## 9. Rollback

Revert the one-line declaration. Doing so brings the bug back, so there is no reason to unless `extend` itself misbehaves.

## 10. Known limits

- **Earlier validations** that used repeated flags ran only the last class. Spot them by comparing the posted test count with the sum of the named classes.
- The script still reports only a count. A gate that printed the class names it actually ran would make this kind of gap visible directly.

## 11. Reuse for similar issues

**How to recognise one:** a check reports success, but the number it reports is smaller than the work you asked for.

- **Compare counts, not colours.** A green status with the wrong count is the clue.
- **Python `argparse`:** for a repeatable list flag, use `action="extend"` (or `append`). Plain `nargs` keeps only the last occurrence.
- **Similar-looking tools can disagree.** `sf` accumulates repeated `--tests`; a wrapper script may not.
- **Test the behaviour, not the source.** Parse real argument lists in the test, and prove the test fails against the old code.
- **A required gate that can quietly narrow** is a check that can't fail. Make it say what it actually did.
