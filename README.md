# renovate-presets

Shared [Renovate](https://docs.renovatebot.com/) configuration presets for `tibuntu` repositories.

Reference them from any repo's `renovate.json` via `github>tibuntu/renovate-presets[:name]`.
`github>tibuntu/renovate-presets` resolves to `default.json`; `:name` resolves to `name.json`.
Parameterized presets take args in parentheses: `:name(arg0, arg1)`.

## Presets

| Preset | Use case | Usage example |
|---|---|---|
| `default` | Baseline every repo should extend: `config:recommended`, dependency dashboard, no rate limiting, `group:recommended`, `renovate` + manager + update-type labels, Europe/Berlin timezone, pre-commit, empty PR footer, semantic commits. | `"extends": ["github>tibuntu/renovate-presets"]` |
| `automerge` | Auto-merge non-major updates (minor/patch/digest/pin/pinDigest/lockFileMaintenance) via PR once required checks pass; majors and 0.x minors stay manual. | `"extends": ["github>tibuntu/renovate-presets:automerge"]` |
| `groupDevDependencies` | Collapse all `devDependencies` bumps into one PR to cut tooling churn. | `"extends": ["github>tibuntu/renovate-presets:groupDevDependencies"]` |
| `semanticCommitType` | Force a commit type for some/all packages. `arg0` = package matcher (`*` = all), `arg1` = commit type. | global: `:semanticCommitType(*, fix)` · one package: `:semanticCommitType(eslint, fix)` |
| `regexAnnotations` | Custom manager for trailing `# renovate: datasource=… depName=… [versioning=…] [registryUrl=…]` comment annotations. `arg0` = file pattern. | `:regexAnnotations(/.+\\.ya?ml$/)` |
| `githubMatrixRunners` | Custom manager that pins/updates GitHub Actions runner versions declared in a build matrix (`strategy.matrix...runner: <os>-<version>`), which the built-in `github-runners` manager misses. | `:githubMatrixRunners` |
| `githubActionsCommitType` | Commit GitHub Actions workflow updates (actions, runner images, workflow Docker images) as `ci(deps):` instead of inheriting a release-triggering `fix:`. Extend **after** `semanticCommitType(*, fix)`. | `"extends": ["github>tibuntu/renovate-presets:githubActionsCommitType"]` |
| `dockerComposeCommitType` | Commit Docker Compose image bumps (service images in `docker-compose*.yml`) as `build(deps):` instead of inheriting a release-triggering `fix:`. Extend **after** `semanticCommitType(*, fix)`. | `"extends": ["github>tibuntu/renovate-presets:dockerComposeCommitType"]` |
| `devDependenciesCommitType` | Commit `devDependencies` bumps as `chore(deps):` instead of inheriting a release-triggering `fix:`. Extend **after** `semanticCommitType(*, fix)`. | `"extends": ["github>tibuntu/renovate-presets:devDependenciesCommitType"]` |

### `default`

**Use case:** the foundation every repo should extend. On top of `config:recommended` it enables the
dependency dashboard, disables rate limiting, applies `group:recommended`, labels every PR with
`renovate` plus the manager and update type, sets the Europe/Berlin timezone, enables the pre-commit
manager, clears the PR footer and turns on semantic commit messages. Rebasing stays on Renovate's
`auto` default: automerged branches rebase when behind, everything else only when conflicted. It
intentionally sets **no** `packageRules`; compose the other presets below for behaviour like
automerge or grouping.

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>tibuntu/renovate-presets"]
}
```

### `automerge`

**Use case:** opt in to hands-off merging of low-risk updates. Minor, patch, digest, pin and
lock-file-maintenance updates are auto-merged through a PR after required CI passes; major bumps
and minor bumps of `0.x` packages (breaking by semver convention) still require manual review.
Repos that prefer to merge everything by hand simply don't extend it.

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

**Use case:** track versions that no built-in manager understands by annotating them inline with a
trailing `# renovate: datasource=… depName=… [versioning=…] [registryUrl=…]` comment on the same
line as the version, e.g. `kubectl: 1.29.0 # renovate: datasource=github-releases depName=kubernetes/kubernetes`.
`versioning` defaults to the datasource's default. `arg0` is the file pattern to
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
(ubuntu/macos/windows) and treats it as a `github-runners` dep. A suffix such as `-arm`, `-large` or
`-xlarge` is part of the version, so only releases with the same suffix are offered
(`macos-14-large` → `macos-15-large`, never `macos-26-large`).
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

### `dockerComposeCommitType`

**Use case:** repos that force every bump to `fix:` via `semanticCommitType(*, fix)` (so app
dependency bumps trigger a semantic-release patch) don't want their **Docker Compose** image bumps
doing the same — the Postgres/Caddy/etc. service images in a `docker-compose*.yml` are local or
reference infrastructure, not the shipped product, so bumping them shouldn't cut a release. This
preset re-types every dependency the built-in `docker-compose` manager extracts to `build`, yielding
`build(deps): update postgres docker tag to v18`. Under semantic-release's Angular preset, `build`
does **not** trigger a release.

It deliberately covers only the `docker-compose` manager. Dockerfile base images (the `dockerfile`
manager) are left on `fix:` because they are the shipped runtime, and workflow images are handled by
`githubActionsCommitType` (the `github-actions` manager) — all three managers are disjoint, so the
presets don't fight over the same dependency.

> **Ordering is required.** Renovate has no rule specificity — `packageRules` are evaluated in
> `extends` order and the **last** matching rule wins. List this preset **after**
> `semanticCommitType(*, fix)` so its rules override the global `*` → `fix` rule.

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "github>tibuntu/renovate-presets",
    "github>tibuntu/renovate-presets:semanticCommitType(*, fix)",
    "github>tibuntu/renovate-presets:dockerComposeCommitType"
  ]
}
```

### `devDependenciesCommitType`

**Use case:** repos that force every bump to `fix:` via `semanticCommitType(*, fix)` don't want
`devDependencies` bumps (linters, test runners, build tooling) to cut a release — dev tooling is not
the shipped product. This preset re-types every `devDependencies` update to `chore`, yielding
`chore(deps): update dependency eslint to v9`. Under semantic-release's Angular preset, `chore` does
**not** trigger a release. Combine with `groupDevDependencies` to land them as one `chore(deps)` PR.

> **Ordering is required.** List this preset **after** `semanticCommitType(*, fix)`; the last
> matching `packageRule` wins.

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "github>tibuntu/renovate-presets",
    "github>tibuntu/renovate-presets:semanticCommitType(*, fix)",
    "github>tibuntu/renovate-presets:devDependenciesCommitType"
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
    "github>tibuntu/renovate-presets:devDependenciesCommitType",
    "github>tibuntu/renovate-presets:githubActionsCommitType",
    "github>tibuntu/renovate-presets:dockerComposeCommitType"
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
