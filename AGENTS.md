# Chess — conversational analysis workspace

A scratchpad for human + LLM chess analysis. The human is the decision-maker; the LLM is a conversational partner that reasons about positions in natural language.

## Philosophy: no engines

We deliberately do NOT use Stockfish, Leela, Maia, or any other chess engine to generate recommendations. The point of this workspace is to converse with the LLM's own chess intelligence, not to outsource analysis to a stronger player.

What we DO use: a "fact server" MCP (`chess-support-mcp`) that exposes deterministic derivations of board state — FEN, legal moves, check/pin status, piece maps — without any evaluation or move suggestion. The LLM's scratchpad, not its oracle.

Reasoning and sources:

- `/tmp/research-chess-reasoning-aids-question.md` — the original brief
- `/tmp/research-chess-reasoning-aids-answer.md` — the research synthesis that shaped this architecture

## Skills (TOC)

Project-local skills live in `.opencode/skills/`. They're auto-discovered when OpenCode runs from this directory — no build step, no registration.

| Skill | Use when |
|---|---|
| [`replaying-a-pgn`](.opencode/skills/replaying-a-pgn/SKILL.md) | Setting up the MCP game state from a SAN PGN, before analysis. |
| [`analyzing-a-position`](.opencode/skills/analyzing-a-position/SKILL.md) | Reasoning about the current position to advise a player. |
| [`improving-the-fact-server`](.opencode/skills/improving-the-fact-server/SKILL.md) | Hitting a gap in `chess-support-mcp` and considering whether to plug it. |

Typical analysis session: `replaying-a-pgn` → `analyzing-a-position`. The third is consulted only when a gap becomes painful.

## The MCP

Configured in `opencode.json`. Tools provided by upstream `danilop/chess-support-mcp`:

- `chess_create_or_reset_game` — start a fresh game
- `chess_add_move` — apply a UCI move (e.g. `e2e4`, `g1f3`, `e7e8q`)
- `chess_get_status` — FEN, side to move, check flags, `pieces` map, last move
- `chess_board_ascii` — ASCII board from White's perspective
- `chess_list_moves` / `chess_list_moves_detailed` — full history
- `chess_last_moves` / `chess_last_moves_detailed` — recent moves
- `chess_is_legal` — check a UCI move's legality in the current position

**Not provided (see `improving-the-fact-server`):** legal-move list, pin detection, attackers-to-square, material count, SAN input, undo/simulate.

## Current games

See `games/` for active PGNs.
