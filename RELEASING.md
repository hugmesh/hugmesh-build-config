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
(hugmesh-generator#347): *"B for the mechanism, A prepared"*.

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
- **Whether anything is already published is unknown, not "nothing".** Confirming needs a
  token with `read:packages`; this box's has none
  (`gh api /orgs/hugmesh/packages?package_type=maven` → HTTP 403).
