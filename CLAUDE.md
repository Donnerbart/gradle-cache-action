# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A GitHub Action that caches the Gradle User Home, written in Kotlin/JS and shipped as a webpack
bundle. Fork of `burrunan/gradle-cache-action`.

## Build and test

```bash
./gradlew --no-parallel --no-daemon --build-cache build   # what CI runs
./gradlew build                                           # local equivalent

./gradlew :layered-cache:jsNodeTest                       # library module tests (node + mocha)
./gradlew :cache-action-entrypoint:jsBrowserTest          # entrypoint tests (karma + webpack)
./gradlew allTests                                        # everything
./gradlew :layered-cache:jsNodeTest --tests '*LayeredCache*'
```

Library modules test on Node, `cache-action-entrypoint` tests in a browser. That split is configured
in the root `build.gradle.kts`, keyed on the module name suffix `-entrypoint`.

CI builds on Java 17. The Node version comes from the Kotlin Gradle plugin default and is downloaded
into the Gradle user home.

`checkKotlinAbi` and `updateKotlinAbi` exist on every module. No reference dumps are committed.

## Release mechanics

`dist/` is committed on `main`, and tags point at commits on `main`. Since `main` is never rewritten,
a pinned commit SHA stays resolvable, which is what a `uses:` reference needs.

Cut a release with `./release.sh v1.2.3`. It validates everything before touching the tree: the tag
is well formed, unused locally and on origin, the checkout is on `main`, and the working tree is
clean. Then it builds, copies the bundle into `dist/`, commits only when the bundle changed, and
creates an annotated tag. Nothing is pushed.

`release.yml` fires on a `v#.#.#` tag. It rebuilds, fails the release when the committed `dist/` does
not match a fresh build, then creates the GitHub release and moves the major tag.

The bundle is byte-reproducible for a given toolchain, which is what makes that check dependable. It
does change when Gradle, the Kotlin plugin or a build script changes, so those updates have to carry
a rebuilt `dist/`.

The source map is deliberately not committed. The runner downloads the repository at the pinned ref
on every job that uses the action.

## Architecture

`action.yml` declares the same bundle for both `main:` and `post:`. `main.kt` in
`cache-action-entrypoint` dispatches on `ActionStage`, so one binary serves restore and save.

`GradleCacheAction.execute(stage)` (`layered-cache/.../gradle/GradleCacheAction.kt`) is the
orchestrator. It assembles a list of `Cache` instances from the action inputs, wraps them in a
`CompositeCache`, then calls `restore()` on `MAIN` and `save()` on `POST`. `read-only` short-circuits
the save.

### Cache implementations

All live in `layered-cache/src/jsMain/kotlin/com/github/burrunan/gradle/cache/` and are built on
`LayeredCache`:

| Implementation | Caches |
| -- | -- |
| `GradleGeneratedJarsCache` | `caches/*/generated-gradle-jars`, keyed on the Gradle version alone |
| `localBuildCache` | `caches/build-cache-1` |
| `dependenciesCache` | `caches/modules-2` (gradle) and `~/.m2/repository` (maven) |

### The layering model

`LayeredCache` holds a `baseline` and a `primaryKey`. `isBaseline` is
`primaryKey.startsWith(baseline)`. On save:

- no existing index and not baseline: refuses, logs `cache saving can't be done`
- no existing index and baseline: writes a single-layer snapshot
- existing index: writes a delta layer on top, up to `maxLayers`

On restore, layers are applied oldest to newest so later layers overwrite earlier files.

The two cache families choose baselines differently, which is the single most important thing to know
in this codebase:

- `dependenciesCache` uses the bare key prefix, so a build on any branch can create the first snapshot.
- `localBuildCache` uses `"$prefix-$defaultBranch"`, so only a default-branch build can. Pull requests
  can never seed it, and until something seeds it no pull request saves a build cache at all.

`ActionsTrigger.cacheKey` (`ActionsTriggerExtensions.kt`) supplies the per-trigger key segment: a pull
request yields `PR<number>`, a push to the default branch yields `defaultbranch`, a push elsewhere
yields the branch name, and `schedule` and `workflow_dispatch` both yield `defaultbranch` regardless
of the ref they run on.

### Supporting modules

- `cache-proxy` implements Gradle's remote build cache HTTP API backed by the GitHub Actions cache,
  enabled by `remote-build-cache-proxy-enabled`.
- `gradle-launcher` resolves and downloads the Gradle distribution and supplies the version used in
  cache keys.
- `hashing` computes the file hashes behind dependency cache keys.
- `wrappers/*` are hand-written Kotlin externals for `@actions/*`, octokit and Node APIs.
- `cache-service-mock` and `test-library` provide the test harness.
