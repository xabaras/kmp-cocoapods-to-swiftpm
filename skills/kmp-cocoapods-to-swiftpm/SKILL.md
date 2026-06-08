---
name: kmp-cocoapods-to-swiftpm
description: Refactor a Kotlin Multiplatform / Compose Multiplatform project from CocoaPods-based iOS dependency integration to Swift Package Manager import tooling, preserving compatible versions and updating Gradle/Xcode configuration safely.
version: 1.0.2
author: Paolo Montalto
tags:
  - kotlin-multiplatform
  - compose-multiplatform
  - ios
  - swiftpm
  - cocoapods
  - gradle
---

# KMP CocoaPods -> SwiftPM Migration Skill

Use this skill when refactoring a Kotlin Multiplatform / Compose Multiplatform project that currently uses `kotlin("native.cocoapods")` and/or a `cocoapods {}` block, and the goal is to migrate iOS library integration to Swift Package Manager.

The official Kotlin migration flow is:
1. Add SwiftPM dependencies in Gradle.
2. Reconfigure the Xcode project for direct integration.
3. Remove or reduce CocoaPods integration only after the project builds correctly.

## Main goals

- Replace CocoaPods-based dependency declarations with `swiftPMDependencies {}` equivalents where possible.
- Preserve explicitly declared pod versions when moving to SwiftPM.
- If a pod version is not specified in CocoaPods, research a compatible Swift package version and add it explicitly to `build.gradle.kts`.
- Replace the current Kotlin version used by the project with **2.4.0 final**, because versions preceding 2.4.0 do not support `swiftPMDependencies {}`.
- Move any `cocoapods.framework {}` configuration into `binaries.framework {}` on iOS targets.
- Reconfigure the Xcode project to direct integration before removing CocoaPods.

## Operating rules

1. Never remove CocoaPods declarations first if the project still depends on them for successful builds.
2. If a library version is explicitly declared in `cocoapods`, keep that same version in the SwiftPM declaration whenever the package supports it.
3. If a library version is not explicitly declared in `cocoapods`, search the web for a compatible version and then write that version explicitly in `build.gradle.kts`.
4. Do not guess Swift package URLs, product names, or versions. Verify them.
5. Prefer exact, minimally invasive refactors over broad rewrites.
6. Preserve working target names, framework base names, static/dynamic framework choices, and existing iOS architecture targets unless there is a clear migration issue.
7. Do not hardcode `:composeApp` as the only shared module path. Depending on the project structure, the shared KMP module may be `:composeApp`, `:shared`, `:sharedUI`, or another module path.
8. When Xcode emits a generated integration command, treat that command as the source of truth.

## What to inspect first

When given a project, inspect:

- Root `build.gradle.kts`
- Shared module `build.gradle.kts`
- Any module containing `kotlin("native.cocoapods")`
- `cocoapods {}` blocks
- `Podfile`
- `iosApp.xcodeproj` / iOS Xcode project
- Existing iOS targets such as `iosArm64()`, `iosSimulatorArm64()`, `iosX64()`
- Current imported APIs under `cocoapods.*`

## Migration strategy

### Phase 1: Inventory CocoaPods usage

Extract every pod currently used, including:

- Pod name
- Explicit version, if present
- Extra options such as `linkOnly`, `extraOpts`, subspecs, or module names
- Whether it is referenced from Kotlin code via `cocoapods.<PodName>...`
- Whether the pod maps to one or multiple Swift package products

Create a migration table like:

| Pod | Pod version | Swift package URL | Swift product(s) | Version to use | Notes |
|---|---|---|---|---|---|

Rules for the `Version to use` column:

- If pod version is explicitly present, reuse it.
- If pod version is absent, research and choose a compatible package version, then set it explicitly.

### Phase 2: Update Kotlin/Gradle config

Use Kotlin **2.4.0** final for the migration. Replace whatever Kotlin version the project currently uses if it is older than 2.4.0, because `swiftPMDependencies {}` support arrives with Kotlin 2.4.0.

Typical target refactor pattern:

```kotlin
kotlin {
    iosArm64()
    iosSimulatorArm64()

    swiftPMDependencies {
        swiftPackage(
            url = url("https://github.com/firebase/firebase-ios-sdk.git"),
            version = from("12.5.0"),
            products = listOf(product("FirebaseAnalytics")),
        )
    }

    listOf(
        iosArm64(),
        iosSimulatorArm64(),
    ).forEach { iosTarget ->
        iosTarget.binaries.framework {
            baseName = "Shared"
            isStatic = true
        }
    }
}
```

Important notes:

- Replace the project's current Kotlin version with `2.4.0` whenever it is below 2.4.0, not just when it is a prerelease.
- Prefer keeping the temporary `cocoapods {}` block during the transition until SwiftPM import and Xcode integration are working.
- If the old project uses `cocoapods.framework { baseName = "..."; isStatic = ... }`, move that configuration under `binaries.framework {}` for each relevant iOS target.
- The `embedAndSignAppleFrameworkForXcode` task only exists when `binaries.framework {}` is configured.

### Phase 3: Translate dependencies

For each pod:

1. Find the package repository URL.
2. Determine the Swift package product name(s).
3. Determine the version:
   - explicit CocoaPods version -> keep same version
   - no CocoaPods version -> search and choose a compatible explicit version
4. Add a `swiftPackage(...)` declaration.
5. Keep naming exact and verified.

Pattern:

```kotlin
swiftPMDependencies {
    swiftPackage(
        url = url("PACKAGE_GIT_URL"),
        version = from("EXPLICIT_VERSION"),
        products = listOf(product("PRODUCT_NAME")),
    )
}
```

If multiple products come from the same package:

```kotlin
swiftPMDependencies {
    swiftPackage(
        url = url("PACKAGE_GIT_URL"),
        version = from("EXPLICIT_VERSION"),
        products = listOf(
            product("ProductA"),
            product("ProductB"),
        ),
    )
}
```

### Phase 4: Update imports in Kotlin code

After resolving SwiftPM dependencies, update imports from the CocoaPods namespace to the imported SwiftPM namespace.

Example pattern:

```kotlin
// old
import cocoapods.FirebaseAnalytics.FIRAnalytics

// new
import swiftPMImport.ExampleProject.shared.FIRAnalytics
```

Where:
- `ExampleProject` is based on the Gradle project identity, typically derived from `rootProject.name`.
- `shared` is the KMP shared module name.
- The module segment may be `shared`, `composeApp`, or another actual shared module name used by the project.

Rules for import migration:
- Do not rewrite imports to an app package namespace such as `swiftPMImport.org.example.package.*`.
- Prefer the SwiftPM import namespace shaped like `swiftPMImport.<ProjectName>.<SharedModule>.*`.
- Derive the actual namespace from the generated imports available after Gradle sync / SwiftPM resolution.
- If the generated import root differs from the expected pattern, inspect the generated imports and use the real one from the project.

Do not mass-rewrite blindly; verify the generated package path in the project.

### Phase 4.1: Troubleshooting `$stable` crashes on serialized models

In projects that use Compose together with Ktor and `kotlinx.serialization`, shared data models may sometimes crash at runtime because of Compose stability instrumentation conflicts involving generated `$stable` fields. In that situation, add a Compose stability configuration file and exclude only the affected serialized model packages from instrumentation.

Example `compose-stability.conf`:

```text
// Stability configuration for Compose compiler
// Exclude serialized data models from Compose stability instrumentation
com.example.data.model.**
```

Example shared-module `build.gradle.kts` configuration:

```kotlin
composeCompiler {
    @Suppress("OPT_IN_USAGE")
    stabilityConfigurationFiles.add(
        rootProject.layout.projectDirectory.file("compose-stability.conf")
    )
}
```

Rules:
- Apply this workaround only if the module actually uses Compose compiler instrumentation.
- Scope the exclusion narrowly to DTOs or serialized model packages.
- Do not exclude the whole shared codebase unless there is no narrower safe option.
- Mention this change explicitly in the migration summary as a compatibility fix for Compose stability instrumentation and `kotlinx.serialization`.

### Phase 5: Reconfigure Xcode project

If the project currently relies on the CocoaPods Gradle plugin, reconfigure the Xcode project for direct integration before deleting CocoaPods.

Expected workflow:

1. Open the iOS project in Xcode.
2. Build the project.
3. Inspect the build error output in Xcode.
4. Copy the generated integration command from the error output.
5. Run that command from the terminal.
6. Resolve SwiftPM dependencies from IntelliJ IDEA / Android Studio.
7. Rebuild and verify.

#### Xcode integration command

The generated command from Xcode is the source of truth and should be preferred over any hardcoded example.

Typical generated form:

```bash
XCODEPROJ_PATH='/path/to/project/iosApp/iosApp.xcodeproj' \
GRADLE_PROJECT_PATH=':shared-module-name' \
'/path/to/project/gradlew' -p '/path/to/project' \
':shared-module-name:integrateEmbedAndSign' \
':shared-module-name:integrateLinkagePackage'
```

Project-specific examples:

```bash
XCODEPROJ_PATH='./iosApp/iosApp.xcodeproj' ./gradlew :composeApp:integrateLinkagePackage
```

```bash
XCODEPROJ_PATH='./iosApp/iosApp.xcodeproj' ./gradlew :shared:integrateLinkagePackage
```

Rules:

- Replace `:composeApp` with the actual shared KMP module path.
- In older Compose Multiplatform templates this is often `:composeApp`.
- In newer project structures this may be `:shared`.
- In some projects it may also be another module such as `:sharedUI`.
- Prefer the exact command emitted by Xcode, especially when it includes both `integrateEmbedAndSign` and `integrateLinkagePackage`.
- If the generated command includes `GRADLE_PROJECT_PATH`, preserve it.
- Do not assume that only `integrateLinkagePackage` is needed; use the exact generated command where possible.

### Phase 6: Remove CocoaPods integration

Only after the project builds successfully with SwiftPM import:

- Remove the KMP module line from `Podfile`, or fully remove CocoaPods usage if no pods remain.
- Run `pod install` if partial CocoaPods usage remains.
- Remove the `cocoapods {}` block entirely when all migrated dependencies are handled by SwiftPM.
- Remove `kotlin("native.cocoapods")` from the relevant Gradle plugins if the project no longer needs CocoaPods at all.
- If fully migrating away from CocoaPods direct integration leftovers, clean the iOS setup as needed, for example with `pod deintegrate` where appropriate.

## Refactoring instructions for Claude

When applying this skill to a repository, follow this exact behavior:

1. Detect every CocoaPods declaration and build a migration map.
2. For each pod, determine whether a Swift Package Manager equivalent exists.
3. Preserve exact versions when the pod declaration specifies one.
4. When no version is specified, search for a compatible version and add it explicitly.
5. Update Gradle Kotlin plugin references to `2.4.0`, replacing the project's existing Kotlin version when it is below 2.4.0 because earlier versions do not support `swiftPMDependencies {}`.
6. Add `swiftPMDependencies {}` without prematurely deleting CocoaPods during the intermediate state.
7. Move framework settings from `cocoapods.framework {}` to `binaries.framework {}`.
8. Update Kotlin imports that change from `cocoapods.*` to SwiftPM-imported namespaces.
9. Reconfigure Xcode using the generated command from the Xcode build error output.
10. Treat `:composeApp` and `:shared` as examples, not universal constants.
11. Explain each change briefly in a migration summary.
12. If a pod has no SwiftPM support, stop and report it instead of fabricating a migration.

## Output format

When refactoring, produce:

### 1. Migration summary
A short explanation of:
- which pods were found
- which ones were migrated
- which versions were preserved
- which versions were researched because none were specified
- what Xcode integration command should be run
- whether CocoaPods can already be removed or must remain temporarily

### 2. File-by-file changes
For each changed file:
- path
- reason for change
- exact diff or rewritten block

### 3. Verification checklist
Include:
- Gradle sync succeeds
- SwiftPM dependencies resolve
- Xcode project reconfigured
- generated integration command executed
- Kotlin imports updated
- iOS app builds
- CocoaPods removed only if no longer needed

## Guardrails

- Do not convert versioned pods to unversioned SwiftPM declarations.
- Do not silently upgrade versions unless compatibility requires it and the reason is explained.
- Do not delete Podfile content unrelated to the KMP module if native iOS code still uses CocoaPods.
- Do not assume package product names equal pod names.
- Do not assume one pod equals one Swift package repository.
- Do not remove `iosX64()` / simulator targets unless there is a justified modernization step.
- Do not rewrite unrelated Gradle or source code.
- Do not hardcode `:composeApp`; derive the actual shared module path from the project.

## Example decision rules

### Example A: Explicit pod version
Input:

```kotlin
cocoapods {
    pod("FirebaseAnalytics") {
        version = "12.5.0"
    }
}
```

Required behavior:
- Keep `12.5.0` in the SwiftPM dependency declaration.

### Example B: No explicit pod version
Input:

```kotlin
cocoapods {
    pod("SomeLibrary")
}
```

Required behavior:
- Search for the Swift package repository and a compatible release version.
- Add that version explicitly in `swiftPMDependencies`.
- Mention that the original pod had no fixed version, so the chosen version was researched and pinned.

### Example C: Xcode command with composeApp
Command:

```bash
XCODEPROJ_PATH='./iosApp/iosApp.xcodeproj' ./gradlew :composeApp:integrateLinkagePackage
```

Required behavior:
- Accept this as valid when `:composeApp` is the actual shared module.
- Mention that the full generated command from Xcode may also include `:composeApp:integrateEmbedAndSign`.

### Example D: Xcode command with shared
Command:

```bash
XCODEPROJ_PATH='./iosApp/iosApp.xcodeproj' ./gradlew :shared:integrateLinkagePackage
```

Required behavior:
- Accept this as valid when `:shared` is the actual shared module in the newer project structure.
- Prefer the exact generated Xcode command over simplified examples.

## Suggested prompt template

Use this skill with a prompt such as:

> Migrate this KMP project from CocoaPods to SwiftPM. Inspect all `build.gradle.kts`, `Podfile`, and iOS integration files. Preserve any explicitly declared pod versions. If a pod has no version, search for a compatible Swift package version and pin it explicitly. Replace the project's current Kotlin version with 2.4.0 final when it is below 2.4.0, because earlier versions do not support `swiftPMDependencies {}`. Keep CocoaPods only as long as needed for an intermediate working state, then remove it cleanly. When reconfiguring Xcode, use the generated integration command and substitute the real shared module path, such as `:composeApp` or `:shared`.

## Success criteria

The migration is complete when:

- Swift packages are declared and resolved in Gradle.
- Xcode is reconfigured for direct integration.
- The generated integration command has been identified and executed.
- Framework configuration lives under `binaries.framework {}` instead of `cocoapods.framework {}`.
- Kotlin imports compile against SwiftPM-imported APIs.
- CocoaPods integration has been removed or minimized appropriately.
