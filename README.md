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

### `js-publish-npm.yml`

Build and publish a package to the **public** npmjs.org registry.

```yaml
jobs:
  publish:
    uses: loash-industries/public-workflows/.github/workflows/js-publish-npm.yml@v0.1.0
    with:
      node_version: 22
      npm_client: yarn
    secrets:
      TRINARYEX_NPM_TOKEN: ${{ secrets.TRINARYEX_NPM_TOKEN }}
```

**Inputs:**

| Input | Required | Default | Description |
|---|---|---|---|
| `node_version` | yes | - | Node.js version |
| `npm_client` | no | `yarn` | Package manager |
| `environment` | no | - | GitHub environment for deployment protection |
| `access` | no | `public` | npm access level (`public` or `restricted`) |
| `working_directory` | no | `.` | Working directory |

**Secrets:**

| Secret | Required | Description |
|---|---|---|
| `TRINARYEX_NPM_TOKEN` | yes | npmjs.org authentication token |

---

### `workflow-lint.yml`

Lints workflow YAML files, embedded shell scripts, and validates composite action metadata. Runs automatically on PRs and pushes that modify `.github/workflows/` or `.github/actions/`.

## Composite Actions

### `node-base-actions`

Shared setup action used by the workflows above. Handles Node.js setup, caching, and dependency installation for public repos (no private registry auth needed).

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
    secrets:
      TRINARYEX_NPM_TOKEN: ${{ secrets.TRINARYEX_NPM_TOKEN }}
```
