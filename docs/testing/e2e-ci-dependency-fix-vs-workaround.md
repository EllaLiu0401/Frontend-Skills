# E2E Tests & CI Dependencies: Root Cause Fix vs Workaround

**Date**: 2026-02-19
**Context**: PR review feedback — e2e tests failing in CI pipeline

---

## The Problem

E2E tests were failing in CI because they depend on Docker images that may not exist. The root cause: Docker build jobs were **conditionally skipped** when no relevant code changes were detected.

```yaml
# Before: Docker build only runs when code changes are detected
docker_api:
  if: needs.detect_changes.outputs.api == 'true'  # Skipped if no API changes

docker_web:
  if: needs.detect_changes.outputs.web == 'true'  # Skipped if no web changes

# E2E tests then fail — no Docker image available to run against
e2e_tests:
  needs: [docker_api, docker_web]
  if: ${{ !failure() && !cancelled() }}  # Runs even when docker was skipped
```

**What happened downstream**: The e2e job would start, try to pull images that were never built, and fail.

---

## Two Approaches to Fix It

### Approach A: Workaround at the E2E Layer

Change e2e behaviour to work around the problem rather than fixing the root cause:

```yaml
# Make e2e only run if docker builds actually succeeded
e2e_tests:
  if: ${{ needs.docker_api.result == 'success' && needs.docker_web.result == 'success' }}
```

```ts
// In playwright.config.ts — skip onboarding tests if credentials already exist
const USE_EXISTING_CREDENTIALS = !!(
  process.env['E2E_TEST_EMAIL'] && process.env['E2E_TEST_PASSWORD']
);
```

**What this does**: If docker is skipped, e2e is also skipped. Avoids the crash but the e2e tests simply don't run.

---

### Approach B: Root Cause Fix at the CI Layer

Remove the conditional entirely — always build Docker images regardless of whether code changed:

```yaml
# After: Docker always builds — no condition
docker_api:
  needs: [detect_changes]
  # if: condition removed

docker_web:
  needs: [detect_changes]
  # if: condition removed
```

**What this does**: E2E tests always have a valid image to run against. Problem is gone at the source.

---

## Comparison

| | Root Cause Fix (Approach B) | Workaround (Approach A) |
|---|---|---|
| **Level** | CI config — upstream | E2E test layer — downstream |
| **Approach** | Always run docker builds | Skip e2e when docker is skipped |
| **Result** | Problem eliminated | Problem bypassed |
| **E2E coverage** | Always runs | May be silently skipped |
| **Complexity** | Simpler (remove a condition) | Adds extra logic and env var dependency |

---

## Key Lessons

### 1. Fix at the Lowest Layer Possible

When something breaks, trace the dependency chain to find the actual root cause:

```
E2E tests fail
  └── because: Docker image missing
        └── because: Docker build was skipped
              └── because: Conditional build step
                    └── FIX HERE ← Remove the condition
```

Fixing at the E2E layer means you're patching the symptom, not the cause.

### 2. Workarounds Add Invisible Risk

The workaround approach (`needs.docker_api.result == 'success'`) silently skips e2e when docker is skipped. This means:
- You get a green pipeline ✅
- But your e2e tests didn't actually run
- No test coverage on those PRs

A root cause fix avoids this false confidence.

### 3. Understand What "Skipped" Means in CI

In GitHub Actions, a job can end with `success`, `failure`, `cancelled`, or `skipped`. These are different:

```yaml
# This runs after a skipped job — "not failure" is true for skipped
if: ${{ !failure() && !cancelled() }}

# This only runs if the job actually ran and succeeded
if: ${{ needs.docker_api.result == 'success' }}
```

When debugging CI failures, always check whether upstream jobs were **skipped** vs **failed** — they behave very differently downstream.

### 4. Stay in Sync After Time Off

If you take a few days off, the codebase may have already received fixes for issues you were about to work on. Before implementing a solution:

- Check recent merged PRs that might be related
- Ask teammates if the issue was already addressed
- Look at the git log for the relevant files (`git log --oneline -- <file>`)

```bash
# Quickly check recent changes to a CI config file
git log --oneline -10 -- .github/workflows/pr.yml
```

### 5. Your Workaround Still Showed Good Thinking

The workaround approach was not wrong in isolation — it identified the real dependency problem and tried to handle it defensively. The only issue was that a cleaner upstream fix was already in place. Understanding both approaches deepens your knowledge of CI dependency graphs.

---

## General Pattern: CI Job Dependency Graph

When e2e tests depend on Docker images, the dependency chain is:

```
detect_changes
      │
      ├── docker_api (build image)
      │
      └── docker_web (build image)
                │
                └── e2e_tests (needs images to exist)
```

**Rule of thumb**: If a downstream job (`e2e_tests`) has a hard runtime dependency on an upstream job (`docker_*`), the upstream job should not be skippable unless the downstream job also skips cleanly.

---

## Quick Reference

| Situation | Recommendation |
|-----------|---------------|
| E2E fails because docker image missing | Fix docker build condition, not e2e condition |
| Job silently skips in CI | Check `if:` condition on upstream jobs |
| Defensive `result == 'success'` check | Fine, but ask: why might it not succeed? Fix that first |
| Returning from time off | Check recent merged PRs before implementing fixes |

---

## Testing Checklist

When setting up or debugging e2e tests in CI:

- [ ] Confirm Docker images are always built before e2e jobs run
- [ ] Check whether upstream jobs can be `skipped` and how downstream handles that
- [ ] Verify e2e tests actually ran (not just "passed") in CI output
- [ ] Check `git log` for the CI workflow file when debugging pipeline issues
- [ ] Sync with team after absences before implementing fixes for known issues
