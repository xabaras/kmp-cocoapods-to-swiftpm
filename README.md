# kmp-cocoapods-to-swiftpm

This is a reusable AI agent skill for migrating Kotlin Multiplatform / Compose Multiplatform projects from CocoaPods to Swift Package Manager.

This repository packages the migration workflow as an agent skill so tools like Claude Code and other SKILL.md-compatible agents can apply the same refactoring guidance consistently.

## What this skill does

The skill helps an agent migrate a KMP project from CocoaPods to SwiftPM by guiding it to:

- upgrade the project to Kotlin 2.4.0 when needed, because earlier versions do not support `swiftPMDependencies {}`
- translate CocoaPods dependencies into `swiftPMDependencies {}` declarations
- preserve explicit pod versions, and pin researched versions when the original pod had none
- update Xcode integration
- update Kotlin imports for SwiftPM-based generated APIs
- fully remove CocoaPods from the project after migration succeeds
- handle known Compose stability workarounds when needed

## Repository layout

This repository follows the directory layout commonly used by Claude Code and the broader Agent Skills ecosystem:

```text
skills/
  kmp-cocoapods-to-swiftpm/
    SKILL.md
```

That `skills/<skill-name>/SKILL.md` structure is widely used several agents and tools that support the open **Agent Skills** / `SKILL.md` format, including Claude Code, Cursor, Gemini in Android Studio and other SKILL.md-aware environments.

## Who this is for

This skill is useful if you:

- maintain a Kotlin Multiplatform or Compose Multiplatform app with iOS dependencies still managed through CocoaPods
- want to move to Swift Package Manager import support introduced in Kotlin 2.4.0
- use Claude Code, Cursor, Gemini in Android Studio, or another agent that can discover and apply `SKILL.md` skills
- want a repeatable migration flow instead of rewriting the same instructions in each repository

## How to use the skill

### Option 1: Copy the skill into a project

Copy the `kmp-cocoapods-to-swiftpm` folder into your agent's skills directory.

Typical examples:

- Claude Code: put the skill in a Claude-recognized skills directory as a folder containing `SKILL.md`; Claude discovers skills from filesystem directories and uses the skill metadata in the frontmatter to decide when to apply them.[cite:155][cite:160]
- Cursor: place the folder under `.cursor/skills/` for a project-local install or `~/.cursor/skills/` for a user-level install.[cite:167][cite:164]

Example project-local layout in Cursor:

```text
.cursor/
  skills/
    kmp-cocoapods-to-swiftpm/
      SKILL.md
```

### Option 2: Keep it as a shared Git dependency

Because this repository already uses the standard skill folder layout, you can also keep it in Git and sync or copy the `skills/kmp-cocoapods-to-swiftpm/` folder into your preferred agent environment as needed. The Agent Skills format is designed as a portable directory-based standard built around `SKILL.md` files.[cite:165][cite:60]

## Example prompts

Once installed, use prompts like:

- `Migrate this KMP project from CocoaPods to SwiftPM using the installed skill.`
- `Inspect this project and replace CocoaPods with swiftPMDependencies, then remove CocoaPods completely.`
- `Use the kmp-cocoapods-to-swiftpm skill to refactor this Compose Multiplatform project.`

A good prompt usually mentions:

- the repository or module to inspect
- that the project should move to Kotlin 2.4.0 if needed
- that explicit pod versions must be preserved
- that missing pod versions must be researched and pinned explicitly
- that CocoaPods must be fully removed at the end

## What the skill assumes

The skill is opinionated and assumes a **full** migration away from CocoaPods.

That means it is written to:

- remove the `cocoapods {}` block
- remove the CocoaPods Gradle plugin
- remove the CocoaPods plugin alias from `libs.versions.toml` when version catalogs are used
- run `pod deintegrate`
- delete `Podfile`, `Pods/`, and `iosApp.xcworkspace`
- open the iOS app through `.xcodeproj`

If the project still depends on a library that only works through CocoaPods, the migration should be reconsidered before applying the skill.

## Why SKILL.md

`SKILL.md` is the core file format used by Claude's custom skills system, where each skill lives in its own directory and starts with YAML frontmatter plus markdown instructions.

The same format is now used more broadly as an open Agent Skills standard, which is why a repository like this can be useful across multiple coding agents instead of only one.

## Compatibility note

This repository aims to stay close to the portable core of the `SKILL.md` format so it can be reused across different agent tools. Some tools add their own features or installation paths, but the fundamental pattern remains the same: a folder named after the skill, containing a `SKILL.md` file with metadata and instructions.

## Contributing

Suggestions and fixes are welcome, especially if you find:

- additional SwiftPM migration edge cases in KMP projects
- better handling for specific CocoaPods-to-SwiftPM library mappings
- import naming differences across project structures
- Compose or serialization-related migration pitfalls

## License

This skill is released under the MIT license.
