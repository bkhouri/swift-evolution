# Workspaces for Swift Package Manager

* Proposal: SE-NNNN
* Authors: Sam Khouri ([GitHub](https://github.com/bkhouri)) ([Swift Forums](https://forums.swift.org/u/bkhouri/))
* Review Manager: TBD
* Status: **Pitch — awaiting swift-evolution discussion**
* Implementation: Phases 0–15 landed across `bkhouri/t/main/poc_workspaces_phase*` branches, plus follow-ups covering per-member trait isolation and cross-package cycle detection. See [Implementation status](#implementation-status) for the concrete surface shipped so far.
* Swift-evolution thread: TBD

> **Note on version numbers.** References to Swift tools version `999.0` and `_PackageDescription` availability `999.0` throughout this proposal are illustrative placeholders. The actual version in which the API becomes available depends on when this proposal is accepted and when the implementation lands in a released Swift toolchain. The final version will be substituted before this document is merged and again at implementation time.

## Introduction

We propose first-class support for **workspaces** in Swift Package Manager: an organizational unit that groups multiple Swift packages together for unified dependency resolution, shared build state, and consistent developer workflows.

A workspace is declared via a new `Workspace.swift` manifest at a workspace root. It lists member packages (each with its own `Package.swift`), and optionally declares workspace-wide dependencies that members can inherit. Under a workspace, SwiftPM commands (`swift build`, `swift test`, `swift run`, `swift package resolve`) operate across all members with a single shared `Package.resolved` and a unified `.build/` directory.

## Motivation

Modern Swift codebases increasingly consist of multiple related packages. Two use cases are common today and poorly served:

**Monorepos with many internal packages.** A team has an application composed of many internal libraries — `app`, `logging`, `networking`, `storage`. Today each is a separate SwiftPM package, and building the system requires one of:

- Publishing internal libraries to a registry (unnecessary friction for internal-only code).
- Using `.package(path: "../lib-a")` in every downstream package (fragile — each package independently resolves its own graph, so a shared external dependency like swift-nio can drift to different versions across members).
- Using `--multiroot-data-file` with an Xcode workspace file (Xcode-specific format, hidden flag, unclear semantics for non-Xcode users).

**Cross-repo local development.** A developer clones package A and its dependency B side-by-side and wants A to consume the local checkout of B rather than the pinned registry version. Today this requires `swift package edit`, path overrides, or careful `Package.swift` manipulation.

Cargo (Rust), Yarn (JavaScript), and Pants (Python) all offer workspace concepts with unified dependency resolution as first-class citizens. Swift lacks a first-class equivalent.

SwiftPM already contains architectural machinery for multi-root graphs. `PackageGraphRootInput.packages` is a plural `[AbsolutePath]`. PubGrub's `<synthesized-root>` handles N unversioned roots. `--multiroot-data-file` uses this machinery today to consume Xcode workspaces. What is missing is a **SwiftPM-native, first-class user surface** for declaring workspaces.

## Proposed solution

Introduce a `Workspace.swift` manifest file, placed at the workspace root and optionally coexisting with `Package.swift`. The manifest lists members and workspace-wide dependencies:

```swift
// swift-tools-version: 999.0
import PackageDescription

let workspace = Workspace(
    members: [
        "packages/app",
        "packages/lib-a",
        .member(
            path: "packages/lib-b",
            ignoredStateDirectories: [.packageResolved],
        ),
    ],
    dependencies: [
        .package(url: "https://github.com/apple/swift-nio", from: "2.0.0"),
        .package(url: "https://github.com/apple/swift-log", from: "1.0.0"),
    ],
)
```

Members reference each other and workspace-inherited dependencies through new DSL:

```swift
// packages/app/Package.swift
let package = Package(
    name: "app",
    dependencies: [
        .package(workspaceMember: "lib-a"),
        .package(workspaceInherited: "swift-nio"),
        .package(workspaceInherited: "swift-log", traits: ["structured"]),
    ],
    ...
)
```

At the CLI, SwiftPM commands operate on the workspace when the current directory is at (or under) a workspace root:

```
$ cd workspace-root
$ swift build                    # builds all members with unified resolution
$ swift test                     # runs all members' test targets
$ cd packages/app
$ swift build                    # builds only 'app', using workspace-shared graph
$ swift build --package lib-a    # builds lib-a from anywhere
$ swift package workspace init --members packages/lib-a packages/app
                                 # scaffolds a workspace
$ swift package workspace override add path some-lib ../local-some-lib
                                 # redirects a declared dep to a local checkout
```

State is unified at the workspace root: single `Package.resolved`, single `.build/`, single `.swiftpm/configuration/`.

## Detailed design

### Workspace manifest DSL

New types in `PackageDescription`:

```swift
public struct Workspace: Sendable {
    public let members: [Member]
    public let dependencies: [Package.Dependency]

    public init(
        members: [Member],
        dependencies: [Package.Dependency] = [],
    )

    public struct Member: Sendable, ExpressibleByStringLiteral {
        public let path: String
        public let ignoredStateDirectories: Set<StateDirectoryKind>

        public init(stringLiteral value: String)
    }

    public static func member(
        path: String,
        ignoredStateDirectories: Set<StateDirectoryKind> = [],
    ) -> Member

    public enum StateDirectoryKind: Sendable {
        case build              // .build/
        case packageResolved    // Package.resolved
        case packages           // Packages/
        case swiftpmConfig      // .swiftpm/configuration/
    }
}
```

New `Package.Dependency` factory methods:

```swift
extension Package.Dependency {
    @available(_PackageDescription, introduced: 999.0)
    public static func package(
        workspaceMember id: String,
        traits: Set<Package.Dependency.Trait> = [.defaults],
    ) -> Package.Dependency

    @available(_PackageDescription, introduced: 999.0)
    public static func package(
        workspaceInherited id: String,
        traits: Set<Package.Dependency.Trait> = [.defaults],
    ) -> Package.Dependency
}
```

### Manifest evaluation

`Workspace.swift` is evaluated via the existing `ManifestLoader.evaluateManifest()` pathway, sharing the subprocess-based approach used for `Package.swift`. A new `WorkspaceManifest` model type mirrors `Manifest`:

```swift
public struct WorkspaceManifest: Sendable {
    public let path: AbsolutePath
    public let toolsVersion: ToolsVersion
    public let members: [Member]
    public let dependencies: [PackageDependency]

    public struct Member: Sendable {
        public let identity: PackageIdentity
        public let path: AbsolutePath
        public let ignoredStateDirectories: Set<StateDirectoryKind>
    }
}
```

The tools-version header (`// swift-tools-version: X.Y`) is required in `Workspace.swift`, using the existing filename-agnostic `ToolsVersionParser`. This governs which Workspace-DSL features are available and provides a clean error path when older toolchains encounter newer manifests.

### Model changes

Two new cases on `PackageDependency.Kind` (public, forward-compatible under Swift's unfrozen-enum semantics):

```swift
extension PackageDependency {
    public enum Kind {
        // Existing:
        case fileSystem(FileSystem)
        case sourceControl(SourceControl)
        case registry(Registry)

        // New:
        case workspaceMember(WorkspaceMember)
        case workspaceInherited(WorkspaceInherited)
    }

    public struct WorkspaceMember {
        public let identity: PackageIdentity
        /// The resolved absolute path of the workspace member. `nil`
        /// in parse-time output; populated at workspace load time
        /// from the workspace's member map. Non-nil after successful
        /// workspace load.
        public let path: AbsolutePath?
        public let productFilter: ProductFilter
        // traits, etc.
    }

    public struct WorkspaceInherited {
        public let identity: PackageIdentity
        /// The resolved concrete workspace-level source this member's
        /// `.workspaceInherited` is bound to. `nil` in parse-time
        /// output; populated at workspace load time from the workspace's
        /// `dependencies:` declaration. Non-nil after successful
        /// workspace load.
        public let resolved: ResolvedInherited?
        public let productFilter: ProductFilter
        // traits, etc.

        public enum ResolvedInherited {
            case sourceControl(
                location: SourceControl.Location,
                requirement: SourceControl.Requirement,
                nameForTargetDependencyResolutionOnly: String?,
                registryIdentity: PackageIdentity?,
            )
            case registry(requirement: Registry.Requirement)
            case fileSystem(
                path: AbsolutePath,
                nameForTargetDependencyResolutionOnly: String?,
            )
        }
    }
}
```

**Lifecycle of the two cases:**

- `.workspaceMember` is a first-class dependency kind that flows through the entire codebase. At workspace load time, its `WorkspaceMember` payload is *augmented in place*: the `path` field (nil in parse-time output) is populated from the workspace's member map. Downstream code (resolver, graph builder, build system) handles `.workspaceMember` natively using the augmented path — semantically equivalent to a local `.fileSystem` dependency, but with the "this is a workspace member" affordance preserved for diagnostics and tooling.
- `.workspaceInherited` is likewise a first-class dependency kind that flows through the entire codebase. At workspace load time, its `WorkspaceInherited` payload is *augmented in place*: the `resolved` field (nil in parse-time output) is populated with the concrete source (source-control URL + requirement, registry requirement, or file-system path) taken from the matching workspace-level `dependencies:` entry. Member-supplied traits are unioned with the workspace's traits; the member's `productFilter` is preserved. Downstream code handles `.workspaceInherited` natively by dispatching on `resolved` — semantically equivalent to handling the underlying concrete `.sourceControl` / `.registry` / `.fileSystem` case, but with the "this is inherited from the workspace" affordance preserved for diagnostics, tooling, and the workspace-level "declared but not inherited" audit.

Documentation on `.workspaceMember`: *"After successful workspace load, `WorkspaceMember.path` is non-nil. Consumers observing this case with `path == nil` are looking at a parse-time manifest that has not yet been resolved against a workspace."*

Documentation on `.workspaceInherited`: *"After successful workspace load, `WorkspaceInherited.resolved` is non-nil and identifies the concrete workspace-level source this member is bound to. Consumers observing this case with `resolved == nil` are looking at a parse-time manifest that has not yet been resolved against a workspace."*

`swift package describe` participates in the case-preservation contract: `DescribedPackage.DescribedPackageDependency` (the Codable shape driving `describe --format json`) gains a `workspaceInherited(identity:, resolved:)` case alongside the existing `workspaceMember(identity:, path:)`. The nested `ResolvedInherited` mirrors the `PackageDependency.WorkspaceInherited.ResolvedInherited` variants (`.fileSystem`, `.sourceControl` with URL-form location, `.registry`), so tooling reading describe output can distinguish inherited from directly-declared deps while still seeing the concrete resolved source the workspace bound them to.

### Discovery and hybrid loading

CLI commands walk up from the current working directory looking for `Workspace.swift` and `Package.swift`. The closest match determines behavior:

| Situation | Behavior |
|---|---|
| Only `Package.swift` in ancestors | Standard single-package operation. |
| Only `Workspace.swift` in ancestors, CWD at workspace root | Workspace operation on all members. |
| Both found, `Workspace.swift` above and `Package.swift` closer (CWD inside member) | Command operates on that member; resolution uses workspace graph. |
| Both found in the same directory (workspace root also a member) | Allowed; the root must be listed explicitly in `members:`. |
| Two `Workspace.swift` files in the ancestor path | Error: nested workspaces not supported. |
| Member's subtree contains its own `Workspace.swift` | Error at load: nested workspaces not supported. Nested workspaces outside declared member subtrees are ignored. |

Empty `members: []` triggers a hard error at load. Workspaces must declare at least one member.

Member paths must be relative to `Workspace.swift`. Absolute paths → hard error. `../` and paths resolving outside the workspace directory tree → allowed with a load-time warning about portability.

### Resolution and workspace lifecycle

`PackageWorkspace` (the renamed internal `Workspace` class — see "Renaming the internal `Workspace` class" below) gains new responsibilities in this order:

1. **Load workspace manifest.** Read `Workspace.swift` via `ManifestLoader`; produce a `WorkspaceManifest`.
2. **Resolve member paths.** Normalize relative paths against `Workspace.swift`'s directory; derive `PackageIdentity` per member from the last path component; error on collisions.
3. **Load member manifests.** Each member's `Package.swift` evaluated as normal, producing a `Manifest` per member.
4. **Resolve workspace-scoped deps.** For each member manifest, walk the dependency list:
   - `.workspaceMember(...)` entries: look up the resolved absolute path in the workspace member map and populate the `path` field in place. The case stays as `.workspaceMember`.
   - `.workspaceInherited(...)` entries: look up the matching workspace-level `dependencies:` entry by identity and populate the `resolved` field with the concrete source (source-control URL + requirement, registry requirement, or file-system path). Traits are unioned (workspace-authoritative versions); the member's `productFilter` is preserved. The case stays as `.workspaceInherited`.
5. **Feed the resolved manifests to the existing graph loader**, with `PackageGraphRootInput.packages: [AbsolutePath]` set to all member paths. PubGrub's existing `<synthesized-root>` path handles N unversioned roots — identical to how `--multiroot-data-file` operates today. `.workspaceMember` and `.workspaceInherited` deps are handled by the resolver using their augmented payloads: `.workspaceMember` behaves as an unversioned local (`.fileSystem`) package at its resolved `path`; `.workspaceInherited` dispatches on `resolved` to behave as the corresponding concrete `.sourceControl` / `.registry` / `.fileSystem` case.
6. **Single shared `Package.resolved`** at workspace root, using the existing V3 schema unchanged. The schema is already root-agnostic (a flat `[PackageIdentity: ResolvedPackage]` map).

### Trait union and error semantics

A member consuming `.package(workspaceInherited: "swift-nio", traits: X)` where the workspace declared the same dependency with `traits: Y` receives the union `X ∪ Y`. Version constraints are workspace-authoritative — members cannot override versions.

- Unknown identity in `workspaceInherited` → hard error at resolve: `"no workspace-level dependency 'foo' declared in Workspace.swift"`.
- Workspace-level `dependencies:` entry that no member inherits → warning at end of resolve.
- `.package(workspaceMember: ...)` or `.package(workspaceInherited: ...)` in a `Package.swift` loaded **outside** a workspace context (no `Workspace.swift` discoverable) → hard error at graph-load: *"this API requires a Workspace.swift in an ancestor directory."*

### Member-level state files

Workspace state (`Package.resolved`, `.build/`, `Packages/`, `.swiftpm/configuration/`) lives at the workspace root under a workspace. Any member with pre-existing state files gets a **trailing warning** at end of each command listing the ignored directories. Members that legitimately need to preserve state (e.g. those also published as standalone packages) suppress warnings per-kind via `Workspace.member(path:ignoredStateDirectories:)`.

Member `Package.resolved` files are read-once-and-ignored — the workspace's resolver populates the workspace-root `Package.resolved` freshly. There is no migration of member pins; users adopting a workspace may see version drift on first resolve and should review the resulting workspace `Package.resolved`.

### CLI surface

**`swift build` / `swift test` / `swift run`:**

- From workspace root, default is "all members".
- From inside a member (Case A), default is "current member only" with resolution via workspace graph.
- `--package <identity>` selects a specific member from anywhere.
- `swift run` searches only the current member's executables when invoked from inside a member. From the workspace root, ambiguous executable names produce an error listing the candidates and their owning members.
- `swift test` output is a single combined xUnit report at the workspace root with `package="<member>"` attributes on each `<testsuite>`.

**`swift package workspace init`:**

- New: `swift package workspace init [--members <path>[:type] ...]`.
- `--members packages/lib-a packages/app:executable path/to/libc` — space-separated variadic; optional `:type` suffix per member specifies package type (default `library`; accepted types match `swift package init --type`).
- Missing paths are scaffolded as full `InitPackage` output for the specified type. Existing paths with `Package.swift` are left alone (info line). Existing paths without `Package.swift` get one created.
- Placement rationale: nesting the new subcommand under the existing `swift package workspace` command tree (alongside `override`) avoids restructuring `swift package init` with `defaultSubcommand` backwards-compat shims, and keeps every workspace-scope command discoverable under one prefix (`swift package workspace {init, add-member, list-members, remove-member, dump-workspace, override, resolve, update, clean, reset}`). The four workspace-aware commands that also live at the top level (`resolve`, `update`, `clean`, `reset`) are registered from the same struct type in both parent trees — `swift package workspace resolve` invokes the identical implementation as `swift package resolve` rather than a duplicated command.

**`swift package workspace override`:**

- Manages a developer-local `.swiftpm/configuration/workspace-overrides.json` file that redirects a workspace-declared dependency to a different target without editing `Workspace.swift`.
- Overrides are folded into the workspace `originHash`, so editing the overrides file triggers a re-resolve on the next command.
- Subcommand-per-source shape (Argument Parser enforces mutual exclusion at parse time):
  - `swift package workspace override add path <identity> <path>` — redirect to a local filesystem path (relative paths resolve against the workspace root).
  - `swift package workspace override add url <identity> <url> (--exact | --branch | --revision | --from | --up-to-next-minor-from) [--to <version>]` — redirect to a source-control URL, using the same version-qualifier vocabulary as `swift package add-dependency`.
  - `swift package workspace override add registry <identity> (--exact | --from | --up-to-next-minor-from) [--to <version>]` — redirect to be resolved via the package registry. The identity plays a dual role — declared dep to redirect AND registry identity to resolve — since the parser reconstructs the redirected dep using the override's identity as the registry key.
- `swift package workspace override remove <identity>` — removes an entry. When the last entry is removed, the file itself is deleted rather than left as an empty `{"version":1,"overrides":[]}` document.
- `swift package workspace override list [--format text|json]` — text default emits a multi-line block per override (identity, kind, location, requirement); `--format json` emits a stable JSON array of per-override records for downstream tooling.

**`swift package update`:**

- Under a workspace, updates write to `<workspace-root>/Package.resolved` (single shared pin file).
- `--package <identity>` restricts fresh pins to that member's transitive dependency subtree; existing pins for identities outside the subtree are preserved.
- From inside a workspace member, the update is implicitly restricted to that member's subtree — same semantics as `swift build`'s Case A.

**`swift package show-dependencies`:**

- Under a workspace, every in-scope member is rendered:
  - `--format text` (default): sequential per-member trees separated by `--- <identity> ---` headers; dep entries that resolve to another workspace member carry a trailing `[workspace member]` tag.
  - `--format flatlist`: deduplicated union of every in-scope member's transitive dep identities.
  - `--format dot`: each member wrapped in a `subgraph cluster_<sanitized-identity> { ... }` block inside the outer `digraph`; node/edge dedup preserved across clusters.
  - `--format json`: multi-root output is a JSON array of per-root objects; single-root output keeps the pre-workspaces top-level object shape for backwards compat.
- Scope selection matches the other workspace-aware commands: inside a member scopes to that member's tree; `--package X` overrides both defaults.

**`swift package describe`:**

- Under a workspace, iterates every in-scope member instead of silently picking the first root:
  - `--type text` (default): sequential per-member descriptions separated by `--- <identity> ---` headers and blank lines.
  - `--type json`: multi-member output is a JSON array of per-member `DescribedPackage` records; single-member output keeps the pre-workspaces top-level object shape for backwards compat with existing `swift package describe --type json` consumers.
  - `--type mermaid`: per-member diagrams concatenated with a blank-line separator (each rendered diagram already names its package).
- Scope selection matches the other workspace-aware commands: inside a member scopes to that member's description; `--package X` overrides both defaults; unknown identity emits `.unknownWorkspaceMember` listing the known members.

**`swift package clean`:**

- Under a workspace, removes the single shared `<workspace-root>/.build/`.
- Emits an info-level diagnostic naming the workspace-root `.build/` path so users invoking `clean` from a member subdirectory get an explicit signal of what got removed.
- Accepts `--package <identity>` for parity with other workspace-aware commands. Under a workspace the flag is a no-op — the shared `.build/` is cleaned regardless — and the CLI emits an info line saying so. Outside a workspace, `--package` surfaces the same "requires a Workspace.swift" error as the other commands.

**`swift package edit` / `swift package unedit`:** deferred under a workspace. Both commands hard-error with an actionable message directing the user at `swift package workspace override` — the workspace-scoped replacement for the "redirect a dep to a local checkout" use case. `edit`'s per-dependency editable-checkout model collides with the workspace's shared `.build/` and `workspaceMember` / workspace-level dependency declarations; `workspace override` supersedes it cleanly.

**CLI flag renaming (coincidental cleanup):** `--package-path` is renamed to `--project-path`, with a deprecation cycle. `--project-path` accepts either a package or a workspace root — the flag is functionally universal, and the old name was misleading. `--package-path` continues to work; emits a deprecation warning; last-wins semantics if both are provided.

> The new alias is `--project-path` rather than `--path` because `swift package edit --path <checkout>` is public API (an evolution-locked flag on the `edit` subcommand). A global `--path` alias would collide with `edit`'s flag at the ArgumentParser level. `--project-path` avoids the collision without needing an evolution proposal to rename `edit`'s flag.

**`--multiroot-data-file`:** deprecated with a removal timeline. When co-specified with a discovered `Workspace.swift`, hard error.

### IDE / libSwiftPM API

Minimal additions to public API to enable IDE integration:

```swift
public struct WorkspaceManifest { ... }   // As above, in PackageModel

extension PackageWorkspace {
    public static func discoverWorkspaceRoot(
        from path: AbsolutePath,
        fileSystem: FileSystem,
    ) -> AbsolutePath?

    public func loadWorkspaceManifest(
        at path: AbsolutePath,
        observabilityScope: ObservabilityScope,
    ) async throws -> WorkspaceManifest
}
```

These enable IDEs (SourceKit-LSP, Xcode) to display workspace-aware UI: breadcrumb navigation ("Workspace › packages/lib-a"), workspace member list in a sidebar, workspace-level configuration UI. Graph loading is otherwise transparent — the existing `PackageWorkspace.loadPackageGraph(rootPath:)` internally routes to workspace loading when `Workspace.swift` is discovered.

### Renaming the internal `Workspace` class

SwiftPM has an internal `Workspace` class (a stateful orchestrator for graph loading, checkouts, and resolution) that would collide *cognitively* with the new user-facing `Workspace` DSL type. Although the two are in different modules and don't produce compile errors, documentation, autocomplete, and LSP navigation would suffer.

The internal class is renamed **`PackageWorkspace`**. The `Workspace` *module* keeps its name (module rename is 10× the churn for marginal additional clarity). A deprecated `public typealias Workspace = PackageWorkspace` is retained in the `Workspace` module for one release to give downstream consumers (SourceKit-LSP, third-party tools) a migration window.

This rename is scoped as a separate preparatory PR landed before workspace feature work begins.

## Implementation status

The proposal has been implemented incrementally across a sequence of
proof-of-concept branches; the design in this document is what the
implementation actually ships (with a small number of documented
deviations noted below). The plan and per-phase status live in the
SwiftPM repo at `plans/2026-08-24-swiftpm-workspaces-implementation.md`.

**Shipped (Phases 0–15):**

- **Phase 0** — Internal `Workspace` class renamed to `PackageWorkspace`; deprecated typealias retained.
- **Phase 1** — Minimal workspace: `Workspace.swift` DSL, `WorkspaceManifest` model, discovery, multi-root loader wiring.
- **Phase 2** — `.package(workspaceMember:)` DSL. Kind preserved through the graph load; `WorkspaceMember.path` augmented in place.
- **Phase 3** — `.package(workspaceInherited:)` DSL. Kind preserved; `WorkspaceInherited.resolved` augmented in place. `resolved` covers `.sourceControl`, `.registry`, and `.fileSystem`.
- **Phase 4** — CWD-inside-member Case A for `swift build`. Scratch stays at workspace root even when the command is invoked from a member subdirectory.
- **Phase 5** — `--package <identity>` selector for `swift build`, threading `PackageIdentity` through `BuildSubset`.
- **Phase 6** — `swift test` and `swift test list` in a workspace, including combined xUnit report + `--package` selector.
- **Phase 7** — `swift run` collision handling. Ambiguous executable names produce an error listing candidates + owning members; `--package <identity> <exec>` resolves cross-member from anywhere.
- **Phase 8** — `swift package resolve` writes `Package.resolved` at the workspace root, trailing warning aggregator lists per-member state files SwiftPM found but ignored, `ignoredStateDirectories` DSL suppresses per-kind warnings, `originHash` covers the union of member manifests + workspace-level `dependencies:`, and `BuildPlan`'s build-input tracking references the workspace-root `Package.resolved`.
- **Phase 8B** — Workspace dependency overrides via `.swiftpm/configuration/workspace-overrides.json`. Overrides fold into `originHash`.
- **Phase 8C** — `swift package workspace override` subcommand tree (`add path`/`add url`/`add registry`, `remove`, `list [--format text|json]`).
- **Phase 9** — `swift package workspace init [--members <path>[:type] ...]` scaffolds a new workspace. Placement is under the existing `swift package workspace` command tree; the pre-existing `swift package init` command is untouched.
- **Phase 10** — `swift package show-dependencies` extended for all four output formats (text with `[workspace member]` tag, deduplicated flatlist, dot subgraph clusters, per-root JSON array), plus CWD focus and `--package` selector.
- **Phase 11** — `swift package update` writes to workspace-root `Package.resolved`; `--package <identity>` and CWD focus restrict fresh pins to that member's transitive dep subtree while preserving pins outside the subtree.
- **Phase 12** — `swift package clean` operates on the shared workspace-root `.build/`, emits an info line naming the cleaned path, and treats `--package` as an info-level no-op under a workspace.
- **Phase 13** — `swift package describe` iterates every in-scope member for all three output formats (text with per-member `--- <identity> ---` headers, JSON array of per-member `DescribedPackage` records, per-member mermaid diagrams concatenated by a blank line), plus CWD focus and `--package` selector. `DescribedPackageDependency` gains a first-class `workspaceInherited(identity:, resolved:)` case so describe consumers can distinguish inherited deps from directly-declared ones while still seeing the concrete resolved source; the prior `preconditionFailure` on `.workspaceInherited` (which crashed describe on any workspace using `.package(workspaceInherited:)`) is replaced by the case-preserving handling.
- **Phase 14** — `swift package dump-package` disambiguates in a workspace: from a workspace root with 2+ members without `--package`, the command errors with the known member identities; from inside a member, CWD auto-selects; `--package <identity>` overrides. Single-member workspaces auto-select and non-workspace usage is unchanged. Landed alongside the sibling command `swift package workspace dump-workspace`, which dumps the parsed `Workspace.swift` (path, tools version, members `{identity, path}`, workspace-level dependencies) as JSON via a new `Encodable` conformance on `WorkspaceManifest`. The same slice also re-registers the four workspace-aware top-level commands (`resolve`, `update`, `clean`, `reset`) under `swift package workspace` so `swift package workspace resolve` invokes the identical struct as `swift package resolve` — no implementation duplication, chosen so every workspace-scope operation is discoverable under one prefix.
- **Phase 15** — Error paths + edge cases. Landed as four sub-slices:
  - **15a/15b** — Load-time validation errors on `Workspace.swift` read like advice: empty members, absolute member paths, out-of-tree member paths, duplicate member identities, missing member directory, member without `Package.swift`, nested `Workspace.swift` in a member subtree, and nested `Workspace.swift` in an ancestor. Each case has a fixture under `Fixtures/Workspaces/S15_ErrorPaths/` and an e2e test asserting the actionable phrasing. `WorkspaceResolveError` gets `CustomStringConvertible` so runtime output reads as prose, not enum reflection.
  - **15c** — `--multiroot-data-file` co-specified with a discoverable `Workspace.swift` hard-errors at CLI init. `--package-path` → `--project-path` rename with a deprecation cycle (aliased `@Option` gives ArgumentParser last-wins semantics; a separate argv-scan detects the deprecated spelling and emits the warning).
  - **15d** — `swift package edit` / `swift package unedit` under a workspace hard-error with an actionable message directing the user at `swift package workspace override` (not "will be addressed in a follow-up" — the workspace-scoped replacement is already shipped in Phase 8C). `.package(workspaceMember:)` and `.package(workspaceInherited:)` used in a standalone `Package.swift` (no ancestor `Workspace.swift`) surface a hard error at load time by propagating `WorkspaceResolveError` through `loadRootManifests` instead of silently returning an empty manifest set.

**Follow-up coverage landed after Phase 15:**

- **Per-member trait isolation for `.workspaceInherited`.** `resolveInherited` already computed each member's `workspace ∪ member` trait union independently, but the multi-member case was not exercised. Unit and e2e tests (`S15_InheritedTraits` fixture) now lock in that two members inheriting the same workspace dep with divergent per-member `traits:` sets get their own trait unions with no cross-contamination.
- **Per-member trait isolation for `.workspaceMember`.** Unlike `.workspaceInherited`, `.workspaceMember` has no workspace-level counterpart to merge with, so `resolveWorkspaceMemberPaths` preserves each consumer's authored trait set verbatim. Unit and e2e tests (`S15_MemberTraits` fixture) lock in per-consumer isolation for sibling-member references.
- **Cross-package cycle detection through workspace-scoped edges.** `.workspaceMember` and `.workspaceInherited` edges participate in the module-level cross-package cycle scan just like ordinary `.package(path:)` / source-control / registry edges. E2E tests (`S15_WorkspaceMemberCycle`, `S15_WorkspaceInheritedCycle` fixtures) lock in that a target-level cycle traversing either workspace-scoped edge kind is rejected at graph load with a `cyclic dependency declaration` diagnostic naming the cycle path.

**Not yet implemented:**

- **`--multiroot-data-file` deprecation timeline** — the co-specification hard error is in (Phase 15c); the standalone deprecation warning that steers users toward `Workspace.swift` as the migration target is still pending.
- **`swift package edit` / `swift package unedit` under a workspace** — deferred as a design decision, superseded by `swift package workspace override`. Not planned for the MVP.

**Deviations from the pitch as drafted:**

- Phase 9 nests `swift package workspace init` under the existing
  `swift package workspace` subcommand tree instead of restructuring
  `swift package init` into `init package` / `init workspace`
  subcommands with a `defaultSubcommand` shim. The user-visible
  surface is `swift package workspace init …`; the pre-existing
  `swift package init` command is untouched. This avoids adding a
  backwards-compat shim for the existing `init` flag surface and
  keeps every workspace-scope command discoverable under a single
  prefix.
- Phase 8C landed a subcommand-per-source shape for `override add`
  (`add path`, `add url`, `add registry`) rather than a single
  flag-based `override add` with mutually-exclusive `--path` /
  `--url` / `--registry` options. Argument Parser has no first-class
  mutual-exclusion between options, and the subcommand shape lets
  the parser enforce it structurally — e.g. `--branch` never
  appears on the `path` subcommand's help output at all.
- Phase 12's `--package` under-workspace behaviour is `info`-level,
  not `warning`-level. The request is benign (the shared `.build/`
  is cleaned regardless) and a warning would incorrectly imply user
  error.
- Phase 15's global path-flag rename lands as `--package-path` →
  `--project-path` rather than the pitched `--package-path` →
  `--path`. `swift package edit --path <checkout>` is public API on
  the `edit` subcommand and would collide with a global `--path` at
  the ArgumentParser level. Renaming `edit`'s flag would require a
  separate evolution proposal; naming the global alias
  `--project-path` avoids the collision entirely.
- Phase 15's `swift package edit` / `unedit` deferred-feature
  diagnostic points at `swift package workspace override` (the
  already-shipped Phase 8C replacement for the "redirect a
  dependency to a local checkout" use case) rather than emitting a
  generic "will be addressed in a follow-up" placeholder. Users
  hitting the guard get a concrete next step, not a promise.
- `DescribedPackageDependency` gains a first-class
  `workspaceInherited(identity:, resolved:)` case in Phase 13 rather
  than unpacking `.workspaceInherited` into its concrete resolved
  kind. Preserves the "inherited from workspace" signal for describe
  consumers and IDE tooling; mirrors the case-preservation choice
  taken for `.workspaceMember` in Phase 2.

## Source compatibility

- **`PackageDescription` DSL:** All additions gated on `@available(_PackageDescription, introduced: 999.0)`. Packages with tools-version < 999.0 continue to compile unchanged.
- **`PackageDependency.Kind` new cases:** source-compatible under Swift's unfrozen-enum semantics — exhaustive switches on `Kind` in third-party code produce `@unknown default` warnings, not errors. Third-party code observing these cases with the augmentation payloads unpopulated (`.workspaceMember(path: nil)`, `.workspaceInherited(resolved: nil)`) is looking at a parse-time manifest and can defer handling to the post-load `packageRef` / `locationString` code paths.
- **`Workspace` class rename:** deprecated typealias preserves compat for one release.
- **`--package-path` flag:** continues to work; emits deprecation warning; renamed to `--path`.
- **`--multiroot-data-file` flag:** continues to work; deprecation warning strengthened to reference `Workspace.swift` as migration target.

## Effect on ABI stability / API resilience

SwiftPM's libSwiftPM is documented as unstable API (`Package.swift`: *"This API is unstable and may change at any time"*). The changes above follow standard SwiftPM API evolution patterns and require no ABI accommodations.

## Alternatives considered

**Xcode-style workspaces (no unified resolution).** A workspace as a pure editing container, with each package resolving independently. Rejected — fails the primary motivation (drift across members' external deps).

**Extending `Package.swift` with a `workspace:` field.** Rejected — conflates package definition with workspace definition; forces every workspace root to also be a package, restricting layouts.

**JSON/TOML `.swiftpm/workspace.json`.** Rejected — breaks SwiftPM's Swift-DSL convention; loses type-checking, extensibility, and consistency with `Package.swift`.

**No new `PackageDependency.Kind` cases; workspace state injected into the manifest subprocess.** Rejected — couples manifest evaluation to workspace loading, complicates caching, breaks standalone member loading, requires new IPC surface for injecting workspace state.

**Skip the internal `Workspace` rename.** Rejected — the collision is real in documentation, autocomplete, and LSP navigation despite living in different modules. The rename cost is bounded and mechanical; the clarity benefit is durable.

**Workspace-authoritative version overrides.** We considered allowing members to override the workspace-level version pin. Rejected — allowing this defeats the primary purpose of workspace-wide dependencies (a single source of truth for versions). Members may layer traits but not versions.

**Migrating member `Package.resolved` files into workspace `Package.resolved`.** We considered auto-unioning member pins on first workspace resolve to preserve version stability during adoption. Rejected in favor of the safer "ignore + trailing warning" approach: migration risks surprising conflicts across members, and users adopting a workspace should consciously review the resulting resolution.

**Global `--multiroot-data-file` removal.** We considered removing the flag entirely as part of shipping workspaces. Rejected — Xcode's `.xcworkspace` is Xcode's format for Xcode's own needs; unilateral removal breaks Xcode integration. The flag is deprecated with a removal timeline instead, letting Xcode migrate on its own schedule.

**Rewrite `.workspaceMember` to `.fileSystem` at load time.** An earlier revision of this proposal specified that `.workspaceMember` cases would be transformed to `.fileSystem` at the workspace load boundary, so downstream code (resolver, graph builder, build system) would remain workspace-unaware. Rejected in favor of the augmentation approach because kind preservation enables tooling affordances (e.g. the `[workspace member]` tag in `swift package show-dependencies` text output falls out naturally) and keeps the manifest observable downstream a faithful record of what the user wrote. The trade-off is workspace-awareness in ~7 downstream files; each site's handling is straightforward (treat `.workspaceMember` analogously to `.fileSystem(member.path!)`).

**Rewrite `.workspaceInherited` to a concrete `.sourceControl` / `.registry` / `.fileSystem` at load time.** An earlier revision of this proposal specified that `.workspaceInherited` cases would be transformed to their concrete workspace-declared source at the workspace load boundary (mirroring the initial `.workspaceMember` design), so downstream code would remain workspace-unaware. Rejected in favor of augmentation — mirroring the reversed `.workspaceMember` decision above — for three reasons. First, kind preservation keeps `.workspaceInherited` observable for diagnostics ("resolution failed because workspace-inherited 'swift-nio' …") and IDE tooling that wants to distinguish "member declared this dependency locally" from "member is consuming the workspace's version." Second, the workspace-level "declared but not inherited by any member" audit needs to see `.workspaceInherited` entries at audit time to compute "used vs. unused"; with the rewrite approach, that audit either had to run pre-rewrite (fragile ordering) or on a separately-tracked identity set (an error-prone parallel data path). Third, the trade-off is small: only ~3 downstream sites (`toConstraintRequirement`, `packageRef`, `locationString`) need to handle the case, each by dispatching on the augmented `resolved` field to the corresponding concrete-kind logic.

## Acknowledgments

Design shaped through iterative design review; captured in commit history and design records on the `bkhouri/t/main/poc_workspaces` branch. Thanks to the SwiftPM maintainers for prior work on the multi-root graph plumbing (`PackageGraphRootInput`, `WorkspaceLoader`, PubGrub `<synthesized-root>`) that this proposal builds on.
