# snapdragon-build-config

A tiny, **resource-only JAR** that carries the Snapdragon ecosystem's shared
static-analysis rulesets:

```
groupId:    systems.snapdragon
artifactId: snapdragon-build-config
version:    0.3.0
packaging:  jar
```

It contains no Java — only ruleset XML under
`src/main/resources/snapdragon/`:

| File | Tool | Conventions enforced |
|------|------|----------------------|
| `pmd-ruleset.xml`     | PMD       | **A** `FinalFieldCouldBeStatic` (constants → `static final`), **B** `ImmutableField` (write-once fields → `final`) |
| `checkstyle.xml`      | Checkstyle | **F** `VisibilityModifier` (private fields), **G** `FinalClass` + `HideUtilityClassConstructor` + `MutableException` (final/utility/immutable-exception classes), **H** `LambdaBodyLength` (oversized lambda guardrail) |
| `spotbugs-include.xml`| SpotBugs + FindSecBugs | **D** `EI_EXPOSE_REP`/`EI_EXPOSE_REP2`/`MS_EXPOSE_REP`/`MS_MUTABLE_*` (defensive copies / no rep exposure), **E** `MS_SHOULD_BE_FINAL` (no mutable static) |
| `spotbugs-exclude.xml`| SpotBugs | The known-pattern **baseline**: entity/dto/model packages (EI_EXPOSE_REP/EI_EXPOSE_REP2), generated sources, and test classes. Also suppresses EI_EXPOSE_REP2 (but NOT EI_EXPOSE_REP) for Spring-managed component packages (controller/service/client/config/repository/etc.) where constructor-injected beans are not an encapsulation leak. |

## Why a separate artifact (the distribution mechanism)

Static-analysis plugins resolve a `classpath:` ruleset from **their own plugin
classpath**, not the project's compile classpath. So `snapdragon-parent` adds
this jar to each analysis plugin's `<dependencies>` and points the plugin at
`classpath:/snapdragon/<file>.xml`. Because the parent inherits into every Java
repo, every repo gets the same audited rulesets automatically — **no per-repo XML
copy**.

This is kept **separate from `snapdragon-commons`** (the runtime security library)
on purpose: a ruleset tweak must not force a version bump of a runtime artifact,
and build-config must not drag Spring onto any plugin classpath. It is the standard
enterprise "shared build config artifact" pattern (cf. how `google-checks`/
`sun_checks` ship inside the checkstyle jar).

## Conventions mapping (the standard)

The conventions come from the org's secure/clean-Java + reactive-readability
standard. Each item maps to the most precise tool:

- **A** static-final constants → PMD `FinalFieldCouldBeStatic`
- **B** final write-once fields → PMD `ImmutableField`
- **D** defensive copies / no rep exposure → SpotBugs + FindSecBugs `EI_EXPOSE_REP*` / `MS_EXPOSE_REP` / `MS_MUTABLE_*`
- **E** mutable static → SpotBugs `MS_SHOULD_BE_FINAL`
- **F** field visibility → Checkstyle `VisibilityModifier`
- **G** final/utility/immutable-exception classes → Checkstyle `FinalClass` + `HideUtilityClassConstructor` + `MutableException`
- **H** reactive readability → Checkstyle `LambdaBodyLength` (size guardrail only)

**C** (enum for fixed string sets) and the *defensive-copy fix* / *lambda→method-ref*
/ *nesting* halves of D and H have **no reliable low-noise static rule**, so they
stay agent-pass conventions — we deliberately do not add a rule that flags every
lambda or every collection field.

## Rollout: partial fail-mode

Checkstyle now runs in **fail-mode** (severity=error; failOnViolation=true) — the fleet was verified clean before this flip. PMD and SpotBugs still run in **warn-first** mode (report only). See `snapdragon-parent/pom.xml` for the documented one-line flip to fail-mode for PMD/SpotBugs once the fleet is clean.

## Publishing

CI packages and (on the default branch / a tag) deploys the jar to the GitLab
Maven Package Registry via `mvn deploy -s settings.xml`, authenticated with the
GitLab-injected `CI_JOB_TOKEN` — **no secret committed**. There is no GitHub
Actions pipeline; CI is GitLab. For local dev, `mvn install` writes the jar to
`~/.m2` so sibling repos resolve it offline.
