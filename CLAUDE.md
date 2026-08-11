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

CI builds on Java 17. The Kotlin/JS toolchain pins Node 22.0.0 in the root build script.

`checkKotlinAbi` and `updateKotlinAbi` exist on every module. No reference dumps are committed.

## Release mechanics

`main` does not contain `dist/`. The compiled bundle lives only on the `release` branch and on `v*`
tags. `.github/workflows/main.yml` builds, then on pushes to `main` copies
`cache-action-entrypoint/build/dist/js/productionExecutable/cache-action-entrypoint.js*` into `dist/`
on the `release` branch, commits `Publish release from <sha>`, and **amends and force-pushes** that
branch.

Consequences worth knowing before touching the release path:

- A commit SHA on `release` can be orphaned by the next publish, so pins against it are not stable.
- `rel/*` tags point at source commits and contain no `dist/`, so the action cannot execute from them.
- `v*` tags contain `dist/` but the newest one declares an older Node runtime than `action.yml` on
  `main`.

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
