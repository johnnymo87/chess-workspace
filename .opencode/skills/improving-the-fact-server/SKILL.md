---
name: improving-the-fact-server
description: Use when a chess analysis session hits a gap in chess-support-mcp (missing legal-move list, no pins, no undo, etc.) and you're tempted to plug it. Catalogues known gaps, their priority, and the constraint that keeps this workspace engine-free.
---

# Improving the Fact Server

`chess-support-mcp` is doing ~70% of the job. This skill is the standing list of gaps between what the research brief (`/tmp/research-chess-reasoning-aids-answer.md`) said we need and what the upstream MCP actually exposes. It exists so that a future session can quickly see what's been considered, what's been decided, and what would be worth actually implementing.

## The hard constraint

**No engine.** No Stockfish, no Leela, no evaluation, no "best move" tool. That's the point of the whole workspace. Anything added must be a mechanical derivation from `python-chess` (or equivalent) — a fact, not a judgment. If a proposed improvement would give the LLM a move recommendation, reject it.

See `../../AGENTS.md` and the research answer for the philosophical reasoning. It's not arbitrary.

## Known gaps (current state: 2026-04)

Ranked roughly by leverage (how much each would have helped real sessions). Verify the current state of the upstream tool before acting — the toolset listed in `../../AGENTS.md` is what we actually have.

### 1. No `legal_moves` in `get_status`  — HIGH LEVERAGE

**What's missing:** `get_status` returns FEN + `pieces` + check flags, but not the list of legal moves in the current position.

**Why it matters:** This is the single highest-leverage change in the research answer. Without a legal-move list, the LLM generates candidates from imagination and validates them one by one with `is_legal`. That works but leaves a gap where the LLM forgets to validate and hallucinates.

**What we want:** A `legal_moves_uci` and `legal_moves_san` field in `get_status` (or a dedicated `list_legal_moves` tool). Cost: trivial — python-chess has `board.legal_moves` and `board.san(move)`.

**Status:** Not in upstream. Worth a PR to `danilop/chess-support-mcp`, or a local fork.

### 2. No pins / checkers / attackers derivation  — HIGH LEVERAGE

**What's missing:** No `pinned_white`, `pinned_black`, `checkers`, `attackers_to_square`.

**Why it matters:** These are exactly the fields that would have caught the original motivating bugs (pinned-Nd2, Bxa2-off-diagonal). Calling `is_legal` catches illegality after the fact; exposing pins lets the LLM see the constraint up front.

**What we want:** python-chess has `board.pin(square)`, `board.attackers(color, square)`, `board.checkers()`. A pair of tools (`get_pins`, `get_attackers_to`) or a single enriched-status tool would do it.

**Status:** Not in upstream. Same fix path as (1).

### 3. No SAN input — MEDIUM LEVERAGE

**What's missing:** `add_move` takes UCI only; a PGN or SAN move list must be hand-converted by the LLM.

**Why it matters:** See `replaying-a-pgn` — SAN→UCI conversion is tedious and error-prone, and a single mistake cascades badly. python-chess's `board.parse_san("Nxe4")` would eliminate this.

**What we want:** Either make `add_move` accept SAN, or add a sibling `add_move_san`, or a `load_pgn(pgn_string)` that replays the whole game server-side.

**Status:** Not in upstream. `load_pgn` is probably the highest-leverage version — one tool call instead of 40.

### 4. No material count — LOW LEVERAGE

**What's missing:** No piece-count summary.

**Why it matters:** Easy to count by hand from `pieces`, but the LLM makes arithmetic errors under load. Cheap to provide.

**What we want:** `material: {white: {Q:n,R:n,B:n,N:n,P:n}, black: {...}}` in status. Trivial.

**Status:** Not in upstream. Bundle with (1) and (2) in a single "enriched status" PR.

### 5. No simulate / undo — MEDIUM LEVERAGE, HIGH COMPLEXITY

**What's missing:** Any way to play a move and then unplay it. The only way to explore a line is `add_move`, then `create_or_reset_game` + replay everything to back out.

**Why it matters:** When analyzing tactical lines, the LLM wants to see "what does the board look like after Nxa5 Rxb5 Kxa2?" Right now you either reason about it in your head (risk of errors) or destroy your position to check.

**What we want options:**
- `undo_last_move` — simple, adds some state complexity.
- `peek(uci_sequence)` — applies the moves, returns the resulting status, restores original state. No mutation visible to caller.
- A parallel "sandbox" position separate from the main game.

**Status:** Not in upstream. `peek` is the cleanest API but requires thinking about state isolation. `undo_last_move` is easier and covers 80% of the use case.

### 6. `board_ascii` is bare  — LOW LEVERAGE

**What's missing:** No file/rank coordinate labels, no Unicode pieces, White's perspective only.

**Why it matters:** Minor readability win for humans and possibly for the LLM's spatial reasoning. The research answer is lukewarm on Unicode vs. ASCII.

**What we want:** Optional `coords: true` parameter. Optional `flip: true` for Black's perspective. Optional Unicode mode.

**Status:** Not in upstream. Cosmetic — defer.

## How to act on this

Three paths, in order of preference:

### Path A: PR upstream

`danilop/chess-support-mcp` is small (~5 commits as of last check) and the maintainer's stated philosophy matches ours. An "enriched status" PR bundling gaps 1, 2, 4 would be small and low-risk. Worth trying first.

### Path B: Local fork

If upstream doesn't respond or diverges, fork and update `opencode.json`:

```json
"command": ["uvx", "--from", "git+https://github.com/<fork>/chess-support-mcp.git", "chess-support-mcp"]
```

All the python-chess primitives are already in the upstream's dependencies. Changes are ~50 lines of Python.

### Path C: Thin wrapper MCP

Write a sibling MCP server that wraps `chess-support-mcp` and adds the missing derivations. Higher complexity, no benefit over (B).

## What NOT to add

Keep the engine constraint hard. Specifically **don't** add:

- `best_move` / `evaluate` / `get_score` / anything returning a numeric evaluation.
- `threatened_squares` framed as "which squares should I be worried about" — that's evaluation, not derivation. (`attackers_to_square` is fine — pure geometry.)
- Opening-book lookups that name a move.
- Tablebase queries in endgames.
- Any integration with Lichess cloud analysis, chess.com analysis, etc.

These all turn the MCP into a second intelligence in the loop, which is exactly what the research rejected.

## Decisions log

*Add entries here when the human or a session makes a decision about any of the above. Format: date, decision, reasoning.*

- **2026-04-19:** Initial gap inventory. No changes committed yet. Decided to document first, implement second — a session that hits a gap in real use will have better intuition about priority than a speculative implementation would.
- **2026-04-19:** Pre-compaction handoff. A plan for closing the top-priority gaps lives at [`docs/plans/2026-04-19-fact-server-gaps.md`](../../../docs/plans/2026-04-19-fact-server-gaps.md). Next session should read that plan before implementing anything.
