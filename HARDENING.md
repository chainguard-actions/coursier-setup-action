<!-- markdownlint-disable -->

# Hardening Report: coursier--setup-action/v2.0.3

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **coursier--setup-action/v2.0.3** was hardened automatically. 4 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Multiple workflow files reference external actions using mutable tags instead of full 40-character SHA commit hashes, making them vulnerable to supply-chain attacks if the tag is moved.

Failing references:
- .github/workflows/no-dist-changes.yml: `uses: actions/checkout@v6`
- .github/workflows/release.yml: `uses: actions/checkout@v6`, `uses: tibdex/github-app-token@v2`
- .github/workflows/test.yml: `uses: actions/checkout@v6`, `uses: actions/setup-node@v6`
- .github/workflows/update-dist.yml: `uses: actions/checkout@v6`, `uses: actions/setup-node@v6`, `uses: peter-evans/create-pull-request@v8`

Locations:

- `.github/workflows/no-dist-changes.yml:11`
- `.github/workflows/release.yml:6`
- `.github/workflows/release.yml:12`
- `.github/workflows/test.yml:18`
- `.github/workflows/test.yml:20`
- `.github/workflows/update-dist.yml:8`
- `.github/workflows/update-dist.yml:10`
- `.github/workflows/update-dist.yml:19`

### missing-permissions (severity: medium)

None of the workflow files define a top-level `permissions:` key, and no job within them defines job-level permissions either. Without explicit permissions, workflows run with the default (potentially broad) token permissions, violating the principle of least privilege.

Locations:

- `.github/workflows/no-dist-changes.yml:1`
- `.github/workflows/release.yml:1`
- `.github/workflows/test.yml:1`
- `.github/workflows/update-dist.yml:1`

### script-injection (severity: high)

Multiple `run:` blocks directly interpolate `${{ ... }}` expressions into shell commands. Before the shell executes the command, GitHub Actions performs template substitution, allowing an attacker-controlled value to inject arbitrary shell commands.

(a) `.github/workflows/no-dist-changes.yml` line 16: `${{ github.base_ref }}` is interpolated directly into a shell command. On a `pull_request` trigger, `github.base_ref` is attacker-controlled via the PR branch name.
  Offending line: `changed=$(git diff --name-only "origin/${{ github.base_ref }}...HEAD" -- dist/)`

(b) `.github/workflows/test.yml` line 37: `${{ steps.extra-jvm-args-test.outcome }}` is interpolated directly into a `run:` block.
  Offending line: `if [ "${{ steps.extra-jvm-args-test.outcome }}" = "failure" ]; then`

(c) `.github/workflows/test.yml` line 56: `${{ steps.cs-setup.outputs.cs-version }}` is interpolated directly into a `run:` block.
  Offending line: `- run: echo cs-version=${{ steps.cs-setup.outputs.cs-version }}`

(d) `.github/workflows/release.yml` line 15: `${{ steps.generate-token.outputs.token }}` is interpolated directly into a `run:` block.
  Offending line: `echo "GH_TOKEN=${{ steps.generate-token.outputs.token }}" >> $GITHUB_ENV`

(e) `.github/workflows/update-dist.yml` lines 24–25: `${{ steps.cpr.outputs.pull-request-number }}` and `${{ steps.cpr.outputs.pull-request-url }}` are interpolated directly into `run:` blocks.
  Offending lines: `echo "Pull Request Number - ${{ steps.cpr.outputs.pull-request-number }}"`
                   `echo "Pull Request URL - ${{ steps.cpr.outputs.pull-request-url }}"`

Locations:

- `.github/workflows/no-dist-changes.yml:16`
- `.github/workflows/test.yml:37`
- `.github/workflows/test.yml:56`
- `.github/workflows/release.yml:15`
- `.github/workflows/update-dist.yml:24`
- `.github/workflows/update-dist.yml:25`

### github-env-injection (severity: high)

In `.github/workflows/release.yml`, the value of `${{ steps.generate-token.outputs.token }}` — a step output — is written directly to `$GITHUB_ENV` without sanitization (no `printf '%s' ... | tr -d '\n\r'` applied before the write). A newline embedded in the token value could inject arbitrary environment variables into subsequent steps.

Offending line: `echo "GH_TOKEN=${{ steps.generate-token.outputs.token }}" >> $GITHUB_ENV`

Locations:

- `.github/workflows/release.yml:15`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, missing-permissions, script-injection, github-env-injection

**Notes:**

Fixed all four workflow files:

1. **unpinned-uses**: Pinned all action references to full SHA hashes with tag comments:
   - actions/checkout@v6 → @d23441a48e516b6c34aea4fa41551a30e30af803
   - tibdex/github-app-token@v2 → @3beb63f4bd073e61482598c45c71c1019b59b73a
   - actions/setup-node@v6 → @249970729cb0ef3589644e2896645e5dc5ba9c38
   - peter-evans/create-pull-request@v8 → @5f6978faf089d4d20b00c7766989d076bb2fc7f1

2. **missing-permissions**: Added top-level `permissions: {}` to all four workflows, with minimal job-level permissions (contents: read for read-only, contents: write and pull-requests: write where needed).

3. **script-injection**: Moved all ${{ }} expressions from run: blocks into env: blocks and referenced them as plain environment variables:
   - no-dist-changes.yml: github.base_ref → BASE_REF
   - test.yml: steps.extra-jvm-args-test.outcome → OUTCOME; steps.cs-setup.outputs.cs-version → CS_VERSION
   - release.yml: steps.generate-token.outputs.token → TOKEN
   - update-dist.yml: steps.cpr.outputs.pull-request-number → PR_NUMBER; steps.cpr.outputs.pull-request-url → PR_URL

4. **github-env-injection**: In release.yml, sanitized the token with `printf '%s' "$TOKEN" | tr -d '\n\r'` before writing to $GITHUB_ENV to prevent newline injection.

