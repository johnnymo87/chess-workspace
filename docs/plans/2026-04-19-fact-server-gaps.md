# Plan: Close the biggest gaps in `chess-support-mcp`

**Status:** Superseded for gaps 1, 2, 4 by the enriched-status work — see
[`2026-04-19-enriched-status-design.md`](2026-04-19-enriched-status-design.md)
and [`2026-04-19-enriched-status-plan.md`](2026-04-19-enriched-status-plan.md).
Implemented on fork branch `github.com/johnnymo87/chess-support-mcp@enriched-status`.
Gaps 3 (SAN input / `load_pgn`) and 5 (undo/simulate) are still open and
would be the natural next targets for the same fork.

The original pre-compaction plan text follows for historical reference.

**Created:** 2026-04-19, at the end of a session that (a) successfully analyzed a real game position using the current MCP, (b) wrote project-local skills documenting the workspace, and (c) enumerated gaps in the fact server.

## Why this plan exists

The user ran a real analysis session with me (the "Nxa5 wins the queen" game — PGN + the full analysis transcript are in the pre-compaction conversation history; the position that was analyzed came from `games/` at the time). During it, I hit real pain points. Afterwards, we wrote documentation. Now we're preparing for compaction and the user specifically asked:

> pass your future self everything it needs to **track all and tackle some of the pain points** next session (possibly with first some brainstorming and planning post-compaction)

This plan is the "track all and tackle some" handoff. Read this first, then `.opencode/skills/improving-the-fact-server/SKILL.md`, then decide scope with the user.

## Hard constraints (DO NOT VIOLATE)

1. **No chess engine.** Not Stockfish, not Leela, not Maia, not a cloud API. Not hidden behind an "evaluation" abstraction. If a proposed change would give the LLM a move recommendation or a numeric eval, reject it. This is the entire reason the workspace exists — re-read `AGENTS.md` and `/tmp/research-chess-reasoning-aids-answer.md` before questioning this.
2. **Fact server, not oracle.** Everything we add must be a mechanical derivation (what python-chess can compute from the board alone). No heuristics, no learned models, no opening-book lookups that name moves.
3. **PR upstream before forking.** The maintainer (`danilop/chess-support-mcp`) has stated philosophy alignment; a small, clean PR is the preferred first path. Fork only if the PR stalls or is declined.

## Where to start the next session

**Don't touch code first.** Ask the user:

1. "Do you want to scope this down to one gap, or do a broader 'enriched status' PR?" — my recommendation below is #2, but it's their call.
2. "Are we PR'ing upstream or forking?" — start with PR.
3. "Do you want me to use the `brainstorming` skill first to pin down the exact shape of the new tools?" — probably yes, especially around simulate/undo which has real API design tradeoffs.

Only after those answers should you start designing or coding.

## Gap ranking (recap — full details in `improving-the-fact-server` skill)

Ordered by leverage-per-complexity, most valuable first. My recommendation: **tackle items 1 + 2 + 4 as a single bundled PR** ("enriched status"), then 3 as a separate PR, then 5 later if it keeps coming up.

1. **`legal_moves_uci` / `legal_moves_san` in `get_status`** — highest leverage, trivially cheap. The research answer calls this out by name. Without it, the LLM generates moves from imagination and validates one-by-one with `is_legal`. With it, candidate generation is constrained at the source.

2. **Pins + checkers + attackers** — fields: `pinned_white`, `pinned_black`, `checkers`, plus a tool `get_attackers_to_square(square)`. python-chess has `board.pin(sq)`, `board.checkers()`, `board.attackers(color, sq)`. These are the fields that would have caught the two motivating bugs (Nb3-while-pinned, Bxa2-off-diagonal).

3. **SAN input** — either `add_move` accepts SAN, or add `add_move_san`, or the bigger win: **`load_pgn(pgn_string)`** that replays the whole game server-side. In the session transcript, SAN→UCI conversion cost me a ~40-call cascade of failures that burned real tokens and required a full reset+replay.

4. **Material count** — `material: {white: {Q,R,B,N,P}, black: {...}}` in status. Trivial; prevents arithmetic drift errors.

5. **Undo / simulate** — medium leverage, real API design work. Three options (from `improving-the-fact-server`):
   - `undo_last_move` — simplest, covers 80%.
   - `peek(uci_sequence)` — applies + returns status + restores. Cleanest UX but needs careful state isolation.
   - Parallel sandbox position.

   My weak preference: `undo_last_move` first (ship something), `peek` later if needed.

## Suggested first PR shape

**Title:** "Enrich `get_status` with legal moves, pins, checkers, material"

**Changes:**
- Add to `get_status()` return object:
  - `legal_moves_uci: string[]`
  - `legal_moves_san: string[]`
  - `pinned_white: string[]` (squares)
  - `pinned_black: string[]` (squares)
  - `checkers: string[]` (squares of pieces giving check, if any)
  - `material: {white: {...}, black: {...}}`
- New tool: `get_attackers_to(square: string)` → `{white: string[], black: string[]}` listing source squares of attackers of each color.

**Rationale to include in the PR:** The research synthesis at <link or summary> identifies legal-move visibility as the single highest-leverage change for reducing LLM legality errors; this PR adds that plus the minimal set of derived-fact fields the companion skills need.

**Size estimate:** ~80–120 lines of Python. All primitives exist in python-chess. Tests should mirror existing test style in `tests/`.

## Testing approach (post-implementation)

Before claiming the PR is ready:

1. Set `opencode.json` to point at a local checkout or your fork branch.
2. Run the same PGN from the compacted session's analysis (should be accessible from git history or the user can re-paste it). Replay up to the decision position.
3. Confirm: `get_status` now returns `legal_moves_san` including `Nxa5`.
4. Confirm: with a constructed position where Nd2 is pinned by Bh6, `pinned_white` contains `d2`.
5. Run the existing upstream test suite — don't regress.

## What NOT to do

- Don't add `best_move`, `evaluate`, `threatened_squares` (as evaluation, vs. `attackers_to` which is derivation), opening-book lookups, or tablebase queries. See `improving-the-fact-server` §"What NOT to add" for the full list.
- Don't unilaterally rewrite `chess-support-mcp` into something bigger. The maintainer's minimal surface is a feature, not a bug.
- Don't skip the brainstorming/plan step for item 5 (undo/simulate) — there's real design choice there.

## Handoff

If the user wants to proceed:
- First invoke `brainstorming` skill to lock down scope + API shape.
- Then invoke `writing-plans` to produce a concrete implementation plan.
- Then `executing-plans` or `subagent-driven-development` depending on size.

If the user wants to defer (e.g. "just keep using what we have"), that's fine — the skills already document the discipline that makes the current toolset workable.
