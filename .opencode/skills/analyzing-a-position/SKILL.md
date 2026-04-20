---
name: analyzing-a-position
description: Use when reasoning about a chess position to advise a player, after the position has been set up via chess_ MCP tools. Covers the discipline that prevents legality hallucinations (off-diagonal moves, pin blindness, miscounted material).
---

# Analyzing a Position

The MCP gives you authoritative board state. The failure mode this skill prevents is **still reasoning from your head** after the tools have spoken. Use the tools as the source of truth; use your brain for plans and explanations.

Before you advise a move: **ground, then reason**. Never skip grounding.

## The discipline

Follow these steps in order. For a complex position, literally write the intermediate results out before proposing a move; don't just think them.

### 1. Ground yourself in the position

- Call `chess_get_status` and read the FEN, `side_to_move`, `is_check`, `last_move_san`, and the `pieces` map.
- Call `chess_board_ascii`.
- **Do both.** The ASCII board catches geometry errors; the `pieces` map catches square-mixups. They're redundant on purpose.

### 2. Read material from `get_status`, then summarize it

`get_status` now returns `material` (per-color piece counts) and `material_diff` (per-piece white-minus-black). Copy those into your response verbatim; don't recount from `pieces`. Example:

```
material: white={Q:1,R:2,B:1,N:1,P:7}, black={Q:1,R:2,B:2,N:0,P:6}
diff: {B:-1, N:+1, P:+1} → roughly even, White up ~1 pawn, with imbalanced minor pieces.
```

If you *do* want a point-value total, compute it once from the field (don't hallucinate one). But a human narrative of the imbalance ("up a knight, down a bishop, tied in pawns") is usually more useful than a single number.

### 3. Enumerate threats, **per square**, using `get_attackers_to`

For each candidate target square (yours that might be hanging, or the opponent's that might be takeable), call `chess_get_attackers_to` and write down the full attacker/defender picture:

```
e5: white attackers = [{f3:N}], black attackers = []
    → e5 is attacked by Nf3, no black defender, hanging if a piece sits there.
```

**Read the docstring warning carefully:** `get_attackers_to` is the *static* attack map. Pinned pieces still count as attackers, and x-ray through the king is not modeled. So:

- "Attacker exists" does NOT mean "the attacker can legally capture right now." A pinned attacker cannot actually move. To confirm a capture is legal, cross-check against `list_legal_moves_detailed`.
- "No attacker" does NOT mean "safe forever" — an unpin or a tempo move might change the picture.

Used well, this is much faster than walking rays by hand and much less error-prone. Do it explicitly for every non-trivial target. Don't speed-run this — it's where errors hide.

### 4. Read `absolute_pins` from `get_status` before moving any non-king piece

`get_status.absolute_pins` lists every absolutely-pinned piece (pinned to its own king), with the pin's ray squares ordered from king outward. Before proposing any candidate non-king move, check that square against `absolute_pins`. If the piece is pinned, the only legal moves for it are along the ray.

**Important caveats:**

- **Absolute pins only.** Relative pins (through the queen or a rook) are NOT reported. If you're worried about a piece being pinned to the queen, you must reason about it manually — walk the ray from the enemy slider through your piece and see what's behind it.
- `list_legal_moves_detailed` is the ground truth. `absolute_pins` tells you *why* a piece's moves might be restricted, but `list_legal_moves_detailed` tells you *exactly what's legal*. Use them together — the pin field informs the plan, the legal-moves list verifies the execution.

This was the failure mode that motivated the entire workspace (the pinned-Nd2 bug). Take it seriously.

### 5. Verify the candidate move appears in `list_legal_moves_detailed`

- Call `chess_list_legal_moves_detailed` once per position you're analyzing. It returns every legal move as `{uci, san}`.
- Look up each candidate. If a candidate move isn't in the list, it's illegal — figure out why (usually a pin, a check you missed, or a blocked square) before proposing something else.
- This is cheaper and stronger than one-off `chess_is_legal` calls: you get the full menu, so you can also *discover* moves you wouldn't have considered.
- Still use `chess_is_legal` if you want a binary check on a single move without enumerating the whole list — both work.

### 6. Only then, reason about the plan

- What are Black's best replies? For each, what's your response?
- Go two or three plies deep for tactical positions, then summarize.
- Keep the explanation in natural language, tied to concrete squares and pieces.

## Simulating lines

If you want to see what a position looks like after a hypothetical move, the cheapest option is to just reason from the `pieces` map and your `absolute_pins`/`material` snapshot by mentally updating it. Your brain is fine at "remove piece from X, add to Y" — the errors come from the initial grounding, not from one-ply imagination.

**Avoid** calling `chess_add_move` to simulate, because there's no undo. To get back to the analysis position you'd have to `create_or_reset_game` and replay every ply. If you absolutely must play out a line on the board, do it at the end of your analysis, not mid-reasoning. (See `improving-the-fact-server` — undo/simulate is an open gap, tracked as #5.)

## Output shape for advice

A good analysis response has this structure:

1. **Position summary** — FEN, whose turn, material count.
2. **What's attacked / hanging** — both sides.
3. **Candidate moves** — 1–3 of them, with `is_legal` verified.
4. **The best move, with justification** — concrete attackers/defenders, response to best enemy reply.
5. **Plan** — one or two sentences of what you want to do over the next few moves.

## Anti-patterns

- **"Let me just quickly check…"** — quickly always means in-head. Use the tools.
- **Reasoning about a move without naming its source and destination squares** — if you can't write "Nc4 → a5" you'll mix up knights.
- **"The bishop defends this" without `get_attackers_to`** — call the tool. Walking rays by hand is the error-prone path `get_attackers_to` replaces.
- **Reading `get_attackers_to` as a legality oracle** — it's the static attack map. Pinned attackers still appear. If you care whether a capture is legal, look it up in `list_legal_moves_detailed`.
- **Recounting material from memory** — `material` and `material_diff` are fields in `get_status`. Read them.
- **Assuming `absolute_pins == []` means no pins at all** — it means no pins to the king. A piece can still be relatively pinned to the queen or rook. For those, you still have to reason manually.
- **Batching `chess_add_move` calls in parallel during analysis** — if one fails all subsequent fail (wrong side-to-move). See `replaying-a-pgn`.
