<!-- markdownlint-disable -->

# Hardening Report: coursier--setup-action/v3.0.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **coursier--setup-action/v3.0.0** was hardened automatically. 4 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

All workflow files use mutable tag-based refs instead of pinned 40-character SHA commit hashes, making the workflows vulnerable to supply-chain attacks if the referenced action tags are moved. Failing references: no-dist-changes.yml: actions/checkout@v6; release.yml: actions/checkout@v6, tibdex/github-app-token@v2; test.yml: actions/checkout@v6, actions/setup-node@v6; update-dist.yml: actions/checkout@v6, actions/setup-node@v6, peter-evans/create-pull-request@v8.

Locations:

- `.github/workflows/no-dist-changes.yml:11`
- `.github/workflows/release.yml:12`
- `.github/workflows/release.yml:17`
- `.github/workflows/test.yml:22`
- `.github/workflows/test.yml:24`
- `.github/workflows/update-dist.yml:9`
- `.github/workflows/update-dist.yml:11`
- `.github/workflows/update-dist.yml:22`

### script-injection (severity: high)

Multiple run: blocks directly interpolate ${{ ... }} expressions into shell commands, allowing script injection. (a) no-dist-changes.yml: `git diff --name-only "origin/${{ github.base_ref }}...HEAD"` — github.base_ref is attacker-controlled on pull_request events and is interpolated directly into the shell command. (b) release.yml: `echo "GH_TOKEN=${{ steps.generate-token.outputs.token }}" >> $GITHUB_ENV` — steps output interpolated directly in run:. (c) test.yml: `if [ "${{ steps.extra-jvm-args-test.outcome }}" = "failure" ]` and `echo cs-version=${{ steps.cs-setup.outputs.cs-version }}` — steps outputs interpolated directly in run:. (d) update-dist.yml: `echo "Pull Request Number - ${{ steps.cpr.outputs.pull-request-number }}"` and `echo "Pull Request URL - ${{ steps.cpr.outputs.pull-request-url }}"` — steps outputs interpolated directly in run:.

Locations:

- `.github/workflows/no-dist-changes.yml:16`
- `.github/workflows/release.yml:21`
- `.github/workflows/test.yml:38`
- `.github/workflows/test.yml:55`
- `.github/workflows/update-dist.yml:28`

### github-env-injection (severity: high)

In release.yml, the run: block writes a value derived from a steps output directly to $GITHUB_ENV without sanitization: `echo "GH_TOKEN=${{ steps.generate-token.outputs.token }}" >> $GITHUB_ENV`. The value from steps.generate-token.outputs.token is interpolated directly into the shell command and written to GITHUB_ENV without the required `printf '%s' ... | tr -d '\n\r'` sanitization step. A newline embedded in the token value could inject arbitrary environment variables into subsequent steps.

Locations:

- `.github/workflows/release.yml:21`

### missing-permissions (severity: medium)

None of the four workflow files define a top-level `permissions:` block, and no individual job within any of these files defines a `permissions:` block either. Without explicit permissions, workflows run with the default repository permissions (which may be read/write for GITHUB_TOKEN), violating the principle of least privilege.

Locations:

- `.github/workflows/no-dist-changes.yml:1`
- `.github/workflows/release.yml:1`
- `.github/workflows/test.yml:1`
- `.github/workflows/update-dist.yml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, github-env-injection, missing-permissions

**Notes:**

Fixed all four findings across all four workflow files:

1. unpinned-uses: Pinned all action refs to full 40-char SHAs with tag comments preserved: actions/checkout@d23441a48e516b6c34aea4fa41551a30e30af803 # v6, tibdex/github-app-token@3beb63f4bd073e61482598c45c71c1019b59b73a # v2, actions/setup-node@249970729cb0ef3589644e2896645e5dc5ba9c38 # v6, peter-evans/create-pull-request@5f6978faf089d4d20b00c7766989d076bb2fc7f1 # v8.

2. script-injection: Moved all ${{ }} expressions from run: blocks into env: blocks and referenced them as plain env vars in shell scripts (BASE_REF, GH_TOKEN_VALUE, OUTCOME, CS_VERSION, PR_NUMBER, PR_URL).

3. github-env-injection: In release.yml, the token is now sanitized with printf '%s' "$GH_TOKEN_VALUE" | tr -d '\n\r' before being written to $GITHUB_ENV.

4. missing-permissions: Added top-level permissions blocks to all four files with least-privilege grants: contents:read for no-dist-changes.yml and test.yml; contents:write for release.yml; contents:write + pull-requests:write for update-dist.yml.

