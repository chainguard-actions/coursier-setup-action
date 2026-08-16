<!-- markdownlint-disable -->

# Hardening Report: coursier--setup-action/v2.0.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **coursier--setup-action/v2.0.0** was hardened automatically. 4 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Multiple workflow files reference actions using mutable tags instead of full 40-character SHA commit digests, making the workflow vulnerable to supply-chain attacks if the tag is moved.

- `.github/workflows/release.yml` line 12: `uses: actions/checkout@v6`
- `.github/workflows/release.yml` line 17: `uses: tibdex/github-app-token@v2`
- `.github/workflows/test.yml` line 19: `uses: actions/checkout@v6`
- `.github/workflows/test.yml` line 21: `uses: actions/setup-node@v6`
- `.github/workflows/update-dist.yml` line 11: `uses: actions/checkout@v6`
- `.github/workflows/update-dist.yml` line 13: `uses: actions/setup-node@v6`
- `.github/workflows/update-dist.yml` line 27: `uses: peter-evans/create-pull-request@v7`

All should be pinned to a full SHA, e.g. `actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4`.

Locations:

- `.github/workflows/release.yml:12`
- `.github/workflows/release.yml:17`
- `.github/workflows/test.yml:19`
- `.github/workflows/test.yml:21`
- `.github/workflows/update-dist.yml:11`
- `.github/workflows/update-dist.yml:13`
- `.github/workflows/update-dist.yml:27`

### permissions (severity: medium)

None of the three workflow files define a top-level `permissions:` block, and no job within them defines job-level `permissions:` either. Without explicit permissions, workflows run with the repository's default token permissions (often `write-all`), granting unnecessary access. Each workflow should declare minimal required permissions.

Locations:

- `.github/workflows/release.yml:1`
- `.github/workflows/test.yml:1`
- `.github/workflows/update-dist.yml:1`

### script-injection (severity: high)

Direct `${{ }}` expression interpolation inside `run:` shell command strings — violates rule (a). The GitHub Actions expression is substituted into the shell command before the shell parses it, allowing an attacker who controls the value to inject arbitrary shell commands.

1. `.github/workflows/release.yml` line 21: `echo "GH_TOKEN=${{ steps.generate-token.outputs.token }}" >> $GITHUB_ENV` — `steps.generate-token.outputs.token` is interpolated directly into the shell command.

2. `.github/workflows/test.yml` line 39: `run: echo cs-version=${{ steps.cs-setup.outputs.cs-version }}` — `steps.cs-setup.outputs.cs-version` is interpolated directly.

3. `.github/workflows/update-dist.yml` line 34: `echo "Pull Request Number - ${{ steps.cpr.outputs.pull-request-number }}"` — step output interpolated directly.

4. `.github/workflows/update-dist.yml` line 35: `echo "Pull Request URL - ${{ steps.cpr.outputs.pull-request-url }}"` — step output interpolated directly.

Fix: move the value into an `env:` variable and reference it as a quoted shell variable, e.g. `echo "GH_TOKEN=$GH_TOKEN_VALUE" >> $GITHUB_ENV` with `env: GH_TOKEN_VALUE: ${{ steps.generate-token.outputs.token }}`.

Locations:

- `.github/workflows/release.yml:21`
- `.github/workflows/test.yml:39`
- `.github/workflows/update-dist.yml:34`
- `.github/workflows/update-dist.yml:35`

### github-env-injection (severity: high)

In `.github/workflows/release.yml` line 21, the value of `${{ steps.generate-token.outputs.token }}` — a `steps.*.outputs.*` value, which is an untrusted/workflow-controllable source — is written directly to `$GITHUB_ENV` without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`). A newline embedded in the token value could inject arbitrary environment variables into subsequent steps.

Offending line: `echo "GH_TOKEN=${{ steps.generate-token.outputs.token }}" >> $GITHUB_ENV`

Fix:
```yaml
env:
  TOKEN_VALUE: ${{ steps.generate-token.outputs.token }}
run: |
  safe=$(printf '%s' "$TOKEN_VALUE" | tr -d '\n\r')
  echo "GH_TOKEN=$safe" >> "$GITHUB_ENV"
```

Locations:

- `.github/workflows/release.yml:21`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, permissions, script-injection, github-env-injection

**Notes:**

Fixed all four findings across three workflow files:

1. **unpinned-uses**: Pinned all 7 action references to full SHA digests with tag comments preserved: actions/checkout@df4cb1c069e1874edd31b4311f1884172cec0e10 # v6, tibdex/github-app-token@3beb63f4bd073e61482598c45c71c1019b59b73a # v2, actions/setup-node@249970729cb0ef3589644e2896645e5dc5ba9c38 # v6, peter-evans/create-pull-request@22a9089034f40e5a961c8808d113e2c98fb63676 # v7.

2. **permissions**: Added top-level `permissions: {}` to all three workflows. Added job-level minimal permissions: `contents: write` for release and update-dist jobs, `contents: read` for test job, and `pull-requests: write` for update-dist job.

3. **script-injection**: Moved all `${{ }}` expressions from `run:` blocks into `env:` blocks and referenced them as plain shell variables: TOKEN_VALUE in release.yml, CS_VERSION in test.yml, PR_NUMBER and PR_URL in update-dist.yml.

4. **github-env-injection**: In release.yml, the token value is now sanitized with `printf '%s' "$TOKEN_VALUE" | tr -d '\n\r'` before being written to $GITHUB_ENV to prevent newline injection attacks.

