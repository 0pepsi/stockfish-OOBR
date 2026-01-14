## Overview
A malformed FEN string containing invalid piece characters can corrupt internal Square values in Stockfish, leading to out-of-bounds reads on global bitboard tables (BetweenBB, LineBB).
This results in a global buffer overflow detected by AddressSanitizer during normal engine operation.

The issue is caused by insufficient validation of FEN input before indexing fixed-size global arrays.

## Build
- Stockfish dev-20251203-c109a88e
- Built with AddressSanitizer
- Issue reproducible on current master

## Asan output
```
==2550==ERROR: AddressSanitizer: global-buffer-overflow on address 0x64a29c6a6cd0 at pc 0x64a2959e7311 bp 0x73eb448fe3a0 sp 0x73eb448fe398
READ of size 8 at 0x64a29c6a6cd0 thread T1
    #0 0x64a2959e7310 in set /home/grover/fuzzing/Stockfish/src/bitboard.h:180
    #1 0x64a2959fd54a in _M_invoke /home/grover/fuzzing/Stockfish/src/thread.cpp:290
    #2 0x64a295a133b5 in idle_loop /usr/include/c++/15/bits/std_function.h:593
    #3 0x64a2959f6d3c in _FUN /usr/include/c++/15/bits/std_function.h:593
    #4 0x77eb47e5c0f5 in asan_thread_start ../../../../src/libsanitizer/asan/asan_interceptors.cpp:239
    #5 0x77eb4789cb7a in start_thread nptl/pthread_create.c:448
    #6 0x77eb4791a7b7 in __clone3 ../sysdeps/unix/sysv/linux/x86_64/clone3.S:78

0x64a29c6a6cd0 is located 16 bytes before global variable 'LineBB' defined in 'bitboard.cpp:32:10' (0x64a29c6a6ce0) of size 32768
0x64a29c6a6cd0 is located 16 bytes after global variable 'BetweenBB' defined in 'bitboard.cpp:33:10' (0x64a29c69ecc0) of size 32768
This confirms an out-of-bounds array access between two adjacent global arrays.
```

## PoC
```sh
printf "uci\nisready\nposition fen r1bqkb1r/pppp1ppp/2b2b2/1B2p3/4P3/ffffN1/PPPP1PPP/RNBQK2R w KQkq - 0 1\nposition fen r1bqkb1r/pppp1ppp/2b2b2/1B2p3/4P3/ffffffK1/PPPP1PPP/RNBQK2R w KQkq - 0 1\ngo depth 1\nquit\n" | ./stockfish

Or enter stockfish ./stockfish exec uci + isready + position fen r1bqkb1r/pppp1ppp/2b2b2/1B2p3/4P3/ffffN1/PPPP1PPP/RNBQK2R w KQkq - 0 1 + go depth 10
```

## Root Cause
1> The FEN parser allows invalid piece characters to propagate into the internal board representation instead of rejecting the position early.

as a result, one or more internal Square values become invalid (outside the range 0–63).

2> druing search initialization and move generation stockfish computes line and between bitboards using global lookup tables:
```c
extern Bitboard BetweenBB[SQUARE_NB][SQUARE_NB];
extern Bitboard LineBB[SQUARE_NB][SQUARE_NB];
```
these tables are indexed directly using Square values.

3> Unsafe access in line_bb() / between_bb()

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
resulting in

- assert(is_ok(...)) is compiled out in release builds
- No runtime validation exists
- Invalid Square values are used directly as array indices
- the crash occurs in a worker thread during normal search execution

Thread T1
→ Thread::idle_loop()
→ worker search setup
→ bitboard line computation

This demonstrates that the invalid state survives parsing and affects normal engine execution, not just initialization.

## Security Impact
Type: Global out-of-bounds read
Primitive: Invalid index → uncontrolled memory read
Trigger: Malformed FEN input
Attack Surface: UCI interface / external input

Impact
Engine crash (DoS)

||Potential|| information disclosure in non-ASan builds -> Havent had the time to debug the bug deeper to check whether its exploitable to get a leak or not but looks promising.

