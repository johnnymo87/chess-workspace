---
name: replaying-a-pgn
description: Use when setting up the MCP game state from a SAN PGN, before analyzing a position. Covers the SAN→UCI conversion discipline and the "one illegal move cascades" gotcha that can waste a session's worth of tokens.
---

# Replaying a PGN into the MCP

The MCP accepts **UCI only** (`e2e4`, `g1f3`, promotions like `e7e8q`). PGNs from Chess.com / Lichess / email are in **SAN** (`e4`, `Nf3`, `O-O-O`). Every replay is a pile of SAN→UCI conversions where a single mistake can waste tens of minutes. Do it carefully.

The research that motivated this workspace (`/tmp/research-chess-reasoning-aids-answer.md`) explicitly names PGN-reconstruction as the LLM's weak spot. Don't trust your head on the tactical replay either — if in doubt, verify.

## The gotcha that will bite you first

**Do not batch `chess_add_move` calls in parallel.** If you make one mistake mid-replay, every subsequent call in the batch will fail with `"reason": "illegal", "expected_turn": "<color>"` — not because those moves are illegal, but because the side-to-move never advanced. You can burn hundreds of tool calls this way.

Worse: OpenCode lets you fire all the tool calls in a single message block. A 40-ply game with one error at move 15 will produce 25+ phantom failures. Ask me how I know.

**Safe patterns:**

- **Small batches (~5 moves) per message**, then `chess_get_status` to confirm the expected `last_move_san` before the next batch. Balances throughput vs. blast radius.
- **One at a time** for opening theory moves where you're rusty on the SAN→UCI mapping.
- **Accept one large batch** only if you've verified every UCI by hand or you're replaying a game you've replayed before.

If a batch does fail mid-way, the easiest recovery is `chess_create_or_reset_game` and start over. Don't try to patch up a half-applied game state — the ply counts in the failure traces are confusing and you'll make a second mistake.

## SAN → UCI conversion cheatsheet

- **Pawn moves:** `e4` → needs context. If white, it's `e2e4` (from start) or `e3e4` (if pawn is already on e3). **Always look at where the pawn actually is** in the `pieces` map before the move. Don't assume.
- **Pawn captures:** `exd5` → the e-pawn captures on d5. Source is `e<rank>`, destination is `d5`. Figure out which rank the e-pawn is on.
- **Piece moves:** `Nf3` → find the knight that can reach f3 legally. If both knights can, the SAN would have disambiguated (`Nbf3` or `N1f3`).
- **Captures:** `Nxe4` → same as Nf3 but with capture. UCI doesn't mark captures; just source-destination.
- **Castling:** `O-O` → `e1g1` (white) or `e8g8` (black); `O-O-O` → `e1c1` / `e8c8`.
- **Promotions:** `e8=Q` → `e7e8q` (lowercase piece letter).
- **Checks and mates:** SAN `+` and `#` have no UCI equivalent. Ignore them.
- **En passant:** `exd6 e.p.` → `e5d6` (capturing pawn moves to the en-passant square, even though the captured pawn is on d5).

## Common ambiguity traps

These are the moves most likely to trip you up in a replay:

1. **The "same pawn" rabbit hole.** If Black's c-pawn plays `c6` on move 12 and then `c5` on move 13, the move 13 UCI is `c6c5`, **not** `c7c5`. The pawn is no longer on c7. Always read the current `pieces` map — don't pattern-match against starting position.
2. **Knight/rook disambiguation.** If SAN says `Nbd2` or `R1e2`, the hint tells you which source piece. `Nbd2` = the knight currently on the b-file moves to d2.
3. **Bishop vs. pawn capture.** `Bxa2` is bishop to a2; `bxa2` (lowercase b) is b-pawn takes a2. SAN is case-sensitive. Don't normalize.

## Recommended workflow

1. `chess_create_or_reset_game`.
2. Take the PGN, split by move number, and mentally walk through it once before firing any tool call. For long openings, you probably remember the UCI for the first 5–10 moves — those can batch.
3. For any move where you're not 100% sure, write the UCI next to the SAN *before* calling the tool.
4. Fire a batch (≤5 moves). Then call `chess_get_status` and check `last_move_san` matches the SAN for that ply. If it doesn't, stop and diagnose.
5. Repeat until you've reached the analysis position.
6. Final check before analysis: `chess_list_moves_detailed` — compare the full SAN list against the PGN. They should match ply-for-ply.
7. Hand off to `analyzing-a-position`.

## Why this isn't fixed upstream

`chess-support-mcp` could accept SAN directly (python-chess has `board.parse_san()`), which would eliminate nearly all of this pain. A `chess_load_pgn` tool that takes the whole PGN and replays it server-side would eliminate even more. Both are tracked in `improving-the-fact-server`.

Until then: batch small, verify after each batch, and don't trust your head.
