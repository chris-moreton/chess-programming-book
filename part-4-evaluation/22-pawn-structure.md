# Chapter 22: Pawn Structure

Pawns form the "skeleton" of a chess position. Unlike pieces, pawn moves are irreversible—once advanced, they can't retreat. Pawn structure evaluation identifies strengths and weaknesses.

## Pawn Structure Terms

### Doubled Pawns

Two pawns of the same color on the same file:

```c
int count_doubled_pawns(uint64 pawns) {
    int doubled = 0;

    for (int file = 0; file < 8; file++) {
        uint64 file_mask = FILE_A << file;
        int pawns_on_file = popcount(pawns & file_mask);
        if (pawns_on_file > 1) {
            doubled += pawns_on_file - 1;
        }
    }

    return doubled;
}

int evaluate_doubled_pawns(Position* pos) {
    int white_doubled = count_doubled_pawns(pos->white_pawns);
    int black_doubled = count_doubled_pawns(pos->black_pawns);

    return -DOUBLED_PAWN_PENALTY * (white_doubled - black_doubled);
}
```

Doubled pawns can't protect each other and clog the file.

### Isolated Pawns

A pawn with no friendly pawns on adjacent files:

```c
uint64 isolated_pawns(uint64 pawns) {
    uint64 adjacent = ((pawns & ~FILE_A) >> 1) | ((pawns & ~FILE_H) << 1);
    uint64 files_with_pawns = file_fill(pawns);
    uint64 supported = files_with_pawns & ((files_with_pawns >> 1) | (files_with_pawns << 1));

    return pawns & ~supported;
}

// file_fill: spread pawns to fill their files
uint64 file_fill(uint64 bb) {
    bb |= bb >> 8;
    bb |= bb >> 16;
    bb |= bb >> 32;
    bb |= bb << 8;
    bb |= bb << 16;
    bb |= bb << 32;
    return bb;
}
```

Isolated pawns are weak because they can't be defended by other pawns.

### Backward Pawns

A pawn that cannot advance safely because an enemy pawn controls the advance square, and no friendly pawn can support it:

```c
uint64 backward_pawns(uint64 our_pawns, uint64 their_pawns) {
    // Pawns that have no friendly pawn beside or behind them
    // and face an enemy pawn on an adjacent file one rank ahead

    // This is complex; simplified version:
    uint64 backward = 0;

    // For each pawn, check if it can be supported by advancement
    // ... detailed logic ...

    return backward;
}
```

Backward pawns are potential targets.

### Passed Pawns

A pawn with no enemy pawns blocking or able to capture it on the way to promotion:

```c
uint64 passed_pawns_white(uint64 white_pawns, uint64 black_pawns) {
    uint64 passed = 0;

    uint64 all_black_fronts = black_pawns;
    // Include adjacent files
    all_black_fronts |= (black_pawns & ~FILE_A) >> 1;
    all_black_fronts |= (black_pawns & ~FILE_H) << 1;

    // Fill downward (black pawn control)
    all_black_fronts |= all_black_fronts >> 8;
    all_black_fronts |= all_black_fronts >> 16;
    all_black_fronts |= all_black_fronts >> 32;

    // White pawns not blocked
    passed = white_pawns & ~all_black_fronts;

    return passed;
}
```

Passed pawns are valuable—they threaten to promote.

### Connected Pawns

Pawns that protect each other:

```c
uint64 connected_pawns(uint64 pawns) {
    uint64 attacking_left = (pawns & ~FILE_A) << 7;   // White attack pattern
    uint64 attacking_right = (pawns & ~FILE_H) << 9;
    uint64 defended = pawns & (attacking_left | attacking_right);
    return defended;
}
```

Connected pawns are stronger than isolated ones.

### Pawn Chains

A diagonal line of connected pawns. The base is a potential target.

## Pawn Evaluation Function

```c
int evaluate_pawns(Position* pos) {
    int score = 0;

    uint64 white_pawns = pos->white_pawns;
    uint64 black_pawns = pos->black_pawns;

    // Doubled pawns
    score -= DOUBLED_PENALTY * count_doubled(white_pawns);
    score += DOUBLED_PENALTY * count_doubled(black_pawns);

    // Isolated pawns
    score -= ISOLATED_PENALTY * popcount(isolated_pawns(white_pawns));
    score += ISOLATED_PENALTY * popcount(isolated_pawns(black_pawns));

    // Backward pawns
    score -= BACKWARD_PENALTY * popcount(backward(white_pawns, black_pawns));
    score += BACKWARD_PENALTY * popcount(backward(black_pawns, white_pawns));

    // Passed pawns (bonus by rank)
    uint64 white_passed = passed_pawns_white(white_pawns, black_pawns);
    uint64 black_passed = passed_pawns_black(white_pawns, black_pawns);

    while (white_passed) {
        int sq = bitscan_forward(white_passed);
        int rank = sq / 8;  // 0-7
        score += PASSED_BONUS[rank];
        white_passed &= white_passed - 1;
    }

    while (black_passed) {
        int sq = bitscan_forward(black_passed);
        int rank = 7 - (sq / 8);  // Flip for black
        score -= PASSED_BONUS[rank];
        black_passed &= black_passed - 1;
    }

    return score;
}

// Passed pawn bonus by rank (rank 2=almost useless, rank 7=very strong)
int PASSED_BONUS[8] = {0, 10, 15, 25, 50, 100, 150, 0};
```

## Pawn Hash Table

Pawn structure changes infrequently. Cache the evaluation:

```c
typedef struct {
    uint64 key;
    int mg_score;
    int eg_score;
    uint64 passed_pawns[2];
    // Other cached info...
} PawnEntry;

PawnEntry pawn_table[PAWN_TABLE_SIZE];

int evaluate_pawns_cached(Position* pos) {
    uint64 key = pos->pawn_hash;  // Hash of just pawn positions
    PawnEntry* entry = &pawn_table[key % PAWN_TABLE_SIZE];

    if (entry->key == key) {
        return entry->mg_score;  // Or interpolate with eg_score
    }

    // Compute and store
    int mg_score, eg_score;
    evaluate_pawn_structure(pos, &mg_score, &eg_score, entry);

    entry->key = key;
    entry->mg_score = mg_score;
    entry->eg_score = eg_score;

    return mg_score;
}
```

Hit rates of 95%+ are common.

## Passed Pawn Details

Passed pawns deserve detailed evaluation:

### Distance to Promotion

```c
int rank = rank_of(pawn_sq);
int distance = (color == WHITE) ? (7 - rank) : rank;
bonus = base_bonus * (7 - distance) / 7;  // More advanced = bigger bonus
```

### Blockers

A passed pawn blocked by an enemy piece is less valuable:

```c
uint64 front_sq = (color == WHITE) ? (pawn_sq + 8) : (pawn_sq - 8);
if (pos->all_pieces & (1ULL << front_sq)) {
    bonus /= 2;  // Blocked
}
```

### King Support

A passed pawn supported by its king is stronger:

```c
int pawn_rank = rank_of(pawn_sq);
int king_rank = rank_of(king_sq);

if ((color == WHITE && king_rank >= pawn_rank - 1) ||
    (color == BLACK && king_rank <= pawn_rank + 1)) {
    bonus += KING_SUPPORT_BONUS;
}
```

### Rook Behind Passed Pawn

A rook behind a passed pawn gains power as the pawn advances:

```c
int file = file_of(pawn_sq);
uint64 file_bb = FILE_A << file;

if (rooks & file_bb) {
    int rook_sq = bitscan(rooks & file_bb);
    if ((color == WHITE && rank_of(rook_sq) < rank_of(pawn_sq)) ||
        (color == BLACK && rank_of(rook_sq) > rank_of(pawn_sq))) {
        bonus += ROOK_BEHIND_PASSER;
    }
}
```

## Sample Values

Typical penalties and bonuses (centipawns):

| Term | Value |
|------|-------|
| Doubled pawn | -10 to -20 |
| Isolated pawn | -15 to -25 |
| Backward pawn | -10 to -15 |
| Passed pawn (rank 4) | +25 |
| Passed pawn (rank 6) | +100 |
| Connected passers | +20 extra |
| Blocked passer | -50% of bonus |

These vary by engine and are tuned empirically.

## Summary

Pawn structure evaluation:

- **Doubled**: Penalty for pawns on same file
- **Isolated**: Penalty for pawns without neighbors
- **Backward**: Penalty for pawns that can't advance safely
- **Passed**: Large bonus, increasing with advancement
- **Connected**: Bonus for mutually supporting pawns

Pawn structure is expensive to compute but changes rarely—use a pawn hash table.

Next: Piece-specific evaluation—mobility, outposts, and coordination.
