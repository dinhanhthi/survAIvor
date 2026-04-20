# SurvAIvor — Project Brief

> Survive the Vibe-Coding era. Learn from your AI sessions.
> The boss is the human. AI is the employee.

This document is the **single source of truth** for what SurvAIvor is, why it exists, and how it should be built. It must be loaded into Claude Code via `CLAUDE.md` at the start of every session. Every feature decision must trace back to a principle here.

> **Document version:** v2 (2026-04-26). Updates from v1: split active learning into 4 mechanisms (FSRS / Recap / Normal teach-back / Deep teach-back), added weekly digest model, added capture pipeline detail (granularity, detection rules, inbox lifecycle, conflict resolution), added versioning policy, added lint operation, added iOS companion as first-class platform, added platform split.

---

## 1. Identity

- **Name:** SurvAIvor
- **One-liner:** A local-first macOS + iOS app that converts your AI-coding sessions into durable, queryable, actively-rehearsed knowledge — so you stay smarter than your tools.
- **Owner:** Solo developer (fullstack web + DS/AI).
- **Primary platform:** macOS 14+ on Apple Silicon (build first, stabilize first).
- **Companion platform (Phase 4+):** iOS 17+, review-only client. Must be planned for in Phase 0 architecture (shared domain packages).
- **Cross-platform later:** Linux/Windows after macOS+iOS are stable. Architecture must isolate UI from domain logic to make this rewrite cheap.
- **License intent:** Personal use first. Open-source consideration later.

---

## 2. The Problem (research-backed)

Vibe coding produces working software while hollowing out the human's ability to understand, modify, or extend it. This is not opinion — it is measured.

**Cognitive offloading is real and physiological:**
- Kosmyna et al. (2025) used EEG to show LLM-assisted writers exhibit weaker neural network connectivity than search-engine or unassisted writers. They also score lowest on ownership of their output.
- Gerlich (2025): significant negative correlation between frequent AI use and critical thinking, mediated by cognitive offloading. Strongest in ages 17–25.
- Lee et al. (2025): knowledge workers using generative AI for analytical tasks report reduced critical thinking effort and problem-solving confidence.

**The forgetting curve is brutal:**
- ~50% of new information is lost within 1 hour, ~100% within a week, without active reinforcement (Ebbinghaus, replicated repeatedly).

**Vibe coding produces "illusion of competence":**
- Vibe-Check Protocol paper (2026) measured this: vibe coders successfully produce functional apps while showing a troubling inability to explain, modify, or extend the code. Functional output is not learning.

**What works (also research-backed):**
- **Desirable difficulties** (Bjork & Bjork): retention requires effort during encoding. Removing the struggle removes the learning.
- **Productive failure** (Kapur 2008): brief unsuccessful attempts before being given the answer measurably improve retention.
- **Metacognitive scripts + teach-back** (Mitigating Epistemic Debt paper, 2026): forcing learners to explain back at SOLO Level 3 depth restores germane processing measurably (verified via EEG).
- **FSRS** (Free Spaced Repetition Scheduler): state-of-the-art SRS, ~99.6% superiority over SM-2 on a 350M-review benchmark. Reduces reviews 20–30% for the same retention.
- **Karpathy LLM-Wiki pattern:** the agent compiles a structured wiki of decisions and patterns; the human owns the canonical knowledge. **Important nuance:** Karpathy advocates compiling and maintaining (cross-references, lint, bookkeeping), NOT autonomous rewriting. Self-feeding rewrites cause "knowledge laundering" — append-only versioning prevents this.

These findings define the design philosophy below.

---

## 3. Anti-Cognitive-Offloading Principles (the design philosophy)

These are non-negotiable. Every feature is judged against them.

1. **The human is the boss, the AI is the employee.** UI language, defaults, and friction must reinforce this. Never frame AI output as authority.

2. **Capture is cheap. Understanding is earned.** Saving content is one click. Marking it "understood" requires a teach-back gate. The two are different operations and must be visually distinct.

3. **No silent acceptance.** Anything that flows from an AI session into the knowledge base must pass through a human-attended moment — even a 5-second reflection. Pure auto-pipe creates the same offloading problem we are trying to solve.

4. **Productive struggle before reveal.** When the user asks a question the system can answer from its KB, default to giving the user 30–120 seconds to attempt it themselves first. Skippable, but skipping is logged.

5. **Skipping has consequences, not friction.** Users can skip teach-back or productive failure — but the affected concept gets marked `shallow` and its quiz frequency doubles. We respect autonomy; we charge for it.

6. **Forgetting is the default; remembering is engineered.** Every persisted concept enters the FSRS schedule. No "save and forget" path exists.

7. **Source attribution is mandatory.** Every node carries provenance: which session, which URL, which file. No floating facts. This kills AI slop accumulation.

8. **Local-first, file-over-app.** Knowledge is stored as plain Markdown that survives the app. The database is an index, never the source of truth. The user can `grep` their vault.

9. **Latest tech, but verified.** When implementing, the agent must verify current library APIs and versions via Context7 / docs / web search rather than relying on memory.

10. **The app is honest about uncertainty.** When semantic search returns weak matches, say so. When the local LLM is unsure, say so. Confidence theatre is offloading by another name.

11. **Append-only knowledge evolution.** Nodes are never silently rewritten. Updates create new versions; old versions remain accessible. This prevents self-feeding LLM drift ("knowledge laundering").

12. **No daily nag.** App is silent by default. One weekly digest at user-chosen time. User pulls value from app; app does not push guilt.

---

## 4. Target User

A working developer (web/data) who:
- Uses Claude Code / Cursor / Copilot daily.
- Notices their understanding decaying as their output speeds up.
- Wants to remain employable in a world where AI writes most code.
- Values privacy and ownership of their notes.
- Is comfortable with a slightly opinionated tool over a fully customizable one.

**Not the target:** people who want a frictionless productivity booster. SurvAIvor is intentionally not frictionless where research says friction helps.

---

## 5. Vision (a day with SurvAIvor)

**Monday morning:** weekly digest notification arrives. "Tuần qua: 23 nodes captured. 8 cards due this week. 2 normal teach-backs ready. 1 deep teach-back invitation. Lint found 3 things to review." User opens app when ready. Spends 15 minutes. Done.

**Tuesday-Sunday:** App is silent. User uses Claude Code normally. Hooks fire in background, sending session data to SurvAIvor inbox. User never interrupted mid-flow.

**End of a meaningful session:** A small inbox sheet appears in SurvAIvor: "5 candidate captures from this session". User reviews quickly: 2 decisions, 1 pattern, 1 gotcha — accepts 4, edits 1, rejects 0. Each becomes a node. The session itself becomes 1 `journal` node parent.

**Mid-week, user gets curious:** clicks "Challenge me" button. App picks a random mode (today: 3 multi-choice questions from recent nodes). User scores 2/3. The failed concept gets marked `shallow`, quiz frequency x2.

**User needs to recall something old:** Cmd+K → keyword → hybrid search returns the right node in <200ms. Opens it, reads the rationale they wrote themselves. Uses it. Continues coding.

**Weekly (deep teach-back):** Sunday afternoon, user opts into deep teach-back. 20 minutes on one central concept. Writes 300 words "in my own words". App saves as Feynman artifact.

**On commute (iOS app):** user reviews recap takeaways one-handed, scrolls through 5 cards, marks 1 as "still unclear" → sent to desktop's normal teach-back queue.

That is success.

---

## 6. Feature Roadmap

Phases are sequential. Each phase has a strict Definition of Done before the next begins. No phase merges into the next prematurely.

### Phase 0 — Foundation (target: 1–2 weeks)
Goal: a runnable empty shell with clean architecture and the storage layer working.
- SwiftUI macOS app + Swift Package Manager packages for domain logic.
- **Domain packages must be iOS-compatible from day one** — no AppKit, no macOS-only APIs in `Packages/`.
- Markdown vault directory + file watcher (FSEvents on macOS).
- SQLite schema (GRDB.swift) + FTS5 + sqlite-vec for vectors.
- Basic node CRUD via paste/manual entry.
- Frontmatter convention defined and enforced — including version markers and shallow flag.

### Phase 1 — Core knowledge management (target: 3–4 weeks)
Goal: the app is a usable personal knowledge base, even before any AI.
- Auto-embedding on save using `NLContextualEmbedding`.
- Hybrid search: FTS5 (BM25) + vector cosine + tag filter, blended ranking.
- Bidirectional links via `[[wiki-style]]` syntax + backlinks panel.
- Knowledge graph view (SwiftUI Canvas).
- Cmd+K quick-open, markdown editor with live preview, tag autocomplete.
- Source/provenance metadata UI.
- **Version history viewer** for nodes (UI to see previous versions inline).

### Phase 2 — Capture pipeline (target: 2–3 weeks)
Goal: knowledge flows in from real AI workflows, not just manual entry.

**Granularity model (B1.c hybrid):**
- Each Claude Code session → one auto-generated `journal` node (raw context preserved).
- During inbox review, user picks specific concepts → child nodes (decision/pattern/gotcha) linked to journal parent.
- Journal node always exists even if user picks no concepts.

**Detection rules:**
- **Decisions**: rule-based detection of phrases like "I'll use X because", "chose X over Y" + LLM extracts tradeoff rationale.
- **Patterns**: same code structure appearing ≥3 times in session OR cross-session.
- **Debug sessions**: ≥3 consecutive turns about the same error/issue. Output: issue + final fix + WHY it failed.

**Components:**
- Local HTTP server in the app (Hummingbird 2 or NIO).
- Claude Code hooks (SessionStart / Stop / PreCompact / PostToolUse) → POST to local server.
- Browser extension (WXT framework, separate repo) → POST to local server.
- MCP server (separate repo) exposing `capture_knowledge` tool for any MCP client.

**Inbox lifecycle:**
- Auto-archive after 14 days unreviewed (move to "Expired" folder, not deleted).
- Soft pressure banner when inbox >50 items.
- Archive remains searchable.

**Conflict resolution (3-tier by cosine similarity):**

| Cosine | Action |
|---|---|
| <0.7 | Create new node |
| 0.7–0.85 | Inbox flag "Possibly related to [Node X]". User chooses: New / Link / Merge |
| 0.85–0.95 | Inbox flag "Likely duplicate of [Node X]". Default: append as version. User can override |
| ≥0.95 | Auto-append as version (never rewrite original) |

**Versioning storage:**
- Inline markers (e.g., `<!-- version 2, captured 2026-04-25 from claude_code_session_xyz -->`) within the same Markdown file.
- Keep last 5 versions inline. Older versions auto-archived to `<vault>/.history/<node-id>.md`.
- Source attribution preserved per version.

### Phase 3 — Active learning (target: 4 weeks; this is the heart of SurvAIvor)
Goal: the anti-cognitive-offloading mechanics work end-to-end.

**Four learning mechanisms:**

| Mechanism | Cadence | Mode | Effort | Purpose |
|---|---|---|---|---|
| **FSRS reviews** | Cards due per FSRS schedule (mostly weekly+ intervals) | Active recall (flashcard) | Variable | Atomic facts, fight forgetting curve |
| **Recap (takeaways)** | User-initiated only, no schedule | Passive consume | 1–2 min | Refresh surface knowledge |
| **Normal teach-back** | App invites every 3 days | Active write + LLM grade | 5–10 min | Verify concept understanding |
| **Deep teach-back** | App invites every 7 days | Active write Feynman artifact | 15–20 min | Refine understanding, create artifact |

**Notification policy:**
- **No daily notification ever.**
- Weekly digest on user-chosen day (default Monday). Single push notification + optional email.
- Digest content: captures count, cards due, teach-backs due, lint findings, "Did you know..." snippet.
- User can also enable random "Did you know..." pull (1 takeaway from vault, weekly, optional).

**Productive failure (cross-cutting):**
- Mode toggle: **Focus mode** (productive failure off) vs **Learning mode** (on).
- Auto-detect: Claude Code session active → auto-switch to Focus mode.
- User can manually override.

**Teach-back grading stack (combo):**
1. **Embedding cosine baseline** filter: <0.3 fail immediately, >0.85 flag suspicious.
2. **Adversarial rubric**: LLM is asked to find weaknesses (3 specific gaps), not grade. Lenient-grader problem mitigated.
3. **Multi-question per concept**: 2–3 questions per concept, must pass 60–80%.

**Failure handling:**
- 1st failure: mark `shallow`, reschedule sooner, do NOT show answer.
- 2nd failure on same concept: show the gap (covered X, missing Y).
- Pass next teach-back → clear `shallow`.

**Question selection (which nodes enter teach-back queue):**
- Hybrid: priority queue by recency + importance (node with many backlinks = central concept = high priority; `shallow` = high priority) + 1 random injection per session to prevent gaming.

**Deep teach-back specifics:**
- 1 central concept per week
- 5–7 questions: definition, example, counter-example, trade-offs, edge case, "when NOT to use", connection with other concepts
- Force user to link this concept to ≥2 other concepts in vault if not already linked
- Output: 200–500 word "in your own words" explanation saved as Feynman artifact (a new linked node, type `journal`)

**Recap presentation:**
- Cards-style takeaways (1–2 sentences each)
- User clicks to expand to full node
- No grading, no skip penalty — pure passive consume

**Challenge me button:**
- App randomly picks one mode each click: random flashcard / mini-quiz of 5 / 1 productive failure question / topic-pick (user types a topic).
- User-initiated only.

**Auto-flashcard generation:**
- MLX Swift LLM (Qwen3-4B-Instruct or similar) generates 2–3 flashcard candidates per node on save.
- User reviews and approves in inbox before cards enter FSRS.

### Phase 4 — Metacognition, lint, mobile companion (target: 4–5 weeks)

**Weekly lint operation:**
- Background job, Sunday night or before Monday digest.
- Detects: orphan nodes (no in/out links), broken links, contradictions between nodes (LLM check), high-cosine clusters (5+ nodes >0.9 = consolidation candidates), mentioned-but-missing concepts.
- **Only proposes**, never auto-fixes.
- Findings shown in weekly digest.

**Analytics & metacognition:**
- Skill decay heatmap (concept-level).
- Weekly retrospection prompt: "What did you actually learn this week?"
- VCP-inspired metrics: comprehension score, false-alarm rate, modify/extend score.
- Diff with 7/30/90 days ago: which concepts have you forgotten?

**iOS companion:**
- iOS 17+, Apple Silicon devices (iPhone, iPad).
- **Capabilities:** recap, FSRS reviews, normal teach-back (short answers), challenge me, push notifications, view nodes (read-only).
- **Not on mobile:** capture, deep teach-back, graph view, import, vault editing, lint review.
- Sync with desktop via iCloud Drive (default) or local Git-based (advanced). User chooses.

### Phase 5 — Polish, import, & extensibility (ongoing)

**Import features (first-class, not one-time):**
- Import from Obsidian vault (markdown + frontmatter conversion).
- Import from Notion export (HTML/Markdown).
- Generic markdown import with format auto-fix (headers, links, frontmatter normalization).
- Deduplicate against existing vault using same 3-tier cosine logic as capture pipeline.
- Conflict UI: "12 of 247 imported notes match existing nodes. Review."

**Other:**
- Plugin system (Swift dynamic frameworks or scripting bridge — TBD).
- Cross-platform expansion: at this point, port domain packages to Tauri or rewrite UI per platform.

---

## 7. Tech Stack & Rationale

Every choice below has a "why this and not the alternative" attached. If a future decision overturns one of these, the rationale must be explicitly addressed.

| Layer | Choice | Why this | Why not the alternative |
|---|---|---|---|
| App shell (macOS) | SwiftUI (macOS 14+) | Native perf, accessibility, system integration | Tauri/Electron: cross-platform but extra runtime; user prefers macOS-first native |
| App shell (iOS) | SwiftUI (iOS 17+) | Same SwiftUI codebase, share views where possible | UIKit: more verbose, less code reuse |
| Architecture | MVVM with `@MainActor` ViewModels + pure-Swift domain packages | Apple's recommended pattern; testable; UI/logic separation = cheap port | TCA: powerful but high learning curve and lock-in |
| Persistence | GRDB.swift (SQLite) | Battle-tested, FTS5, thread-safe, actor-friendly, SQL is portable, works on iOS | SwiftData: newer, less mature, less control over FTS/vec extensions |
| Vector store | sqlite-vec (loadable extension via GRDB) | Simple, embedded, ~10–50k items perf is fine, works on iOS | LanceDB: third-party Swift binding (single maintainer = risk) |
| Vector search algo | brute-force cosine (sqlite-vec built-in) | Adequate <100k items; simpler; no index maintenance | HNSW/IVF: premature for personal scale |
| Markdown | Apple `swift-markdown` + custom front-matter (YAML) parser | Apple-official, AST access, fast | Pure regex: brittle |
| Embeddings (Phase 0/1) | `NLContextualEmbedding` (Apple, built-in) | Zero deps, on-device, multi-language, deterministic, works on iOS | Ollama / MLX embeddings: more complex setup; quality gain not worth the risk early |
| LLM (Phase 3+) | mlx-swift + mlx-swift-lm, in-process, Qwen3-4B-Instruct-4bit (or similar) | Native, fast on Apple Silicon, no separate process, works on iOS | Ollama HTTP: separate process, slower cold start, doesn't work on iOS |
| SRS | `swift-fsrs` (open-spaced-repetition org) | Official OSR org, FSRS-5; FSRS-6 not yet in Swift but FSRS-5 is excellent | Custom impl: don't reinvent SRS |
| Local HTTP (Phase 2) | Hummingbird 2 | Modern Swift HTTP server, async-await native | Vapor: heavier; we don't need an ORM here |
| File watching | FSEvents (macOS) / DispatchSource (iOS, fallback) | Native | Polling: wasteful |
| Sync (Phase 4 mobile) | iCloud Drive (default) + Git-based (advanced) | Both portable, both local-first compatible | Custom server: violates local-first |
| Logging | OSLog (`os.Logger`) | Native, structured, integrates with Console.app | Print statements, third-party loggers |
| Tests | Swift Testing framework + XCTest where needed | Modern Swift testing API | XCTest only: older style |

**Verification rule:** when implementing, the agent must check current versions and APIs of `mlx-swift`, `mlx-swift-lm`, `swift-fsrs`, `GRDB.swift`, `sqlite-vec`, `Hummingbird`, and `swift-markdown` via web search or Context7. Do not commit to versions in this doc — they change.

---

## 8. Glossary

Precise vocabulary prevents drift.

- **Vault:** the directory of Markdown files on disk. Source of truth for knowledge content.
- **Node:** one Markdown file in the vault representing one unit of knowledge. Has YAML frontmatter (id, type, source, created, tags, links, fsrs_state, comprehension_score, shallow flag, version count).
- **Node type:** `concept` | `decision` | `pattern` | `gotcha` | `reference` | `journal`. Drives default behavior (quiz format, schedule).
- **Version:** an append-only revision of a node's body. Stored inline with markers. Last 5 inline; older archived to `.history/<node-id>.md`. Original source attribution preserved per version.
- **Capture:** the act of pulling content from outside (Claude Code session, browser, paste) into the inbox, not yet a node.
- **Inbox:** queue of captured-but-not-reviewed items. Human must review before they become nodes. Auto-archive after 14 days.
- **Promotion:** the act of converting an inbox item into one or more nodes. Always human-attended.
- **Journal node:** auto-generated parent node for one Claude Code session. Always exists even if no child concepts are extracted.
- **FSRS reviews:** active recall flashcards scheduled per FSRS-5 algorithm. Mostly weekly+ intervals. No daily nag.
- **Recap:** passive consume of recent takeaways. User-initiated only. No grading, no skip penalty.
- **Normal teach-back:** active write exercise every 3 days, 2–3 nodes, 2 questions each, LLM-graded.
- **Deep teach-back:** active write Feynman artifact weekly. 5–7 questions on 1 central concept. Output saved as new linked node.
- **Productive failure:** a 30–120s timed window where the user attempts to answer before the answer is revealed. Cross-cutting feature, toggle via Focus/Learning mode.
- **Focus mode / Learning mode:** toggle for productive failure. Auto-switches to Focus when Claude Code session is active.
- **Challenge me:** user-initiated mid-week practice. App randomly picks one mode (flashcard / mini-quiz / productive failure / topic-pick).
- **Shallow:** flag on a node meaning the user skipped or failed teach-back/productive failure. Doubles quiz frequency until cleared by passing a teach-back.
- **Comprehension score:** 0–1, derived from teach-back evaluations + quiz performance + recency. Drives prioritization.
- **FSRS state:** per-node scheduling state (stability, difficulty, last review, next due). Standard FSRS-5 fields.
- **Provenance:** the source of a node — `claude_code_session_id` | `url` | `file_path` | `manual`. Required, never optional.
- **Hybrid search:** FTS5 keyword + vector cosine + tag/type filter, blended into a single ranked result.
- **Capture source:** the integration that produced a capture (`hook:claude_code` | `extension:chrome` | `mcp:claude_chat` | `manual`).
- **Lint:** weekly background pass detecting graph health issues (orphans, broken links, contradictions, consolidation candidates). Proposes; never auto-fixes.
- **Weekly digest:** single notification summarizing the week (captures, due cards, teach-backs, lint findings). User-chosen day. Default Monday.
- **Companion (mobile):** iOS app for review only. Read + review + light teach-back. No capture, no deep work.

---

## 9. Anti-Patterns / Non-Goals

Things the app must **not** do, even if they would be easy or popular.

- **No auto-summarize-and-save.** Every capture passes through the inbox for human review.
- **No "ask AI" everywhere.** Chat-with-vault feature, if implemented, must gate behind productive failure (not yet finalized; pending discussion).
- **No gamification rewards (streaks, points) before Phase 4.** They reward showing up, not learning.
- **No daily notifications.** Weekly digest only.
- **No cloud sync in Phase 1–3.** iCloud comes only in Phase 4 for mobile sync; even then it's optional, not required.
- **No telemetry.** Zero. The user's learning is private.
- **No real-time collaboration.** Personal tool. Multi-user is out of scope forever.
- **No Markdown rendering bells and whistles.** GitHub-flavored Markdown + math + mermaid is plenty. We're not building Notion.
- **No proprietary file format.** Markdown + frontmatter, period.
- **No "AI suggests links you should make" as default.** Available as opt-in only. Default is user creates links.
- **Don't allow easy skip of teach-back.** Skipping is allowed (autonomy) but visibly marked `shallow`.
- **No silent rewriting of node content.** All updates are append-only versions.
- **No auto-merge of inbox items into existing nodes** below cosine 0.95. And even at 0.95+, append as version, never rewrite.

---

## 10. Success Metrics (for the developer-user, measured monthly)

- **Recall accuracy** on FSRS reviews stays >85% over 6 months for non-shallow concepts.
- **Capture-to-promotion ratio**: at least 60% of captured items become nodes (if lower, capture quality is poor).
- **Teach-back pass rate** trends up over time per node type.
- **Time-to-find** a known concept via Cmd+K stays under 5 seconds.
- **Cross-session reuse**: the user references nodes from past sessions in new Claude Code sessions, measurably increasing.
- **Subjective**: the user feels they understand their codebase better than they did one month ago.
- **App engagement**: user opens app ≥2 times per week (target). If <1/week → digest is failing or app is forgotten.

---

## 11. Out of Scope (forever or for now)

| Item | Status |
|---|---|
| Multi-user / team features | Forever out |
| Cloud-only mode | Forever out |
| Windows/Linux native | Out for v1; cross-platform later via Tauri or rewrite |
| Built-in code execution | Out (use existing IDEs) |
| Built-in PDF annotation | Out for v1 |
| Audio/video recording | Out for v1 |
| Real-time collaboration | Forever out |
| Custom LLM training | Forever out |
| Daily streak gamification | Forever out |

---

## 12. Open questions (still in active design)

These are explicitly NOT decided yet. Phase 0/1 do not depend on them; Phase 2/3 will.

- **Chat-with-vault**: should the app have a chat interface to query the vault? If yes, how is productive failure enforced? Multi-choice mode? Recall mode? Direct mode with logging? (Pending discussion; will be settled before Phase 3.)
- **Knowledge graph use cases**: graph view exists in Phase 1. But should it support prerequisite tracking? Auto-suggest links? (Pending discussion.)
- **Failure modes**: what does the app do when the user is inactive 30+ days? When user skips teach-back 5 times in a row? (Pending discussion.)
- **Import details**: which fields to map from Obsidian/Notion to SurvAIvor frontmatter? Conflict policy at import time? (Pending discussion.)

When these are resolved, this section will move them to relevant sections above.

---

## 13. How to use this document

When starting a session in Claude Code:
1. Load this file via `CLAUDE.md` reference (or paste relevant section).
2. Load the active phase prompt (`PHASE_0_1_PROMPT.md`, `PHASE_2_PROMPT.md`, etc.).
3. At session end, capture decisions back into this brief or a CHANGELOG. Do not let the AI decide what is canonical — the human does.

When in doubt, return to **Section 3 (principles)**. If a feature can't be justified by a principle, don't build it.
