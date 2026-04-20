# SurvAIvor — Phase 0 + 1 Mega Prompt for Claude Code

> Read `PROJECT_BRIEF.md` first. This document is operational; the brief is constitutional. If anything here contradicts the brief, the brief wins.

> **Document version:** v2 (2026-04-26). Updates from v1: added iOS-compatibility constraint to all domain packages, added versioning fields to schema, added `shallow` and `comprehension_score` to schema from day one, expanded ADR list, clarified that vault is the future iCloud sync target.

---

## How to use this prompt

**You are building Phase 0 and Phase 1 of SurvAIvor.** Phase 0 is the foundation; Phase 1 makes it usable as a personal knowledge base without any AI in the loop yet. AI features arrive in Phase 2+.

**Working agreement:**
1. Before writing any code, read `PROJECT_BRIEF.md` end-to-end.
2. Verify current versions and APIs of every external dependency (GRDB.swift, sqlite-vec, swift-markdown, Hummingbird if used, Yams) via web search or Context7. **Do not** assume APIs from your training data.
3. Work phase by phase. Do not start Phase 1 work until Phase 0's Definition of Done passes.
4. Within a phase, work task by task. After each task, run tests and ask the user to verify acceptance criteria before moving on.
5. Use the Swift Testing framework (`import Testing`) for new tests. Fall back to XCTest only when needed.
6. Commit after each acceptance criterion passes. Conventional Commits style (`feat:`, `fix:`, `chore:`, `test:`, `docs:`).
7. Never bypass an anti-pattern in `PROJECT_BRIEF.md` Section 9. If you think one is wrong, raise it with the user — don't silently override.

**iOS-readiness constraint (NEW in v2):**
- All code in `Packages/` MUST compile for both `.macOS(.v14)` and `.iOS(.v17)` from day one.
- No `import AppKit`, no `NSEvent`, no macOS-only Foundation APIs in `Packages/`.
- Where platform-specific code is needed (e.g., FSEvents on macOS, DispatchSource on iOS), gate it with `#if os(macOS)` / `#if os(iOS)`.
- The macOS app target is the only place where AppKit/macOS-specific code may live.
- This is non-negotiable. Phase 4 will add an iOS app target reusing `Packages/`. We don't want to refactor then.

**When stuck:** stop and ask the user. Don't fabricate API signatures.

---

## 0. Repository structure (target end-state of Phase 1)

```
survaivor/
├── README.md
├── CLAUDE.md                          # Loads PROJECT_BRIEF + this prompt + per-phase notes
├── PROJECT_BRIEF.md
├── PHASE_0_1_PROMPT.md
├── CHANGELOG.md
├── SurvAIvor.xcodeproj/               # Or SurvAIvor.xcworkspace if SPM-only is awkward
├── SurvAIvor/                         # SwiftUI macOS app target (only target in Phase 0/1)
│   ├── App/
│   │   └── SurvAIvorApp.swift
│   ├── Views/                         # SwiftUI views, no business logic
│   ├── ViewModels/                    # @MainActor view models
│   ├── Resources/
│   └── Info.plist
├── Packages/                          # Local Swift packages — pure logic, iOS-compatible
│   ├── Core/                          # Models, protocols, errors
│   │   ├── Package.swift              # supports .macOS(.v14), .iOS(.v17)
│   │   ├── Sources/Core/
│   │   └── Tests/CoreTests/
│   ├── Storage/                       # GRDB schema, sqlite-vec, repositories
│   ├── VaultIO/                       # Markdown read/write + frontmatter + watcher (platform-gated)
│   ├── Search/                        # Hybrid search (FTS5 + vector)
│   ├── Embeddings/                    # NLContextualEmbedding wrapper
│   └── KnowledgeGraph/                # Node/link domain ops
├── docs/
│   ├── architecture.md
│   ├── schema.md
│   └── decisions/                     # ADRs, one per significant decision
└── scripts/
```

The `Packages/` directory is the cross-platform-friendly portion. **No SwiftUI, no AppKit imports inside `Packages/`. iOS compatibility is enforced from day one.**

---

## 1. Phase 0 — Foundation

### Goal
A runnable empty SwiftUI shell with the storage layer working. The user can open the app, see an empty state, manually create a node by typing, and the node is persisted to disk as a Markdown file and indexed in SQLite.

### Tasks

#### 0.1 — Project init
- Create the Xcode project: macOS app, SwiftUI lifecycle, deployment target macOS 14.0, language Swift 6 (or latest stable as of build time — verify), bundle id `com.<your-handle>.survaivor`.
- Initialize Git, write `.gitignore` (Xcode + DerivedData + user-specific files).
- Create the `Packages/` folder structure with empty SPM packages: `Core`, `Storage`, `VaultIO`, `Search`, `Embeddings`, `KnowledgeGraph`.
- Each package's `Package.swift` MUST declare both macOS and iOS support: `platforms: [.macOS(.v14), .iOS(.v17)]`.
- Wire each package as a local SPM dependency of the app target.
- App displays a placeholder window with title "SurvAIvor" and a single button "Verify setup" that prints to OSLog.

**Acceptance:**
- ✅ App builds and runs on macOS 14+.
- ✅ Each `Packages/<n>` builds independently with `swift build` for both macOS and iOS targets (`swift build --triple arm64-apple-ios17.0` or via Xcode iOS Simulator scheme).
- ✅ Each package has at least one passing placeholder test on both platforms.
- ✅ `git log` shows clean commits per task.

#### 0.2 — Vault directory + frontmatter convention
In `Packages/VaultIO`:
- Define struct `VaultLocation` (a URL) and a function to create the vault dir if missing. Default location: `~/Library/Application Support/SurvAIvor/vault/` on macOS; iOS will use `Documents/SurvAIvor/vault/` (placeholder for Phase 4).
- Define the canonical frontmatter schema for a node (use YAML). Fields:
  ```yaml
  ---
  id: "01HXYZ..."             # ULID, required
  type: "concept"              # concept|decision|pattern|gotcha|reference|journal
  title: "..."                 # required
  created: 2026-04-25T10:00:00Z
  updated: 2026-04-25T10:00:00Z
  tags: ["swift", "fsrs"]
  links: []                    # array of node ids this links to
  source:
    kind: "manual"             # manual|claude_code_session|url|file
    ref: ""                    # session id, URL, or path; empty for manual
  fsrs:
    stability: 0
    difficulty: 0
    due: null
    last_review: null
    state: "new"               # new|learning|review|relearning
  comprehension_score: 0.0     # 0..1
  shallow: false
  version_count: 1             # increments on each new version
  ---
  
  # Body in Markdown follows
  
  <!-- version 1, 2026-04-25T10:00:00Z, source=manual -->
  Body content here...
  ```
- Implement `FrontmatterCodec`: encode/decode between `Node` (Swift struct) and Markdown-with-frontmatter file content.
- Use a verified YAML library — search current options for Swift (`Yams` is a common choice; verify its current version and Swift 6 compatibility before pinning).
- Versioning markers (`<!-- version N, ... -->`) are part of the body. Phase 0 only handles version 1 (no actual versioning logic yet — that's Phase 2). But schema and parser must handle the marker format from day one.

**Acceptance:**
- ✅ Round-trip test: encode a `Node` to a string, decode it back, all fields equal.
- ✅ Test with edge cases: emoji in title, empty tags, multi-line body, body containing `---`.
- ✅ Decoding a malformed file returns a typed error, never crashes.
- ✅ Version marker is preserved on round-trip even though Phase 0 doesn't manipulate versions.

#### 0.3 — SQLite schema + GRDB integration
In `Packages/Storage`:
- Verify current GRDB.swift version and Swift 6 concurrency support before pinning.
- Define the database file location: `~/Library/Application Support/SurvAIvor/index.sqlite` on macOS; iOS placeholder for Phase 4.
- Use a `DatabaseQueue` (or `DatabasePool` if reads dominate writes — choose and justify in an ADR).
- Schema migrations via `DatabaseMigrator`. First migration:
  - `nodes` table: `id TEXT PRIMARY KEY`, `path TEXT NOT NULL UNIQUE`, `type TEXT NOT NULL`, `title TEXT NOT NULL`, `created INTEGER NOT NULL`, `updated INTEGER NOT NULL`, `comprehension_score REAL NOT NULL DEFAULT 0`, `shallow INTEGER NOT NULL DEFAULT 0`, `version_count INTEGER NOT NULL DEFAULT 1`.
  - `tags` table: `node_id TEXT NOT NULL REFERENCES nodes(id) ON DELETE CASCADE`, `tag TEXT NOT NULL`, primary key `(node_id, tag)`.
  - `links` table: `from_id TEXT NOT NULL`, `to_id TEXT NOT NULL`, `kind TEXT NOT NULL DEFAULT 'wiki'`, indexes on both columns.
  - `sources` table: `node_id TEXT NOT NULL REFERENCES nodes(id) ON DELETE CASCADE`, `version INTEGER NOT NULL DEFAULT 1`, `kind TEXT NOT NULL`, `ref TEXT NOT NULL`, `captured_at INTEGER NOT NULL`, primary key `(node_id, version)`.
  - `fsrs_state` table: `node_id TEXT PRIMARY KEY REFERENCES nodes(id) ON DELETE CASCADE`, `stability REAL`, `difficulty REAL`, `due INTEGER`, `last_review INTEGER`, `state TEXT NOT NULL DEFAULT 'new'`.
- FTS5 virtual table `nodes_fts` on `(title, body)` with content rowid linked to `nodes` (use external content for sync triggers).
- Insert/update/delete triggers to keep FTS5 in sync.
- Repository pattern: `NodeRepository` with `upsert(node:)`, `delete(id:)`, `find(id:)`, `all()`.
- **Note:** `sources` table supports versioning from day one (composite PK on `node_id + version`). Phase 0 only writes version=1 rows. Phase 2 will write more.

**Acceptance:**
- ✅ Migration runs idempotently; second run is a no-op.
- ✅ `upsert` creates or updates correctly.
- ✅ `delete` cascades to tags, links, sources, fsrs_state.
- ✅ FTS5 search returns the right node after insert/update.
- ✅ Tests run against an in-memory SQLite (use GRDB's `:memory:` queue).
- ✅ Schema works on both macOS and iOS Simulator.

#### 0.4 — sqlite-vec extension
- Verify current `sqlite-vec` distribution method for Swift. As of writing, it is a SQLite loadable extension distributed as a `.dylib` (or static library for iOS). Confirm current best practice for loading it via GRDB on both macOS and iOS before implementing — search `sqlite-vec GRDB Swift` or check the project's README.
- Add a `vec_nodes` virtual table for node embeddings. Dimension is determined by `NLContextualEmbedding`'s output — verify per locale, but plan around 512 (do not hardcode; query the model at runtime and persist the dimension in a config table).
- Implement `EmbeddingRepository` with `upsert(nodeId:, vector:)`, `delete(nodeId:)`, `nearest(vector:, k:)`.

**Acceptance:**
- ✅ Insert 100 random vectors, query for k=5 nearest by cosine, results are sane (closest matches actual nearest in a brute check).
- ✅ Deleting a node deletes its embedding.
- ✅ App launches with the extension loaded; failure is logged and surfaced clearly.
- ✅ Extension loads on iOS Simulator (verify in CI or local).

#### 0.5 — Vault ↔ DB sync
In `Packages/VaultIO`:
- File watcher using FSEvents (Apple framework) wrapped in a Swift async sequence on macOS. Use `#if os(macOS)`.
- iOS placeholder: leave a `NoOpWatcher` implementation conforming to the same protocol. Phase 4 will replace.
- On vault change (file added / modified / deleted / renamed): update DB.
- On DB change (write a node): write the Markdown file.
- Conflict policy for now: **last writer wins**, with the source of the change recorded. Document this in `docs/decisions/0001-sync-policy.md`.

**Acceptance:**
- ✅ Manually editing a Markdown file in Finder updates the DB within 2 seconds (macOS).
- ✅ Creating a node in the app writes a Markdown file matching the frontmatter spec.
- ✅ Deleting a Markdown file from Finder deletes the row.
- ✅ Rename = delete + insert (acceptable for v1).
- ✅ `WatcherProtocol` is platform-agnostic; iOS noop implementation compiles.

### Phase 0 Definition of Done
All of:
1. App builds, runs, persists, and reads back a manually created node.
2. Markdown vault and SQLite index stay in sync both directions on macOS.
3. All packages have ≥80% test coverage on the touched code paths.
4. **All packages compile cleanly for iOS too (verified via `xcodebuild -scheme <Package> -destination 'platform=iOS Simulator,...' build` or equivalent).**
5. Architecture decisions are recorded as ADRs in `docs/decisions/`.
6. `CLAUDE.md` exists and references `PROJECT_BRIEF.md` + this prompt.
7. The user has manually verified each acceptance bullet above.

**Stop here. Get explicit user sign-off before starting Phase 1.**

---

## 2. Phase 1 — Core knowledge management

### Goal
The app is a usable personal Markdown knowledge base, comparable to a stripped-down Obsidian. No AI yet.

### Tasks

#### 1.1 — `NLContextualEmbedding` integration
In `Packages/Embeddings`:
- Wrap `NaturalLanguage.NLContextualEmbedding` in a protocol `EmbeddingProvider`. Methods: `dimension: Int`, `embed(_ text: String) async throws -> [Float]`.
- Provide concrete `AppleNLEmbeddingProvider` for `.english` (verify supported locales at runtime; default to English, allow override via setting).
- Auto-embed on node create/update: pipeline `Node → EmbeddingProvider → EmbeddingRepository`.
- Re-embed strategy: only when body changes substantially (hash compare). Log otherwise.
- Persist the embedding dimension in a `app_config` table on first run; if it ever changes (locale switch), trigger a full re-embed with user confirmation.

**Acceptance:**
- ✅ Creating a node populates `vec_nodes` within 500ms for short text.
- ✅ Two semantically similar nodes have higher cosine similarity than two unrelated nodes — write a test with deliberate examples.
- ✅ Embedding fails gracefully if the model is unavailable (older locale, etc.) — node is saved without embedding, flagged in DB.
- ✅ Works on iOS Simulator (verify NLContextualEmbedding availability).

#### 1.2 — Hybrid search
In `Packages/Search`:
- Define `SearchQuery` (text, optional tags, optional types).
- Define `SearchResult` (nodeId, score, snippets, matched tags).
- Implementation: run FTS5 BM25 query and vector top-k in parallel, blend scores. Suggested initial blend: `score = 0.5 * normalized_bm25 + 0.5 * normalized_cosine`. Document the blend in the ADR; tune later.
- Tag/type filters apply as SQL WHERE clauses, not post-filter.
- Return at most 50 results sorted by blended score.

**Acceptance:**
- ✅ Pure keyword query returns expected node first.
- ✅ Pure semantic query (no exact keyword overlap) returns the right node.
- ✅ Combined query outperforms either alone on a small fixture corpus (write a test with ~20 fixture nodes).
- ✅ Search latency <200ms for a 5000-node vault on M-series Mac (benchmark, document the result).

#### 1.3 — `[[wiki-style]]` links + backlinks
In `Packages/KnowledgeGraph`:
- Parser that extracts `[[node-title]]` and `[[node-id|alias]]` from Markdown body.
- On node save, resolve link targets (by id or by title). Unresolved links get persisted as "broken" with the literal target string for later resolution.
- `LinkRepository`: `linksFrom(id:)`, `backlinksOf(id:)`, `unresolved()`.
- Title changes propagate: when a node is renamed, find references to the old title and update them in source files (with user confirmation; no silent rewrite).

**Acceptance:**
- ✅ Backlinks panel for a node X lists every node that references X.
- ✅ Renaming a node prompts the user before rewriting references.
- ✅ Broken links are listed in a dedicated view.

#### 1.4 — UI: Cmd+K quick-open + markdown editor
In `SurvAIvor/Views`:
- Three-pane layout: left sidebar (vault tree + tags + recent), center editor, right backlinks/metadata panel.
- Cmd+K opens a palette over the window. Type to search (uses Phase 1.2 hybrid search). Enter opens; Shift+Enter opens in a new tab; Cmd+Enter creates a new node with that title.
- Markdown editor: use a verified library or Apple's `NSTextView` with custom highlighting. Verify current best-of-breed Swift markdown editor before committing — search recent Swift markdown editor libraries.
- Live preview: split view with rendered HTML on right. Use `swift-markdown` to render.
- Tag autocomplete on `#` keystroke; node-link autocomplete on `[[`.

**Acceptance:**
- ✅ Cmd+K consistently lands on the right node in <5s for a 5000-node vault (benchmark).
- ✅ Editor handles 100kb+ files without lag.
- ✅ Preview updates within 100ms of a keystroke (debounced).
- ✅ Keyboard navigation works without a mouse end-to-end (create node, edit, link, search, delete).

#### 1.5 — Knowledge graph view
In `SurvAIvor/Views`:
- A separate view (Cmd+G) showing nodes as circles, links as edges, force-directed layout. Use Apple's `Canvas` + `TimelineView` for animation; no third-party graph libraries unless verified small and Swift 6 ready.
- Click a node = focus + open it.
- Filter by tag, type. Limit visible to 200 nodes for perf; show a banner if truncated.
- Color by node `type`.

**Acceptance:**
- ✅ Renders a 200-node graph at 60fps on M-series.
- ✅ Layout converges within 3 seconds.
- ✅ Click navigation works.

#### 1.6 — Source/provenance UI
- A "Source" section in the right panel for each node: shows kind + ref + a link to open it (URL → browser, file → Finder, manual → "Manually entered on …").
- Provenance is **mandatory** at node creation time. Default for manually-created nodes is `kind=manual`. UI must always show the field, never hide it.
- The Source section shows source per version (Phase 0 stores only v1; Phase 2 will populate more).

**Acceptance:**
- ✅ Cannot create a node without a source value.
- ✅ Editing source updates DB and frontmatter.

#### 1.7 — Version history viewer (NEW in v2)
In `Packages/VaultIO` and `SurvAIvor/Views`:
- Parser that splits a node body by `<!-- version N, ... -->` markers.
- "Show version history" button in the node metadata panel.
- Side-by-side or accordion view of all inline versions. Read-only in Phase 1 (no diff editor yet).
- Older archived versions in `.history/<node-id>.md` are loaded on demand if user clicks "Load older versions".

**Acceptance:**
- ✅ A node with 3 inline versions displays all 3 in correct order (newest first).
- ✅ Clicking "Load older" reads from `.history/` and displays.
- ✅ Phase 0 only-v1 nodes display normally with just 1 version visible.

### Phase 1 Definition of Done
All of:
1. The user can capture, edit, link, search, and visualize knowledge entirely manually.
2. Hybrid search latency <200ms on a 5000-node fixture vault.
3. All vault-DB sync flows tested end-to-end.
4. ≥80% test coverage on touched paths.
5. ADRs exist for: sync policy, blend ratio for hybrid search, vector dimension choice, version storage format.
6. User has used the app daily for at least one week without showstopper bugs.
7. CHANGELOG entries for every visible feature.
8. **All packages still compile for iOS Simulator after Phase 1 (no regression).**

---

## 3. CLAUDE.md template (create at the start of Phase 0)

```markdown
# Claude Code instructions for SurvAIvor

## Always

- Read `PROJECT_BRIEF.md` before any non-trivial change.
- Follow the active phase prompt (Phase 0/1: `PHASE_0_1_PROMPT.md`).
- Verify external library APIs against current docs (Context7 / web search) before using them. Do not trust your training data for versions or signatures.
- Write tests first when reasonable. Use Swift Testing.
- Commit per task with Conventional Commits.
- Honor Section 9 anti-patterns in the brief. If unsure, ask the user.
- All `Packages/` code must compile for both macOS 14+ and iOS 17+.

## Never

- Bypass the inbox / human-review step for any data flow (post-Phase 1, this matters most).
- Add cloud sync, telemetry, or third-party analytics.
- Introduce a dependency without justifying it in an ADR.
- Use `print` for production logging — use `os.Logger`.
- Skip writing tests because "it's a small change."
- Use AppKit, NSEvent, or macOS-only APIs inside `Packages/`. Gate platform code with `#if os(...)`.
- Silently rewrite node content. All updates are append-only versions.

## When stuck

Stop and ask. Do not fabricate APIs.

## Structure

- App code: `SurvAIvor/`
- Domain logic: `Packages/<n>/Sources/<n>/`
- Tests: `Packages/<n>/Tests/<n>Tests/`
- Decisions: `docs/decisions/NNNN-<slug>.md`
```

---

## 4. ADR template

Save as `docs/decisions/NNNN-<slug>.md`. Use sequential numbers.

```markdown
# ADR NNNN: <title>

- Status: proposed | accepted | superseded by ADR-XXXX
- Date: YYYY-MM-DD

## Context
Why this decision needs to be made.

## Decision
What we decided.

## Consequences
What follows from this. Trade-offs accepted.

## Alternatives considered
Briefly, with reasons rejected.
```

Required ADRs in Phase 0/1:
- `0001-vault-and-index-sync-policy`
- `0002-vector-dimension-and-provider`
- `0003-hybrid-search-blend-ratio`
- `0004-grdb-database-pool-vs-queue`
- `0005-markdown-editor-choice`
- `0006-version-storage-format` (NEW: inline markers + .history fallback)
- `0007-ios-compatibility-strategy` (NEW: how Packages/ stays iOS-clean)

---

## 5. Definition of "verify the latest"

When the prompt says "verify current version/API," do this:

1. Search the project's GitHub `README.md` and `CHANGELOG.md` (use web_fetch).
2. Search `https://swiftpackageindex.com/<owner>/<package>` for current Swift compatibility matrix.
3. If a Context7 MCP server is connected, use it. Otherwise, fetch the docs page directly.
4. Record the version chosen in `Package.swift` with a fixed minor (e.g., `from: "x.y.0"`) and note why in an ADR if it's not the latest.
5. Never copy code snippets older than 12 months without checking they still work.

---

## 6. Anti-patterns specific to this implementation

- ❌ Putting business logic in SwiftUI views.
- ❌ Force-unwrapping (`!`) outside of test fixtures.
- ❌ Catching errors with `try?` and silently dropping them.
- ❌ Singleton database instances accessed from anywhere — pass repositories via init.
- ❌ Doing I/O on `@MainActor`. Use `Task` and async/await; only update UI back on main.
- ❌ Hardcoding the vault path or DB path. They are configurable from day one (even if no UI yet).
- ❌ Using `String` for ids. Use a typed `NodeID` wrapper.
- ❌ Premature optimization (HNSW indexes, custom allocators). Brute-force first; benchmark; optimize only if proven necessary.
- ❌ macOS-only API usage in `Packages/`. Always platform-gate.
- ❌ Rewriting node body in place when content changes from a non-user source. Always append a new version.

---

## 7. What to do when Phase 1 finishes

1. Tag a release `v0.1.0`.
2. Write a `RETROSPECTIVE.md` covering: what worked, what didn't, what should change in Phase 2 plan.
3. Wait for the user to write `PHASE_2_PROMPT.md` (capture pipeline). Do not start Phase 2 without it.

---

## 8. One last thing

If at any point you find yourself thinking "the user probably wants X, I'll just do it," stop. Ask. The whole point of SurvAIvor is to keep the human as the boss. The same rule applies to building it.
