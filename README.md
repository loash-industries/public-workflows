# public-workflows

Reusable GitHub Actions workflows for **public** npm packages built by Trinary Exchange


## Workflows

### `js-validate-code.yml`

Lint, typecheck, and run tests for JavaScript/TypeScript projects.

```yaml
jobs:
  validate-code:
    uses: loash-industries/public-workflows/.github/workflows/js-validate-code.yml@v0.1.0
    with:
      node_version: 22
      npm_client: yarn
      npm_client_production_flag: --frozen-lockfile
```

**Inputs:**

| Input | Required | Default | Description |
|---|---|---|---|
| `node_version` | yes | - | Node.js version |
| `npm_client` | yes | `yarn` | Package manager (`npm`, `yarn`, `pnpm`) |
| `npm_client_production_flag` | no | `--frozen-lockfile` | Install flag |
| `yarn_version` | no | `1` | Yarn version (`1` or `4`) |
| `test_names` | no | `['test:cov']` | JSON array of test script names |
| `test_timeout_minutes` | no | `30` | Timeout for test jobs |
| `eslint_enabled` | no | `true` | Run ESLint |
| `prettier_enabled` | no | `true` | Run Prettier |
| `typecheck_enabled` | no | `true` | Run TypeScript type checking |
| `typecheck_command` | no | `tscheck` | Type check script name |
| `working_directory` | no | `.` | Working directory (monorepo support) |

---

### `js-create-release-tag.yml`

Bump version using `standard-version`, commit, and push a git tag.

```yaml
jobs:
  create-tag:
    uses: loash-industries/public-workflows/.github/workflows/js-create-release-tag.yml@v0.1.0
    with:
      package_json_path: package.json
```

**Inputs:**

| Input | Required | Default | Description |
|---|---|---|---|
| `package_json_path` | yes | - | Path to `package.json` |
| `node_version` | no | `22` | Node.js version |
| `npm_client` | no | `yarn` | Package manager |
| `add_v_prefix_to_version` | no | `true` | Prefix tag with `v` |

**Outputs:**

| Output | Description |
|---|---|
| `tag` | The created release tag |

---

### `js-semantic-release.yml`

Version, changelog, tag, GitHub release and npm publish with
[semantic-release](https://semantic-release.gitbook.io/), driven by the
caller's `.releaserc`.

The caller must grant everything the `release` job requests, including
`id-token: write` even when publishing with `NPM_TOKEN`:

```yaml
permissions:
  id-token: write
  contents: write
  issues: write
  pull-requests: write

jobs:
  release:
    uses: loash-industries/public-workflows/.github/workflows/js-semantic-release.yml@main
    with:
      node_version: "22"
      environment: prd
      npm_trusted_publishing: true
    secrets:
      GH_AUTH_TOKEN: ${{ secrets.GH_AUTH_TOKEN }}
```

**Inputs:**

| Input | Required | Default | Description |
|---|---|---|---|
| `node_version` | yes | - | Node.js version |
| `environment` | no | - | GitHub environment for the `release` job |
| `npm_client` | no | `yarn` | Package manager (`npm`, `yarn`, `pnpm`) |
| `npm_client_production_flag` | no | `--frozen-lockfile` | Install flag |
| `yarn_version` | no | `1` | Yarn version (`1` or `4`) |
| `npm_trusted_publishing` | no | `false` | Publish with [npm trusted publishing](#npm-trusted-publishing) instead of `NPM_TOKEN` |
| `build_output_path` | no | `dist` | Directory the build writes, relative to `working_directory` |
| `working_directory` | no | `.` | Working directory (monorepo support) |
| `npm_registry_url` | no | - | Deprecated and ignored: the registry comes from `publishConfig.registry` or `.npmrc` |

**Secrets:**

| Secret | Required | Description |
|---|---|---|
| `GH_AUTH_TOKEN` | yes | GitHub token that pushes the release commit and tag and creates the GitHub release |
| `NPM_TOKEN` | unless `npm_trusted_publishing` | npm token. Ignored under trusted publishing |

---

### `js-publish-npm.yml`

Build and publish a package to the **public** npmjs.org registry.

```yaml
jobs:
  publish:
    uses: loash-industries/public-workflows/.github/workflows/js-publish-npm.yml@v0.1.0
    with:
      node_version: 22
      npm_client: yarn
      npm_trusted_publishing: true
```

**Inputs:**

| Input | Required | Default | Description |
|---|---|---|---|
| `node_version` | yes | - | Node.js version |
| `npm_client` | no | `yarn` | Package manager |
| `environment` | no | - | GitHub environment for the `publish` job |
| `access` | no | `public` | npm access level (`public` or `restricted`) |
| `npm_trusted_publishing` | no | `false` | Publish with [npm trusted publishing](#npm-trusted-publishing) instead of `TRINARYEX_NPM_TOKEN` |
| `build_output_path` | no | `dist` | Directory the build writes, relative to `working_directory` |
| `working_directory` | no | `.` | Working directory |

**Secrets:**

| Secret | Required | Description |
|---|---|---|
| `TRINARYEX_NPM_TOKEN` | unless `npm_trusted_publishing` | npmjs.org token. Ignored under trusted publishing |

---

### How the publishing workflows are split

`js-semantic-release.yml` and `js-publish-npm.yml` each run two jobs:

1. **`build`** installs dependencies and runs `build` with a read-only
   `GITHUB_TOKEN` and no OIDC token, then uploads `build_output_path` as an
   artifact.
2. **`release`** / **`publish`** installs again with dependency lifecycle
   scripts disabled, downloads that build output and publishes it. Only this
   job can mint an npm publish token.

So a dependency's install or build script can't publish or push. It can still
tamper with the build output, as it could before.

### npm trusted publishing

With `npm_trusted_publishing: true`, npm exchanges a short-lived GitHub OIDC
token for a publish token, and no npm token is stored anywhere. npm also adds
provenance to public packages built from public repos. To set it up:

1. On npmjs.com, open the package's **Settings → Trusted publisher** and add
   GitHub Actions with the org, the repo, the **calling** workflow's file name
   (e.g. `publish.yml`, not this repo's workflow) and the `environment` you
   pass, if any. The package must already exist on npm.
2. In `package.json`, set `repository.url` to the GitHub repo, which npm checks
   for provenance. For `js-semantic-release.yml`, `publishConfig.registry` must
   be unset or exactly `https://registry.npmjs.org/`, **with** the trailing
   slash: `@semantic-release/npm` compares the string as-is and doesn't try OIDC
   for anything else.
3. For `js-semantic-release.yml`, use semantic-release 25
   (`@semantic-release/npm` 13.1.0 or later), which brings npm 11.5.1 or later.
   `js-publish-npm.yml` upgrades npm itself.
4. Set `npm_trusted_publishing: true` and stop passing the npm token.

`js-semantic-release.yml` checks the registry and npm version before
releasing, so a misconfiguration fails before any tag is pushed.

---

### `workflow-lint.yml`

Lints workflow YAML files, embedded shell scripts, and validates composite action metadata. Runs automatically on PRs and pushes that modify `.github/workflows/` or `.github/actions/`.

## Composite Actions

### `node-base-actions`

Shared setup action used by the workflows above. Handles Node.js setup, caching, and dependency installation for public repos.

## Example: Full Publish Pipeline

```yaml
name: Publish

permissions:
  id-token: write
  contents: write
  actions: read
  checks: write

on:
  push:
    branches: [main]

jobs:
  validate-code:
    if: github.actor != 'github-actions[bot]'
    uses: loash-industries/public-workflows/.github/workflows/js-validate-code.yml@v0.1.0
    with:
      node_version: 22
      npm_client: yarn
      npm_client_production_flag: --frozen-lockfile

  create-tag:
    uses: loash-industries/public-workflows/.github/workflows/js-create-release-tag.yml@v0.1.0
    needs: [validate-code]
    with:
      package_json_path: package.json

  publish:
    uses: loash-industries/public-workflows/.github/workflows/js-publish-npm.yml@v0.1.0
    needs: [create-tag]
    with:
      node_version: 22
      npm_client: yarn
      npm_trusted_publishing: true
```
