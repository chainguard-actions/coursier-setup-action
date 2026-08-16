<!-- markdownlint-disable -->

# Hardening Report: coursier--setup-action/v2.0.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **coursier--setup-action/v2.0.1** was hardened automatically. 4 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

All workflow files reference GitHub Actions using mutable version tags instead of full 40-character commit SHAs. This exposes the workflow to supply-chain attacks if a tag is moved or a dependency is compromised. Failing references:
- release.yml: `actions/checkout@v6`, `tibdex/github-app-token@v2`
- test.yml: `actions/checkout@v6`, `actions/setup-node@v6`
- update-dist.yml: `actions/checkout@v6`, `actions/setup-node@v6`, `peter-evans/create-pull-request@v7`

Locations:

- `.github/workflows/release.yml:11`
- `.github/workflows/release.yml:16`
- `.github/workflows/test.yml:18`
- `.github/workflows/test.yml:20`
- `.github/workflows/update-dist.yml:8`
- `.github/workflows/update-dist.yml:10`
- `.github/workflows/update-dist.yml:26`

### missing-permissions (severity: medium)

None of the three workflow files define a top-level `permissions:` block, and no job within them defines job-level permissions. Without explicit permissions, workflows run with the default (often overly broad) GITHUB_TOKEN permissions, violating the principle of least privilege.

Locations:

- `.github/workflows/release.yml:1`
- `.github/workflows/test.yml:1`
- `.github/workflows/update-dist.yml:1`

### script-injection (severity: high)

GitHub Actions expressions (`${{ ... }}`) are interpolated directly inside `run:` shell command strings, violating rule (a). The expression values are substituted into the shell command before the shell parses them, allowing an attacker who controls the expression value to inject arbitrary shell commands.

- release.yml: `echo "GH_TOKEN=${{ steps.generate-token.outputs.token }}" >> $GITHUB_ENV` — `steps.*.outputs.*` expression directly in run block.
- test.yml: `run: echo cs-version=${{ steps.cs-setup.outputs.cs-version }}` — `steps.*.outputs.*` expression directly in run block.
- update-dist.yml: `echo "Pull Request Number - ${{ steps.cpr.outputs.pull-request-number }}"` and `echo "Pull Request URL - ${{ steps.cpr.outputs.pull-request-url }}"` — `steps.*.outputs.*` expressions directly in run block.

Locations:

- `.github/workflows/release.yml:21`
- `.github/workflows/test.yml:44`
- `.github/workflows/update-dist.yml:33`
- `.github/workflows/update-dist.yml:34`

### github-env-injection (severity: high)

In release.yml, the `run:` block writes a value derived from `steps.generate-token.outputs.token` (an untrusted `steps.*.outputs.*` source) directly to `$GITHUB_ENV` without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`). A newline embedded in the token value could allow an attacker to inject arbitrary environment variable definitions into subsequent steps.

Offending line: `echo "GH_TOKEN=${{ steps.generate-token.outputs.token }}" >> $GITHUB_ENV`

Locations:

- `.github/workflows/release.yml:21`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, missing-permissions, script-injection, github-env-injection

**Notes:**

Fixed all four findings across three workflow files:

1. **unpinned-uses**: Pinned all action references to full commit SHAs: actions/checkout@df4cb1c069e1874edd31b4311f1884172cec0e10, tibdex/github-app-token@3beb63f4bd073e61482598c45c71c1019b59b73a, actions/setup-node@249970729cb0ef3589644e2896645e5dc5ba9c38, peter-evans/create-pull-request@22a9089034f40e5a961c8808d113e2c98fb63676. Original tags preserved in comments.

2. **missing-permissions**: Added top-level `permissions: {}` to all three workflows, plus job-level minimal permissions: `contents: write` for release.yml (git/release ops), `contents: read` for test.yml (checkout), `contents: write` + `pull-requests: write` for update-dist.yml (PR creation).

3. **script-injection**: Moved all `${{ steps.*.outputs.* }}` expressions from `run:` blocks into `env:` blocks and referenced them as plain environment variables in shell scripts.

4. **github-env-injection**: In release.yml, the token value is now sanitized with `printf '%s' "$TOKEN" | tr -d '\n\r'` before being written to GITHUB_ENV, preventing newline injection attacks.

