# OpenClaw Improvements

## Issues Found

### 1. Formatting Issue: docs/session-storage.md

**Severity:** Low

**Description:** The file `docs/session-storage.md` has formatting issues detected by `pnpm format:check`. The oxfmt linter reports formatting problems in this file.

**Evidence:**

```
pnpm format:check
Checking formatting...

docs/session-storage.md (63ms)

Format issues found in above 1 files. Run without `--check` to fix.
```

**Impact:** The full `pnpm check` command fails due to this formatting issue, blocking CI/PR validation.

**Fix Required:** Run `pnpm format:fix` or manually format the file according to oxfmt conventions.

---

## Summary

| Issue                              | Severity | Status    |
| ---------------------------------- | -------- | --------- |
| docs/session-storage.md formatting | Low      | Needs fix |
