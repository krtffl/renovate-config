# krtffl/renovate-config

Shared Renovate presets for every repo under the `krtffl` owner. Polyrepo: each repo carries a
one-line `renovate.json` that points here, and policy changes happen in this repo only.

Validated against Renovate `44.12.0` (`renovate-config-validator --strict`) and against
`renovate-schema.json` `x-renovate-version 44.11.8`.

## Files

| Preset | Reference | Purpose |
| --- | --- | --- |
| `default.json` | `github>krtffl/renovate-config` | Org baseline. Cadence, commit style, grouping, rate limits, security fast lane. Extends the six manager fragments. |
| `npm.json` | `…:npm` | npm. Apps not libraries, so ranges are bumped. Astro and Cloudflare toolchains grouped. |
| `gomod.json` | `…:gomod` | Go modules. Indirect deps security-only, `gomodTidy`. |
| `cargo.json` | `…:cargo` | Cargo workspaces. One PR per workspace. |
| `python.json` | `…:python` | pep621 / uv / requirements.txt. |
| `github-actions.json` | `…:github-actions` | SHA digest pinning for every `uses:`. |
| `docker.json` | `…:docker` | Dockerfile and compose. Digest pinning, base-image majors held. |
| `automerge.json` | `…:automerge` | Tier A. Repos with a real test suite on `pull_request`. |
| `automerge-ci.json` | `…:automerge-ci` | Tier A-narrow. Repos whose PR gate is a build or lint, not tests. |
| `manual.json` | `…:manual` | Tier C. Nothing merges without a human. |
| `dormant.json` | `…:dormant` | Tier D. Security updates only. |

The manager fragments are extended by `default.json`, so a repo never references them directly.
A Go-only repo still loads `npm.json`; its rules simply never match.

## What each repo gets

```json
{"$schema":"https://docs.renovatebot.com/renovate-schema.json","extends":["github>krtffl/renovate-config"]}
```

Tiered repos swap the single `extends` entry for `…:automerge`, `…:automerge-ci`, `…:manual` or
`…:dormant`. Each tier preset extends the baseline itself, so there is never more than one entry.

Presets are intentionally **not** pinned to a tag. A policy change here reaches all 35 repos on the
next run. To freeze a repo during a migration, pin it: `github>krtffl/renovate-config#v1`.

## Cadence

Routine updates run Mondays 04:00–07:59 Europe/Madrid, capped at 3 concurrent PRs and 2 per hour
per repo. Lockfile refreshes run on the first Monday of the month.

Security fixes ignore all of that. Renovate documents that a `vulnerabilityAlerts` PR bypasses
`branchConcurrentLimit`, `commitHourlyLimit`, `prConcurrentLimit`, `prHourlyLimit` and `schedule` —
they "skip the line". The org-wide 3-day release quarantine is also nulled for them, so a fix is not
held back by the malware-detection delay that applies to routine bumps.

Two independent feeds drive that lane:

- **GitHub Dependabot alerts** — requires the dependency graph and Dependabot alerts enabled per
  repo, plus `read` access to Dependabot alerts on the Renovate app installation.
- **`osvVulnerabilityAlerts`** — osv.dev, queried offline, **direct dependencies only**, and only
  for the crate / go / npm / pypi (and a few other) datasources. It is a backstop, not a
  replacement: it will not see the transitive `brace-expansion` and `undici` findings that GitHub
  surfaces today.

### What neither feed fixes: transitive npm dependencies

A GitHub alert only becomes a PR if Renovate has *extracted* the vulnerable package, and the npm
manager extracts from `package.json` alone — `dependencies`, `devDependencies`, `overrides`. A
package that exists only in `package-lock.json` is never a Renovate dependency, so its alert
matches nothing and no PR is raised, silently. Measured 2026-09-25: roma-aeterna's alert for
`devalue` (pulled in by astro) sat open a week with no PR, and a `--dry-run=extract` of that
commit listed 17 npm dependencies, none of them devalue. yodidac had nine such alerts.

- Anything pinned through `overrides` **is** covered — Renovate reads overrides as dependencies —
  but only inside the major the override pins; see the next section. The exception is an override
  whose own key carries a version selector (`"undici@7.29.0": "7.29.1"`): Renovate skips it as
  `invalid-name`, so it gets no update of any kind; a selector on a parent key
  (`"miniflare@5.x": { "undici": … }`) is extracted normally.
- `transitiveRemediation` is not the answer: it was removed in Renovate v38 and now fails
  `renovate-config-validator --strict`.
- Upstream is building it (renovatebot/renovate#46377, #46391): lockfile-only fixes for
  transitive alerts under `rangeStrategy: update-lockfile`, which `vulnerabilityAlerts` here
  already sets. Re-check when it ships; it may land behind a new option.

Until then, fix a transitive alert by hand: `npm update <pkg>` when the parent's range allows the
patched version, or a scoped `overrides` entry when the parent pins it exactly. Ticking "lock file
maintenance" on the repo's Dependency Dashboard covers the first case.

Keeping Dependabot *alerts* on is what makes the fast lane work at all. It is a different feature
from Dependabot *security updates* and Dependabot *version updates*, both of which must be off — see
the runbook.

### Overrides never move to a later major

An `overrides` entry pins a transitive dependency on behalf of a parent that needs that major, so
the next major of an override is never a valid update — the parent has to move first. `npm.json`
drops every override update whose new major is above the one written in the override. Updates
inside the override's major still arrive, security ones included, and ordinary dependencies are
untouched.

Measured 2026-10-02: yodidac #142 rewrote the `miniflare` > `undici` override from `^7.29.1` to
`^8.0.0` as a `[SECURITY]` fix and went red at `npm ci`: the PR dropped undici 7.29.1 from the
lockfile and added no 8.x entry for miniflare, so npm refused the lockfile as out of sync
(`Missing: undici@8.11.2 from lock file`) and lint, build and tests were skipped. The override was
already patched. A second, 8.x copy of undici elsewhere in the lockfile still had open alerts, and
on the GitHub path Renovate folds every alert for a package into one rule that asks for the
**highest** first-patched version (`8.10.2`) and applies it to every copy below it.

Two things about the rule look odd and are deliberate. Both were measured with
`--platform=local --dry-run=lookup` on Renovate `44.127.0`, not assumed:

- **It sits under the `npm` manager key, not in `packageRules`.** An alert becomes a synthetic rule
  appended after the top-level `packageRules` with `enabled: true` in its `force` key, so the same
  rule in the top-level list stops routine override majors and is switched back on for security
  ones, on both feeds. Manager-level rules are evaluated after the synthetic one.
- **It compares majors itself instead of using `matchUpdateTypes: ["major"]`.** Renovate reads an
  override's locked version from the top-level `node_modules/<name>` lockfile entry, whichever
  parent the override is scoped to. la-ruina pins `miniflare` > `undici` at `7.29.1` while an
  unrelated undici `8.11.0` is hoisted to the top level, so Renovate labels `7.29.1 → 8.11.2` a
  *patch* — bound for the weekly `npm non-major` PR — and `7.29.1 → 7.30.0` a *major*.
  `matchUpdateTypes` dropped the safe one and kept the breaking one.

| Override update | Before | Same rule in `packageRules` | Rule under `npm` |
| --- | --- | --- | --- |
| to a later major, routine | raised | dropped | dropped |
| to a later major, GitHub alert | raised | **raised** | dropped |
| to a later major, OSV alert | raised | **raised** | dropped |
| inside its major, routine or security | raised | raised | raised |

What it costs:

- The ordering is Renovate internals, not a documented contract, and the validator cannot see it
  break. Re-test on a Renovate major upgrade.
- A repo's own `packageRules` cannot undo it, for the same reason. An override major that is really
  wanted is re-enabled under that repo's `npm.packageRules`, or simply edited by hand.
- **The gap: no config can make the GitHub path offer the in-major fix.** While an alert for a
  *later* major of the same package is open, Renovate asks for that major's fix for the override
  too, so the override gets no PR at all — not even the in-major fix it may itself need. The
  Dependabot alert stays open and is the signal: fix the override by hand. Once only in-major
  alerts remain, the in-major security PR is raised as usual.
- The mislabelling is still there for what does get through: in the la-ruina shape the in-major
  update arrives labelled `major`, and in `…:manual` it waits on the dashboard like one.
- A 0.x minor of an override (`fflate ^0.7.5 → ^0.8.3`) is not a major. It is still proposed,
  labelled `breaking-0x`.
- The major is read as the first number in the override's value, so a range written only as an
  exclusive upper bound on the next major is not protected: for `<8` that number is 8, and an
  update to 8.x is not dropped (`<=7` is protected).

## Grouping

`separateMajorMinor` is true and `separateMinorPatch` is false, so minor and patch travel together
and majors are always split out.

- Non-major, per manager: one PR each — `npm non-major`, `go non-major`, `cargo non-major`,
  `python non-major`, `github actions`, `docker non-major`.
- Digest-only refreshes get their own PRs (`github actions digests`, `docker digests`) so a
  mechanical SHA churn never hides a version change.
- Majors are ungrouped, labelled `major`, and quarantined 7 days.
- **0.x minors are ungrouped and labelled `breaking-0x`.** Under semver a `0.x` minor may break, and
  Renovate classifies it as `minor`. This matters here: most of the Rust crates in `logmole`,
  `stintlab`, `tablero` and `mcp-servers` are pre-1.0.

`config:recommended` brings `group:monorepos`, which keeps things like the Astro or AWS SDK families
together across the grouping above.

## Commit messages

`semanticCommits` is `enabled` explicitly rather than left on `auto`, because auto-detection samples
recent commits and would silently disable itself on the dormant repos.

`config:recommended` supplies `:semanticPrefixFixDepsChoreOthers`, giving `fix(deps):` for runtime
dependencies and `chore(deps):` for everything else. GitHub Actions updates are overridden to
`ci(deps):`.

## Automerge policy

Automerge is unsafe without a pull-request gate, and the failure mode is not the one people assume.
Renovate refuses to automerge until it sees a passing branch status. A commit with no check runs
returns combined state `pending` from GitHub — verified directly against
`repos/krtffl/logmole/commits/{sha}/status`, which returns `{"state":"pending","total_count":0}` —
and Renovate maps that to yellow. So a repo without PR CI does not silently automerge; it
accumulates branches that never merge. That is still a bad outcome, and it is why the tier is
assigned per repo rather than globally.

`ignoreTests` is `false` everywhere and must stay that way. Setting it to `true` is the one switch
that turns "no CI" from "stalls" into "merges blind".

### Repos that MAY automerge

Preconditions, all three required:

1. a workflow with a `pull_request:` trigger, so a Renovate branch produces check runs;
2. that workflow runs a gate that a bad dependency would actually fail;
3. the gate is green on the default branch today, so a red PR means the update and not the backlog.

**Tier A — `…:automerge`** (real test suite, green today):

| Repo | Gate | HEAD checks |
| --- | --- | --- |
| `mojodojo` | `make format`, `make unit-test` | 6/6 pass |
| `torrons` | `go build`, `go vet`, `go test ./...` | 3/3 pass |
| `fontes-antiquae` | `cargo fmt`, `clippy -D warnings`, `cargo nextest`, `cargo-deny`, release build | 5/5 pass |

**Tier A-narrow — `…:automerge-ci`** (build or lint gate, no tests, green today):

| Repo | Gate | HEAD checks |
| --- | --- | --- |
| `roma-aeterna` | `npm run check`, `docs:check-links`, `npm run build` | 3/3 pass |
| `krtffl.dev` | `hugo --gc --minify --printPathWarnings` | 3/3 pass |
| `torrons-infra` | `check-pins` (every `uses:` SHA-pinned), shell/Dockerfile lint | 2/2 pass |

Scope in Tier A: patch updates, lockfile maintenance, CI action versions and digests, and dev-only
npm/cargo dependencies. Scope in Tier A-narrow: CI actions and dev-only npm packages only.
Excluded from both, always: majors, 0.x minors, every security fix, and **everything docker,
digests included** — no repo here builds its image on a `pull_request` (`mojodojo` and `casahouse`
both gate `build-and-push` on the push/tag event), so a green PR says nothing about a new base layer.

### Repos that MUST NOT automerge

**No `pull_request` trigger — 21 repos.** Renovate would open branches that never go green:

- `la-ruina` — `deploy.yml` is `push: [main]`, `data-refresh.yml` is a cron.
- `nadie-sabe-nada` — `rebuild.yml` is `push: [main]` + cron + dispatch.
- `llobrekatzen` — **archived**; Renovate cannot open a PR at all. Excluded from the rollout.
- `krtffl` — no workflows.
- The 17 with no CI whatsoever: `web-blueprint`, `portfolio-mailer`, `mojodojo.club`, `dotfiles`,
  `logmole`, `salesfly`, `mojodojo-vps`, `potatoblog`, `dx`, `checking-automata`, `poop`,
  `travelling-f1`, `javascript`, `go-mixtape-trading`, `remindf1`, `playground`, `gws`.
  (`gws` has `build.yaml`, but it triggers only on `push` to `main`.)

**PR CI exists but is failing on the default branch — 9 repos.** Automerge is meaningless while the
baseline is red: every Renovate PR inherits the failure, so nothing merges and the red signal stops
carrying information. These start on `…:manual` and get promoted once CI is green:

| Repo | HEAD checks | Gate it will have once fixed |
| --- | --- | --- |
| `casahouse` | 3 of 8 failing | lint, `tsc --noEmit`, `npm run test` → Tier A |
| `yodidac` | 1 of 4 failing | lint, build, unit + API + API-e2e tests → Tier A |
| `didacperezescrich` | 5 of 7 failing | lint, check, build, `npm run test` → Tier A |
| `posteguillo_imperator` | 5 of 8 failing | `lint:tokens`, build → Tier A-narrow |
| `tender` | 4 of 6 failing | Go lint + tests → Tier A |
| `tablero` | 2 of 2 failing | fmt, clippy, `cargo test`, `cargo audit` → Tier A |
| `stintlab` | 2 of 2 failing | fmt, clippy, `cargo test`, wasm build → Tier A |
| `anomalog` | 3 of 3 failing | ruff, pytest, cargo fmt/clippy → Tier A |
| `mcp-servers` | 1 of 1 failing | fmt, clippy, `cargo test` → Tier A |

**Excluded from the rollout entirely — 2 repos:** `Fast-F1` (a fork of an upstream project;
managing its dependencies means diverging from upstream) and `llobrekatzen` (archived).

### Hardening that would widen the policy

`platformAutomerge` is off because GitHub only offers native auto-merge when a PR is not already
mergeable — meaning at least one required check or review. Private repos on this account cannot have
branch protection or rulesets (the API returns *"Upgrade to GitHub Pro or make this repository
public"*), and the five public repos have `rulesets: 0` and `Branch not protected`.

The cheap win: the public repos (`torrons`, `tablero`, `stintlab`, `anomalog`, `mcp-servers`) can
have a ruleset requiring the CI check on `main` for free. Add one, then flip `platformAutomerge` to
`true` for those repos and GitHub — not Renovate — enforces the gate.

Separately, `allow_auto_merge` is `false` on every repo checked; it must be turned on before native
auto-merge can ever be used.

## Validating a change to this repo

```sh
npx --yes --package renovate renovate-config-validator --strict
```

Run it from a directory containing the file renamed to `renovate.json`; the validator treats other
filenames as *global* config and applies a different, weaker ruleset. It catches unknown option
names, including inside `packageRules` — but it does **not** verify that a preset named in `extends`
exists, so a typo in `security:…` passes silently. `validate.py` in the phase-2 workspace checks
preset existence against the installed `renovate` package as a second pass.
