# Chapter 28: Debugging Techniques

Chess engines are complex. Bugs can be subtle—wrong results, not crashes. This chapter covers systematic debugging approaches.

## Perft Testing

**Perft** (Performance Test) counts nodes at a given depth, verifying move generation and make/unmake:

```c
uint64_t perft(Position* pos, int depth) {
    if (depth == 0) return 1;

    MoveList moves;
    generate_legal_moves(pos, &moves);

    uint64_t nodes = 0;
    for (int i = 0; i < moves.count; i++) {
        make_move(pos, moves.moves[i]);
        nodes += perft(pos, depth - 1);
        unmake_move(pos, moves.moves[i]);
    }

    return nodes;
}
```

### Known Results

Starting position:
```
Depth 1:        20
Depth 2:       400
Depth 3:     8,902
Depth 4:   197,281
Depth 5: 4,865,609
Depth 6: 119,060,324
```

If your results differ, there's a bug in move generation or make/unmake.

### Divide and Conquer

Use "divide" to isolate bugs:

```c
void perft_divide(Position* pos, int depth) {
    MoveList moves;
    generate_legal_moves(pos, &moves);

    uint64_t total = 0;
    for (int i = 0; i < moves.count; i++) {
        make_move(pos, moves.moves[i]);
        uint64_t count = perft(pos, depth - 1);
        unmake_move(pos, moves.moves[i]);

        printf("%s: %llu\n", move_to_uci(moves.moves[i]), count);
        total += count;
    }
    printf("Total: %llu\n", total);
}
```

Compare against a known-good engine. When counts differ, recurse into that move.

### Tricky Positions

Test edge cases:
```
// Position with en passant
fen: rnbqkbnr/ppp1p1pp/8/3pPp2/8/8/PPPP1PPP/RNBQKBNR w KQkq f6 0 3

// Position with castling rights
fen: r3k2r/pppppppp/8/8/8/8/PPPPPPPP/R3K2R w KQkq - 0 1

// Position with promotions
fen: 8/P7/8/8/8/8/p7/8 w - - 0 1
```

## Hash Verification

Periodically verify incremental hash matches recomputed hash:

```c
void verify_hash(Position* pos) {
    uint64_t computed = compute_hash_from_scratch(pos);
    if (pos->hash != computed) {
        printf("Hash mismatch!\n");
        printf("Stored: %llx\n", pos->hash);
        printf("Computed: %llx\n", computed);
        abort();
    }
}

// In search, occasionally:
if (DEBUG && (nodes & 0xFFFF) == 0) {
    verify_hash(pos);
}
```

## Position Integrity Checks

Verify position consistency:

```c
void verify_position(Position* pos) {
    // Check piece bitboards match board array
    for (int sq = 0; sq < 64; sq++) {
        int piece = pos->board[sq];
        if (piece != EMPTY) {
            int color = piece_color(piece);
            int type = piece_type(piece);
            if (!(pos->pieces[color][type] & (1ULL << sq))) {
                printf("Bitboard mismatch at %d\n", sq);
                abort();
            }
        }
    }

    // Check aggregate bitboards
    uint64_t all_white = 0;
    for (int pt = 0; pt < 6; pt++) {
        all_white |= pos->pieces[WHITE][pt];
    }
    if (all_white != pos->all_pieces[WHITE]) {
        printf("White aggregate mismatch\n");
        abort();
    }

    // Check king count
    if (popcount(pos->pieces[WHITE][KING]) != 1) {
        printf("White has %d kings\n", popcount(pos->pieces[WHITE][KING]));
        abort();
    }
}
```

## Reproducible Bugs

When a bug occurs:

1. **Save the position** (FEN string)
2. **Save the search parameters** (depth, time)
3. **Save the random seed** if using randomness

```c
void log_search_start(Position* pos, SearchLimits* limits) {
    char fen[256];
    position_to_fen(pos, fen);
    FILE* f = fopen("debug_log.txt", "a");
    fprintf(f, "Position: %s\n", fen);
    fprintf(f, "Depth: %d, Time: %d\n", limits->depth, limits->movetime);
    fclose(f);
}
```

## Common Bug Patterns

### Off-by-One in Square Indexing

```c
// WRONG
for (int sq = 0; sq <= 64; sq++)  // Should be < 64

// WRONG
int rank = sq / 7;  // Should be / 8
```

### Missing Unmake

```c
// WRONG
for (each move) {
    make_move(pos, move);
    int score = -search(pos, depth - 1);
    if (score > best) {
        best = score;
        // BUG: Forgot to unmake before continuing!
    }
    unmake_move(pos, move);  // This should always execute
}
```

### Sign Errors in Negamax

```c
// WRONG
score = search(pos, depth - 1, -beta, -alpha);  // Missing negation

// CORRECT
score = -search(pos, depth - 1, -beta, -alpha);
```

### Castling Rights After Capture

```c
// WRONG: Only checking 'from' square
update_castling_rights(from);

// CORRECT: Also check 'to' square (rook capture)
update_castling_rights(from);
update_castling_rights(to);
```

### En Passant Hash

```c
// WRONG: Forgetting to unhash old EP square
pos->en_passant = new_ep_square;
pos->hash ^= EP_HASH[new_ep_square];

// CORRECT
if (pos->en_passant != NO_SQUARE) {
    pos->hash ^= EP_HASH[pos->en_passant];  // Remove old
}
pos->en_passant = new_ep_square;
if (new_ep_square != NO_SQUARE) {
    pos->hash ^= EP_HASH[new_ep_square];  // Add new
}
```

## Assertion Strategies

Use assertions liberally during development:

```c
#ifdef DEBUG
#define ASSERT(cond) do { if (!(cond)) { \
    printf("Assertion failed: %s at %s:%d\n", #cond, __FILE__, __LINE__); \
    abort(); \
}} while(0)
#else
#define ASSERT(cond)
#endif

// Usage
ASSERT(score > -INFINITY && score < INFINITY);
ASSERT(depth >= 0);
ASSERT(move != MOVE_NONE);
```

## Logging

Add conditional logging:

```c
#ifdef DEBUG_SEARCH
#define SEARCH_LOG(...) printf(__VA_ARGS__)
#else
#define SEARCH_LOG(...)
#endif

// In search:
SEARCH_LOG("Depth %d, alpha=%d, beta=%d, move=%s\n",
           depth, alpha, beta, move_to_uci(move));
```

## Test Positions

### Tactical Suites

Run standard test suites:
- WAC (Win at Chess)
- STS (Strategic Test Suite)
- ECM (Encyclopedia of Chess Middlegames)

```c
void run_test_suite(const char* filename) {
    FILE* f = fopen(filename, "r");
    char line[1024];

    int passed = 0, failed = 0;
    while (fgets(line, sizeof(line), f)) {
        char fen[256], best_move[8];
        parse_epd(line, fen, best_move);

        set_fen(&position, fen);
        Move found = search_to_depth(&position, 12);

        if (move_matches(found, best_move)) {
            passed++;
        } else {
            printf("FAILED: %s, expected %s, got %s\n",
                   fen, best_move, move_to_uci(found));
            failed++;
        }
    }
    printf("Passed: %d, Failed: %d\n", passed, failed);
}
```

### Regression Testing

After any change, run perft and test positions to catch regressions.

## Memory Issues

Use tools like Valgrind (Linux) or AddressSanitizer:

```bash
# Compile with sanitizers
gcc -fsanitize=address -g -o engine engine.c

# Run with Valgrind
valgrind --leak-check=full ./engine
```

## Summary

Debugging techniques:

- **Perft**: Verify move generation counts
- **Divide**: Isolate bugs by comparing per-move counts
- **Hash verification**: Ensure incremental hash matches full computation
- **Position checks**: Verify bitboard consistency
- **Assertions**: Catch invalid states during development
- **Test suites**: Verify tactical correctness
- **Regression testing**: Prevent reintroducing bugs

Systematic debugging saves time. Build verification into your development process.

Part 5 complete. Next: Testing and tuning your engine.
