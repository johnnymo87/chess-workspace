---
name: improving-the-fact-server
description: Use when a chess analysis session hits a gap in chess-support-mcp (missing legal-move list, no pins, no undo, etc.) and you're tempted to plug it. Catalogues known gaps, their priority, and the constraint that keeps this workspace engine-free.
---

# Improving the Fact Server

`chess-support-mcp` is the fact server. This skill is the standing list of gaps between what the research brief (`/tmp/research-chess-reasoning-aids-answer.md`) said we need and what the MCP actually exposes. It exists so that a future session can quickly see what's been considered, what's been decided, and what would be worth actually implementing.

We run a fork (`github.com/johnnymo87/chess-support-mcp`, branch `enriched-status`) that closes gaps 1, 2, and 4. `opencode.json` already points at it. Open gaps are 3 (SAN input) and 5 (undo/simulate).

## The hard constraint

**No engine.** No Stockfish, no Leela, no evaluation, no "best move" tool. That's the point of the whole workspace. Anything added must be a mechanical derivation from `python-chess` (or equivalent) — a fact, not a judgment. If a proposed improvement would give the LLM a move recommendation, reject it.

See `../../AGENTS.md` and the research answer for the philosophical reasoning. It's not arbitrary.

## Known gaps (current state: 2026-04)

Ranked roughly by leverage (how much each would have helped real sessions). Verify the current state of the upstream tool before acting — the toolset listed in `../../AGENTS.md` is what we actually have.

### 1. Legal-move enumeration — DONE (2026-04-19)

`list_legal_moves_detailed` tool now returns `[{uci, san}, ...]` sorted by UCI. Shipped on the fork branch `enriched-status` of `johnnymo87/chess-support-mcp`; `opencode.json` points there.

Design rationale: [`docs/plans/2026-04-19-enriched-status-design.md`](../../../docs/plans/2026-04-19-enriched-status-design.md) §"New tool: list_legal_moves_detailed".

### 2. Pins / checkers / attackers — DONE (2026-04-19)

`get_status` now returns `absolute_pins` and `checkers`. New tool `get_attackers_to(square)` exposes the static attack map. Same fork branch.

Caveats baked into the design:
- `absolute_pins` is absolute-only (pinned-to-king). Relative pins (through queen/rook) are out of scope; would be a separate field if ever wanted.
- `get_attackers_to` is the *static* attack map: pinned pieces still count as attackers, and x-ray through the king is not modeled. The tool's docstring makes this explicit; don't use it as a legality oracle.

Design rationale: same design doc, §"Surface changes".

### 3. No SAN input — MEDIUM LEVERAGE, OPEN

**What's missing:** `add_move` takes UCI only; a PGN or SAN move list must be hand-converted by the LLM.

**Why it matters:** See `replaying-a-pgn` — SAN→UCI conversion is tedious and error-prone, and a single mistake cascades badly. python-chess's `board.parse_san("Nxe4")` would eliminate this.

**What we want:** Either make `add_move` accept SAN, or add a sibling `add_move_san`, or a `load_pgn(pgn_string)` that replays the whole game server-side.

**Status:** Not implemented. `load_pgn` is probably the highest-leverage version — one tool call instead of 40. Would be the obvious next PR to the fork.

### 4. Material count — DONE (2026-04-19)

`get_status` now includes `material` (per-color piece counts, kings excluded, uncapped for promotions) and `material_diff` (per-piece `white[k] - black[k]`). Same fork branch.

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

Work on the fork: `github.com/johnnymo87/chess-support-mcp`, branch `enriched-status`. Local checkout at `/home/dev/projects/chess-support-mcp`. `opencode.json` already points there.

TDD is the expected workflow (see the existing commits on the branch for style). Each new field or tool:

1. Write the failing test against the stdio MCP client (mirror the style in `tests/test_chess_support_mcp.py`).
2. Implement the smallest change in `src/chess_support_mcp/server.py` that makes it pass.
3. Run `uv run pytest -q` — full suite should stay green.
4. Commit. Push to the `enriched-status` branch.

Restart OpenCode to pick up the new server binary (or run `uvx --refresh --from 'git+...@enriched-status' chess-support-mcp --help` to warm the cache first).

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
- **2026-04-19:** Gaps 1, 2, 4 implemented on the fork. Design at [`docs/plans/2026-04-19-enriched-status-design.md`](../../../docs/plans/2026-04-19-enriched-status-design.md), 13-task plan at [`docs/plans/2026-04-19-enriched-status-plan.md`](../../../docs/plans/2026-04-19-enriched-status-plan.md). Fork `github.com/johnnymo87/chess-support-mcp` branch `enriched-status`; 7 commits atop upstream `f75662d`. End-to-end validation against the motivating PGN passed all 9 assertions (absolute pin of Nd2 by Bh6 detected correctly, material diff, legal-moves enumeration). Decided to stay on the fork indefinitely rather than PR upstream — the fork is our runtime and nothing forces the decision. Gaps 3 (SAN input / `load_pgn`) and 5 (undo/simulate) remain open and are the obvious next candidates.
