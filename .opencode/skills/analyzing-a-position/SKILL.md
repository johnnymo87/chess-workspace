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

### 2. Count material by hand, once, and write it down

There is no `material` field in `get_status` (as of this writing — see `improving-the-fact-server`). Count from the `pieces` map. Write it as a line item in your response:

```
White: B + N + 2R + 7P = 16 + 7 = 23 (plus K)
Black: Q + 2B + 2R + 6P = 22 + 6 = 28 (plus K)
Black up ~5 points.
```

Recount if you do anything non-trivial (imagined captures, line evaluation). Arithmetic drift is real.

### 3. Enumerate threats, **per piece**, from the data

For the side not to move, list what each piece attacks. For the side to move, list what is hanging and what is attacked. **Don't speed-run this** — this is where errors hide.

For each candidate target square, list:
- **Attackers** (with piece + color)
- **Defenders** (with piece + color)

Do it explicitly. "The bishop on b5 is attacked by Qa5 and Rb7. It is defended by: nothing. It is hanging."

If you're unsure whether piece X attacks square Y, you can reason it out from the `pieces` map (check the ray is unblocked for sliders, check the knight-jump table for knights). For sliding pieces, explicitly walk the ray square by square using the `pieces` map.

### 4. Check pins before moving any non-king piece

For every candidate move that is not a king move:

- Identify your king's square.
- For each enemy sliding piece (Q, R, B), check whether there's a ray from that piece through your candidate piece to your king, with no other piece on the ray.
- If yes, the piece is pinned. If it's an absolute pin, your candidate piece may only move along the pin ray.

This was the failure mode that motivated the entire workspace (the pinned-Nd2 bug). Take it seriously.

### 5. Verify the candidate move is legal

- Call `chess_is_legal` on every candidate before recommending it.
- Treat this as non-negotiable, not as a double-check. Your "obvious" moves are exactly the ones where you skip the verification.
- If `is_legal` says false, figure out why before trying something else. The reason often reveals a tactical feature (pin, check, blocked square).

### 6. Only then, reason about the plan

- What are Black's best replies? For each, what's your response?
- Go two or three plies deep for tactical positions, then summarize.
- Keep the explanation in natural language, tied to concrete squares and pieces.

## Simulating lines

If you want to see what a position looks like after a hypothetical move, the cheapest option is to just reason from the `pieces` map by mentally updating it. Your brain is fine at "remove piece from X, add to Y" — the errors come from the initial grounding, not from one-ply imagination.

**Avoid** calling `chess_add_move` to simulate, because there's no undo. To get back to the analysis position you'd have to `create_or_reset_game` and replay every ply. If you absolutely must play out a line on the board, do it at the end of your analysis, not mid-reasoning. (See `improving-the-fact-server` — a `simulate_move` / undo capability is on the wishlist.)

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
- **"The bishop defends this" without checking the diagonal** — walk the ray square by square.
- **Counting material from memory of the captures** — count from the current `pieces` map. Captures accumulate errors; current state doesn't.
- **Batching `chess_add_move` calls in parallel during analysis** — if one fails all subsequent fail (wrong side-to-move). See `replaying-a-pgn`.
