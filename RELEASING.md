# Releasing hugmesh-build-config

`net.hugmesh:hugmesh-build-config` is **link 1 of 4** in the org's Maven publish chain:

```
hugmesh-build-config 0.5.0  ->  hugmesh-parent 2.8.0  ->  hugmesh-commons 2.1.0  ->  hugmesh-auth-core 2.5.1
```

It is first because it is the only one of the four with **no `net.hugmesh` upstream at
all** — `pom.xml` declares no `<parent>` and no `<dependencies>`, only a pinned
`maven-jar-plugin`. Maven Central is a sufficient repository list for it. Everything after
it needs this jar: `hugmesh-parent` puts it on the `maven-pmd-plugin` /
`maven-checkstyle-plugin` / `spotbugs-maven-plugin` classpaths, so **every Java build in the
org fails at `verify` until this artifact resolves.**

The full story — what a release IS in this org, the CI-only resolution route the other three
links carry, and the operator's launch-day click list — is in
**`hugmesh-auth-core/RELEASING.md`**. Written 2026-09-17 against operator decision 27
(hugmesh-generator#347): *"B for the mechanism, A prepared"*. Decision 27's letters are its
own: its A is "publish the chain to GitHub Packages", unrelated to the read "route A" below.

## Why this repo has no `.github/maven-ci-settings.xml`

The other three links each ship one, declaring the GitHub Packages URLs of the links before
them, because `maven.pkg.github.com` needs authentication even to **read** and therefore
cannot go in a pom. This repo resolves nothing from there. A settings file here would be
decoration, and decoration in a release path is a thing future readers trust.

The deploy credential is not decoration and is already handled: `actions/setup-java`'s
`server-id: github` writes a `<server>` into the generated `~/.m2/settings.xml`, and since
this workflow passes no `-s`, that generated file is the one Maven reads. (The other three
DO pass `-s`, which *replaces* the user settings file — which is why they each carry a
`<server><id>github</id>` of their own.)

## This is the only link that needs no `PACKAGES_READ_TOKEN`

Links 2–4 each READ packages published from another repository, and
`secrets.GITHUB_TOKEN` is scoped to the repository whose workflow is running — so those
builds read their upstreams with **`PACKAGES_READ_TOKEN`**: one organisation secret holding a
PAT with `read:packages`, separate from `RELEASE_CHAIN_TOKEN`. Their `release.yml` already
prefers it and falls back to `GITHUB_TOKEN` only when it is absent. That is **the** route,
and it is **kept** as a standing secret — not a launch-day stopgap to be retired afterwards
(operator decision 90a = B, hugmesh-generator#347, answered 2026-09-23). Going public does
**not** create that secret; the operator does. See `hugmesh-auth-core/RELEASING.md` §6.1
blocker 4 and §6.2 step 2.

This repo resolves no `net.hugmesh` artifact at all, so none of it applies here — which is
the same asymmetry as the missing settings file above, and it is worth stating out loud so
that nobody "aligns" this workflow with its three siblings by adding a `PACKAGES_READ_TOKEN`
it has no use for. That matters more once the org secret exists, because unless its
repository access is narrowed it will be *visible* to this repo's workflow too. What this
repo *is*, in the read picture, is the **first thing read**: every downstream link puts its
ruleset jar on an analyser plugin classpath, so on launch day `hugmesh-parent`,
`hugmesh-commons` and `hugmesh-auth-core` all read this package with that one secret.

Per-package *Manage Actions access* grants — "route A", which this file described before
2026-09-23 — are **not the plan**. Where another link's `RELEASING.md`, or a `release.yml`
preflight error message, still offers route A as an option, that text predates decision 90a.
GitHub's documentation lists that setting only for container, npm, NuGet and RubyGems
packages and says Maven packages inherit their repository's permissions, so it is probably
not offered for this package at all. That is read from the docs, not tested — no
`net.hugmesh` package is known to have been published (see the last bullet below).

## Publishing

`workflow_dispatch` only, on `Release`. Never on push or tag: republishing over an
already-published, immutable coordinate is what forced `hugmesh-auth-core`'s 2.4.2 re-pin.
Bump `<version>` in the same PR as any ruleset change — a ruleset change published at the
same coordinate silently changes what every repo in the fleet is analysed against.

`project.build.outputTimestamp` is pinned in `pom.xml`, so the jar is reproducible: the same
source produces the same hash. **Bump it to the release date when you bump the version**, or
two different rulesets ship with the same normalized timestamp.

## What a human must still verify

- **This workflow has never run.** `gh run list --repo hugmesh/hugmesh-build-config` → empty.
- **No Actions job in this org starts at all today.** The annotation on the most recent runs
  across the org is *"The job was not started because recent account payments have failed or
  your spending limit needs to be increased."* Private-repo Actions minutes are billed;
  public-repo minutes are free, so this dissolves when the repos go public.
  `hugmesh-auth-core/RELEASING.md` §6.1 has the exact `gh api` calls that measured it.
- **There is no automated guard on this workflow.** `hugmesh-auth-core` pins its own release
  workflow's shape with `ReleaseMetadataGuardTest` (9 checks, mutation-tested). This repo has
  no test framework on its classpath — `pom.xml` has no `<dependencies>` by design — so the
  equivalent pin would have to live in `hugmesh-generator`'s shell suite, outside the file
  set this change was allowed to touch. **Until that exists, the only thing stopping this
  workflow gaining a `push` trigger is review.**
- **Whether `PACKAGES_READ_TOKEN` exists is not verified from here.** Creating it is
  operator-owned (decision 90a); listing org secrets needs org-admin and this box cannot
  confirm it. It does not gate this repo's own publish, only links 2–4.
- **Whether anything is already published is unknown, not "nothing".** Confirming needs a
  token with `read:packages`; this box's has none
  (`gh api /orgs/hugmesh/packages?package_type=maven` → HTTP 403).
