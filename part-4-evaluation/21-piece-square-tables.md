# Chapter 21: Piece-Square Tables

**Piece-square tables** (PSTs) assign a bonus or penalty based on where a piece stands. A knight on e5 is better than a knight on a1. PSTs encode this knowledge compactly.

## The Concept

For each piece type and square, store a value:

```c
// Knight piece-square table (from white's perspective)
// Values in centipawns, a1 = index 0, h8 = index 63
int KNIGHT_PST[64] = {
    -50, -40, -30, -30, -30, -30, -40, -50,  // Rank 1
    -40, -20,   0,   0,   0,   0, -20, -40,  // Rank 2
    -30,   0,  10,  15,  15,  10,   0, -30,  // Rank 3
    -30,   5,  15,  20,  20,  15,   5, -30,  // Rank 4
    -30,   0,  15,  20,  20,  15,   0, -30,  // Rank 5
    -30,   5,  10,  15,  15,  10,   5, -30,  // Rank 6
    -40, -20,   0,   5,   5,   0, -20, -40,  // Rank 7
    -50, -40, -30, -30, -30, -30, -40, -50,  // Rank 8
};
```

Knights are penalized on edges, rewarded in the center.

## Tables for Each Piece Type

### Pawns

Pawns should advance; central pawns are valuable:

```c
int PAWN_PST[64] = {
      0,   0,   0,   0,   0,   0,   0,   0,  // Rank 1 (impossible)
     50,  50,  50,  50,  50,  50,  50,  50,  // Rank 2 (start)
     10,  10,  20,  30,  30,  20,  10,  10,  // Rank 3
      5,   5,  10,  25,  25,  10,   5,   5,  // Rank 4
      0,   0,   0,  20,  20,   0,   0,   0,  // Rank 5
      5,  -5, -10,   0,   0, -10,  -5,   5,  // Rank 6
      5,  10,  10, -20, -20,  10,  10,   5,  // Rank 7
      0,   0,   0,   0,   0,   0,   0,   0,  // Rank 8 (promotes)
};
```

### Knights

As shown above—centralize, avoid edges:

```c
int KNIGHT_PST[64] = { /* ... */ };
```

### Bishops

Long diagonals, not blocked by own pawns:

```c
int BISHOP_PST[64] = {
    -20, -10, -10, -10, -10, -10, -10, -20,
    -10,   0,   0,   0,   0,   0,   0, -10,
    -10,   0,   5,  10,  10,   5,   0, -10,
    -10,   5,   5,  10,  10,   5,   5, -10,
    -10,   0,  10,  10,  10,  10,   0, -10,
    -10,  10,  10,  10,  10,  10,  10, -10,
    -10,   5,   0,   0,   0,   0,   5, -10,
    -20, -10, -10, -10, -10, -10, -10, -20,
};
```

### Rooks

Seventh rank, open files:

```c
int ROOK_PST[64] = {
      0,   0,   0,   0,   0,   0,   0,   0,
      5,  10,  10,  10,  10,  10,  10,   5,
     -5,   0,   0,   0,   0,   0,   0,  -5,
     -5,   0,   0,   0,   0,   0,   0,  -5,
     -5,   0,   0,   0,   0,   0,   0,  -5,
     -5,   0,   0,   0,   0,   0,   0,  -5,
     -5,   0,   0,   0,   0,   0,   0,  -5,
      0,   0,   0,   5,   5,   0,   0,   0,
};
```

### Queen

Generally centralize but not early:

```c
int QUEEN_PST[64] = {
    -20, -10, -10,  -5,  -5, -10, -10, -20,
    -10,   0,   0,   0,   0,   0,   0, -10,
    -10,   0,   5,   5,   5,   5,   0, -10,
     -5,   0,   5,   5,   5,   5,   0,  -5,
      0,   0,   5,   5,   5,   5,   0,  -5,
    -10,   5,   5,   5,   5,   5,   0, -10,
    -10,   0,   5,   0,   0,   0,   0, -10,
    -20, -10, -10,  -5,  -5, -10, -10, -20,
};
```

### King (Middlegame)

Castled, behind pawns:

```c
int KING_MG_PST[64] = {
    -30, -40, -40, -50, -50, -40, -40, -30,
    -30, -40, -40, -50, -50, -40, -40, -30,
    -30, -40, -40, -50, -50, -40, -40, -30,
    -30, -40, -40, -50, -50, -40, -40, -30,
    -20, -30, -30, -40, -40, -30, -30, -20,
    -10, -20, -20, -20, -20, -20, -20, -10,
     20,  20,   0,   0,   0,   0,  20,  20,
     20,  30,  10,   0,   0,  10,  30,  20,
};
```

### King (Endgame)

Active, centralized:

```c
int KING_EG_PST[64] = {
    -50, -40, -30, -20, -20, -30, -40, -50,
    -30, -20, -10,   0,   0, -10, -20, -30,
    -30, -10,  20,  30,  30,  20, -10, -30,
    -30, -10,  30,  40,  40,  30, -10, -30,
    -30, -10,  30,  40,  40,  30, -10, -30,
    -30, -10,  20,  30,  30,  20, -10, -30,
    -30, -30,   0,   0,   0,   0, -30, -30,
    -50, -30, -30, -30, -30, -30, -30, -50,
};
```

## Computing PST Score

```c
int evaluate_pst(Position* pos) {
    int score = 0;

    // White pieces
    uint64 pieces = pos->white_pawns;
    while (pieces) {
        int sq = bitscan_forward(pieces);
        score += PAWN_PST[sq];
        pieces &= pieces - 1;
    }

    pieces = pos->white_knights;
    while (pieces) {
        int sq = bitscan_forward(pieces);
        score += KNIGHT_PST[sq];
        pieces &= pieces - 1;
    }
    // ... other white pieces ...

    // Black pieces (flip square vertically)
    pieces = pos->black_pawns;
    while (pieces) {
        int sq = bitscan_forward(pieces);
        score -= PAWN_PST[FLIP_VERTICAL(sq)];
        pieces &= pieces - 1;
    }
    // ... other black pieces ...

    return score;
}
```

For black, flip the square index vertically (rank 1 ↔ rank 8):

```c
#define FLIP_VERTICAL(sq) ((sq) ^ 56)
// sq ^ 56 swaps rank: a1(0) <-> a8(56), e4(28) <-> e5(36)
```

## Incremental PST Updates

Recomputing PST every evaluation is slow. Update incrementally:

```c
struct Position {
    int pst_score[2];  // [WHITE, BLACK] PST totals
};

void make_move(Position* pos, Move move) {
    int from = move_from(move);
    int to = move_to(move);
    int piece = pos->board[from];
    int us = pos->side_to_move;

    // Remove PST for piece leaving 'from'
    pos->pst_score[us] -= PST[piece][from];

    // Add PST for piece arriving at 'to'
    int moved_piece = is_promotion(move) ? promotion_piece(move) : piece;
    pos->pst_score[us] += PST[moved_piece][to];

    // Handle captures
    if (captured != EMPTY) {
        int them = us ^ 1;
        pos->pst_score[them] -= PST[captured][to];
    }

    // ... castling rook PST updates ...
}
```

Then:

```c
int evaluate_pst(Position* pos) {
    return pos->pst_score[WHITE] - pos->pst_score[BLACK];
}
```

## Tapered Evaluation

Piece-square values differ between middlegame and endgame. **Tapered evaluation** blends them:

```c
int PST_MG[12][64];  // Middlegame values
int PST_EG[12][64];  // Endgame values

int evaluate(Position* pos) {
    int mg_score = 0;  // Middlegame score
    int eg_score = 0;  // Endgame score

    // Compute both scores
    mg_score += pos->pst_mg[WHITE] - pos->pst_mg[BLACK];
    eg_score += pos->pst_eg[WHITE] - pos->pst_eg[BLACK];
    // ... other terms ...

    // Determine game phase (0 = endgame, 256 = opening/middlegame)
    int phase = compute_phase(pos);

    // Interpolate
    int score = (mg_score * phase + eg_score * (256 - phase)) / 256;

    return (pos->side_to_move == WHITE) ? score : -score;
}

int compute_phase(Position* pos) {
    // Based on piece counts (excluding pawns and kings)
    int phase = 0;
    phase += popcount(pos->white_knights | pos->black_knights) * 1;
    phase += popcount(pos->white_bishops | pos->black_bishops) * 1;
    phase += popcount(pos->white_rooks | pos->black_rooks) * 2;
    phase += popcount(pos->white_queens | pos->black_queens) * 4;

    // Max phase = 2*1 + 2*1 + 2*2 + 2*4 = 16 (for one side)
    // Total max = 32, but we scale to 256
    return min(phase * 8, 256);
}
```

## Designing PST Values

Principles:
- **Centralization**: Most pieces are better in center
- **Development**: Pieces should leave starting squares
- **King safety**: King should be safe (different for MG/EG)
- **Pawn advance**: Pawns generally better advanced

Start with published tables, then tune.

## Summary

Piece-square tables:

- **Simple idea**: Bonus/penalty by piece and square
- **Cheap to compute**: Just array lookups
- **Incremental**: Update on make/unmake
- **Tapered**: Different values for middlegame vs endgame
- **Significant impact**: ~10-15% of evaluation

PSTs are a simple way to encode positional knowledge. Combined with material, they give a reasonable evaluation with minimal code.

Next: Pawn structure—the skeleton of the position.
