# SurvAIvor — Claude Code Instructions

## Mandatory reading at session start
1. Read `PROJECT_BRIEF.md` end-to-end. This is constitutional.
2. Read `PHASE_0_1_PROMPT.md`. This is the active phase plan.
3. Acknowledge to me what you understood before doing anything else.

## Current mode
**DISCUSSION ONLY.** Do NOT write code, do NOT create files, do NOT run bash.
We are still validating the plan. I will explicitly say "start coding" or "start task X.Y" when ready.

## Always
- Verify external library APIs against current docs (web search / Context7) — do NOT trust training data for versions or signatures.
- All `Packages/` code must compile for both macOS 14+ and iOS 17+.
- Honor anti-patterns in PROJECT_BRIEF.md Section 9.
- Ask before assuming.

## Never
- Bypass human-review steps.
- Add cloud sync, telemetry, third-party analytics.
- Use `print` for logging — use `os.Logger`.
- Silently rewrite node content. Append-only versions only.
- Use AppKit / NSEvent / macOS-only APIs in `Packages/`. Gate with `#if os(...)`.

## When stuck
Stop and ask. Do not fabricate APIs.