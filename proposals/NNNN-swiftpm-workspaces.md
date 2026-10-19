# Workspaces for Swift Package Manager

* Proposal: SE-NNNN
* Authors: Sam Khouri ([GitHub](https://github.com/bkhouri)) ([Swift Forums](https://forums.swift.org/u/bkhouri/))
* Review Manager: TBD
* Status: **Pitch — awaiting swift-evolution discussion**
* Implementation: TBD
* Swift-evolution thread: TBD

> **Note on version numbers.** References to Swift tools version `999.0` and `_PackageDescription` availability `999.0` throughout this proposal are illustrative placeholders. The actual version in which the API becomes available depends on when this proposal is accepted and when the implementation lands in a released Swift toolchain. The final version will be substituted before this document is merged and again at implementation time.

## Introduction

We propose first-class support for **workspaces** in Swift Package Manager: an organizational unit that groups multiple Swift packages together for unified dependency resolution, shared build state, and consistent developer workflows.

A workspace is declared via a new `Workspace.swift` manifest at a workspace root. It lists member packages (each with its own `Package.swift`), optionally telling SwiftPM where to fetch a member from if it isn't on disk yet. `swift build`, `swift test`, `swift run`, and every `swift package` subcommand are never workspace-aware: each always operates on exactly one package, found by walking up from the current directory to the nearest `Package.swift`, regardless of any enclosing `Workspace.swift`. A new `swift workspace` command tree (`swift workspace build`, `swift workspace test`, `swift workspace run`, `swift workspace resolve`, ...) provides the workspace-aware equivalents: operating across every member with a single shared `Package.resolved` and a unified `.build/` directory, while still allowing a single member to be targeted.

This proposal establishes first-class workspace support in SwiftPM: the manifest DSL, resolver plumbing, CLI surface, and IDE hooks needed for the primary monorepo and cross-repo local-development use cases. Several follow-on capabilities — cross-workspace composition, a workspace-level configuration surface, completing the `--multiroot-data-file` deprecation timeline, and an "adopt an existing package into a workspace" command — are deliberately out of scope here and captured in [Future direction](#future-direction).

## Motivation

Modern Swift codebases increasingly consist of multiple related packages. Two use cases are common today and poorly served:

### Monorepos with many internal packages

A team may have an application composed of many internal libraries — `app`, `logging`, `networking`, `storage`. Today each is a separate SwiftPM package, and building the system requires one of:

- Publishing internal libraries to a registry (unnecessary friction for internal-only code).
- Using `.package(path: "../lib-a")` in every downstream package. This resolves consistently *within* one build — SwiftPM already resolves a package's whole graph, path dependencies included, together. There could be fragility across builds: a monorepo typically has more than one independently-built root (say `app` and `other-app`), each with its own `Package.resolved`, both path-depending on the same internal `networking` library. If `app` declares `swift-nio from: "2.0.0"` and `other-app` declares `swift-nio from: "2.5.0"`, each root resolves `networking`'s `swift-nio` dependency on its own — nothing ties the two roots' resolutions together, so they can legitimately land on different `swift-nio` versions even though they share the same internal library.
- Using `--multiroot-data-file` with an Xcode workspace file (Xcode-specific format, hidden flag, unclear semantics for non-Xcode users).
- Collapsing everything into a single `Package.swift` with dozens of targets and products instead of a package per library. This sidesteps the resolution-drift and path-dependency problems above, but trades them for a different one: one manifest file that every team edits, that grows slower to evaluate as targets and products accumulate, and that erases per-library ownership — there's no independent `CODEOWNERS` boundary, platform floor, or Swift-tools-version per library once they're all targets in one `Package.swift`. A workspace lets a monorepo keep one small `Package.swift` per library — each independently owned and scoped — while still resolving and building as a unit.

### Cross-repo local development

A developer clones package A and its dependency B side-by-side and wants A to consume the local checkout of B rather than the pinned registry version. Today this requires `swift package edit`, path overrides, or careful `Package.swift` manipulation.

Cargo (Rust), Yarn (JavaScript), and Pants (Python) all offer workspace concepts with unified dependency resolution as first-class citizens. Swift lacks a first-class equivalent.

SwiftPM already knows how to resolve multiple unversioned package roots together — `--multiroot-data-file` relies on exactly this to consume Xcode workspaces today. What is missing is a **SwiftPM-native, first-class user surface** for declaring workspaces.

## Proposed solution

Introduce a `Workspace.swift` manifest file, placed at the workspace root and optionally coexisting with `Package.swift`. The manifest lists members. A member's source lives either on disk already, or is fetched by SwiftPM on demand from source control:

```swift
// swift-tools-version: 999.0
import PackageDescription

let workspace = Workspace(
    members: [
        .member(path: "packages/app"),
        .member(path: "packages/lib-a"),
        .member(
            path: "packages/lib-b",
            ignoredStateDirectories: [.packageResolved],
        ),
        .member(
            path: "packages/lib-c",
            from: .sourceControl(
                url: "https://github.com/example/lib-c",
                requirement: .branch("main"),
            ),
        ),
    ],
)
```

`lib-c` need not exist on disk; the first command that loads the workspace clones it into `packages/lib-c` before proceeding. A member already present on disk ignores its declared `from:` source — it only governs initial acquisition, not ongoing updates.

Members reference sibling members the same way they reference any other local package today, with `.package(path:)`. Because the workspace resolves every member together, a sibling reference is unified with the rest of the graph without a new dependency kind:

```swift
// packages/app/Package.swift
let package = Package(
    name: "app",
    dependencies: [
        .package(path: "../lib-a"),
        .package(url: "https://github.com/apple/swift-nio", from: "2.0.0"),
        .package(url: "https://github.com/apple/swift-log", from: "1.0.0"),
    ],
    ...
)
```

At the CLI, `swift build`, `swift test`, `swift run`, and every `swift package` subcommand are never workspace-aware — each always acts on exactly one package, found by walking up from the current directory to the nearest `Package.swift`, with any enclosing `Workspace.swift` ignored entirely. A new `swift workspace` command tree provides the workspace-aware equivalents:

```
$ cd workspace-root
$ swift workspace init --members packages/lib-a packages/app
                                   # scaffolds a workspace
$ swift workspace build             # builds all members with unified resolution
$ swift workspace test              # runs all members' test targets
$ cd packages/app
$ swift workspace build             # builds only 'app', using the workspace-shared graph
$ swift workspace build --package lib-a
                                   # builds lib-a from anywhere
$ swift build                       # builds 'app' as a fully standalone package; workspace ignored
$ swift package resolve             # resolves and pins 'app' alone, into its own Package.resolved; workspace ignored
$ swift workspace resolve           # resolves the whole workspace into the shared, workspace-root Package.resolved
$ swift workspace override add path some-lib ../local-some-lib
                                   # redirects a declared dep to a local checkout
```

State is unified at the workspace root for `swift workspace` commands: single `Package.resolved`, single `.build/`, single `.swiftpm/configuration/`. A plain `swift build` invoked from inside `packages/app` resolves and builds `app` entirely independently, with its own `Package.resolved` and `.build/` local to `packages/app`, exactly as if `Workspace.swift` did not exist — which also makes it the natural way to sanity-check that a member still builds correctly on its own, outside the workspace's unified resolution.

## Detailed design

### Workspace manifest DSL

New types in `PackageDescription`:

```swift
public struct Workspace: Sendable {
    public let members: [Member]

    public init(
        members: [Member],
    )

    public struct Member: Sendable {
        public let path: String
        public let ignoredStateDirectories: Set<StateDirectoryKind>
        public let remoteSource: RemoteSource?
    }

    public static func member(
        path: String,
        ignoredStateDirectories: Set<StateDirectoryKind> = [],
        from remoteSource: RemoteSource? = nil,
    ) -> Member

    public enum StateDirectoryKind: Sendable {
        case build              // .build/
        case packageResolved    // Package.resolved
        case packages           // Packages/
        case swiftpmConfig      // .swiftpm/configuration/
    }

    /// Where to fetch a member's source from when `path` does not already
    /// exist on disk. Ignored for members already present on disk.
    public enum RemoteSource: Sendable {
        case sourceControl(url: String, requirement: SourceControlRequirement)
    }

    public enum SourceControlRequirement: Sendable {
        case exact(Version)
        case branch(String)
        case revision(String)
        case from(Version)
        case upToNextMinor(from: Version)
    }
}
```

This mirrors the version-qualifier vocabulary `swift workspace override add url` already uses (see [CLI surface](#cli-surface)), so the two ways of pointing SwiftPM at a remote source read the same way.

### Manifest evaluation

`Workspace.swift` uses the same manifest-loading mechanism as `Package.swift`, so it supports the same expressive DSL, diagnostics, and sandboxing guarantees. A `WorkspaceManifest` model mirrors the shape of `Workspace`, for consumption by tooling (see [IDE / libSwiftPM API](#ide--libswiftpm-api)):

```swift
public struct WorkspaceManifest: Sendable {
    public let path: AbsolutePath
    public let toolsVersion: ToolsVersion
    public let members: [Member]

    public struct Member: Sendable {
        public let identity: PackageIdentity
        public let path: AbsolutePath
        public let ignoredStateDirectories: Set<StateDirectoryKind>
        public let remoteSource: Workspace.RemoteSource?
    }
}
```

The tools-version header (`// swift-tools-version: X.Y`) is required in `Workspace.swift`. This governs which Workspace-DSL features are available and provides a clean error path when older toolchains encounter newer manifests.

### Sibling-member references need no new dependency kind

A member declares a dependency on a sibling member the same way it declares a dependency on any other local package: `.package(path:)`. Because every member in a workspace is resolved together as part of one graph (see [Resolution](#resolution)), a sibling reference unifies with the rest of the workspace's dependencies automatically — no new `PackageDependency` API is required, and a `.package(path:)` dependency behaves identically whether or not it happens to point at a workspace sibling.

### Workspace discovery

CLI commands walk up from the current working directory looking for `Workspace.swift` and `Package.swift`. The closest match determines behavior:

This discovery only applies to the `swift workspace` command tree. `swift build`, `swift test`, `swift run`, and every `swift package` subcommand always walk up from the current directory to the nearest `Package.swift` and operate on that single package, full stop — none of them ever look for a `Workspace.swift`, even when one exists above it.

| Situation (for a workspace-aware command) | Behavior |
|---|---|
| Only `Package.swift` in ancestors | Standard single-package operation. |
| Only `Workspace.swift` in ancestors, CWD at workspace root | Workspace operation on all members. |
| Both found, `Workspace.swift` above and `Package.swift` closer (CWD inside member) | Command operates on that member; resolution uses workspace graph. |
| Both found in the same directory (workspace root also a member) | Allowed; the root must be listed explicitly in `members:`. |
| Two `Workspace.swift` files in the ancestor path | Error: nested workspaces not supported. |
| Member's subtree contains its own `Workspace.swift` | Error at load: nested workspaces not supported. Nested workspaces outside declared member subtrees are ignored. |

Empty `members: []` triggers a hard error at load. Workspaces must declare at least one member.

Member paths must be relative to `Workspace.swift`. Absolute paths → hard error. `../` and paths resolving outside the workspace directory tree → allowed with a load-time warning about portability.

If a declared member's `path` does not exist on disk: when the member declares a `remoteSource`, SwiftPM fetches it (checking out the given source-control requirement) into `path` before continuing to load the workspace; when it does not, this is a hard error naming the missing path.

### Resolution

Loading a workspace proceeds in this order:

1. **Load the workspace manifest**, producing a `WorkspaceManifest`.
2. **Resolve member paths.** Normalize relative paths against `Workspace.swift`'s directory; derive a `PackageIdentity` per member from the last path component; error on collisions. Members missing from disk are fetched per their declared `remoteSource` at this point (see [Workspace discovery](#workspace-discovery)).
3. **Load member manifests.** Each member's `Package.swift` is evaluated normally, exactly as it would be standalone.
4. **Resolve the whole workspace as one graph.** Every member is treated as a root package being built together, the same way `--multiroot-data-file` already resolves multiple Xcode-workspace roots together today. A member's external dependencies (`.package(url:)` / `.package(id:)` / `.package(path:)`) are unified with every other member's — if two members declare the same external dependency, one version is chosen that satisfies both. A sibling-member `.package(path:)` reference resolves to that sibling's on-disk location like any other local path dependency.
5. **Produce a single shared `Package.resolved`** at the workspace root.

### Error semantics

- A declared member whose `remoteSource` fails to fetch (bad URL, unreachable host, unsatisfiable requirement) → hard error at workspace load naming the member and the underlying failure.

### Member-level state files

Workspace state (`Package.resolved`, `.build/`, `Packages/`, `.swiftpm/configuration/`) lives at the workspace root under a workspace. Any member with pre-existing state files gets a **trailing warning** at end of each command listing the ignored directories. Members that legitimately need to preserve state (e.g. those also published as standalone packages) suppress warnings per-kind via `Workspace.member(path:ignoredStateDirectories:)`.

Member `Package.resolved` files are read-once-and-ignored — the workspace's resolver populates the workspace-root `Package.resolved` freshly. There is no migration of member pins; users adopting a workspace may see version drift on first resolve and should review the resulting workspace `Package.resolved`.

### CLI surface

#### `swift workspace build` / `swift workspace test` / `swift workspace run`

- From the workspace root, default is "all members".
- From inside a member (Case A), default is "current member only" with resolution via the workspace graph.
- `--package <identity>` selects a specific member from anywhere.
- `swift workspace run` searches only the current member's executables when invoked from inside a member. From the workspace root, ambiguous executable names produce an error listing the candidates and their owning members.
- `swift workspace test` output is a single combined xUnit report at the workspace root with `package="<member>"` attributes on each `<testsuite>`.
- `swift build`, `swift test`, and `swift run` are unaffected: they never discover `Workspace.swift`, even when CWD is inside a member directory. `swift build` inside `packages/app` builds `app` as a fully standalone package — its own resolution, its own `.build/` — exactly as if no workspace existed.

#### `swift workspace init`

- New: `swift workspace init [--members <path>[:type] ...]`.
- `--members packages/lib-a packages/app:executable path/to/libc` — space-separated variadic; optional `:type` suffix per member specifies package type (default `library`; accepted types match `swift package init --type`).
- Missing paths are scaffolded with the same output `swift package init --type` would produce for the specified type. Existing paths with `Package.swift` are left alone (info line). Existing paths without `Package.swift` get one created.
- Placement rationale: the workspace-scope subcommand tree is a top-level `swift workspace` command (a new `swift-workspace` executable), mirroring the precedent set by `swift package-registry`, `swift package-collection`, and `swift sdk` — each of which is a top-level SwiftPM command for its own scope. Every workspace-scope operation stays discoverable under one prefix (`swift workspace {init, add-member, list-members, remove-member, dump-workspace, override, config, resolve, update, clean, reset, build, test, run, show-dependencies, describe}`). These are independent commands from their `swift build` / `swift test` / `swift run` / `swift package` counterparts, which remain single-package only and are never workspace-aware. The pre-existing `swift package init` command is untouched.

#### `swift workspace add-member` / `swift workspace list-members` / `swift workspace remove-member`

Manifest-editing commands that maintain the `members:` array in `Workspace.swift` without hand-editing:

- `swift workspace add-member <path> [--scaffold [--type <type>]]` — adds a member entry. With `--scaffold`, scaffolds a `Package.swift` at that path if one does not exist (the same scaffolding `swift workspace init` produces for a missing member); if a manifest already exists at that path, the flag is a no-op with an info line.
- `swift workspace list-members` — prints the workspace's member identities and paths.
- `swift workspace remove-member <path>` — removes a member entry. Files under the member directory are not deleted; the entry is dropped from `members:` only.

#### `swift workspace override`

- Manages a developer-local `.swiftpm/configuration/workspace-overrides.json` file that redirects a member-declared dependency to a different target without editing the member's `Package.swift`.
- Overrides are picked up as a dependency change, triggering a re-resolve the next time a `swift workspace` command runs.
- Subcommand-per-source shape (one source kind per subcommand, so each subcommand's help only shows the flags relevant to that source):
  - `swift workspace override add path <identity> <path>` — redirect to a local filesystem path (relative paths resolve against the workspace root).
  - `swift workspace override add url <identity> <url> (--exact | --branch | --revision | --from | --up-to-next-minor-from) [--to <version>]` — redirect to a source-control URL, using the same version-qualifier vocabulary as `swift package add-dependency`.
  - `swift workspace override add registry <identity> (--exact | --from | --up-to-next-minor-from) [--to <version>]` — redirect to be resolved via the package registry. The identity plays a dual role — declared dep to redirect AND registry identity to resolve.
- `swift workspace override remove <identity>` — removes an entry. When the last entry is removed, the file itself is deleted rather than left as an empty `{"version":1,"overrides":[]}` document.
- `swift workspace override list [--format text|json]` — text default emits a multi-line block per override (identity, kind, location, requirement); `--format json` emits a stable JSON array of per-override records for downstream tooling.

#### `swift workspace config`

- `swift workspace config {set-mirror, get-mirror, unset-mirror}` anchors to a single `<workspace-root>/.swiftpm/configuration/mirrors.json`. A mirror set from anywhere in the workspace is visible from anywhere in the workspace, including from the workspace root itself and from any member subdirectory.
- Mirror substitution applies at dependency-resolution time to member-declared external deps: `swift workspace resolve` and `swift workspace update` fetch from the mirror URL as expected.
- `swift package config` is unaffected: run from inside a member, it continues to manage a mirrors file local to that single package, even inside a workspace.

#### `swift workspace update`

- Updates write to `<workspace-root>/Package.resolved` (single shared pin file).
- `--package <identity>` restricts fresh pins to that member's transitive dependency subtree; existing pins for identities outside the subtree are preserved.
- From inside a workspace member, the update is implicitly restricted to that member's subtree — same scoping as `swift workspace build`'s Case A.
- `swift package update` is unaffected: it always updates only the current package's own `Package.resolved`.

#### `swift workspace show-dependencies`

- Every in-scope member is rendered:
  - `--format text` (default): sequential per-member trees separated by `--- <identity> ---` headers; dep entries that resolve to another workspace member carry a trailing `[workspace member]` tag.
  - `--format flatlist`: deduplicated union of every in-scope member's transitive dep identities.
  - `--format dot`: each member wrapped in a `subgraph cluster_<sanitized-identity> { ... }` block inside the outer `digraph`; node/edge dedup preserved across clusters.
  - `--format json`: multi-root output is a JSON array of per-root objects.
- Scope selection matches the other workspace-aware commands: inside a member scopes to that member's tree; `--package X` overrides both defaults.
- `swift package show-dependencies` is unaffected: always single-package.

#### `swift workspace describe`

- Iterates every in-scope member instead of picking just one:
  - `--type text` (default): sequential per-member descriptions separated by `--- <identity> ---` headers and blank lines.
  - `--type json`: multi-member output is a JSON array of per-member `DescribedPackage` records.
  - `--type mermaid`: per-member diagrams concatenated with a blank-line separator (each rendered diagram already names its package).
- Scope selection matches the other workspace-aware commands: inside a member scopes to that member's description; `--package X` overrides both defaults; an unknown identity lists the known members.
- `swift package describe` is unaffected: always single-package.

#### `swift workspace clean`

- Removes the single shared `<workspace-root>/.build/`.
- Emits an info-level diagnostic naming the workspace-root `.build/` path so users invoking `clean` from a member subdirectory get an explicit signal of what got removed.
- Accepts `--package <identity>` for parity with other workspace-aware commands, but the flag is a no-op — the shared `.build/` is cleaned regardless — and the CLI emits an info line saying so.
- `swift package clean` is unaffected: removes only the current package's own `.build/`, even inside a workspace.

#### `swift package edit` / `swift package unedit`

Not workspace-aware, and need no special-casing: both operate on the current package exactly as they do standalone, creating an editable checkout and local state scoped to that one package, workspace or not. A developer wanting to redirect a dependency within the workspace's shared, unified graph should use `swift workspace override` instead; the two serve the same goal at different scopes.

#### `--multiroot-data-file`

Deprecated with a removal timeline, in favor of `Workspace.swift` as the SwiftPM-native way to express the same multi-root use case. Since `swift build` / `swift test` never discover `Workspace.swift`, the two surfaces don't overlap at runtime — there's no precedence rule to define, just a migration path to document.

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

These enable IDEs (SourceKit-LSP, Xcode) to display workspace-aware UI: breadcrumb navigation ("Workspace › packages/lib-a"), workspace member list in a sidebar, workspace-level configuration UI. Package-graph loading automatically takes the workspace into account whenever a `Workspace.swift` is discovered, transparently to existing callers.

### Renaming the internal `Workspace` class

SwiftPM has an internal `Workspace` class (a stateful orchestrator for graph loading, checkouts, and resolution) that would collide *cognitively* with the new user-facing `Workspace` DSL type. Although the two are in different modules and don't produce compile errors, documentation, autocomplete, and LSP navigation would suffer.

The internal class is renamed **`PackageWorkspace`**. The `Workspace` *module* keeps its name (module rename is a large amount of churn for marginal additional clarity). A deprecated `public typealias Workspace = PackageWorkspace` is retained in the `Workspace` module for one release to give downstream consumers (SourceKit-LSP, third-party tools) a migration window.

## Source compatibility

- **`PackageDescription` DSL:** All additions gated on `@available(_PackageDescription, introduced: 999.0)`. Packages with tools-version < 999.0 continue to compile unchanged.
- **`Workspace` class rename:** deprecated typealias preserves compat for one release.
- **`swift build` / `swift test` / `swift run` / `swift package` subcommands:** behaviorally unchanged, including inside a workspace member directory — none of them discover or consult `Workspace.swift`.
- **`swift package workspace <subcommand>` invocations:** the workspace-scope subcommand tree lives at the top-level `swift workspace <subcommand>` command. There is no `swift package workspace` nesting.
- **`--multiroot-data-file` flag:** continues to work; deprecation warning strengthened to reference `Workspace.swift` as migration target.

## Effect on ABI stability / API resilience

SwiftPM's libSwiftPM is documented as unstable API (`Package.swift`: *"This API is unstable and may change at any time"*). The changes above follow standard SwiftPM API evolution patterns and require no ABI accommodations.

## Alternatives considered

### Explicit workspace-level shared deps (`.workspaceInherited`)
We considered a design where `Workspace.swift` declared a workspace-wide `dependencies:` array and members opted in via `.package(workspaceInherited: <identity>)` — a single committed place to author shared external version constraints, with member-side trait layering unioned at resolve time, and a "declared but not inherited" audit. Rejected for three reasons. First, resolution unification is already free: every member is resolved together as one graph (see [Resolution](#resolution)), so divergent member declarations of the same external dep already unify — "drift" is a declaration-hygiene concern, not a resolution bug. Second, `swift workspace override` already handles local redirection of member-declared deps, covering one of the strongest motivations for a workspace-level dep surface. The "one-file version bump across N members" ergonomic — the main user-facing win of the dropped design — is addressable without a new DSL surface: `swift workspace update-dependencies` (future direction below) rewrites version constraints across every member's `Package.swift` in one invocation. Third, the dropped design asks the workspace author to pre-declare every external dep every member might want, which scales poorly and discourages per-member constraint tightening. Accepted tradeoff: no central authored version floor in committed source; drift is a hygiene concern caught by review, CI, or lint — not by the type system.

### A dedicated `.package(workspaceMember:)` dependency kind
Early drafts of this proposal had members reference siblings through a dedicated `.package(workspaceMember:)` API rather than plain `.package(path:)`, with the sibling's absolute path populated once the workspace loaded. Rejected: since every member is already resolved together as one graph, a path-based sibling reference gets the same unified resolution a dedicated kind would have offered, for no behavioral difference a plain path dependency doesn't already provide. The one thing a dedicated kind made easier was tagging a dependency as "this resolves to a sibling" in reporting output (e.g. a `[workspace member]` tag in `swift workspace show-dependencies`); that's recoverable by comparing a resolved path dependency's location against the workspace's member paths at render time, without needing a dedicated manifest-level kind.

### Implicit workspace-awareness via a flag on `swift build` / `swift test` / `swift run`
We considered making `swift build`, `swift test`, and `swift run` workspace-aware by default whenever a `Workspace.swift` is discoverable — matching Cargo, where plain `cargo build` already behaves this way inside a workspace, with `-p <package>` to scope to one member — and adding an opt-out flag for a standalone build. Rejected: scope would then depend on whether a `Workspace.swift` happens to exist somewhere above the current directory, so adding one to a repo (or simply running a command from a directory that later gains one) would silently change the behavior of every pre-existing `swift build` / `swift test` invocation and CI script underneath it — different resolved dependency versions, a different (shared) `.build/` — without any change to the command line that ran them. The explicit `swift workspace build` / `swift workspace test` / `swift workspace run` verb keeps that scope decision visible in the invocation itself, and incidentally gives a standalone-build sanity check for free: plain `swift build` inside a member always builds it exactly as a consumer outside the workspace would see it.

### Xcode-style workspaces (no unified resolution)
A workspace as a pure editing container, with each package resolving independently. Rejected — fails the primary motivation (drift across members' external deps).

### Extending `Package.swift` with a `workspace:` field
Rejected — conflates package definition with workspace definition; forces every workspace root to also be a package, restricting layouts.

### JSON/TOML `.swiftpm/workspace.json`
Rejected — breaks SwiftPM's Swift-DSL convention; loses type-checking, extensibility, and consistency with `Package.swift`.

### Skip the internal `Workspace` rename
Rejected — the collision is real in documentation, autocomplete, and LSP navigation despite living in different modules. The rename cost is bounded and mechanical; the clarity benefit is durable.

### Migrating member `Package.resolved` files into workspace `Package.resolved`.
We considered auto-unioning member pins on first workspace resolve to preserve version stability during adoption. Rejected in favor of the safer "ignore + trailing warning" approach: migration risks surprising conflicts across members, and users adopting a workspace should consciously review the resulting resolution.

### Global `--multiroot-data-file` removal
We considered removing the flag entirely as part of shipping workspaces. Rejected — Xcode's `.xcworkspace` is Xcode's format for Xcode's own needs; unilateral removal breaks Xcode integration. The flag is deprecated with a removal timeline instead, letting Xcode migrate on its own schedule.

## Future direction

### Adopt an existing package into a workspace.

This proposal's `swift workspace add-member --scaffold` scaffolds a new member. The reverse — an existing single-package project deciding to become a workspace — is currently hand-edited. A `swift workspace init --from-package .` or a `swift workspace adopt <path>` companion could:

- Read the existing `Package.swift` at the target path.
- Scaffold a sibling `Workspace.swift` with that package as the sole member.

Out of scope for this proposal; a documented adoption story is a natural follow-up once the workspace DSL is in wide use.

### `swift workspace update-dependencies`

> **Note:** There is a PR that introduces a `swift package update-depdencices` in [swiftlang/swift-package-manager#10212](https://github.com/swiftlang/swift-package-manager/pull/10212)

With no workspace-level `dependencies:` surface, bumping a shared external dep across many members is a per-member edit. A `swift workspace update-dependencies [--package <identity>...] <dep-identity> --<requirement>` command would rewrite the version constraint on every member's declaration of the named external dep in one invocation, analogous to `swift package upgrade-dependencies` added in [swift-package-manager#10212](https://github.com/swiftlang/swift-package-manager/pull/10212). Scope selection matches the other workspace-aware commands: no `--package` edits every member that declares the dep; `--package <identity>` (repeatable) restricts to the named members. The command edits each member's `Package.swift` via the existing manifest-editing infrastructure. Deferred so the initial workspace proposal stays focused on the manifest-and-resolver surface.

### Common build settings

A workspace could declare build settings — Swift/C/C++/linker flags, preprocessor defines, warning treatments, unsafe-flag audit exemptions, strict-concurrency level — once and have every member inherit them, avoiding the current pattern where the same setting is copy-pasted into each member's `Package.swift`. Members could opt out or override on a per-target basis. This is a higher-value slice of the broader workspace-level configuration surface sketched above: build settings are the axis that otherwise churns across every member manifest whenever a project-wide flag changes (e.g. bumping strict concurrency, adding a new warning group, or introducing a shared define).

### Build combinations

Today each `swift build` invocation composes its configuration ad hoc from CLI flags (`-c`, `-Xswiftc`, sanitizers, triples). A workspace could declare named *build combinations* — for example `release-strict` (release + strict concurrency + warnings-as-errors), `debug-sanitized` (debug + ASan + TSan), or a cross-compilation matrix — and expose them as `swift workspace build --combination release-strict`. This lifts per-invocation flag juggling into a declarative, reviewable artifact in the workspace manifest, gives CI a shared vocabulary with local development, and makes matrix builds easier to express without shell orchestration.

### Test combinations

Analogous to build combinations, a workspace could declare named *test combinations* that fan a test run across platforms, Swift language modes, sanitizers, and member subsets. For example, a `release-integration` combination might run only integration-tagged tests across every member with release optimizations; a `pr-fast` combination might run only unit tests on the members touched by a diff. This gives CI a single entry point (`swift workspace test --combination pr-fast`) instead of orchestrating filtering and iteration across many `swift test` invocations, and keeps the combination definitions in-repo and reviewable alongside the code they exercise.

### Implicit workspace members instead of explicit

For the initial proposal, it was opted to be explicit. However, there is no reason a workspace couldn't search for `Package.swift` in the directory tree to determine its members. This would require additional considerations, such as not scanning directories under a nested workspace's directory. As such, it was opted to pursue this in a future direction if desirable.

### Complete the `--multiroot-data-file` deprecation timeline.

`--multiroot-data-file` is unaffected by this proposal — it remains a `swift build`/`swift test` flag, and those commands never discover `Workspace.swift`, so the two surfaces don't interact. Still to do, as a separate follow-up:

- **Standalone deprecation warning.** Emit a deprecation warning whenever `--multiroot-data-file` is used, pointing at `Workspace.swift` as the SwiftPM-native migration target. The warning wording should acknowledge the Xcode integration case rather than reading as a bare "please stop using this flag."
- **Migration guidance.** Document how to convert an `.xcworkspace` referencing SwiftPM packages into an equivalent `Workspace.swift`. Whether this is a tool (`swift workspace migrate --from-xcworkspace <path>`) or prose in the migration guide is a design decision the follow-up will settle.
- **Removal timeline.** Xcode's `.xcworkspace` is Xcode's own format; SwiftPM removing `--multiroot-data-file` unilaterally would break the Xcode workflow. Any removal window must be coordinated with the Xcode team, and the target release must ship in a Swift toolchain Xcode is willing to adopt as its floor. This proposal specifies deprecation now and defers removal to that coordinated follow-up.

**Workspace-level configuration surface.** Today a workspace lists members but has no place to declare shared build settings, tools-version floors, or defaultable per-member configuration. A `configuration:` block on `Workspace(...)` — or a sibling type like `WorkspaceSettings` — could hoist "same setting across every member" without asking every member's `Package.swift` to repeat itself. Concrete candidates: default swift-tools-version, default platform floors, shared unsafe-flag audit exemptions. Out of scope for this proposal; a natural next chunk once the workspace surface stabilizes in-ecosystem.


## Acknowledgments

Thanks to the SwiftPM maintainers for prior work on multi-root graph support — the same support that `--multiroot-data-file` relies on today — that this proposal builds on.
