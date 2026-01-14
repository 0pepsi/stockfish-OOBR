## Stockfish FEN Parsing Out of bounds read

---

## Summary

A vulnerability exists in the Stockfish chess engine whereby malformed FEN (Forsyth–Edwards Notation) input containing invalid piece characters can corrupt internal board state. This corrupted state propagates into later execution phases, resulting in out-of-bounds memory accesses in multiple locations, including:

- `std::array<Piece, 64>` accesses during position setup
- Global bitboard lookup tables (`BetweenBB`, `LineBB`) during search initialization and execution

The issue is reproducible on the current master branch and has been confirmed via AddressSanitizer, libstdc++ debug mode, and debugger-assisted analysis.

---

## Affected Product

- **Product:** Stockfish Chess Engine
- **Tested Version:** `dev-20251203-c109a88e`
- **Repository:** https://github.com/official-stockfish/Stockfish
- **Platform:** Linux x86_64
- **Build Variants Affected:**
  - ASan builds (global-buffer-overflow)
  - libstdc++ debug builds (container OOB abort)
  - Release builds (silent corruption / undefined behavior)

---

## Attack Vector

- **Local:** Passing a malformed FEN via stdin / UCI interface
- **Remote:** Any service or application exposing Stockfish and accepting untrusted FEN input (e.g. analysis servers, web backends, chess platforms)

No authentication or special privileges are required.

---

## Root Cause

### Location

The vulnerability originates in the FEN parsing logic implemented in:

```
Position::set()
```

specifically in the loop responsible for processing the *piece placement* portion of the FEN string.

---

## Root Cause Details

### Expected Behavior (per FEN specification)

Valid characters in the piece placement field are strictly limited to:

```
K Q R B N P
k q r b n p
1–8
/
```

Any other character must cause the FEN to be rejected.

---

### Actual Behavior

Invalid characters (e.g. lowercase `'f'`) are **not rejected**.

Instead, they fall through the parsing logic, causing internal state corruption.

---

### Vulnerable Parsing Logic

```c
// FEN parser loop - processes rank characters
while ((ss >> token) && !isspace(token))
{
    if (isdigit(token))
        sq += (token - '0') * EAST;  // Skip empty squares

    else if (token == '/')
        sq += 2 * SOUTH;  // Next rank

    else if ((idx = PieceToChar.find(token)) != string::npos)
    {
        put_piece(Piece(idx), sq);  // Place piece
        ++sq;
    }
    // BUG: No 'else' clause to reject invalid characters
}
```

#### Key Issue

There is **no final `else` clause** to handle invalid, non-digit, non-piece characters.

As a result:
- Invalid characters are silently accepted
- `sq` (the current square index) is not advanced or validated correctly
- Subsequent iterations operate on a corrupted `sq` value

---

### corruption

From GDB:

```
sq = 250
```

Valid square indices must be in the range:

```
0 <= sq <= 63
```

This invalid value propagates into downstream logic.

---

## out of bounds access in `Position::put_piece`

```c
void Position::put_piece(Piece pc, Square s) {
    board[s] = pc;  // OOB when s < 0 or s >= 64
    byTypeBB[ALL_PIECES] |= byTypeBB[type_of(pc)] |= s;
    byColorBB[color_of(pc)] |= s;
    pieceCount[pc]++;
    pieceCount[make_piece(color_of(pc), ALL_PIECES)]++;
}
```

### libstdc++ debug mode

```
Error: attempt to subscript container with out-of-bounds index -6
std::array<Stockfish::Piece, 64>::operator[]
```

Backtrace confirms the fault originates in `Position::put_piece()` called from `Position::set()`.

---

## Global Bitboard Table OOB access

Even when the engine survives `Position::set()`, the corrupted board state persists and later causes out-of-bounds access in global lookup tables.

### Affected Tables

```c
extern Bitboard BetweenBB[SQUARE_NB][SQUARE_NB];
extern Bitboard LineBB[SQUARE_NB][SQUARE_NB];
```

### Accessors (no runtime validation)

```c
inline Bitboard line_bb(Square s1, Square s2) {
    assert(is_ok(s1) && is_ok(s2));
    return LineBB[s1][s2];
}

inline Bitboard between_bb(Square s1, Square s2) {
    assert(is_ok(s1) && is_ok(s2));
    return BetweenBB[s1][s2];
}
```

#### Why this fails

- `assert()` is compiled out in release builds
- Invalid `Square` values derived from malformed FEN are used directly as indices
- Results in out-of-bounds reads between adjacent global arrays

---

## Observed ASan Failure

```
ERROR: AddressSanitizer: global-buffer-overflow
READ of size 8
```

Memory layout confirms the read occurs **between** `BetweenBB` and `LineBB`, proving an index underflow/overflow on the first dimension.

---

## Threaded Execution Context

The crash is observed in a worker thread:

```
Thread T1
→ Thread::idle_loop()
→ worker initialization
→ search setup
→ bitboard computation
```

This demonstrates that:
- The invalid state survives FEN parsing
- The engine proceeds into multithreaded search
- The vulnerability is not confined to initialization logic

---

## Behavioral Spectrum (Empirical)

Based on repeated testing:

1. **Few invalid characters (e.g. 1–4 `'f'`)**
   - Silent corruption
   - Engine continues
   - Incorrect internal state

2. **Moderate amount (≈10–20 `'f'`)**
   - Heap / container corruption
   - Deterministic crashes in debug builds

3. **Large amount (20+ `'f'`)**
   - ASan global-buffer-overflow
   - Hangs / deadlocks due to corrupted control structures

---

## Proof of Concept

```sh
printf "uci\nposition fen r1bqkb1r/pppp1ppp/2b2b2/1B2p3/4P3/ffffffffffffffffffffffN1/PPPP1PPP/RNBQK2R w KQkq - 0 1\ngo depth 1\n" | ./stockfish
```

---

## Security Impact

- **Type:** Out-of-bounds read and write
- **Trigger:** Malformed FEN input
- **Attack Surface:** UCI interface / any FEN-consuming API
- **Impact:**
  - Denial of Service (crash / hang)
  - Silent state corruption
  - Undefined behavior
  - Potential information disclosure in non-sanitized builds

---
