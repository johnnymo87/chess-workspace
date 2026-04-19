# Design: Enriched status for `chess-support-mcp`

**Status:** Design approved. Implementation plan to be written next via `writing-plans` skill. Likely to span multiple compactions during execution — this document is designed to be self-contained for a cold-start agent.

**Date:** 2026-04-19

**Supersedes / refines:** [`2026-04-19-fact-server-gaps.md`](2026-04-19-fact-server-gaps.md) §"Suggested first PR shape". That earlier plan contained a rougher sketch; this document replaces its field shapes and scope after a brainstorming round and external review.

---

## Resumption checklist (read first if resuming post-compaction)

A future agent picking up this work should, in order:

1. **Read [`AGENTS.md`](../../AGENTS.md)** for workspace philosophy (the "no engines" constraint is non-negotiable and shapes every decision below).
2. **Read [`.opencode/skills/improving-the-fact-server/SKILL.md`](../../.opencode/skills/improving-the-fact-server/SKILL.md)** for the gap catalogue this design sits within (addresses gaps 1 + 2 + 4).
3. **Read this whole document end-to-end.** All decisions have inlined rationale. Don't re-litigate them unless you find a correctness bug.
4. **Check the "Current state" section below** to see what has/hasn't been done yet (fork created? branch pushed? PR opened? `opencode.json` flipped?).
5. **Check the implementation plan at** `docs/plans/2026-04-19-enriched-status-plan.md` if it exists — that's the concrete task list produced by the `writing-plans` skill. If it doesn't exist yet, producing it is your next step.

Do not start coding until you've done at least steps 1–3.

---

## Current state (as of initial write — update this section on every material change)

**Our repo (`/home/dev/projects/chess`):**
- Branch `main`, commit `ea29ebf` ("Initial commit: workspace, skills, and fact-server gaps plan"). No remote configured.
- This design doc committed on top of that (commit hash to be assigned when committed).
- `opencode.json` still points at upstream `danilop/chess-support-mcp` (not yet flipped to fork).
- Relevant PGN for end-to-end validation: `games/2026-04-12-mohrbacher-dauromonte.pgn`.

**Upstream `danilop/chess-support-mcp`:**
- Pinned reference commit for this design: `f75662d` (on `main`, dated 2025-08-13, "Update README.md"). If upstream has moved since, re-base from their latest `main` before PR; rebase pain should be minimal given how small the diff area is.
- 3 stars, 0 issues, 0 prior PRs, 8 months quiet as of 2026-04-19. Philosophy-aligned ("intentionally does not suggest moves") but response time unknown.
- Single-file server: `src/chess_support_mcp/server.py` (306 LoC). `GameState.status()` is at lines 95–115 of that file in `f75662d` — that's the one method we modify. New tools are added near the existing `@server.tool()`-decorated functions (lines 126+).
- Tests: `tests/test_chess_support_mcp.py` (98 LoC), async pytest + stdio MCP client, asserting on `resp.structuredContent["result"]`.

**Fork status:**
- GitHub account: `johnnymo87`.
- Fork not yet created. Target: `github.com/johnnymo87/chess-support-mcp`, branch `enriched-status`.
- After fork: point `opencode.json` at `git+https://github.com/johnnymo87/chess-support-mcp.git@enriched-status` so we dogfood the branch during development.

**What has NOT been done yet:**
- Fork not created.
- Branch not created.
- No code written.
- Implementation plan file not yet written.
- `opencode.json` not yet updated.
- No PR opened.

---

## Scope

**In:** Augment `chess-support-mcp` with derived-fact fields in `get_status()` and two new tools, all mechanical derivations from `python-chess`. No evaluation, no move recommendation, no engine.

**Out of scope for this PR:** SAN input (`add_move_san`, `load_pgn`), undo/simulate, relative pins (pins not to the king), point-value material totals, `board_ascii` enhancements, any `verbose`/opt-out flag. These are deferred to future PRs — see [`improving-the-fact-server/SKILL.md`](../../.opencode/skills/improving-the-fact-server/SKILL.md) for the full gap catalogue.

## Hard constraints

1. **No engine.** See [`AGENTS.md`](../../AGENTS.md) and the skill file linked above. This is non-negotiable. Any proposed addition that returns a move recommendation, a numeric evaluation, or a judgment-laden signal ("threatened", "advantageous", "best") is rejected on sight. Only mechanical derivations from `python-chess`.
2. **Additive only.** No existing field renamed, removed, or changed in shape. Existing clients must continue to work unchanged.
3. **Small PR.** Target ~80–120 LoC in `server.py`, ~120–150 LoC in tests. Fits in a stale-but-not-archived repo's review budget.
4. **Python 3.13, python-chess only.** No new heavy dependencies. (`chess>=1.10.0` already provides every primitive we need.)

---

## External review findings (inlined — do not require `/tmp/*` files)

The design went through an external review via ChatGPT (ask-question skill). The full exchange was written to `/tmp/research-enriched-chess-status-question.md` and `/tmp/research-enriched-chess-status-answer.md` at the time of the review, but `/tmp` is ephemeral — the substantive findings are reproduced below so a future agent can act on them without re-running the review.

Corrections that reshaped the design:

**1. Pin scope: `python-chess` pins are absolute only.** `board.pin(color, sq)` and `board.is_pinned(color, sq)` are documented and implemented as pins-to-king only. Our original "walk the ray to find pinned_to piece" algorithm was over-engineered — the pinned-to piece is always the king by construction. Relative pins through a queen or rook are not what those primitives detect; we'd need a separate algorithm, which is out of scope for this PR. Result: field renamed `pins` → `absolute_pins`, `pinned_to_square/piece` → `king_square`, dropped redundant `absolute` boolean, stopped trying to model relative pins. If relative pins become useful later, they get a separate field name.

**2. `board.attackers()` semantic trap.** python-chess's `attackers()` counts pinned pieces as attackers and does not model x-ray attacks through the king. It's a static attack map, not a "who can legally capture here right now" query. Without an explicit docstring warning, the LLM will misuse `get_attackers_to` as a legality oracle — which is exactly the reasoning-bug class this PR exists to prevent. Result: mandatory warning in the docstring (inlined below in §Surface changes).

**3. No ordering guarantee on `board.legal_moves`.** python-chess's docs describe it as a dynamic generator but make no stable-ordering promise. Tests built against its natural order would be brittle against upstream library updates. Result: we sort `list_legal_moves_detailed` output by UCI ascending and document the ordering contract.

**4. Legal moves as array-of-objects, not parallel arrays.** Our original design had `legal_moves_uci: [...]` and `legal_moves_san: [...]` as parallel arrays. ChatGPT's counter: self-describing objects (`[{uci, san}, ...]`) are harder to desync in future edits, better for LLM one-shot consumption, and "easier to produce in one loop" was a weak reason for parallel arrays anyway (object-per-iteration is still one loop). Result: switched to array-of-objects.

**5. Legal moves belong in a separate tool, not bundled into `get_status`.** Upstream's existing API style separates a lean `get_status` snapshot from heavier `*_detailed` tools (`list_moves_detailed`, `last_moves_detailed`). Bundling ~50 SAN-rendered moves into every `get_status` call cuts against that style and is the single most likely upstream-reviewer objection. Result: `list_legal_moves_detailed()` is its own tool. Decouples it from the `get_status` additions — if the maintainer objects to legal-moves-as-a-thing, we don't lose `absolute_pins` / `checkers` / `material` with it. Cost to us: one extra round-trip per position; negligible.

**6. `material_imbalance` → `material_diff`.** "Imbalance" carries a faint evaluation whiff; "diff" is unambiguously arithmetic. Same data (`white[k] - black[k]` per piece type), better name. Mergeability-focused.

**7. Promotion-aware tests.** Material counts must not silently assume orthodox starting-position maxima. Test fixtures must exercise post-promotion positions (e.g., 2+ queens, 3+ knights).

**8. Double-check coverage.** `board.checkers()` can return up to two squares in a valid position (discovered double check). Tests must cover 0, 1, and 2 checkers cases.

**9. Ordering of ray squares.** `board.pin()` returns a `SquareSet`, which is an unordered bitboard. If we include `ray` in the pin object, we must impose an ordering (we chose: from king outward along the pin line).

All nine findings are incorporated into the design sections below.

---

## Surface changes

### `GameState.status()` — new fields (existing fields unchanged)

```jsonc
{
  // ...all existing fields unchanged (fen, pieces, is_check, is_game_over, etc.)...

  "absolute_pins": [
    {
      "color": "white",           // color of the pinned piece
      "pinned_square": "d2",
      "pinned_piece": "N",         // piece symbol (upper=white, lower=black)
      "king_square": "b2",         // always the own-color king (absolute pin definition)
      "pinner_square": "h6",
      "pinner_piece": "b",
      "ray": ["b2", "c3", "d4", "e5", "f6", "g7", "h8"]
      // The full pin line from king through pinned piece to pinner and beyond.
      // Ordered: from the king outward toward and past the pinner.
    }
  ],
  // Empty [] if no absolute pins. Absolute-only: pins to the king.
  // Pins through queen/rook (relative pins) are out of scope for this PR
  // and would be a separate future feature with a distinct field name.

  "checkers": [
    { "square": "h4", "piece": "q" }
  ],
  // Pieces giving check to the side-to-move king. Empty [] if not in check.
  // Can contain up to 2 entries in a valid position (discovered double check).

  "material": {
    "white": { "Q": 1, "R": 2, "B": 2, "N": 2, "P": 8 },
    "black": { "Q": 1, "R": 2, "B": 2, "N": 2, "P": 8 }
  },
  // Piece counts per color. Uncapped — promotion can produce >1 queen, etc.
  // No king count (always 1 per side). No point-value totals.

  "material_diff": { "Q": 0, "R": 0, "B": 0, "N": 0, "P": 0 }
  // Strictly white[k] - black[k] per piece type. Pure arithmetic.
  // Positive = white has more of that piece; negative = black does.
}
```

### New tool: `list_legal_moves_detailed()`

```jsonc
// returns:
[
  { "uci": "a2a3", "san": "a3" },
  { "uci": "a2a4", "san": "a4" },
  { "uci": "b1a3", "san": "Na3" },
  // ...
]
```

- **Sorted by `uci` ascending.** Stable, deterministic, test-friendly. `python-chess` does not document a stable `legal_moves` ordering, so we impose one.
- Empty `[]` in checkmate / stalemate.
- Docstring style mirrors existing `list_moves_detailed` in the upstream codebase.

### New tool: `get_attackers_to(square: str)`

```jsonc
{
  "square": "e5",
  "white": [
    { "square": "d4", "piece": "P" },
    { "square": "f3", "piece": "N" }
  ],
  "black": [
    { "square": "f6", "piece": "n" }
  ]
}
// Parse-error path: { "square": "zz9", "parse_error": "...", "white": [], "black": [] }
```

**Mandatory docstring warning** (reproduce verbatim in the tool's docstring — this is the semantic trap that must not be lost):

> This is the static attack map. Pinned pieces are still counted as attackers, and x-ray attacks through the king are not considered. This tool does not answer "who can legally capture on this square right now" — to answer that, filter `list_legal_moves_detailed()` by destination square.

### Tools NOT changed

`board_ascii`, `add_move`, `create_or_reset_game`, `is_legal`, `list_moves`, `list_moves_detailed`, `last_moves`, `last_moves_detailed` — none modified. `add_move` and `create_or_reset_game` both return a `status` object that now transparently contains the new fields via the shared `GameState.status()` method; no signature change.

---

## Rationale log (why each shape is what it is)

Each decision could have been made differently. Rationale preserved so a future PR reviewer — or a future session — doesn't re-litigate.

**Why `absolute_pins` and not `pins`:** `python-chess`'s pin detection is absolute-only. A broader "pins" field would require either lying about our underlying library's semantics or implementing relative-pin detection ourselves. The latter is out of scope; the former is dishonest. See External review finding #1.

**Why include `ray`:** `board.pin()` returns it anyway — including it in the response is near-zero extra cost and supports LLM reasoning about where the pinned piece can still legally move along the pin line without a follow-up call.

**Why rich objects (with `piece` fields) vs. bare square lists:** The LLM consuming this response should not have to join against the `pieces` map for every downstream reasoning step — that's the kind of extra bookkeeping that causes the hallucination failures this PR exists to prevent. Piece info is included wherever a square is named, even though it slightly duplicates `pieces`. Payload cost is negligible.

**Why array-of-objects for `legal_moves`, not parallel arrays:** Self-describing, harder to desync, better for LLMs. See External review finding #4.

**Why `list_legal_moves_detailed()` is a separate tool, not a `get_status` field:** Upstream API style, decoupling, mergeability. See External review finding #5.

**Why `material_diff` and not `material_imbalance`:** "Imbalance" whiff. See External review finding #6.

**Why sort `legal_moves` by UCI:** Stable tests, predictable output. See External review finding #3.

**Why no `verbose` flag:** Considered, rejected as YAGNI. Payload sizes are small; no consumer has asked for opt-out. Easier to add later than to remove.

**Why no king count in `material`:** Always 1, exposing it is noise.

---

## Derivation algorithms (concrete enough to implement without rediscovery)

### `absolute_pins`

```python
# In GameState.status(), or as a helper method on GameState.
absolute_pins = []
for color in (chess.WHITE, chess.BLACK):
    king_sq = self.board.king(color)
    if king_sq is None:
        continue  # defensive; legal games always have both kings
    color_name = "white" if color == chess.WHITE else "black"
    for square, piece in self.board.piece_map().items():
        if piece.color != color:
            continue
        if not self.board.is_pinned(color, square):
            continue
        ray_set = self.board.pin(color, square)  # chess.SquareSet, king and pinner both included
        # The pinner is the unique enemy slider on the ray.
        # (board.pin() returns the line from king through pinned piece to the first enemy slider.)
        pinner_sq = next(
            sq for sq in ray_set
            if self.board.piece_at(sq) is not None
            and self.board.piece_at(sq).color != color
        )
        ray_squares = _sort_ray_from_king(king_sq, ray_set)
        absolute_pins.append({
            "color": color_name,
            "pinned_square": chess.square_name(square),
            "pinned_piece": piece.symbol(),
            "king_square": chess.square_name(king_sq),
            "pinner_square": chess.square_name(pinner_sq),
            "pinner_piece": self.board.piece_at(pinner_sq).symbol(),
            "ray": [chess.square_name(sq) for sq in ray_squares],
        })
```

### `_sort_ray_from_king(king_sq, ray_set)` — helper

Given the king square (an int 0-63) and a `SquareSet` containing collinear squares including the king, return a list of square ints ordered from the king outward along the shared line.

```python
def _sort_ray_from_king(king_sq: int, ray_set: chess.SquareSet) -> list[int]:
    # All squares in ray_set are collinear with king_sq (property of board.pin()).
    # Determine the step direction by finding any other square and computing a unit vector.
    squares = [sq for sq in ray_set if sq != king_sq]
    if not squares:
        return [king_sq]
    # Pick the nearest non-king square to establish direction.
    # chess.square_file(sq) and chess.square_rank(sq) give 0-7 coords.
    kf, kr = chess.square_file(king_sq), chess.square_rank(king_sq)
    # Use the nearest-by-Chebyshev-distance square to fix direction robustly.
    nearest = min(squares, key=lambda s: max(
        abs(chess.square_file(s) - kf),
        abs(chess.square_rank(s) - kr),
    ))
    nf, nr = chess.square_file(nearest), chess.square_rank(nearest)
    # Unit direction (df, dr) in {-1, 0, 1} each.
    df = (nf - kf) and (1 if nf > kf else -1)
    dr = (nr - kr) and (1 if nr > kr else -1)
    # Sort all ray squares (including king) by projection along (df, dr).
    def projection(sq: int) -> int:
        f, r = chess.square_file(sq), chess.square_rank(sq)
        return (f - kf) * df + (r - kr) * dr
    return sorted(ray_set, key=projection)
```

Tested against: horizontal, vertical, and both diagonal pins. King comes first (projection 0); pinner is last (largest projection).

### `checkers`

```python
checkers = []
for sq in self.board.checkers():
    piece = self.board.piece_at(sq)
    checkers.append({
        "square": chess.square_name(sq),
        "piece": piece.symbol() if piece else "?",  # piece should always exist if it's giving check
    })
```

### `material` and `material_diff`

```python
PIECE_KEYS = ("Q", "R", "B", "N", "P")

def _material_counts(self) -> dict:
    white = {k: 0 for k in PIECE_KEYS}
    black = {k: 0 for k in PIECE_KEYS}
    for piece in self.board.piece_map().values():
        sym_upper = piece.symbol().upper()
        if sym_upper == "K":
            continue
        if piece.color == chess.WHITE:
            white[sym_upper] += 1
        else:
            black[sym_upper] += 1
    return {"white": white, "black": black}

# In status():
material = self._material_counts()
material_diff = {k: material["white"][k] - material["black"][k] for k in PIECE_KEYS}
```

### `list_legal_moves_detailed`

```python
@server.tool()
def list_legal_moves_detailed() -> List[Dict[str, Any]]:
    """Return all legal moves in the current position, sorted by UCI ascending.
    
    Each item: { uci: string, san: string }.
    Ordering: sorted by uci ascending. Empty list in checkmate or stalemate.
    """
    moves = [
        {"uci": m.uci(), "san": _GAME.board.san(m)}
        for m in _GAME.board.legal_moves
    ]
    moves.sort(key=lambda x: x["uci"])
    return moves
```

### `get_attackers_to`

```python
@server.tool()
def get_attackers_to(square: str) -> Dict[str, Any]:
    """Return all pieces attacking the given square, by color.
    
    Parameters:
    - square: algebraic square name like "e4".
    
    Returns:
    - { square, white: [{square, piece}, ...], black: [{square, piece}, ...] }
    - On parse error: { square, parse_error: str, white: [], black: [] }
    
    IMPORTANT: This is the static attack map. Pinned pieces are still counted as
    attackers, and x-ray attacks through the king are not considered. This tool
    does not answer "who can legally capture on this square right now" — to
    answer that, filter list_legal_moves_detailed() by destination square.
    """
    try:
        sq = chess.parse_square(square)
    except ValueError as e:
        return {"square": square, "parse_error": str(e), "white": [], "black": []}
    def format_attackers(color: chess.Color) -> List[Dict[str, str]]:
        result = []
        for s in _GAME.board.attackers(color, sq):
            piece = _GAME.board.piece_at(s)
            result.append({
                "square": chess.square_name(s),
                "piece": piece.symbol() if piece else "?",
            })
        return result
    return {
        "square": square,
        "white": format_attackers(chess.WHITE),
        "black": format_attackers(chess.BLACK),
    }
```

---

## Test plan (concrete fixtures, no semantic-only descriptions)

Mirror upstream's pattern: async pytest, stdio MCP client, one `@pytest.mark.anyio` function per case. All assertions on `resp.structuredContent["result"]`. Tests go in `tests/test_chess_support_mcp.py` alongside the existing three tests.

Each test below gives either the exact UCI move sequence to replay from the initial position, or a FEN to set up (note: upstream does not currently expose a `load_fen` tool, so positions must be reached by legal UCI play; if that's infeasible for a given test, the test should note it and either play a legal approximation or be deferred).

### Legal moves

1. **`test_list_legal_moves_detailed_initial`** — after `create_or_reset_game`, call `list_legal_moves_detailed`. Assert `len(result) == 20`, assert `{"uci": "e2e4", "san": "e4"}` in result, assert result equals `sorted(result, key=lambda x: x["uci"])`.

2. **`test_list_legal_moves_detailed_checkmate`** — Fool's Mate: reset, then apply `f2f3, e7e5, g2g4, d8h4`. Call `list_legal_moves_detailed`. Assert `result == []`. Also assert `get_status().is_game_over is True`.

3. **`test_list_legal_moves_detailed_sorted`** — reset, play `e2e4` (mid-enough for variety). Call `list_legal_moves_detailed`. Extract `[m["uci"] for m in result]`; assert it equals its own sorted copy.

### Pins

4. **`test_absolute_pins_detected`** — Reset, play: `e2e4, e7e5, g1f3, d7d6, f1b5`. After 5. Bb5+, Black's Nc6... wait, there's no Nc6 yet in this sequence. Use this instead: `e2e4, e7e5, g1f3, b8c6, f1b5`. Now Bb5 pins Black's Nc6 to the Ke8 (absolute pin along a4-e8 diagonal). Call `get_status`. Assert `absolute_pins` contains an entry with `pinned_square == "c6"`, `pinned_piece == "n"`, `king_square == "e8"`, `pinner_square == "b5"`, `pinner_piece == "B"`, `color == "black"`, and `ray` starts with `"e8"` and ends with `"a4"` (or the farthest square the bishop's ray reaches on that diagonal — verify against actual board.pin() output during implementation; if the ray stops at the pinner at b5, then `ray == ["e8", "d7", "c6", "b5"]`).
   
   Note: the exact `ray` contents depend on what `board.pin()` returns. During TDD, verify the actual python-chess output first and adjust the assertion. The shape and ordering contract (king first, monotonic along the line) is what we're testing for.

5. **`test_absolute_pins_empty_initial`** — after `create_or_reset_game`, assert `absolute_pins == []`.

6. **`test_relative_pin_not_reported`** — a pin through the queen (not the king) must not appear. Example position: set up where a bishop pins a knight against the opposite queen, but with the king safely off the line. Construct via: `e2e4, d7d5, g1f3, d8d6, f1b5` — here after 5. Bb5+, it's a check, not what we want. Alternative: find a position where there's a relative pin but no absolute pin. If constructing one via legal play proves awkward, this test can be skipped in v1 with a TODO comment — the primary defensive assertion is covered by `test_absolute_pins_empty_initial`. During implementation, try: `d2d4, d7d5, c1f4, b8c6, g1f3, c8g4` — now Bg4 pins Nf3 to... the queen on d1. Relative pin, no check. Assert `absolute_pins == []`.

### Checkers

7. **`test_checkers_zero`** — initial position, `checkers == []`.

8. **`test_checkers_one`** — Scholar's-mate-style check: `e2e4, e7e5, f1c4, b8c6, d1h5, g8f6, h5f7`. After 7. Qxf7+, Black's king on e8 is in check from the queen on f7. Assert `checkers == [{"square": "f7", "piece": "Q"}]` and `is_check is True`. (Note: this is checkmate — Black has no legal moves. Assertion still works.)

9. **`test_checkers_two`** — discovered double check. A classical construction: after some sequence of moves, a piece moves and both it and a piece behind it give check. Example: black king on e8, white rook on e1, white bishop moves off the e-file with check (say, white bishop on c4 — delivers bishop check — while the rook on e1 also checks the king on e8 because the bishop was blocking). Constructing this via legal play from the initial position takes several moves. One workable line: `e2e4, e7e5, g1f3, b8c6, f1b5, g8f6, e1g1, f6e4, b1c3, c6d4, f3d4, e4c3, d4f5, c3d1, f5d6` — gets complicated; this test may be the hardest to construct via legal play from the starting position.
   
   **Implementation guidance:** if legal-play construction is awkward, accept that this test requires a FEN fixture. Since upstream currently has no `load_fen` tool, either (a) defer this test with a TODO and ship the PR without it (losing some coverage), or (b) add a minimal `_load_fen` helper used only by tests, which does not become part of the MCP surface. Option (a) is simpler and doesn't expand PR scope.

### Material

10. **`test_material_initial`** — after reset, both colors have `{Q:1, R:2, B:2, N:2, P:8}`, `material_diff` all zeros.

11. **`test_material_after_capture`** — Reset, play `e2e4, d7d5, e4d5`. Now White has `P:8` (captured pawn is replaced-in-kind on d5), Black has `P:7`, `material_diff.P == 1`.

12. **`test_material_after_promotion`** — Play a sequence that ends with a pawn promotion. One concrete line (14 plies): `e2e4, d7d5, e4d5, d8d5, b1c3, d5a5, g1f3, a5a2, d2d4, a2b1, d1d3, b1a2, d3b5, c7c6, b5b7` — too long and fragile. Simpler: use a promotion-reaching sequence; the exact line is less important than the final state having two white queens. During implementation, pick the shortest legal sequence that reaches a position with `material.white.Q == 2`. Assert the count is 2, not an exception or cap.

### Attackers

13. **`test_get_attackers_to_basic`** — Reset, play `e2e4, e7e5, g1f3`. Call `get_attackers_to("e5")`. Expected: `white` contains `{"square": "f3", "piece": "N"}`; `black` is empty or contains only pieces that attack e5 from Black's side (king on e8 doesn't attack e5; pawn on d7 doesn't; pawn on f7 doesn't — so `black: []` is likely correct). Verify exact expected contents against python-chess during implementation.

14. **`test_get_attackers_to_includes_pinned`** — reuse the position from test 4 (pin on c6). Call `get_attackers_to(<square the pinned knight attacks>)`. Assert the pinned knight appears in the attackers list even though it cannot legally move there. Documents the static-attack-map semantic.

15. **`test_get_attackers_to_parse_error`** — call with `"zz9"`. Assert `"parse_error"` key is present, `white == []`, `black == []`, `square == "zz9"`.

### Regression

16. **Add to existing `test_reset_and_status`**: after the existing assertions, add:
    ```python
    assert isinstance(status["absolute_pins"], list)
    assert isinstance(status["checkers"], list)
    assert isinstance(status["material"], dict)
    assert isinstance(status["material_diff"], dict)
    assert status["material_diff"] == {"Q":0,"R":0,"B":0,"N":0,"P":0}
    ```

The other two existing tests (`test_add_move_and_list`, `test_legality_and_board_ascii`) should pass unchanged — they don't touch the new fields.

### Out of scope for tests

Performance benchmarks, `verbose` paths (doesn't exist), `board_ascii` (unchanged), pathological positions beyond what python-chess itself is documented to handle.

---

## Delivery

### Two parallel tracks

**Track 1: Upstream PR**
1. Fork `danilop/chess-support-mcp` to `johnnymo87/chess-support-mcp`.
2. Branch `enriched-status` off upstream `main` at commit `f75662d` (or latest `main` if upstream has moved — rebase if so).
3. Implement per plan (to be written next).
4. Push branch to fork.
5. Open PR against `danilop/chess-support-mcp:main` with title: `Enrich get_status with derived facts; add list_legal_moves_detailed and get_attackers_to`.
6. PR body covers: motivating LLM-reasoning pain points, additive-only backwards compatibility note, why each field stays on the derivation-not-evaluation side of the line, test coverage summary, link to this design doc if useful.

**Track 2: Fork-as-runtime (dogfooding + fallback)**
1. Same fork, same branch, serves as our runtime immediately.
2. Update `/home/dev/projects/chess/opencode.json` to:
   ```json
   "command": [
     "uvx",
     "--from",
     "git+https://github.com/johnnymo87/chess-support-mcp.git@enriched-status",
     "chess-support-mcp"
   ]
   ```
3. If upstream merges → flip `opencode.json` back to `git+https://github.com/danilop/chess-support-mcp.git` (no branch suffix).
4. If upstream stays silent or declines → stay on fork indefinitely. No loss.

### End-to-end validation (before opening PR)

With `opencode.json` pointed at the fork branch:
1. Replay `games/2026-04-12-mohrbacher-dauromonte.pgn` up to ply 38 using the `replaying-a-pgn` skill.
2. Call `list_legal_moves_detailed()`. Assert the result contains `{"uci": "c4a5", "san": "Nxa5"}` — this is the winning move we identified in the first analysis session, and it must appear in the legal-moves list.
3. Call `get_status()`. Confirm `absolute_pins`, `checkers`, `material`, `material_diff` are present and well-formed.
4. Call `get_attackers_to("a5")`. Confirm the White knight on c4 appears in `white`.

If all four pass, PR is ready.

### Documentation updates after PR opens / after fork is in use

- Update `.opencode/skills/improving-the-fact-server/SKILL.md` decisions log — add dated entry linking to PR URL + fork branch + this design doc.
- Update `AGENTS.md` — update the MCP tool list (add `list_legal_moves_detailed`, `get_attackers_to`) and shrink the "Not provided" callout.
- Update `docs/plans/2026-04-19-fact-server-gaps.md` — mark gaps 1, 2, 4 as in-progress or done; leave gaps 3, 5, 6 as open future work.

---

## Risks

- **Upstream objects to `material_diff`** on "feels evaluative" grounds (despite the rename). Mitigation: drop it from PR, keep `material` only. Client can subtract.
- **Upstream objects to `absolute_pins` / `checkers` / `material` living in `get_status`** (wants them in dedicated tools). Mitigation: offer to split. Low likelihood — the existing `get_status` already bundles derived facts like `is_check`, `is_game_over`, `pieces`.
- **`test_checkers_two` is hard to set up via legal play.** Fallback: defer that test (Option (a) in the test plan) or gate behind a test-only FEN loader. Don't let this block the PR.
- **`test_relative_pin_not_reported` setup may be fragile.** Fallback: if a clean construction eludes us, skip with a TODO comment; primary defensive coverage is in `test_absolute_pins_empty_initial`.
- **`board.san()` edge cases** in exotic positions. python-chess's `san()` is battle-tested; our tests exercise the common paths. Exotic cases (4 knights from the same color on a file all attacking the same square) won't arise in our e2e validation.
- **python-chess's `board.pin()` when the king is missing** has special pseudo-chess fallback behavior. Our algorithm guards against missing king (`if king_sq is None: continue`) and is not reachable from our runtime (we only replay legal moves from the starting position).
- **Upstream rebase pain** if `main` has moved since `f75662d`. Mitigation: the diff area (`status()`, a few new tool functions, new test functions) is small and well-isolated. Any conflict will be minor.

---

## Non-goals (explicitly deferred)

- **Gap 3: SAN input / `load_pgn`** — separate PR if this one lands. Much higher value-per-LoC; eliminates the SAN→UCI cascade failure documented in [`replaying-a-pgn` skill](../../.opencode/skills/replaying-a-pgn/SKILL.md).
- **Gap 5: undo / simulate** — needs its own brainstorming session; three plausible API shapes (`undo_last_move`, `peek(seq)`, parallel sandbox) with real tradeoffs.
- **Gap 6: `board_ascii` enhancements** — cosmetic, defer.
- **Relative pins** — if we ever want them, distinct field name, distinct derivation.
- **Point-value material totals** — would cross the derivation/evaluation line.
- **FEN loader** — not part of this PR even if we temporarily need one for `test_checkers_two`. If we add one for tests, it's a private helper, not an MCP tool.

---

## Handoff

**Next step:** invoke the `writing-plans` skill to produce an ordered, verifiable implementation plan at `docs/plans/2026-04-19-enriched-status-plan.md`.

**After the plan is written and approved:**
1. Create fork and branch (first concrete action — pre-implementation).
2. Execute the plan (likely via `executing-plans` with compaction-aware checkpoints, since the plan is expected to span multiple sessions).
3. End-to-end validation per §Delivery.
4. Open PR per §Delivery.
5. Update documentation per §Delivery.
