<!-- markdownlint-disable -->

# Hardening Report: coursier--setup-action/v3.0.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **coursier--setup-action/v3.0.1** was hardened automatically. 4 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

All workflow files reference actions using mutable tags instead of pinned 40-character SHA commits, making the workflows vulnerable to supply-chain attacks if the referenced tags are moved.

Failing references:
- actions/checkout@v7
- actions/setup-node@v7
- tibdex/github-app-token@v2
- actions/setup-java@v5
- peter-evans/create-pull-request@v8

Locations:

- `.github/workflows/test.yml:18`
- `.github/workflows/test.yml:20`
- `.github/workflows/release.yml:8`
- `.github/workflows/release.yml:14`
- `.github/workflows/no-dist-changes.yml:9`
- `.github/workflows/update-coursier.yml:14`
- `.github/workflows/update-coursier.yml:36`
- `.github/workflows/update-dist.yml:9`
- `.github/workflows/update-dist.yml:11`

### missing-permissions (severity: medium)

Several workflow files have no top-level `permissions:` key and at least one job also lacks a `permissions:` key, meaning the GITHUB_TOKEN is granted default (potentially write) permissions.

- release.yml: no top-level or job-level permissions on the `release` job.
- no-dist-changes.yml: no top-level or job-level permissions on the `dist-changes` job.
- update-dist.yml: no top-level or job-level permissions on the `update-dist` job.
- test.yml: no top-level permissions; the `test` and `test-nightly` jobs lack job-level permissions (only `test-jvm-launchers` has `permissions: contents: read`).

Locations:

- `.github/workflows/release.yml:1`
- `.github/workflows/no-dist-changes.yml:1`
- `.github/workflows/update-dist.yml:1`
- `.github/workflows/test.yml:1`

### script-injection (severity: high)

Multiple `run:` blocks directly interpolate GitHub Actions expressions (`${{ ... }}`), which causes the expression value to be substituted into the shell command string before the shell parses it. An attacker who can influence these values can inject arbitrary shell commands.

(a) no-dist-changes.yml — `${{ github.base_ref }}` interpolated directly into a git diff shell command. On a pull_request trigger, `github.base_ref` is attacker-controlled:
  `git diff --name-only "origin/${{ github.base_ref }}...HEAD"`

(a) release.yml — `${{ steps.generate-token.outputs.token }}` interpolated directly in a `run:` block:
  `echo "GH_TOKEN=${{ steps.generate-token.outputs.token }}" >> $GITHUB_ENV`

(a) test.yml — `${{ steps.extra-jvm-args-test.outcome }}` interpolated in a run block:
  `if [ "${{ steps.extra-jvm-args-test.outcome }}" = "failure" ]`

(a) test.yml — `${{ steps.cs-setup.outputs.cs-version }}` interpolated in a run block:
  `echo cs-version=${{ steps.cs-setup.outputs.cs-version }}`

(a) test.yml — `${{ steps.cs-setup-nightly.outputs.cs-version }}` interpolated in run blocks:
  `echo cs-version=${{ steps.cs-setup-nightly.outputs.cs-version }}`
  `test "${{ steps.cs-setup-nightly.outputs.cs-version }}" = "nightly"`

(a) update-dist.yml — `${{ steps.cpr.outputs.pull-request-number }}` and `${{ steps.cpr.outputs.pull-request-url }}` interpolated in a run block:
  `echo "Pull Request Number - ${{ steps.cpr.outputs.pull-request-number }}"`
  `echo "Pull Request URL - ${{ steps.cpr.outputs.pull-request-url }}"`

Locations:

- `.github/workflows/no-dist-changes.yml:12`
- `.github/workflows/release.yml:19`
- `.github/workflows/test.yml:43`
- `.github/workflows/test.yml:57`
- `.github/workflows/test.yml:130`
- `.github/workflows/test.yml:133`
- `.github/workflows/update-dist.yml:28`

### github-env-injection (severity: high)

In release.yml, the value of `steps.generate-token.outputs.token` (a step output, which is a workflow-controllable value) is written directly to `$GITHUB_ENV` without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`). A newline embedded in the token value could inject arbitrary environment variable definitions into subsequent steps.

Offending line:
  `echo "GH_TOKEN=${{ steps.generate-token.outputs.token }}" >> $GITHUB_ENV`

The fix is to sanitize the value before writing:
  `safe=$(printf '%s' "$GH_TOKEN_VALUE" | tr -d '\n\r')`
  `echo "GH_TOKEN=$safe" >> "$GITHUB_ENV"`

Locations:

- `.github/workflows/release.yml:19`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, missing-permissions, script-injection, github-env-injection

**Notes:**

Fixed all findings across 5 workflow files:

1. unpinned-uses: Pinned all 5 action references to full 40-char SHAs (actions/checkout, actions/setup-node, tibdex/github-app-token, actions/setup-java, peter-evans/create-pull-request) in test.yml, release.yml, no-dist-changes.yml, update-dist.yml, and update-coursier.yml.

2. missing-permissions: Added top-level `permissions: {}` to test.yml, release.yml, no-dist-changes.yml, and update-dist.yml. Added job-level `permissions: contents: read` to test, test-nightly, and test-jvm-launchers jobs in test.yml; `permissions: contents: write, pull-requests: write` to release job in release.yml and update-dist job in update-dist.yml; `permissions: contents: read` to dist-changes job in no-dist-changes.yml.

3. script-injection: Moved all ${{ }} expressions from run: blocks to env: blocks in no-dist-changes.yml (github.base_ref), release.yml (token output), test.yml (outcome and cs-version outputs), and update-dist.yml (PR number and URL outputs).

4. github-env-injection: In release.yml, sanitized the token value with `printf '%s' "$GH_TOKEN_VALUE" | tr -d '\n\r'` before writing to $GITHUB_ENV.

