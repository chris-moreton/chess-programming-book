# Chapter 23: Piece Evaluation

Beyond material and placement, pieces have properties that affect their strength: mobility, coordination, and piece-specific features.

## Mobility

A piece's **mobility** is the number of squares it can reach. More mobility generally means more power.

```c
int evaluate_mobility(Position* pos) {
    int score = 0;
    uint64 occupied = pos->all_pieces[WHITE] | pos->all_pieces[BLACK];
    uint64 safe_white = ~pos->all_pieces[WHITE] & ~pawn_attacks(pos, BLACK);
    uint64 safe_black = ~pos->all_pieces[BLACK] & ~pawn_attacks(pos, WHITE);

    // Knights
    uint64 knights = pos->white_knights;
    while (knights) {
        int sq = bitscan_forward(knights);
        int mobility = popcount(KNIGHT_ATTACKS[sq] & safe_white);
        score += KNIGHT_MOBILITY[mobility];
        knights &= knights - 1;
    }

    knights = pos->black_knights;
    while (knights) {
        int sq = bitscan_forward(knights);
        int mobility = popcount(KNIGHT_ATTACKS[sq] & safe_black);
        score -= KNIGHT_MOBILITY[mobility];
        knights &= knights - 1;
    }

    // Bishops (using magic bitboards)
    uint64 bishops = pos->white_bishops;
    while (bishops) {
        int sq = bitscan_forward(bishops);
        uint64 attacks = bishop_attacks(sq, occupied);
        int mobility = popcount(attacks & safe_white);
        score += BISHOP_MOBILITY[mobility];
        bishops &= bishops - 1;
    }
    // ... similar for black bishops, rooks, queens ...

    return score;
}

// Mobility tables (example values)
int KNIGHT_MOBILITY[9] = {-20, -10, 0, 5, 10, 15, 18, 20, 22};
int BISHOP_MOBILITY[14] = {-25, -15, -5, 0, 5, 10, 14, 17, 20, 22, 24, 26, 27, 28};
int ROOK_MOBILITY[15] = {-15, -10, -5, 0, 5, 8, 11, 14, 16, 18, 20, 21, 22, 23, 24};
int QUEEN_MOBILITY[28] = { /* ... */ };
```

### Safe Mobility

Count only squares not attacked by enemy pawns:

```c
uint64 safe_squares = ~pawn_attacks(pos, enemy_color);
int safe_mobility = popcount(attacks & safe_squares);
```

Pieces on pawn-attacked squares are often driven away.

## Bishop Pair

Having both bishops is worth a bonus—they cover all 64 squares together:

```c
int evaluate_bishop_pair(Position* pos) {
    int score = 0;

    // White bishop pair
    if (popcount(pos->white_bishops) >= 2) {
        score += BISHOP_PAIR_BONUS;  // ~30-50 centipawns
    }

    // Black bishop pair
    if (popcount(pos->black_bishops) >= 2) {
        score -= BISHOP_PAIR_BONUS;
    }

    return score;
}
```

The bishop pair is more valuable in open positions.

## Outposts

An **outpost** is a square protected by a pawn where enemy pawns can no longer attack:

```c
uint64 outpost_squares_white(Position* pos) {
    // Squares attacked by white pawns
    uint64 pawn_support = pawn_attacks(pos, WHITE);

    // Squares that black pawns can't attack (no black pawns on adjacent files)
    uint64 black_pawn_spans = file_fill(pos->black_pawns);
    black_pawn_spans = (black_pawn_spans >> 1) | (black_pawn_spans << 1);
    uint64 safe_from_black_pawns = ~black_pawn_spans;

    // Outpost must be in enemy territory (rank 4-6 for white)
    uint64 enemy_territory = RANK_4 | RANK_5 | RANK_6;

    return pawn_support & safe_from_black_pawns & enemy_territory;
}

int evaluate_knight_outpost(Position* pos) {
    int score = 0;
    uint64 white_outposts = outpost_squares_white(pos);
    uint64 black_outposts = outpost_squares_black(pos);

    score += OUTPOST_BONUS * popcount(pos->white_knights & white_outposts);
    score -= OUTPOST_BONUS * popcount(pos->black_knights & black_outposts);

    return score;
}
```

Knights on outposts are very strong.

## Rooks on Open Files

A file with no pawns is **open**. A file with only enemy pawns is **semi-open**:

```c
int evaluate_rook_files(Position* pos) {
    int score = 0;
    uint64 white_pawns = pos->white_pawns;
    uint64 black_pawns = pos->black_pawns;

    uint64 rooks = pos->white_rooks;
    while (rooks) {
        int sq = bitscan_forward(rooks);
        int file = sq % 8;
        uint64 file_bb = FILE_A << file;

        if (!(file_bb & (white_pawns | black_pawns))) {
            score += ROOK_OPEN_FILE;  // Open file
        } else if (!(file_bb & white_pawns)) {
            score += ROOK_SEMI_OPEN;  // Semi-open
        }

        rooks &= rooks - 1;
    }

    // Similar for black rooks...

    return score;
}
```

## Rooks on Seventh Rank

Rooks on the 7th rank (2nd for black) attack pawns and confine the king:

```c
if (pos->white_rooks & RANK_7) {
    score += ROOK_ON_SEVENTH;
}
if (pos->black_rooks & RANK_2) {
    score -= ROOK_ON_SEVENTH;
}
```

## Trapped Pieces

Pieces with very low mobility might be trapped:

```c
// Bishop trapped on a7/h7 by pawn on b6/g6
uint64 trapped_bishop_a7 = pos->white_bishops & A7_BB;
if (trapped_bishop_a7 && (pos->black_pawns & B6_BB)) {
    score -= TRAPPED_BISHOP_PENALTY;
}
```

## Minor Piece Imbalance

Knights vs bishops depends on the position:
- Closed positions favor knights
- Open positions favor bishops

```c
int evaluate_imbalance(Position* pos) {
    int white_knights = popcount(pos->white_knights);
    int white_bishops = popcount(pos->white_bishops);
    int black_knights = popcount(pos->black_knights);
    int black_bishops = popcount(pos->black_bishops);

    int pawn_count = popcount(pos->white_pawns | pos->black_pawns);

    // In closed positions (many pawns), knights gain value
    int knight_adj = (pawn_count - 8) * 2;  // +value if >8 pawns

    int score = 0;
    score += knight_adj * (white_knights - black_knights);
    score -= knight_adj * (white_bishops - black_bishops);

    return score;
}
```

## Piece Coordination

Pieces that work together are stronger:

### Connected Rooks

```c
if (rooks_connected(pos->white_rooks, occupied)) {
    score += CONNECTED_ROOKS;
}
```

### Battery (Queen + Bishop/Rook)

Queen and rook on same file, or queen and bishop on same diagonal:

```c
// Queen-rook battery
for each white queen sq:
    uint64 file_bb = file_of_sq(sq);
    if (pos->white_rooks & file_bb) {
        score += BATTERY_BONUS;
    }
```

## Complete Piece Evaluation

```c
int evaluate_pieces(Position* pos) {
    int score = 0;

    score += evaluate_mobility(pos);
    score += evaluate_bishop_pair(pos);
    score += evaluate_outposts(pos);
    score += evaluate_rook_files(pos);
    score += evaluate_rooks_on_seventh(pos);
    score += evaluate_trapped_pieces(pos);
    score += evaluate_imbalance(pos);

    return score;
}
```

## Sample Values

| Term | Value (centipawns) |
|------|-------------------|
| Bishop pair | +30 to +50 |
| Knight outpost | +20 to +40 |
| Rook open file | +20 to +30 |
| Rook semi-open file | +10 to +15 |
| Rook on 7th | +20 to +30 |
| Trapped bishop | -50 to -100 |
| Connected rooks | +10 to +20 |

## Summary

Piece evaluation considers:

- **Mobility**: Number of safe squares reachable
- **Bishop pair**: Bonus for having both bishops
- **Outposts**: Protected squares where pieces can't be driven away
- **Open files**: Rooks benefit from open and semi-open files
- **Seventh rank**: Rooks on 7th are powerful
- **Trapped pieces**: Penalty for pieces with no moves
- **Coordination**: Bonus for well-coordinated pieces

These terms add depth to evaluation beyond simple material counting.

Next: King safety—protecting the most important piece.
