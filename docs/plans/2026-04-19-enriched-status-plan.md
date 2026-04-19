# Enriched Status Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Land a purely-additive PR on `danilop/chess-support-mcp` that exposes derived facts (`absolute_pins`, `checkers`, `material`, `material_diff`) in `get_status()`, plus two new tools (`list_legal_moves_detailed`, `get_attackers_to`), strictly on the derivation-not-evaluation side of the no-engine constraint.

**Architecture:** Single-file MCP server in Python (`src/chess_support_mcp/server.py`). All new facts are mechanical derivations from `python-chess` primitives (`board.legal_moves`, `board.pin()`, `board.is_pinned()`, `board.checkers()`, `board.attackers()`, `board.piece_map()`). Tests are async pytest against a live stdio MCP subprocess. Ships via fork + PR, dogfooded via `opencode.json` pointed at the fork branch.

**Tech Stack:** Python 3.13, `python-chess >= 1.10.0`, `mcp >= 1.2.0`, pytest-anyio.

---

## Prerequisites (read before Task 1)

**Must read:**
- [`docs/plans/2026-04-19-enriched-status-design.md`](2026-04-19-enriched-status-design.md) — full design, rationale, and all inlined external-review findings.
- [`AGENTS.md`](../../AGENTS.md) — workspace philosophy.
- [`.opencode/skills/improving-the-fact-server/SKILL.md`](../../.opencode/skills/improving-the-fact-server/SKILL.md) — the no-engine hard constraint.

**Two repos are involved** — don't conflate them:
- **`/home/dev/projects/chess`** — this analysis workspace. Only two files change here, at the very end (documentation updates in `AGENTS.md`, skill decisions log, `opencode.json`).
- **`/home/dev/projects/chess-support-mcp`** — our local clone of the fork. Does not exist yet; Task 1 creates it. All Python code and test changes happen here.

**GitHub account:** `johnnymo87`. Fork target: `johnnymo87/chess-support-mcp`. Branch: `enriched-status`. Upstream pinned reference: `danilop/chess-support-mcp@f75662d`.

**Commit style:** Match upstream. Looking at upstream's `git log` shows short messages ("Update README.md", etc.). Keep commit messages focused and one-line-subject + optional body. Use `feat:` / `test:` / `docs:` prefixes for our own clarity.

---

## Task 0: Verify design-doc assumptions still hold

**Files:** None modified. Read-only verification.

**Step 1: Confirm upstream hasn't drastically changed**

Run: `gh repo view danilop/chess-support-mcp --json pushedAt,isArchived`

Expected: `isArchived: false`. If `pushedAt` is much newer than 2025-08-13, pull the latest `server.py` and compare against the design doc's line references (`status()` at lines 95–115). If the shape has diverged, pause and update the design doc before proceeding.

**Step 2: Confirm python-chess primitives are where we expect**

Run:
```bash
python3 -c "import chess; b = chess.Board(); print(type(b.legal_moves), type(b.checkers()), b.is_pinned(chess.WHITE, chess.E1))"
```

Expected: `<class 'chess.LegalMoveGenerator'> <class 'chess.SquareSet'> False`. If any of these is missing or has a different type, python-chess has changed — adjust accordingly.

**Step 3: No commit**

Nothing to commit. Proceed to Task 1.

---

## Task 1: Fork upstream, clone locally, create branch

**Files:** None modified inside either repo. Creates a new local checkout at `/home/dev/projects/chess-support-mcp`.

**Step 1: Fork on GitHub**

Run: `gh repo fork danilop/chess-support-mcp --clone=false --remote=false`

Expected output: confirmation that `johnnymo87/chess-support-mcp` was created. If it already exists (idempotent re-run), `gh` will say so and continue.

**Step 2: Clone the fork locally**

Run:
```bash
git clone https://github.com/johnnymo87/chess-support-mcp.git /home/dev/projects/chess-support-mcp
```

Expected: clone completes cleanly.

**Step 3: Add upstream as a remote (for future rebases)**

Run (in the fork checkout):
```bash
git -C /home/dev/projects/chess-support-mcp remote add upstream https://github.com/danilop/chess-support-mcp.git
git -C /home/dev/projects/chess-support-mcp fetch upstream
```

Expected: upstream fetched cleanly.

**Step 4: Create and check out branch `enriched-status`**

Run:
```bash
git -C /home/dev/projects/chess-support-mcp checkout -b enriched-status
```

Expected: branch created from `origin/main` (which tracks our fork's main, which mirrors upstream).

**Step 5: Verify baseline tests pass before we change anything**

Run (in the fork checkout):
```bash
cd /home/dev/projects/chess-support-mcp && uv sync && uv run pytest -q
```

Expected: all 3 existing tests pass. If they don't, stop and investigate — we have a contaminated baseline.

**Step 6: No commit yet** — nothing changed in the fork tree, only branch state. Commits start in Task 2.

---

## Task 2: Point `opencode.json` at the fork branch (dogfooding setup)

**Files:**
- Modify: `/home/dev/projects/chess/opencode.json`

This is done early so that once we start pushing commits to `enriched-status`, we can verify behavior from this analysis workspace without any further config changes.

**Step 1: Update `opencode.json`**

Replace the existing `command` array in `opencode.json` with:

```json
"command": [
  "uvx",
  "--from",
  "git+https://github.com/johnnymo87/chess-support-mcp.git@enriched-status",
  "chess-support-mcp"
]
```

**Step 2: Push the branch so `uvx` can fetch it**

Run:
```bash
git -C /home/dev/projects/chess-support-mcp push -u origin enriched-status
```

Expected: branch pushed to fork. Currently identical to upstream main.

**Step 3: Commit the `opencode.json` change in the analysis repo**

Run:
```bash
cd /home/dev/projects/chess && git add opencode.json && git commit -m "Point chess MCP at fork branch for enriched-status dogfooding"
```

---

## Task 3: Add regression-safety assertions to existing test

**Files:**
- Modify: `/home/dev/projects/chess-support-mcp/tests/test_chess_support_mcp.py` (add assertions to existing `test_reset_and_status` function)

**Step 1: Write the new assertions (they will initially fail)**

Append inside the existing `test_reset_and_status` function, after the final existing assertion, before the `status2 = await ...` line:

```python
        # New enriched-status fields (added in this PR)
        assert isinstance(status["absolute_pins"], list)
        assert isinstance(status["checkers"], list)
        assert isinstance(status["material"], dict)
        assert isinstance(status["material_diff"], dict)
        assert status["material_diff"] == {"Q": 0, "R": 0, "B": 0, "N": 0, "P": 0}
        assert status["material"]["white"] == {"Q": 1, "R": 2, "B": 2, "N": 2, "P": 8}
        assert status["material"]["black"] == {"Q": 1, "R": 2, "B": 2, "N": 2, "P": 8}
        assert status["checkers"] == []
        assert status["absolute_pins"] == []
```

**Step 2: Run test, expect failure**

Run: `cd /home/dev/projects/chess-support-mcp && uv run pytest tests/test_chess_support_mcp.py::test_reset_and_status -v`

Expected: FAIL with `KeyError: 'absolute_pins'` or similar. This confirms the test is actually checking something.

**Step 3: No implementation yet** — proceed to Task 4 to add the fields. This test will go green once material/checkers/absolute_pins are added in Tasks 5–7.

**Step 4: Do NOT commit the failing test yet** — commit will come bundled with the implementation in later tasks.

---

## Task 4: Add `material` and `material_diff` to `get_status()`

**Files:**
- Modify: `/home/dev/projects/chess-support-mcp/src/chess_support_mcp/server.py` (extend `GameState.status()` at lines 95–115)

Simplest derivation first — no edge cases.

**Step 1: Add a new test `test_material_initial`**

Append to `tests/test_chess_support_mcp.py`:

```python
@pytest.mark.anyio
async def test_material_initial():
    async with run_client() as session:
        await session.call_tool("create_or_reset_game", {})
        s = (await session.call_tool("get_status", {})).structuredContent["result"]
        assert s["material"]["white"] == {"Q": 1, "R": 2, "B": 2, "N": 2, "P": 8}
        assert s["material"]["black"] == {"Q": 1, "R": 2, "B": 2, "N": 2, "P": 8}
        assert s["material_diff"] == {"Q": 0, "R": 0, "B": 0, "N": 0, "P": 0}
```

**Step 2: Run it, expect failure**

Run: `uv run pytest tests/test_chess_support_mcp.py::test_material_initial -v`

Expected: FAIL with `KeyError: 'material'`.

**Step 3: Add the second test `test_material_after_capture`**

Append to `tests/test_chess_support_mcp.py`:

```python
@pytest.mark.anyio
async def test_material_after_capture():
    async with run_client() as session:
        await session.call_tool("create_or_reset_game", {})
        for uci in ["e2e4", "d7d5", "e4d5"]:
            await session.call_tool("add_move", {"uci": uci})
        s = (await session.call_tool("get_status", {})).structuredContent["result"]
        assert s["material"]["white"]["P"] == 8
        assert s["material"]["black"]["P"] == 7
        assert s["material_diff"]["P"] == 1
```

**Step 4: Run both, expect both to fail**

Run: `uv run pytest tests/test_chess_support_mcp.py -k material -v`

Expected: both new tests FAIL (KeyError).

**Step 5: Implement `material` and `material_diff` in `GameState`**

In `src/chess_support_mcp/server.py`, add the `PIECE_KEYS` constant near the top (after imports, before the `@dataclass`):

```python
PIECE_KEYS = ("Q", "R", "B", "N", "P")
```

Add a new method inside the `GameState` class (e.g., right before `status()`):

```python
    def material_counts(self) -> Dict[str, Dict[str, int]]:
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
```

In `status()`, after the existing `pieces` line and before the closing `}`, add:

```python
            "material": self.material_counts(),
            "material_diff": {
                k: self.material_counts()["white"][k] - self.material_counts()["black"][k]
                for k in PIECE_KEYS
            },
```

Wait — that calls `material_counts()` twice. Refactor: pull into a local variable.

Replace the above with:

```python
        material = self.material_counts()
        material_diff = {
            k: material["white"][k] - material["black"][k] for k in PIECE_KEYS
        }
```

...right before the `return {` statement, and then reference them in the dict:

```python
            "material": material,
            "material_diff": material_diff,
```

**Step 6: Run both material tests, expect both to pass**

Run: `uv run pytest tests/test_chess_support_mcp.py -k material -v`

Expected: both PASS.

**Step 7: Run full test suite to verify no regression**

Run: `uv run pytest -q`

Expected: all existing tests still pass (the regression assertions from Task 3 still fail because `absolute_pins`/`checkers` aren't added yet — that's expected; those assertions will go green in Tasks 5–6).

**Step 8: Commit**

Run:
```bash
git add src/chess_support_mcp/server.py tests/test_chess_support_mcp.py
git commit -m "feat: add material and material_diff to get_status

Counts per color, no kings, no point-value totals. material_diff is
strictly white[k] - black[k] per piece type."
```

---

## Task 5: Add `checkers` to `get_status()`

**Files:**
- Modify: `/home/dev/projects/chess-support-mcp/src/chess_support_mcp/server.py` (extend `GameState.status()`)
- Modify: `/home/dev/projects/chess-support-mcp/tests/test_chess_support_mcp.py` (add 2 tests; defer the double-check test)

**Step 1: Write `test_checkers_zero`**

Append:

```python
@pytest.mark.anyio
async def test_checkers_zero():
    async with run_client() as session:
        await session.call_tool("create_or_reset_game", {})
        s = (await session.call_tool("get_status", {})).structuredContent["result"]
        assert s["checkers"] == []
```

**Step 2: Write `test_checkers_one`**

Append:

```python
@pytest.mark.anyio
async def test_checkers_one():
    # Scholar's Mate: Qxf7#. After 7. Qxf7+, Black's king is in check from the queen.
    async with run_client() as session:
        await session.call_tool("create_or_reset_game", {})
        for uci in ["e2e4", "e7e5", "f1c4", "b8c6", "d1h5", "g8f6", "h5f7"]:
            await session.call_tool("add_move", {"uci": uci})
        s = (await session.call_tool("get_status", {})).structuredContent["result"]
        assert s["is_check"] is True
        assert s["checkers"] == [{"square": "f7", "piece": "Q"}]
```

**Step 3: Run both, expect failure**

Run: `uv run pytest tests/test_chess_support_mcp.py -k checkers -v`

Expected: FAIL (KeyError on `checkers`).

**Step 4: Implement `checkers` in `GameState.status()`**

Add a helper method on `GameState`, near `material_counts`:

```python
    def checkers_info(self) -> List[Dict[str, str]]:
        result: List[Dict[str, str]] = []
        for sq in self.board.checkers():
            piece = self.board.piece_at(sq)
            result.append({
                "square": chess.square_name(sq),
                "piece": piece.symbol() if piece else "?",
            })
        return result
```

In `status()`, add to the returned dict (near `is_check`):

```python
            "checkers": self.checkers_info(),
```

**Step 5: Run both tests, expect pass**

Run: `uv run pytest tests/test_chess_support_mcp.py -k checkers -v`

Expected: both PASS.

**Step 6: Defer `test_checkers_two`**

The discovered-double-check test is awkward to set up via legal play from the initial position. Add this as a skipped test with an explanatory comment:

```python
@pytest.mark.anyio
@pytest.mark.skip(reason="Double-check setup via legal play from initial position is awkward; "
                        "deferred until/unless we add a test-only FEN loader. "
                        "See docs/plans/2026-04-19-enriched-status-design.md Test 9.")
async def test_checkers_two():
    pass
```

This documents intent without burning time on a fragile fixture.

**Step 7: Run full test suite**

Run: `uv run pytest -q`

Expected: all non-skipped tests pass. Skipped tests counted separately.

**Step 8: Commit**

```bash
git add src/chess_support_mcp/server.py tests/test_chess_support_mcp.py
git commit -m "feat: add checkers to get_status

Rich objects with {square, piece}. Covers 0 and 1 checker cases; the
discovered double-check test is skipped pending a test-only FEN loader."
```

---

## Task 6: Add `absolute_pins` to `get_status()`

**Files:**
- Modify: `/home/dev/projects/chess-support-mcp/src/chess_support_mcp/server.py`
- Modify: `/home/dev/projects/chess-support-mcp/tests/test_chess_support_mcp.py`

Most complex derivation. Do the helper first, pin-detection second.

**Step 1: Write `test_absolute_pins_empty_initial`**

Append:

```python
@pytest.mark.anyio
async def test_absolute_pins_empty_initial():
    async with run_client() as session:
        await session.call_tool("create_or_reset_game", {})
        s = (await session.call_tool("get_status", {})).structuredContent["result"]
        assert s["absolute_pins"] == []
```

**Step 2: Write `test_absolute_pins_detected`**

```python
@pytest.mark.anyio
async def test_absolute_pins_detected():
    # After 1. e4 e5 2. Nf3 Nc6 3. Bb5, White's Bb5 pins Black's Nc6 to Ke8
    # along the a4-e8 diagonal.
    async with run_client() as session:
        await session.call_tool("create_or_reset_game", {})
        for uci in ["e2e4", "e7e5", "g1f3", "b8c6", "f1b5"]:
            await session.call_tool("add_move", {"uci": uci})
        s = (await session.call_tool("get_status", {})).structuredContent["result"]
        pins = s["absolute_pins"]
        assert len(pins) == 1
        p = pins[0]
        assert p["color"] == "black"
        assert p["pinned_square"] == "c6"
        assert p["pinned_piece"] == "n"
        assert p["king_square"] == "e8"
        assert p["pinner_square"] == "b5"
        assert p["pinner_piece"] == "B"
        # Ray is the pin line from king outward: e8, d7, c6, b5, and possibly a4.
        assert p["ray"][0] == "e8"
        assert "c6" in p["ray"]
        assert "b5" in p["ray"]
        # All squares on ray should be on the a4-e8 diagonal (strictly collinear).
        # Just check ordering starts at the king.
```

**Step 3: Write `test_relative_pin_not_reported`**

```python
@pytest.mark.anyio
async def test_relative_pin_not_reported():
    # After 1. d4 d5 2. Bf4 Nc6 3. Nf3 Bg4, Bg4 relatively pins Nf3 to Qd1
    # (through the queen, not the king). No king involvement => no absolute pin.
    async with run_client() as session:
        await session.call_tool("create_or_reset_game", {})
        for uci in ["d2d4", "d7d5", "c1f4", "b8c6", "g1f3", "c8g4"]:
            await session.call_tool("add_move", {"uci": uci})
        s = (await session.call_tool("get_status", {})).structuredContent["result"]
        # There may be 0 absolute pins, or there could be one we don't expect.
        # The specific assertion: Nf3 is NOT reported as pinned (it's only relatively pinned).
        for p in s["absolute_pins"]:
            assert p["pinned_square"] != "f3", \
                f"Nf3 is only relatively pinned (to Qd1), should not appear in absolute_pins"
```

**Step 4: Run all three, expect failure**

Run: `uv run pytest tests/test_chess_support_mcp.py -k "absolute_pins or relative_pin" -v`

Expected: all three FAIL with `KeyError: 'absolute_pins'`.

**Step 5: Implement `_sort_ray_from_king` helper**

In `server.py`, add as a module-level function (outside any class):

```python
def _sort_ray_from_king(king_sq: int, ray_set: "chess.SquareSet") -> List[int]:
    """Order squares in ray_set (collinear with king_sq) from king outward."""
    squares = [sq for sq in ray_set if sq != king_sq]
    if not squares:
        return [king_sq]
    kf, kr = chess.square_file(king_sq), chess.square_rank(king_sq)
    # Use the nearest-by-Chebyshev-distance square to fix direction robustly.
    nearest = min(squares, key=lambda s: max(
        abs(chess.square_file(s) - kf),
        abs(chess.square_rank(s) - kr),
    ))
    nf, nr = chess.square_file(nearest), chess.square_rank(nearest)
    df = 0 if nf == kf else (1 if nf > kf else -1)
    dr = 0 if nr == kr else (1 if nr > kr else -1)
    def projection(sq: int) -> int:
        f, r = chess.square_file(sq), chess.square_rank(sq)
        return (f - kf) * df + (r - kr) * dr
    return sorted(ray_set, key=projection)
```

**Step 6: Implement `absolute_pins_info` on `GameState`**

Add the method:

```python
    def absolute_pins_info(self) -> List[Dict[str, Any]]:
        pins: List[Dict[str, Any]] = []
        for color in (chess.WHITE, chess.BLACK):
            king_sq = self.board.king(color)
            if king_sq is None:
                continue
            color_name = "white" if color == chess.WHITE else "black"
            for square, piece in self.board.piece_map().items():
                if piece.color != color:
                    continue
                if not self.board.is_pinned(color, square):
                    continue
                ray_set = self.board.pin(color, square)
                pinner_sq = next(
                    (sq for sq in ray_set
                     if self.board.piece_at(sq) is not None
                     and self.board.piece_at(sq).color != color),
                    None,
                )
                if pinner_sq is None:
                    continue  # defensive; is_pinned() shouldn't return True without a pinner
                ray_squares = _sort_ray_from_king(king_sq, ray_set)
                pinner_piece = self.board.piece_at(pinner_sq)
                pins.append({
                    "color": color_name,
                    "pinned_square": chess.square_name(square),
                    "pinned_piece": piece.symbol(),
                    "king_square": chess.square_name(king_sq),
                    "pinner_square": chess.square_name(pinner_sq),
                    "pinner_piece": pinner_piece.symbol() if pinner_piece else "?",
                    "ray": [chess.square_name(sq) for sq in ray_squares],
                })
        return pins
```

In `status()`, add:

```python
            "absolute_pins": self.absolute_pins_info(),
```

**Step 7: Run the three pin tests**

Run: `uv run pytest tests/test_chess_support_mcp.py -k "absolute_pins or relative_pin" -v`

Expected: all three PASS.

**Step 8: Run full test suite — the Task-3 regression assertions should now ALL go green**

Run: `uv run pytest -q`

Expected: all tests pass except the single explicitly-skipped `test_checkers_two`.

**Step 9: Commit**

```bash
git add src/chess_support_mcp/server.py tests/test_chess_support_mcp.py
git commit -m "feat: add absolute_pins to get_status

Absolute pins only (pinned-to-king). Each entry carries the color,
pinned square/piece, king square, pinner square/piece, and the full
pin ray ordered from the king outward. Relative pins (through queen/
rook) are out of scope and documented as such."
```

---

## Task 7: Add `list_legal_moves_detailed` tool

**Files:**
- Modify: `/home/dev/projects/chess-support-mcp/src/chess_support_mcp/server.py`
- Modify: `/home/dev/projects/chess-support-mcp/tests/test_chess_support_mcp.py`

**Step 1: Write three tests**

```python
@pytest.mark.anyio
async def test_list_legal_moves_detailed_initial():
    async with run_client() as session:
        await session.call_tool("create_or_reset_game", {})
        result = (await session.call_tool("list_legal_moves_detailed", {})).structuredContent["result"]
        assert len(result) == 20
        assert {"uci": "e2e4", "san": "e4"} in result
        # Sorted by UCI ascending
        ucis = [m["uci"] for m in result]
        assert ucis == sorted(ucis)


@pytest.mark.anyio
async def test_list_legal_moves_detailed_checkmate():
    # Fool's Mate
    async with run_client() as session:
        await session.call_tool("create_or_reset_game", {})
        for uci in ["f2f3", "e7e5", "g2g4", "d8h4"]:
            await session.call_tool("add_move", {"uci": uci})
        status = (await session.call_tool("get_status", {})).structuredContent["result"]
        assert status["is_game_over"] is True
        result = (await session.call_tool("list_legal_moves_detailed", {})).structuredContent["result"]
        assert result == []


@pytest.mark.anyio
async def test_list_legal_moves_detailed_sorted():
    async with run_client() as session:
        await session.call_tool("create_or_reset_game", {})
        await session.call_tool("add_move", {"uci": "e2e4"})
        result = (await session.call_tool("list_legal_moves_detailed", {})).structuredContent["result"]
        ucis = [m["uci"] for m in result]
        assert ucis == sorted(ucis)
```

**Step 2: Run, expect failure**

Run: `uv run pytest tests/test_chess_support_mcp.py -k list_legal_moves -v`

Expected: FAIL with tool-not-found errors.

**Step 3: Implement the tool**

In `server.py`, add near the other `@server.tool()` functions:

```python
@server.tool()
def list_legal_moves_detailed() -> List[Dict[str, Any]]:
    """Return all legal moves in the current position, sorted by UCI ascending.

    Each item: { uci: string, san: string }.

    Notes:
    - Ordering: sorted by uci ascending (stable, deterministic).
    - Empty list in checkmate or stalemate.
    - This tool enumerates legality; it does not suggest or score moves.
    """
    moves = [
        {"uci": m.uci(), "san": _GAME.board.san(m)}
        for m in _GAME.board.legal_moves
    ]
    moves.sort(key=lambda x: x["uci"])
    return moves
```

**Step 4: Run, expect pass**

Run: `uv run pytest tests/test_chess_support_mcp.py -k list_legal_moves -v`

Expected: all three PASS.

**Step 5: Run full suite**

Run: `uv run pytest -q`

**Step 6: Commit**

```bash
git add src/chess_support_mcp/server.py tests/test_chess_support_mcp.py
git commit -m "feat: add list_legal_moves_detailed tool

Returns all legal moves as [{uci, san}], sorted by uci ascending for
deterministic output. Separate tool (not bundled into get_status) to
match upstream's existing lean-status + detailed-tools pattern."
```

---

## Task 8: Add `get_attackers_to` tool

**Files:**
- Modify: `/home/dev/projects/chess-support-mcp/src/chess_support_mcp/server.py`
- Modify: `/home/dev/projects/chess-support-mcp/tests/test_chess_support_mcp.py`

**Step 1: Write three tests**

```python
@pytest.mark.anyio
async def test_get_attackers_to_basic():
    # After 1. e4 e5 2. Nf3, White's Nf3 attacks e5.
    async with run_client() as session:
        await session.call_tool("create_or_reset_game", {})
        for uci in ["e2e4", "e7e5", "g1f3"]:
            await session.call_tool("add_move", {"uci": uci})
        result = (await session.call_tool("get_attackers_to", {"square": "e5"})).structuredContent["result"]
        assert result["square"] == "e5"
        white_squares = {a["square"] for a in result["white"]}
        assert "f3" in white_squares
        # Confirm piece info is included
        nf3 = next(a for a in result["white"] if a["square"] == "f3")
        assert nf3["piece"] == "N"


@pytest.mark.anyio
async def test_get_attackers_to_includes_pinned():
    # Reuse the Bb5-pin position from test_absolute_pins_detected.
    # Nc6 is pinned, but it still attacks e5, d4, etc. as a static-map attacker.
    async with run_client() as session:
        await session.call_tool("create_or_reset_game", {})
        for uci in ["e2e4", "e7e5", "g1f3", "b8c6", "f1b5"]:
            await session.call_tool("add_move", {"uci": uci})
        result = (await session.call_tool("get_attackers_to", {"square": "d4"})).structuredContent["result"]
        # The pinned Nc6 attacks d4. It still appears in the attack map.
        black_squares = {a["square"] for a in result["black"]}
        assert "c6" in black_squares, "Pinned knight should still appear as static-map attacker"


@pytest.mark.anyio
async def test_get_attackers_to_parse_error():
    async with run_client() as session:
        await session.call_tool("create_or_reset_game", {})
        result = (await session.call_tool("get_attackers_to", {"square": "zz9"})).structuredContent["result"]
        assert result["square"] == "zz9"
        assert "parse_error" in result
        assert result["white"] == []
        assert result["black"] == []
```

**Step 2: Run, expect failure**

Run: `uv run pytest tests/test_chess_support_mcp.py -k get_attackers -v`

Expected: FAIL with tool-not-found.

**Step 3: Implement the tool**

In `server.py`:

```python
@server.tool()
def get_attackers_to(square: str) -> Dict[str, Any]:
    """Return all pieces attacking the given square, by color.

    Parameters:
    - square: algebraic square name like "e4".

    Returns:
    - { square, white: [{square, piece}, ...], black: [{square, piece}, ...] }
    - On parse error: { square, parse_error: str, white: [], black: [] }

    IMPORTANT: This is the static attack map. Pinned pieces are still counted
    as attackers, and x-ray attacks through the king are not considered. This
    tool does not answer "who can legally capture on this square right now" —
    to answer that, filter list_legal_moves_detailed() by destination square.
    """
    try:
        sq = chess.parse_square(square)
    except ValueError as e:
        return {"square": square, "parse_error": str(e), "white": [], "black": []}

    def format_attackers(color: chess.Color) -> List[Dict[str, str]]:
        entries: List[Dict[str, str]] = []
        for s in _GAME.board.attackers(color, sq):
            piece = _GAME.board.piece_at(s)
            entries.append({
                "square": chess.square_name(s),
                "piece": piece.symbol() if piece else "?",
            })
        return entries

    return {
        "square": square,
        "white": format_attackers(chess.WHITE),
        "black": format_attackers(chess.BLACK),
    }
```

**Step 4: Run, expect pass**

Run: `uv run pytest tests/test_chess_support_mcp.py -k get_attackers -v`

Expected: all three PASS.

**Step 5: Run full suite**

Run: `uv run pytest -q`

Expected: everything passes except the one explicitly-skipped test.

**Step 6: Commit**

```bash
git add src/chess_support_mcp/server.py tests/test_chess_support_mcp.py
git commit -m "feat: add get_attackers_to tool

Returns static attack map {white:[{square,piece}], black:[{square,piece}]}
for a given square. Pinned pieces still count as attackers; docstring
warns callers not to use this as a legality oracle."
```

---

## Task 9: Promotion-aware material test

**Files:**
- Modify: `/home/dev/projects/chess-support-mcp/tests/test_chess_support_mcp.py`

This test verifies that `material` shapes don't silently cap at starting-position maxima. Constructing a promotion via legal play from the initial position takes a non-trivial sequence; we'll use a short forced line.

**Step 1: Find a short sequence that legally promotes a pawn**

A compact line: `h2h4, g7g5, h4g5, f7f6, g5f6, e7e5, f6f7, e8e7, f7f8q` — White plays 5 captures + advances to promote on f8. Verify the line is legal by running it through python-chess interactively, then harden:

Actually, let's prefer: `a2a4, b7b5, a4b5, c7c6, b5c6, d7d6, c6b7, b8d7, b7a8q` — 9 plies, ends with White promoting a pawn to a queen on a8. White now has 2 queens.

Verify this is legal before committing the test. During implementation, run it against the live server first via a throwaway test or interactive session.

**Step 2: Write the test**

```python
@pytest.mark.anyio
async def test_material_after_promotion():
    # a-file pawn marches to a8 with captures, promotes to queen.
    async with run_client() as session:
        await session.call_tool("create_or_reset_game", {})
        moves = ["a2a4", "b7b5", "a4b5", "c7c6", "b5c6", "d7d6", "c6b7", "b8d7", "b7a8q"]
        for uci in moves:
            resp = await session.call_tool("add_move", {"uci": uci})
            assert resp.structuredContent["result"]["accepted"] is True, \
                f"Move {uci} unexpectedly rejected; check the line"
        s = (await session.call_tool("get_status", {})).structuredContent["result"]
        # White now has 2 queens.
        assert s["material"]["white"]["Q"] == 2, \
            f"Expected 2 white queens after promotion, got {s['material']['white']['Q']}"
        assert s["material_diff"]["Q"] == 1  # white has 2, black has 1
```

**Step 3: Run, expect pass (no implementation change needed — Task 4's material code should already handle this)**

Run: `uv run pytest tests/test_chess_support_mcp.py -k promotion -v`

Expected: PASS. If it fails because the move sequence is illegal, adjust the sequence; the material implementation itself should not be the bottleneck.

**Step 4: Run full suite**

Run: `uv run pytest -q`

**Step 5: Commit**

```bash
git add tests/test_chess_support_mcp.py
git commit -m "test: verify material counts handle pawn promotion correctly

Covers the case where a pawn promotes to queen, producing 2+ queens
for one side. material field must not silently cap at starting-position
maxima."
```

---

## Task 10: Update upstream README with new tool documentation

**Files:**
- Modify: `/home/dev/projects/chess-support-mcp/README.md`

The PR should include README updates so the maintainer doesn't have to ask. Keep it minimal — match existing style.

**Step 1: Read existing README**

```bash
cat /home/dev/projects/chess-support-mcp/README.md
```

Look for the section listing tools. Note the format.

**Step 2: Add entries for the two new tools**

In the tools list, append (match the existing format, which typically names the tool and a one-line description):

- `list_legal_moves_detailed` — returns all legal moves in the current position as `[{uci, san}]`, sorted by UCI.
- `get_attackers_to(square)` — returns the static attack map for a square as `{white: [{square, piece}], black: [{square, piece}]}`. Note: pinned pieces still count as attackers; this is not a legality oracle.

In the `get_status` description (if the README describes its return shape), note the new fields: `absolute_pins`, `checkers`, `material`, `material_diff`. Use the exact same descriptive style as existing fields.

**Step 3: No test step — this is documentation**

**Step 4: Commit**

```bash
git add README.md
git commit -m "docs: document new tools and get_status fields"
```

---

## Task 11: End-to-end validation against the real game

**Files:** None modified. Dogfood validation using our analysis repo.

This is where we verify the whole thing actually works as intended for our motivating use case.

**Step 1: Push the branch with all commits**

Run:
```bash
git -C /home/dev/projects/chess-support-mcp push
```

Expected: fast-forward push to `origin/enriched-status` on the fork.

**Step 2: Restart OpenCode / flush uvx cache so the new branch is picked up**

The `uvx --from git+https://...@branch` path caches by commit. Force a refresh by running:

```bash
uvx --from "git+https://github.com/johnnymo87/chess-support-mcp.git@enriched-status" --refresh chess-support-mcp --help 2>&1 | head -5
```

Then restart the OpenCode session so it re-initializes the MCP with the new binary.

**Step 3: Replay the motivating game**

In the analysis session, use the `replaying-a-pgn` skill with `games/2026-04-12-mohrbacher-dauromonte.pgn`. Replay up to ply 38 (the point where White is to move and `Nxa5` is the winning move).

**Step 4: Verify `list_legal_moves_detailed` contains `Nxa5`**

Call the new tool. Assert (manually, by reading output) that `{"uci": "c4a5", "san": "Nxa5"}` is present in the returned array.

**Step 5: Verify `get_status` returns sensible `absolute_pins`, `checkers`, `material`**

Call `get_status`. Confirm:
- `absolute_pins` is an array (probably empty in this position — no pins at ply 38).
- `checkers` is `[]` (White is not in check).
- `material` counts match the ASCII board.
- `material_diff` shows the ~8-point White deficit (but computed as per-piece diffs, so probably shows `{Q:0, R:-1, B:-2, N:0, P:-2}` or similar — the exact imbalance from the analysis session).

**Step 6: Verify `get_attackers_to("a5")` shows the White knight**

Call `get_attackers_to(square="a5")`. Confirm the White knight on c4 appears in the `white` list.

**Step 7: If all four pass, the PR is ready**

No commit in this task — purely runtime validation. If anything fails here, go back and fix the relevant implementation before proceeding.

---

## Task 12: Open the pull request

**Files:** None modified in either repo.

**Step 1: Draft the PR body**

Write the PR body to a scratch file (e.g., `/tmp/pr-body.md`) with these sections:

- **Motivation**: the LLM-legality-drift pain points (legal moves from imagination, pins not visible, arithmetic drift on material). Cite that this is for "LLMs reasoning about chess with the MCP as a deterministic fact server."
- **Scope**: 4 new fields in `get_status`, 2 new tools. Purely additive. Nothing renamed.
- **Why each field is a mechanical derivation, not evaluation**: quote the `material_diff` is `white[k] - black[k]`; quote the `get_attackers_to` docstring warning; note that `absolute_pins` uses `board.pin()` directly.
- **Testing**: list the new tests, note the one skipped test and why.
- **Backwards compatibility**: no existing field changed; no tool signature changed.
- **Acknowledgements**: link to design doc if useful (commit hash in our analysis repo once a remote exists).

**Step 2: Open the PR**

Run:
```bash
cd /home/dev/projects/chess-support-mcp
gh pr create \
  --repo danilop/chess-support-mcp \
  --base main \
  --head johnnymo87:enriched-status \
  --title "Enrich get_status with derived facts; add list_legal_moves_detailed and get_attackers_to" \
  --body-file /tmp/pr-body.md
```

**Step 3: Capture PR URL**

Record the PR URL in `.opencode/skills/improving-the-fact-server/SKILL.md` decisions log (next task).

---

## Task 13: Update documentation in the analysis workspace

**Files:**
- Modify: `/home/dev/projects/chess/AGENTS.md` (MCP tool list)
- Modify: `/home/dev/projects/chess/.opencode/skills/improving-the-fact-server/SKILL.md` (decisions log + gap status)
- Modify: `/home/dev/projects/chess/docs/plans/2026-04-19-fact-server-gaps.md` (mark gaps addressed)

**Step 1: Update `AGENTS.md`**

In the `## The MCP` section:
- Add `list_legal_moves_detailed` and `get_attackers_to` to the tools list with one-line descriptions.
- Update the "Not provided" line to remove: legal-move list, pin detection, attackers-to-square, material count. It should now read something like: "**Not provided:** SAN input, undo/simulate, relative pins, `board_ascii` enhancements."

**Step 2: Update `.opencode/skills/improving-the-fact-server/SKILL.md`**

Append to the "Decisions log" section:

```markdown
- **2026-04-19:** Gaps 1, 2, 4 addressed via fork+PR. Design at
  [`docs/plans/2026-04-19-enriched-status-design.md`](../../../docs/plans/2026-04-19-enriched-status-design.md).
  Plan at [`docs/plans/2026-04-19-enriched-status-plan.md`](../../../docs/plans/2026-04-19-enriched-status-plan.md).
  Fork: `github.com/johnnymo87/chess-support-mcp@enriched-status`. Upstream PR: <URL from Task 12>.
  `opencode.json` points at the fork branch; will flip back to upstream if/when PR merges.
  Gaps 3, 5, 6 remain open.
```

Also update the individual gap sections (1, 2, 4): change `**Status:** Not in upstream.` to `**Status:** Addressed in fork; PR pending at <URL>.`

**Step 3: Update `docs/plans/2026-04-19-fact-server-gaps.md`**

Update the `**Status:**` line at the top:

```markdown
**Status:** Implementation in progress. Design at
[`2026-04-19-enriched-status-design.md`](2026-04-19-enriched-status-design.md).
Plan at [`2026-04-19-enriched-status-plan.md`](2026-04-19-enriched-status-plan.md).
Gaps 1+2+4 addressed in the enriched-status PR; gaps 3, 5, 6 still open.
```

**Step 4: Commit all three changes together**

```bash
cd /home/dev/projects/chess
git add AGENTS.md .opencode/skills/improving-the-fact-server/SKILL.md docs/plans/2026-04-19-fact-server-gaps.md
git commit -m "docs: reflect enriched-status PR in workspace docs

Updates the MCP tool list in AGENTS.md, marks gaps 1/2/4 as addressed
in the improving-the-fact-server skill's decisions log and per-gap
status fields, and annotates the original fact-server-gaps plan with a
pointer to the design and implementation plan."
```

---

## What to do if the PR is merged / declined / stale

**If merged:** flip `opencode.json` back to pointing at upstream `main` (remove the `@enriched-status` suffix). Commit that change in the analysis repo.

**If declined:** stay on the fork indefinitely. Update the decisions log to note the outcome and any maintainer feedback worth preserving for future PRs.

**If stale (>4 weeks no response):** same as declined. Optionally nudge the maintainer with a polite comment on the PR after ~2 weeks.

---

## Compaction-safe resumption notes

If this implementation spans multiple compactions:

- **Always start by reading this plan + the design doc** in the order given in the design doc's "Resumption checklist".
- **Check git status in both repos** (`/home/dev/projects/chess` and `/home/dev/projects/chess-support-mcp`) to see where you are.
- **Run `uv run pytest -q` in the fork checkout** to see which tests pass — that's your ground truth for what's been implemented.
- **The task numbering in this plan is the source of truth for progress.** Each task ends in a commit, so `git log --oneline` in the fork checkout tells you the last completed task.
