<!-- markdownlint-disable -->

# Hardening Report: coursier--setup-action/v3.0.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **coursier--setup-action/v3.0.2** was hardened automatically. 4 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Multiple workflow files reference actions using mutable version tags instead of pinned 40-character SHA digests, making them vulnerable to supply-chain attacks if the tag is moved.

Failing references:
- no-dist-changes.yml: actions/checkout@v7
- release.yml: actions/checkout@v7, tibdex/github-app-token@v2
- test.yml: actions/checkout@v7 (×3), actions/setup-node@v7 (×2), actions/setup-java@v5
- update-coursier.yml: actions/checkout@v7, peter-evans/create-pull-request@v8
- update-dist.yml: actions/checkout@v7, actions/setup-node@v7, peter-evans/create-pull-request@v8

Locations:

- `.github/workflows/no-dist-changes.yml:11`
- `.github/workflows/release.yml:12`
- `.github/workflows/release.yml:17`
- `.github/workflows/test.yml:23`
- `.github/workflows/test.yml:25`
- `.github/workflows/test.yml:97`
- `.github/workflows/test.yml:112`
- `.github/workflows/test.yml:114`
- `.github/workflows/test.yml:135`
- `.github/workflows/test.yml:137`
- `.github/workflows/update-coursier.yml:17`
- `.github/workflows/update-coursier.yml:44`
- `.github/workflows/update-dist.yml:11`
- `.github/workflows/update-dist.yml:13`
- `.github/workflows/update-dist.yml:27`

### script-injection (severity: high)

Multiple run: blocks directly interpolate ${{ ... }} expressions into shell commands. This causes the expression value to be substituted into the shell script before execution, allowing an attacker to inject arbitrary shell commands.

(a) no-dist-changes.yml line 17: `${{ github.base_ref }}` is interpolated directly into a git diff command. An attacker controlling the base_ref (e.g. via a crafted PR) can inject shell metacharacters.
  Offending lines:
    dist_changed=$(git diff --name-only "origin/${{ github.base_ref }}...HEAD" -- dist/)
    non_dist_changed=$(git diff --name-only "origin/${{ github.base_ref }}...HEAD" -- ':(exclude)dist/')

(a) release.yml line 21: `${{ steps.generate-token.outputs.token }}` is interpolated directly into a shell echo command that writes to $GITHUB_ENV.
  Offending line:
    echo "GH_TOKEN=${{ steps.generate-token.outputs.token }}" >> $GITHUB_ENV

(a) update-dist.yml lines 35-36: step outputs are interpolated directly into echo commands.
  Offending lines:
    echo "Pull Request Number - ${{ steps.cpr.outputs.pull-request-number }}"
    echo "Pull Request URL - ${{ steps.cpr.outputs.pull-request-url }}"

(a) test.yml line 46: `${{ steps.extra-jvm-args-test.outcome }}` is interpolated directly into a shell if-condition.
  Offending line:
    if [ "${{ steps.extra-jvm-args-test.outcome }}" = "failure" ]; then

(a) test.yml line 130: `${{ steps.cs-setup-nightly.outputs.cs-version }}` is interpolated directly into a test command.
  Offending line:
    test "${{ steps.cs-setup-nightly.outputs.cs-version }}" = "nightly"

Locations:

- `.github/workflows/no-dist-changes.yml:17`
- `.github/workflows/no-dist-changes.yml:18`
- `.github/workflows/release.yml:21`
- `.github/workflows/update-dist.yml:35`
- `.github/workflows/update-dist.yml:36`
- `.github/workflows/test.yml:46`
- `.github/workflows/test.yml:130`

### github-env-injection (severity: high)

In release.yml, the value of `${{ steps.generate-token.outputs.token }}` is written directly to $GITHUB_ENV without sanitization (no `printf '%s' ... | tr -d '\n\r'` step). A newline embedded in the token value could allow injection of additional environment variable definitions into the runner environment.

Offending line (line 21):
  echo "GH_TOKEN=${{ steps.generate-token.outputs.token }}" >> $GITHUB_ENV

Fix: sanitize the value before writing:
  safe=$(printf '%s' "${{ steps.generate-token.outputs.token }}" | tr -d '\n\r')
  echo "GH_TOKEN=$safe" >> "$GITHUB_ENV"

Locations:

- `.github/workflows/release.yml:21`

### missing-permissions (severity: medium)

The following workflow files have no top-level `permissions:` key and at least one job also lacks a `permissions:` key, meaning jobs run with the default (potentially broad) GITHUB_TOKEN permissions:

- release.yml: no top-level permissions, no job-level permissions on the `release` job.
- no-dist-changes.yml: no top-level permissions, no job-level permissions on the `dist-changes` job.
- update-dist.yml: no top-level permissions, no job-level permissions on the `update-dist` job.
- test.yml: no top-level permissions; the `test` and `test-nightly` jobs have no job-level permissions (only `test-jvm-launchers` has `permissions: contents: read`).

Each workflow should declare minimal required permissions at the top level or on every job.

Locations:

- `.github/workflows/release.yml:1`
- `.github/workflows/no-dist-changes.yml:1`
- `.github/workflows/update-dist.yml:1`
- `.github/workflows/test.yml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, github-env-injection, missing-permissions

**Notes:**

Fixed all 4 findings across 5 workflow files:

1. unpinned-uses: Pinned all action references to full SHA digests with tag comments: actions/checkout@v7→3d3c42e5..., actions/setup-node@v7→820762786..., actions/setup-java@v5→b6effb05..., tibdex/github-app-token@v2→3beb63f4..., peter-evans/create-pull-request@v8→5f6978fa...

2. script-injection: Moved all ${{ }} expressions from run: blocks into step env: blocks and referenced them as plain shell variables: github.base_ref→BASE_REF (no-dist-changes.yml), steps.generate-token.outputs.token→GENERATED_TOKEN (release.yml), steps.cpr.outputs.*→PR_NUMBER/PR_URL (update-dist.yml), steps.extra-jvm-args-test.outcome→OUTCOME (test.yml), steps.cs-setup-nightly.outputs.cs-version→CS_VERSION (test.yml).

3. github-env-injection: In release.yml, sanitized the token before writing to $GITHUB_ENV using printf '%s' "$GENERATED_TOKEN" | tr -d '\n\r'.

4. missing-permissions: Added top-level permissions blocks to no-dist-changes.yml (contents: read), release.yml (contents: write), test.yml (contents: read), and update-dist.yml (contents: write, pull-requests: write). update-coursier.yml already had permissions.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed three script injection vulnerabilities in .github/workflows/test.yml:
1. Line ~68 (test job): Replaced `run: echo cs-version=${{ steps.cs-setup.outputs.cs-version }}` with an `env: CS_VERSION: ${{ steps.cs-setup.outputs.cs-version }}` block and `run: echo "cs-version=$CS_VERSION"`.
2. Line ~113 (test-jvm-launchers job): Same fix for `steps.cs-setup-jvm-launcher.outputs.cs-version`.
3. Line ~133 (test-nightly job): Same fix for `steps.cs-setup-nightly.outputs.cs-version`.
In all cases the `${{ }}` expression is now routed through an environment variable and double-quoted in the shell, preventing shell interpretation of any special characters in the output value.

