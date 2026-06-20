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

## Composing presets

A realistic config stacks the baseline with the building blocks it needs:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "github>tibuntu/renovate-presets",
    "github>tibuntu/renovate-presets:automerge",
    "github>tibuntu/renovate-presets:groupDevDependencies",
    "github>tibuntu/renovate-presets:semanticCommitType(*, fix)"
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
