# renovate-presets

Shared [Renovate](https://docs.renovatebot.com/) configuration presets for `tibuntu` repositories.

Reference them from any repo's `renovate.json` via `github>tibuntu/renovate-presets[:name]`.
`github>tibuntu/renovate-presets` resolves to `default.json`; `:name` resolves to `name.json`.
Parameterized presets take args in parentheses: `:name(arg0, arg1)`.

## Presets

| Preset | Use case | Usage example |
|---|---|---|
| `default` | Baseline every repo should extend: `config:recommended` + empty PR footer + semantic commits enabled. | `"extends": ["github>tibuntu/renovate-presets"]` |
| `automerge` | Auto-merge non-major updates (minor/patch/digest/pin/pinDigest/lockFileMaintenance) via PR once required checks pass; majors stay manual. | `"extends": ["github>tibuntu/renovate-presets:automerge"]` |
| `groupDevDependencies` | Collapse all `devDependencies` bumps into one PR to cut tooling churn. | `"extends": ["github>tibuntu/renovate-presets:groupDevDependencies"]` |
| `semanticCommitType` | Force a commit type for some/all packages. `arg0` = package matcher (`*` = all), `arg1` = commit type. | global: `:semanticCommitType(*, fix)` · one package: `:semanticCommitType(eslint, fix)` |
| `regexAnnotations` | Custom manager for `# renovate: datasource=… depName=… versioning=… [registryUrl=…]` comment annotations. `arg0` = file pattern. | `:regexAnnotations(/.+\\.ya?ml$/)` |
| `githubMatrixRunners` | Custom manager that pins/updates GitHub Actions runner versions declared in a build matrix (`strategy.matrix...runner: <os>-<version>`), which the built-in `github-runners` manager misses. | `:githubMatrixRunners` |
| `githubActionsCommitType` | Commit GitHub Actions workflow updates (actions, runner images, workflow Docker images) as `ci(deps):` instead of inheriting a release-triggering `fix:`. Extend **after** `semanticCommitType(*, fix)`. | `"extends": ["github>tibuntu/renovate-presets:githubActionsCommitType"]` |

### `default`

**Use case:** the foundation every repo should extend — enables Renovate's `config:recommended`,
clears the PR footer, and turns on semantic commit messages. It intentionally sets **no**
`packageRules`; compose the other presets below for behaviour like automerge or grouping.

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>tibuntu/renovate-presets"]
}
```

### `automerge`

**Use case:** opt in to hands-off merging of low-risk updates. Minor, patch, digest, pin and
lock-file-maintenance updates are auto-merged through a PR after required CI passes; major bumps
still require manual review. Repos that prefer to merge everything by hand simply don't extend it.

```json
{
  "extends": ["github>tibuntu/renovate-presets", "github>tibuntu/renovate-presets:automerge"]
}
```

### `groupDevDependencies`

**Use case:** for repos with many `devDependencies`, batch all of their updates into a single
"dev dependencies" PR instead of one PR per package.

```json
{
  "extends": ["github>tibuntu/renovate-presets", "github>tibuntu/renovate-presets:groupDevDependencies"]
}
```

### `semanticCommitType`

**Use case:** override the semantic commit *type* for matched packages — e.g. so dependency bumps
land as `fix:` (and trigger a release) or `build:`. `arg0` is the package matcher (`*` for every
package), `arg1` is the commit type.

```json
{
  "extends": [
    "github>tibuntu/renovate-presets",
    "github>tibuntu/renovate-presets:semanticCommitType(*, fix)"
  ]
}
```

Scope it to a single package instead:

```json
{
  "extends": ["github>tibuntu/renovate-presets:semanticCommitType(eslint, fix)"]
}
```

### `regexAnnotations`

**Use case:** track versions that no built-in manager understands by annotating them inline with
`# renovate: datasource=… depName=… versioning=… [registryUrl=…]`. `arg0` is the file pattern to
scan (a single regex such as `/.+\\.ya?ml$/` already covers both `.yaml` and `.yml`). To scan more
than one distinct pattern, extend the preset more than once with different args.

```json
{
  "extends": [
    "github>tibuntu/renovate-presets",
    "github>tibuntu/renovate-presets:regexAnnotations(/.+\\.ya?ml$/)"
  ]
}
```

### `githubMatrixRunners`

**Use case:** keep GitHub Actions runner images pinned *and* auto-updated when they're chosen via a
build matrix (e.g. a multi-arch job whose `runs-on: ${{ matrix.runner }}` reads from
`strategy.matrix.include[].runner: ubuntu-24.04` / `ubuntu-24.04-arm`). Renovate's built-in
`github-runners` manager only reads literal `runs-on:` values, so matrix-driven runners are
invisible to it. This custom manager scans `.github/workflows/*.y{a,}ml` for `runner: <os>-<version>`
(ubuntu/macos/windows, with an optional `-arm` suffix) and treats it as a `github-runners` dep.
Takes no args — matrix runners only ever live in workflow files.

```json
{
  "extends": [
    "github>tibuntu/renovate-presets",
    "github>tibuntu/renovate-presets:githubMatrixRunners"
  ]
}
```

### `githubActionsCommitType`

**Use case:** repos that force every bump to `fix:` via `semanticCommitType(*, fix)` (so app
dependency bumps trigger a semantic-release patch) don't want their **CI** updates doing the same —
bumping `actions/checkout`, a runner image, or a workflow's container image doesn't change the
released product. This preset re-types every dependency in `.github/workflows` to `ci`, yielding
`ci(deps): update actions/checkout action to v5`. Under semantic-release's Angular preset, `ci`
does **not** trigger a release.

It covers everything the built-in `github-actions` manager extracts (action `uses:`, runner images
in `runs-on:`, Docker images in `container:`/`services:`/`uses: docker://`) plus runner-image bumps
surfaced by the `githubMatrixRunners` custom manager (matched via the shared `github-runners`
datasource).

> **Ordering is required.** Renovate has no rule specificity — `packageRules` are evaluated in
> `extends` order and the **last** matching rule wins. List this preset **after**
> `semanticCommitType(*, fix)` so its rules override the global `*` → `fix` rule.

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "github>tibuntu/renovate-presets",
    "github>tibuntu/renovate-presets:semanticCommitType(*, fix)",
    "github>tibuntu/renovate-presets:githubActionsCommitType"
  ]
}
```

## Composing presets

A realistic config stacks the baseline with the building blocks it needs:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "github>tibuntu/renovate-presets",
    "github>tibuntu/renovate-presets:automerge",
    "github>tibuntu/renovate-presets:groupDevDependencies",
    "github>tibuntu/renovate-presets:semanticCommitType(*, fix)",
    "github>tibuntu/renovate-presets:githubActionsCommitType"
  ],
  "packageRules": [
    { "matchPackageNames": ["some-fragile-dep"], "enabled": false }
  ]
}
```

> **Shorthand vs. full reference:** inside a consuming repo you may write the shorthand `:name`
> only after a `github>tibuntu/renovate-presets` entry resolves the repo. When one preset references
> another *within this repo*, always use the full `github>tibuntu/renovate-presets:name` form —
> `local>` would resolve against the repo being renovated, not this one.
