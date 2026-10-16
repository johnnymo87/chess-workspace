# Chess — conversational analysis workspace

A scratchpad for human + LLM chess analysis. The human is the decision-maker; the LLM is a conversational partner that reasons about positions in natural language.

## Philosophy: no engines

We deliberately do NOT use Stockfish, Leela, Maia, or any other chess engine to generate recommendations. The point of this workspace is to converse with the LLM's own chess intelligence, not to outsource analysis to a stronger player.

What we DO use: a "fact server" MCP (`chess-support-mcp`) that exposes deterministic derivations of board state — FEN, legal moves, pins, checkers, attackers, material — without any evaluation or move suggestion. The LLM's scratchpad, not its oracle.

Reasoning and sources:

- `/tmp/research-chess-reasoning-aids-question.md` — the original brief
- `/tmp/research-chess-reasoning-aids-answer.md` — the research synthesis that shaped this architecture

## Skills (TOC)

Project-local skills live in `.agents/skills/` and are the primary source of truth for how to do things in this workspace. They're auto-discovered when OpenCode runs from this directory — no build step, no registration.

| Skill | Use when |
|---|---|
| [`replaying-a-pgn`](.agents/skills/replaying-a-pgn/SKILL.md) | Setting up the MCP game state from a SAN PGN, before analysis. |
| [`analyzing-a-position`](.agents/skills/analyzing-a-position/SKILL.md) | Reasoning about the current position to advise a player. |
| [`improving-the-fact-server`](.agents/skills/improving-the-fact-server/SKILL.md) | Hitting a gap in `chess-support-mcp` and considering whether to plug it. |

Typical analysis session: `replaying-a-pgn` → `analyzing-a-position`. The third is consulted only when a gap becomes painful.

## Quick start

Launch OpenCode from this directory. `opencode.json` wires up the `chess-support-mcp` server via `uvx`, which pulls and caches our fork (`johnnymo87/chess-support-mcp@enriched-status`) on first run. First run takes ~30 seconds to build; subsequent runs start instantly.

## The MCP

Configured in `opencode.json`; running from our fork at `github.com/johnnymo87/chess-support-mcp` branch `enriched-status` (see `improving-the-fact-server` for what the fork adds and why we're not on upstream). Tools:

- `chess_create_or_reset_game` — start a fresh game
- `chess_add_move` — apply a UCI move (e.g. `e2e4`, `g1f3`, `e7e8q`)
- `chess_get_status` — FEN, side to move, last move, `pieces` map, `is_check`, `checkers`, `absolute_pins`, `material`, `material_diff`
- `chess_board_ascii` — ASCII board from White's perspective
- `chess_list_legal_moves_detailed` — every legal move in the current position as `[{uci, san}]`, UCI-sorted
- `chess_get_attackers_to` — static attack map for a square (**not** a legality oracle: pinned attackers still appear)
- `chess_is_legal` — check a UCI move's legality in the current position
- `chess_list_moves` / `chess_list_moves_detailed` — full move history
- `chess_last_moves` / `chess_last_moves_detailed` — recent moves

**Not provided (see `improving-the-fact-server`):** SAN input / `load_pgn`, undo/simulate, relative pins, `board_ascii` enhancements.

## Current games

See `games/` for active PGNs.
