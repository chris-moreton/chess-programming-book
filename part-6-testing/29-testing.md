# Chapter 29: Testing Your Engine

Thorough testing separates working engines from tournament-ready ones. This chapter covers testing methodologies for chess engines.

## Perft Testing

**Perft** (Performance Test) is the foundation of move generation testing:

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

### Standard Perft Results

Starting position (`rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w KQkq - 0 1`):

| Depth | Nodes |
|-------|-------|
| 1 | 20 |
| 2 | 400 |
| 3 | 8,902 |
| 4 | 197,281 |
| 5 | 4,865,609 |
| 6 | 119,060,324 |
| 7 | 3,195,901,860 |

### Perft Divide

When perft fails, use divide to isolate the bug:

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

Compare output with a known-good engine. When counts differ, recurse into that move.

### Tricky Test Positions

```
// Kiwipete - complex position with many edge cases
r3k2r/p1ppqpb1/bn2pnp1/3PN3/1p2P3/2N2Q1p/PPPBBPPP/R3K2R w KQkq - 0 1
Depth 1: 48
Depth 2: 2039
Depth 3: 97862
Depth 4: 4085603
Depth 5: 193690690

// Position 3 - en passant and castling
8/2p5/3p4/KP5r/1R3p1k/8/4P1P1/8 w - - 0 1
Depth 1: 14
Depth 2: 191
Depth 3: 2812
Depth 4: 43238
Depth 5: 674624

// Position 4 - promotion heavy
r3k2r/Pppp1ppp/1b3nbN/nP6/BBP1P3/q4N2/Pp1P2PP/R2Q1RK1 w kq - 0 1
Depth 1: 6
Depth 2: 264
Depth 3: 9467
Depth 4: 422333
Depth 5: 15833292

// Position 5 - discovered checks
rnbq1k1r/pp1Pbppp/2p5/8/2B5/8/PPP1NnPP/RNBQK2R w KQ - 1 8
Depth 1: 44
Depth 2: 1486
Depth 3: 62379
Depth 4: 2103487
Depth 5: 89941194
```

## Tactical Test Suites

Test that your engine finds the correct moves in tactical positions.

### WAC (Win at Chess)

300 positions from Fred Reinfeld's book:

```
// WAC.001 - Knight fork
2rr3k/pp3pp1/1nnqbN1p/3pN3/2pP4/2P3Q1/PPB4P/R4RK1 w - - 0 1
bm Qg6  // Best move: Qg6 (mate threat)

// WAC.002 - Back rank mate
8/7p/5k2/5p2/p1p2P2/Pr1pPK2/1P1R3P/8 b - - 0 1
bm Rxb2  // Best move: Rxb2
```

```c
void run_wac_test(const char* filename, int depth) {
    FILE* f = fopen(filename, "r");
    char line[1024];

    int passed = 0, failed = 0;

    while (fgets(line, sizeof(line), f)) {
        char fen[256];
        char best_move[16];
        parse_epd(line, fen, best_move);

        set_fen(&position, fen);
        Move found = search_to_depth(&position, depth);

        if (move_matches(found, best_move)) {
            passed++;
        } else {
            printf("FAILED: %s\n", fen);
            printf("Expected: %s, Got: %s\n", best_move, move_to_uci(found));
            failed++;
        }
    }

    printf("WAC Results: %d/%d (%.1f%%)\n",
           passed, passed + failed,
           100.0 * passed / (passed + failed));
}
```

### STS (Strategic Test Suite)

Tests positional understanding across categories:
- Undermining
- Open files
- Knight outposts
- Passed pawns
- Activity
- etc.

### ECM (Encyclopedia of Chess Middlegames)

Challenging tactical positions from master games.

### Mate-in-N Suites

Verify mate finding:

```
// Mate in 1
8/8/8/8/8/5K2/4Q3/7k w - - 0 1
bm Qe1#

// Mate in 2
r1bqkb1r/pppp1ppp/2n2n2/4p2Q/2B1P3/8/PPPP1PPP/RNB1K1NR w KQkq - 4 4
bm Qxf7+  // Scholar's mate

// Mate in 3
r2qkb1r/pp2nppp/3p4/2pNN1B1/2BnP3/3P4/PPP2PPP/R2bK2R w KQkq - 0 1
bm Nf6+
```

## Self-Play Testing

Play your engine against itself to check for:
- Crashes
- Time management issues
- Hash collisions
- Position corruption

```bash
# Using cutechess-cli
cutechess-cli \
    -engine name=MyEngine cmd=./myengine \
    -engine name=MyEngine cmd=./myengine \
    -each tc=1+0.1 proto=uci \
    -games 100 \
    -pgnout selfplay.pgn
```

## Regression Testing

After any change, verify you haven't broken anything:

```bash
#!/bin/bash
# regression.sh

echo "Running perft tests..."
./engine << EOF
position startpos
go perft 5
position fen r3k2r/p1ppqpb1/bn2pnp1/3PN3/1p2P3/2N2Q1p/PPPBBPPP/R3K2R w KQkq - 0 1
go perft 4
quit
EOF

echo "Running tactical tests..."
./run_wac_test wac.epd 10

echo "Running search tests..."
./engine << EOF
position fen r1bqkbnr/pppp1ppp/2n5/4p3/4P3/5N2/PPPP1PPP/RNBQKB1R w KQkq - 2 3
go depth 12
quit
EOF
```

## Search Consistency Tests

Verify search behaves correctly:

### Null Window Test

Search with alpha = beta - 1 should behave like PVS:

```c
void test_null_window() {
    // If score >= beta, return score >= beta
    // If score < beta, return score < beta
    int score = search(pos, depth, beta - 1, beta);
    assert(score >= beta || score < beta);  // Always true, but verify bounds
}
```

### TT Consistency

The same position should give the same result:

```c
void test_tt_consistency() {
    set_fen(&pos, "r1bqkbnr/pppp1ppp/2n5/4p3/4P3/5N2/PPPP1PPP/RNBQKB1R w KQkq - 0 1");

    int score1 = search(&pos, 10);
    Move move1 = get_best_move();

    clear_hash_table();  // Clear TT

    int score2 = search(&pos, 10);
    Move move2 = get_best_move();

    // Should find same move (score might differ due to TT effects)
    assert(move1 == move2);
}
```

### Path Independence

Reaching the same position via different move orders should give identical evaluation:

```c
void test_path_independence() {
    // Path 1: e4 e5 Nf3
    set_fen(&pos, START_FEN);
    make_move(&pos, parse_move("e2e4"));
    make_move(&pos, parse_move("e7e5"));
    make_move(&pos, parse_move("g1f3"));
    int eval1 = evaluate(&pos);

    // Path 2: Nf3 e5 e4
    set_fen(&pos, START_FEN);
    make_move(&pos, parse_move("g1f3"));
    make_move(&pos, parse_move("e7e5"));
    make_move(&pos, parse_move("e2e4"));
    int eval2 = evaluate(&pos);

    assert(eval1 == eval2);
}
```

## Benchmark Positions

Standard positions for measuring NPS (nodes per second):

```c
const char* BENCHMARK_FENS[] = {
    "rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w KQkq - 0 1",
    "r3k2r/p1ppqpb1/bn2pnp1/3PN3/1p2P3/2N2Q1p/PPPBBPPP/R3K2R w KQkq - 0 1",
    "8/2p5/3p4/KP5r/1R3p1k/8/4P1P1/8 w - - 0 1",
    "r3k2r/Pppp1ppp/1b3nbN/nP6/BBP1P3/q4N2/Pp1P2PP/R2Q1RK1 w kq - 0 1",
    "rnbq1k1r/pp1Pbppp/2p5/8/2B5/8/PPP1NnPP/RNBQK2R w KQ - 1 8",
    "r4rk1/1pp1qppp/p1np1n2/2b1p1B1/2B1P1b1/P1NP1N2/1PP1QPPP/R4RK1 w - - 0 10",
    // Add more positions covering different game phases
};

void benchmark() {
    uint64_t total_nodes = 0;
    int64_t total_time = 0;

    for (int i = 0; i < NUM_BENCHMARK_FENS; i++) {
        set_fen(&pos, BENCHMARK_FENS[i]);

        int64_t start = current_time_ms();
        search(&pos, 12);  // Fixed depth
        int64_t elapsed = current_time_ms() - start;

        total_nodes += search_info.nodes;
        total_time += elapsed;
    }

    printf("Total nodes: %llu\n", total_nodes);
    printf("Total time: %lld ms\n", total_time);
    printf("NPS: %llu\n", total_nodes * 1000 / total_time);
}
```

## Engine vs Engine Testing

The most reliable way to test improvements:

### Cutechess-cli

```bash
cutechess-cli \
    -engine name=NewEngine cmd=./new_engine \
    -engine name=OldEngine cmd=./old_engine \
    -each tc=10+0.1 proto=uci hash=64 \
    -games 1000 \
    -openings file=openings.pgn format=pgn \
    -pgnout results.pgn \
    -concurrency 4
```

### SPRT (Sequential Probability Ratio Test)

Statistical test for determining if a change is an improvement:

```bash
cutechess-cli \
    -engine name=New cmd=./new \
    -engine name=Old cmd=./old \
    -each tc=10+0.1 proto=uci \
    -games 10000 \
    -sprt elo0=0 elo1=5 alpha=0.05 beta=0.05 \
    -openings file=openings.epd format=epd \
    -concurrency 8
```

Parameters:
- `elo0`: Null hypothesis (no improvement)
- `elo1`: Alternative hypothesis (improvement exists)
- `alpha`: False positive rate (accept bad change)
- `beta`: False negative rate (reject good change)

### Elo Estimation

From win/loss/draw statistics:

```c
double calculate_elo(int wins, int losses, int draws) {
    int total = wins + losses + draws;
    double score = (wins + 0.5 * draws) / total;

    // Avoid division by zero at extremes
    if (score <= 0.0) return -1000;
    if (score >= 1.0) return 1000;

    // Elo difference formula
    double elo_diff = -400 * log10(1.0 / score - 1.0);

    return elo_diff;
}
```

## Common Testing Mistakes

### 1. Too Few Games

```
100 games: +/- 30 Elo error margin
1000 games: +/- 10 Elo error margin
10000 games: +/- 3 Elo error margin
```

### 2. No Opening Book

Without varied openings, games follow the same lines repeatedly. Use a book:
- Polyglot format
- EPD opening positions
- PGN opening sequences

### 3. Same Hash Size

Both engines should use identical hash table sizes for fair comparison.

### 4. Different Hardware

Test on the same machine to eliminate hardware variables.

## Continuous Integration

Automate testing on every commit:

```yaml
# .github/workflows/test.yml
name: Engine Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      - name: Build
        run: make release

      - name: Perft Tests
        run: ./run_perft_tests.sh

      - name: Tactical Tests
        run: ./run_tactical_tests.sh

      - name: Quick SPRT
        run: |
          cutechess-cli \
            -engine cmd=./new_engine \
            -engine cmd=./baseline_engine \
            -each tc=1+0.1 proto=uci \
            -games 500 \
            -sprt elo0=-10 elo1=5 alpha=0.05 beta=0.05
```

## Summary

Testing methodology:

- **Perft**: Verify move generation correctness
- **Tactical suites**: Test search finds correct moves
- **Self-play**: Check for crashes and stability
- **Regression tests**: Ensure changes don't break existing functionality
- **Engine matches**: Measure actual playing strength
- **SPRT**: Statistical significance for changes
- **Benchmarks**: Track performance (NPS) over time

Thorough testing catches bugs early and verifies improvements are real.

Next: Parameter tuning—optimizing evaluation weights.
