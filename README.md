# NanoFaaS Gradle Plugin

A Gradle **settings plugin** for building [NanoFaaS](https://github.com/Nanofaas/nanofaas)
distributions. It discovers control-plane modules, checks their requirements and
conflicts, composes OpenAPI documents, and validates and assembles YAML recipes.

- Repository: `Nanofaas/nanofaas-gradle-plugin`
- Plugin ID: `io.github.nanofaas.build`
- Implementation artifact: `io.github.nanofaas:nanofaas-gradle-plugin`
- Release: `0.1.1`
- Requirements: **Java 25**, **Gradle 9.7.1** (the included wrapper).
- License: [MIT](LICENSE).

The Java packages retain `it.unimib.datai.nanofaas.gradle` from the original implementation.
The published plugin ID replaces the unpublished, repository-local
`it.unimib.datai.nanofaas.control-plane-modules` ID.
Version `0.1.0` was submitted under `io.github.nanofaas.control-plane-modules`;
from `0.1.1`, use `io.github.nanofaas.build`. The Maven implementation coordinates
remain unchanged. The new ID requires its own Portal approval.

## Apply the plugin

After version `0.1.1` has been published to the Gradle Plugin Portal, apply it in
**`settings.gradle`**, before the project includes:

```groovy
pluginManagement {
    repositories {
        gradlePluginPortal()
        mavenCentral()
    }
}

plugins {
    id 'io.github.nanofaas.build' version '0.1.1'
}
```

This plugin follows the NanoFaaS repository layout; it does not turn an arbitrary
Java project into a FaaS platform. NanoFaaS pins the plugin version through
`nanofaasGradlePluginVersion` in its `gradle.properties`.
The first Portal release is a prerequisite for builds that use the published ID.
For development or before that release, use the local checkout instructions below.

## Tutorial: build a NanoFaaS distribution

### 1. Get NanoFaaS

```bash
git clone https://github.com/Nanofaas/nanofaas.git
cd nanofaas
./gradlew listRecipeFunctions
```

Use a NanoFaaS revision that consumes this plugin version. The catalog lists the
function and service implementations available in that checkout.
Validating a recipe and building a JVM artifact need no Docker daemon or GraalVM.

### 2. Write a recipe

Create `recipes/tutorial.yaml`:

```yaml
# yaml-language-server: $schema=https://raw.githubusercontent.com/Nanofaas/nanofaas-gradle-plugin/v0.1.1/src/main/resources/recipes/recipe-v2.schema.json
schemaVersion: 2
name: tutorial
controlPlane:
  modules: []
  build:
    mode: jvm
```

This selects a core-only JVM control plane. Replace `modules: []` with an exact
list of module IDs when you need optional features. Requirements and conflicts
are checked before compilation.

### 3. Validate and assemble

```bash
./gradlew validateRecipe -Precipe=recipes/tutorial.yaml
./gradlew assembleRecipe -Precipe=recipes/tutorial.yaml
```

Validation prints the build plan. Assembly writes artifacts and
`distribution.json` under `build/recipes/tutorial/`. There is no container image
in this recipe, so assembly does not invoke Docker.

### 4. Add a container image

With a running Docker daemon, extend the recipe:

```yaml
schemaVersion: 2
name: tutorial
registry:
  repository: ghcr.io/YOUR-ORGANIZATION
  tag: tutorial
controlPlane:
  modules: []
  build: {mode: jvm}
  container: {image: control-plane}
```

```bash
./gradlew validateRecipe -Precipe=recipes/tutorial.yaml
./gradlew assembleRecipe -Precipe=recipes/tutorial.yaml
# Log in to the registry first; this command assembles and then pushes the image.
./gradlew publishRecipe -Precipe=recipes/tutorial.yaml
```

Use `-PrecipeTag=<tag>` to replace the image tag in CI. Native recipes use
`build.mode: native` and need GraalVM on the host, or the NanoFaaS container builder.
The complete [recipe guide](https://github.com/Nanofaas/nanofaas/blob/main/docs/recipes.md)
covers functions, services, native options, buildx and output ownership.

## Module selection without recipes

```bash
./gradlew :control-plane:bootJar -PcontrolPlaneModules=none
./gradlew :control-plane:bootJar -PcontrolPlaneModules=all
./gradlew :control-plane:bootJar -PcontrolPlaneModules=async-queue,k8s-deployment-provider
```

When no selector is supplied, default-enabled modules are selected. The
`NANOFAAS_CONTROL_PLANE_MODULES` environment variable is a fallback to the
`-PcontrolPlaneModules` property. With `-Precipe`, the recipe's exact module list
is authoritative: the selector property is rejected and the environment fallback
is ignored.

Modules live under `platform/modules/<id>/` with a `build.gradle` or
`build.gradle.kts` and a descriptor such as:

```properties
schemaVersion=1
id=example
defaultEnabled=false
requires.strong=
requires.weak=
requires.oneOf=
conflicts=
```

`requires.strong` adds implementation dependencies; `requires.weak` adds
compile-only dependencies. `requires.oneOf` validates that at least one provider
is selected, without adding a dependency. All references must name known modules.
The plugin composes `openapi/core.yaml` with selected modules' `openapi.yaml`
fragments for the `:control-plane` project's resources.

## Repository boundary

This repository owns the Java plugin, its tests and the
[recipe v1](src/main/resources/recipes/recipe-v1.schema.json) and
[recipe v2](src/main/resources/recipes/recipe-v2.schema.json) **JSON Schemas**.
Schemas define allowed YAML fields and types. They are packaged in the plugin JAR
and loaded locally; validation never downloads a schema. Version 1 remains supported.

**Dockerfiles remain in NanoFaaS.** The plugin reads `deploy/recipes/Dockerfile.jvm`
and `deploy/recipes/Dockerfile.native` from the consuming checkout. The shared
native builder remains `tools/native-java/Dockerfile`, with its companion scripts
and empty Maven context. These files express NanoFaaS packaging and runtime policy.
Tests of NanoFaaS's real module descriptors and warm-echo build scripts also remain
in NanoFaaS, under `tools/build-tests`.

Configuration Cache and Isolated Projects are currently declared unsupported.

## Develop locally

```bash
git clone https://github.com/Nanofaas/nanofaas-gradle-plugin.git
cd nanofaas-gradle-plugin
./gradlew build validatePlugins
# Optional advisory analysis, or add -PdeadCodeStrict=true to fail on findings:
./gradlew pmdMain
```

The plugin suite uses temporary fixture projects and a fake Docker executable;
it does not need a NanoFaaS checkout, Docker daemon or registry credentials.
To try changes in a sibling NanoFaaS checkout, without publishing:

```bash
cd ../nanofaas
./gradlew validateRecipe -Precipe=recipes/local-demo.yaml \
  -PnanofaasGradlePluginDir=../nanofaas-gradle-plugin
./gradlew test -PnanofaasGradlePluginDir=../nanofaas-gradle-plugin
```

The override is explicit; ordinary builds resolve the pinned Portal release.
You can persist the override as `nanofaasGradlePluginDir=/absolute/path/to/checkout`
in your **user-level** `~/.gradle/gradle.properties` while developing both repositories.

### Check the published marker locally

```bash
./gradlew publishToMavenLocal
./gradlew -p examples/smoke help
```

The smoke project resolves the plugin marker and its implementation through a
Maven repository and applies the settings plugin without TestKit's injected classpath.

## CI and publishing to the Gradle Plugin Portal

[The workflow](.github/workflows/ci.yml) runs `build`, `validatePlugins`, local Maven
publication and a consumer smoke build on main pushes, pull requests and `v*` tags.
Only version tags publish to the Portal, after verification succeeds.

One-time setup:

1. Push this repository to `Nanofaas/nanofaas-gradle-plugin` and make its English
   documentation publicly accessible.
2. Create or use a [Plugin Portal account](https://plugins.gradle.org/user/register)
   associated with the NanoFaaS organization and generate API credentials.
3. Add GitHub Actions secrets **`GRADLE_PUBLISH_KEY`** and
   **`GRADLE_PUBLISH_SECRET`** to the repository's `gradle-plugin-portal` environment.
   Configure environment protection rules if desired.

For a release, set `version` in `gradle.properties`, commit and push the changes,
then push a matching version tag:

```bash
git tag v0.1.1
git push origin v0.1.1
```

The workflow rejects a tag whose version differs from `gradle.properties`, checks
that both secrets exist, and runs `publishPlugins` with credentials supplied only
to that step. Pull requests cannot trigger publication. Portal versions are
immutable; use a new version for every release and do not use `SNAPSHOT` versions.
The [Portal publishing guide](https://plugins.gradle.org/docs/publish-plugin)
explains initial manual approval and namespace ownership. A successful upload
may still be awaiting approval before users can resolve the first release.

For the initial extraction, publish and wait for `0.1.1` to become available
**before merging the NanoFaaS consumer migration**. Until then use the explicit
local checkout override above. Creating the workflow does not publish a release
or configure the secrets automatically.
