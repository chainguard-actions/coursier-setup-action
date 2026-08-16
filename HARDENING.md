<!-- markdownlint-disable -->

# Hardening Report: coursier--setup-action/v2.0.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **coursier--setup-action/v2.0.2** was hardened automatically. 4 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

All workflow files use mutable tag/version refs instead of pinned full SHA commits, making them vulnerable to supply-chain attacks if the referenced action is compromised or a tag is moved.

- release.yml: `uses: actions/checkout@v6` and `uses: tibdex/github-app-token@v2`
- test.yml: `uses: actions/checkout@v6` and `uses: actions/setup-node@v6`
- update-dist.yml: `uses: actions/checkout@v6`, `uses: actions/setup-node@v6`, and `uses: peter-evans/create-pull-request@v8`

All should be replaced with full 40-character hex commit SHAs (e.g. `actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4`).

Locations:

- `.github/workflows/release.yml:12`
- `.github/workflows/release.yml:17`
- `.github/workflows/test.yml:22`
- `.github/workflows/test.yml:24`
- `.github/workflows/update-dist.yml:11`
- `.github/workflows/update-dist.yml:13`
- `.github/workflows/update-dist.yml:27`

### missing-permissions (severity: medium)

None of the three workflow files declare a top-level `permissions:` block, and no job within them declares job-level permissions either. Without explicit permissions, the GITHUB_TOKEN is granted its default (often write) permissions, violating the principle of least privilege.

- release.yml: no permissions block at top-level or job level
- test.yml: no permissions block at top-level or job level
- update-dist.yml: no permissions block at top-level or job level

Locations:

- `.github/workflows/release.yml:1`
- `.github/workflows/test.yml:1`
- `.github/workflows/update-dist.yml:1`

### script-injection (severity: high)

Multiple `run:` blocks interpolate `${{ ... }}` expressions directly inside shell command strings (rule a). Before the shell executes the command, GitHub Actions performs template substitution, allowing an attacker-controlled value to inject arbitrary shell metacharacters.

1. release.yml line 21: `echo "GH_TOKEN=${{ steps.generate-token.outputs.token }}" >> $GITHUB_ENV` — step output interpolated directly in shell.
2. test.yml line 41: `echo cs-version=${{ steps.cs-setup.outputs.cs-version }}` — step output interpolated directly in shell.
3. update-dist.yml line 35: `echo "Pull Request Number - ${{ steps.cpr.outputs.pull-request-number }}"` — step output interpolated directly in shell.
4. update-dist.yml line 36: `echo "Pull Request URL - ${{ steps.cpr.outputs.pull-request-url }}"` — step output interpolated directly in shell.

Fix: move the value into an `env:` variable and reference it as a quoted shell variable (e.g. `"$ENV_VAR"`).

Locations:

- `.github/workflows/release.yml:21`
- `.github/workflows/test.yml:41`
- `.github/workflows/update-dist.yml:35`
- `.github/workflows/update-dist.yml:36`

### github-env-injection (severity: high)

release.yml line 21 writes a value derived from `${{ steps.generate-token.outputs.token }}` (a step output, which is an untrusted/workflow-controlled source) directly to `$GITHUB_ENV` without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`). A newline embedded in the token value could allow an attacker to inject arbitrary environment variable definitions into subsequent steps.

Offending line:
```
echo "GH_TOKEN=${{ steps.generate-token.outputs.token }}" >> $GITHUB_ENV
```

Fix:
```bash
safe=$(printf '%s' "$GENERATED_TOKEN" | tr -d '\n\r')
echo "GH_TOKEN=$safe" >> "$GITHUB_ENV"
```
(with `GENERATED_TOKEN: ${{ steps.generate-token.outputs.token }}` in the step's `env:` block)

Locations:

- `.github/workflows/release.yml:21`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, missing-permissions, script-injection, github-env-injection

**Notes:**

Fixed all 4 findings across 3 workflow files:

1. **unpinned-uses**: Pinned all 7 action references to full SHA commits with tag comments preserved: actions/checkout@df4cb1c069e1874edd31b4311f1884172cec0e10 # v6, tibdex/github-app-token@3beb63f4bd073e61482598c45c71c1019b59b73a # v2, actions/setup-node@249970729cb0ef3589644e2896645e5dc5ba9c38 # v6, peter-evans/create-pull-request@5f6978faf089d4d20b00c7766989d076bb2fc7f1 # v8.

2. **missing-permissions**: Added top-level `permissions: {}` to all three workflows, plus job-level minimal permissions (contents: write for release.yml, contents: read for test.yml, contents: write + pull-requests: write for update-dist.yml).

3. **script-injection**: Moved all ${{ }} expressions from run: shell strings into env: blocks — GENERATED_TOKEN in release.yml, CS_VERSION in test.yml, PR_NUMBER and PR_URL in update-dist.yml.

4. **github-env-injection**: Fixed release.yml to sanitize the token value with `printf '%s' "$GENERATED_TOKEN" | tr -d '\n\r'` before writing to $GITHUB_ENV, and quoted $GITHUB_ENV.

