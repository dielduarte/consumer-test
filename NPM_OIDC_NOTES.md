# npm Trusted Publishers + Shared Workflows: What We Know

## Goal
Publish `diel-testing` to npm using:
- npm Trusted Publishers (OIDC, no long-lived tokens)
- `npm stage publish` (staged publishing, requires npm ≥ 11.15.0)
- Shared workflows architecture: `dielduarte/shared-workflows` enforces pipeline structure, consumer repo owns all execution via composite actions

## Architecture

```
consumer-test/.github/workflows/release-automation.yml  (caller)
  └── calls dielduarte/shared-workflows/.github/workflows/release-automation.yml
        └── jobs: check-version → ci → release → notify
              ci job: runs ./.github/actions/ci      (from consumer-test)
              release job: runs ./.github/actions/release  (from consumer-test)
```

**Rule**: shared-workflows NEVER run commands directly. They only define the job pipeline structure. All actual commands live in the consumer's composite actions.

## npm Trusted Publisher Config (npmjs.com)
Package: `diel-testing`  
Owner: `dielduarte`  
Repository: `consumer-test`  
Workflow: `release-automation.yml`  
Environment: `production`  

The `environment: production` on the shared workflow's `release` job is what puts the `environment` claim into the OIDC token. This must match what's configured on npm.

## Key Learnings

### 1. `actions/setup-node` with `registry-url` breaks OIDC
When you pass `registry-url` to `setup-node`, it:
- Creates a temp `.npmrc` at `$NPM_CONFIG_USERCONFIG` with `//registry.npmjs.org/:_authToken=${NODE_AUTH_TOKEN}`
- Sets `NODE_AUTH_TOKEN` to the GitHub Actions token (not an npm token)
- npm uses this GitHub token → **E401**

**Fix**: Do NOT pass `registry-url` to `setup-node`. Node version setup and npm auth are independent.

### 2. npm does NOT auto-detect OIDC for `npm publish` / `npm stage publish`
Without `actions/setup-node`'s token injection, npm just returns **ENEEDAUTH** — it doesn't automatically fetch the GitHub OIDC token.

**Fix**: Manually fetch the OIDC token and pass it as `NPM_ID_TOKEN`. This is exactly what `resend/react-email` does in `scripts/release.mts`:
```typescript
const npmIdToken = await core.getIDToken('npm:registry.npmjs.org');
// passed as NPM_ID_TOKEN env var to the publish command
```

In bash (composite action):
```bash
NPM_ID_TOKEN=$(curl -sH "Authorization: Bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" \
  "${ACTIONS_ID_TOKEN_REQUEST_URL}&audience=npm:registry.npmjs.org" | jq -r '.value')
NPM_ID_TOKEN="$NPM_ID_TOKEN" NPM_CONFIG_PROVENANCE=true npm publish --access public
```

### 3. `NPM_ID_TOKEN` was still empty as of last test
The curl to fetch the OIDC token was returning empty. We did NOT confirm whether:
- `ACTIONS_ID_TOKEN_REQUEST_URL` is available in the reusable workflow's job
- The curl itself fails (wrong audience, network, permissions)
- `jq` is not on PATH

**Next debugging step**: Add echo statements to print whether `ACTIONS_ID_TOKEN_REQUEST_URL` and `ACTIONS_ID_TOKEN_REQUEST_TOKEN` are set before the curl. We had a debug version ready but the session ended before approval.

### 4. Caller must explicitly grant `id-token: write` to reusable workflows
Reusable workflows from external repos default to no permissions. The caller must pass them explicitly:
```yaml
# consumer-test/.github/workflows/release-automation.yml
jobs:
  release:
    uses: dielduarte/shared-workflows/.github/workflows/release-automation.yml@main
    permissions:
      id-token: write
      contents: read
```
AND the reusable workflow's job must also declare `permissions: id-token: write`.

### 5. `npm stage publish` requires npm ≥ 11.15.0
The default npm on GitHub-hosted runners doesn't have `npm stage`. Must run `npm install -g npm@latest` first.

### 6. Version check: use npm registry, not changeset files
After a changeset version PR merges, the `.changeset/*.md` files are consumed. Using them as a gate always returns false. Use npm registry check instead:
```bash
npm view "${PACKAGE_NAME}@${VERSION}" version 2>/dev/null | grep -q "${VERSION}"
```

### 7. GitHub Actions: can't split PR create vs approve permission
`Settings → Actions → General → "Allow GitHub Actions to create and approve pull requests"` is one setting — you can't allow only creation.

## Current State of Files

### consumer-test/.github/actions/release/action.yml
```yaml
name: Consumer Release
runs:
  using: composite
  steps:
    - uses: actions/setup-node@v6
      with:
        node-version: 24
        package-manager-cache: false   # no registry-url!
    - run: npm install -g npm@latest
      shell: bash
    - name: Publish to npm
      shell: bash
      run: |
        NPM_ID_TOKEN=$(curl -sH "Authorization: Bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" \
          "${ACTIONS_ID_TOKEN_REQUEST_URL}&audience=npm:registry.npmjs.org" | jq -r '.value')
        NPM_ID_TOKEN="$NPM_ID_TOKEN" NPM_CONFIG_PROVENANCE=true npm publish --access public
```

## What to Try Next Session

1. **Debug the OIDC token fetch** — add echo before curl to confirm `ACTIONS_ID_TOKEN_REQUEST_URL` is set in the release job. Run and check logs before approving production gate.

2. **If OIDC URL is NOT available**: the `id-token: write` permission may not be propagating correctly through the reusable workflow chain. Try declaring `permissions: id-token: write` at the top-level of the shared workflow as well.

3. **If OIDC URL IS available but curl returns empty/null**: check the audience string — may need URL-encoding (`npm%3Aregistry.npmjs.org` vs `npm:registry.npmjs.org`).

4. **Once `npm publish` works**: switch back to `npm stage publish` (same OIDC flow, just different command). Staged packages need manual approval at npmjs.com before going live.

5. **Reference**: `resend/react-email/scripts/release.mts` line 178 is the working implementation. It uses `@actions/core`'s `getIDToken('npm:registry.npmjs.org')` — which is the JS equivalent of the curl above.
