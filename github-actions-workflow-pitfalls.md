# GitHub Actions Workflow Pitfalls

## Context

Learned from a PR review on a CI notification workflow. The workflow sends Slack messages after package releases. Three subtle issues were caught by reviewers that apply broadly to any GitHub Actions workflow with multi-job dependencies.

---

## Pitfall 1: `if: always()` and DAG Pruning

### The Problem

When you have chained jobs (`jobA` → `jobB` → `jobC`) and use `if: always()` on `jobC`, you'd expect it to **always** run. But GitHub Actions has a known DAG-pruning bug ([actions/runner#3664](https://github.com/actions/runner/issues/3664)) where `jobC` can be silently **skipped** when an upstream job is skipped (not failed — skipped).

### Example

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: echo "building"

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "deploying"

  notify:
    needs: [build, deploy]
    runs-on: ubuntu-latest
    if: always()  # ⚠️ May be pruned when deploy is skipped
    steps:
      - run: echo "sending notification"
```

If `build` fails → `deploy` is **skipped** (not failed) → `notify` may be pruned from the DAG entirely, even though `always()` should mean "always run."

### The Fix

Use `!cancelled()` instead of `always()`:

```yaml
notify:
  needs: [build, deploy]
  if: !cancelled()
```

| Condition | Runs on success | Runs on failure | Runs on skip | Runs on cancel |
|-----------|:-:|:-:|:-:|:-:|
| `always()` | ✅ | ✅ | ⚠️ Bug | ✅ |
| `!cancelled()` | ✅ | ✅ | ✅ | ❌ |

`!cancelled()` correctly runs on success, failure, and skip — but properly skips when the user manually cancels the workflow.

### Best Practice

For notification jobs that need to run regardless of upstream results, prefer `!cancelled()` over `always()`. Consider further filtering to only notify on meaningful events:

```yaml
notify:
  needs: [build, deploy]
  if: |
    !cancelled() && (
      needs.deploy.outputs.deployed == 'true' ||
      contains(needs.*.result, 'failure')
    )
```

This fires notifications only on actual deployments or actual failures, avoiding noise on "nothing to do" runs.

---

## Pitfall 2: Empty Outputs in Downstream Jobs

### The Problem

When a job is skipped, its `outputs` are empty strings. Any downstream job referencing those outputs will get `''`, which can produce confusing messages or broken logic.

### Example

```yaml
jobs:
  release:
    outputs:
      version: ${{ steps.semver.outputs.version }}
    steps:
      - id: semver
        run: echo "version=2.1.0" >> $GITHUB_OUTPUT

  notify:
    needs: release
    steps:
      - run: echo "Released v${{ needs.release.outputs.version }}"
      # When release is skipped → prints "Released v" (empty)
```

### The Fix

Always provide a fallback for outputs that may be empty:

```yaml
- run: echo "Released v${{ needs.release.outputs.version || 'unreleased' }}"
```

### Rule of Thumb

Any time you reference `needs.<job>.outputs.<name>` in a downstream job, ask: **"What happens if this job was skipped?"** If the answer produces a confusing result, add a `|| 'fallback'` default.

---

## Pitfall 3: Secrets in `if:` Conditionals

### The Problem

GitHub Actions **does not allow** referencing `secrets` directly in `if:` expressions:

```yaml
# ❌ This does NOT work
steps:
  - name: Send notification
    if: ${{ secrets.SLACK_WEBHOOK_URL != '' }}
    run: echo "sending"
```

Unset secrets resolve to empty strings, but the `secrets` context is not available in `if:` conditionals for security reasons.

### The Fix

Export the secret to an environment variable first, then check the `env` context:

```yaml
jobs:
  notify:
    env:
      SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
    steps:
      - name: Send notification
        if: ${{ env.SLACK_WEBHOOK_URL != '' }}
        run: echo "sending"
```

### Why This Works

| Context | Available in `if:`? |
|---------|:---:|
| `secrets.*` | ❌ |
| `env.*` | ✅ |
| `github.*` | ✅ |
| `needs.*` | ✅ |
| `steps.*` | ✅ |

The `env` context is fully accessible in conditionals, so mapping a secret to an env var is the standard workaround.

---

## Summary Checklist

When writing multi-job GitHub Actions workflows:

- [ ] Use `!cancelled()` instead of `always()` for notification/cleanup jobs
- [ ] Add `|| 'fallback'` for any `needs.*.outputs.*` that may be empty
- [ ] Never reference `secrets.*` in `if:` — map to `env` first
- [ ] Ask "what happens when upstream is skipped?" for every downstream job
- [ ] Consider gating notifications to meaningful events only (actual success + actual failure)
