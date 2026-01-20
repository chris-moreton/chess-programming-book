# Chapter 20: Evaluation Basics

The evaluation function assigns a numerical score to a chess position. Positive scores favor the side to move; negative scores favor the opponent. Search uses these scores to compare positions and choose moves.

## What Are We Evaluating?

The evaluation tries to answer: "Who is winning, and by how much?"

A perfect evaluation would return:
- +INFINITY if the side to move has a forced win
- -INFINITY if the opponent has a forced win
- 0 if the game is drawn with best play

Perfect evaluation is impossible without searching to game end. We use heuristics—imperfect but useful approximations.

## The Evaluation Interface

```c
// Returns score in centipawns from side-to-move's perspective
int evaluate(Position* pos) {
    int score = 0;

    score += evaluate_material(pos);
    score += evaluate_pst(pos);
    score += evaluate_pawns(pos);
    score += evaluate_pieces(pos);
    score += evaluate_king_safety(pos);

    // Side-to-move perspective
    return (pos->side_to_move == WHITE) ? score : -score;
}
```

## Centipawns

Scores are measured in **centipawns** (1/100 of a pawn):
- Pawn = 100
- Positional advantage of a pawn = 100
- Winning exchange (rook for bishop) ≈ 170

This gives fine-grained scores without floating point.

## Material Counting

The simplest and most important evaluation term:

```c
// Standard piece values (centipawns)
int PIECE_VALUE[7] = {0, 100, 320, 330, 500, 900, 0};
// Index:              0    1    2    3    4    5   6
// Piece:          NONE PAWN KNIGHT BISHOP ROOK QUEEN KING

int evaluate_material(Position* pos) {
    int score = 0;

    score += PIECE_VALUE[PAWN] * (popcount(pos->white_pawns) -
                                  popcount(pos->black_pawns));
    score += PIECE_VALUE[KNIGHT] * (popcount(pos->white_knights) -
                                    popcount(pos->black_knights));
    score += PIECE_VALUE[BISHOP] * (popcount(pos->white_bishops) -
                                    popcount(pos->black_bishops));
    score += PIECE_VALUE[ROOK] * (popcount(pos->white_rooks) -
                                  popcount(pos->black_rooks));
    score += PIECE_VALUE[QUEEN] * (popcount(pos->white_queens) -
                                   popcount(pos->black_queens));

    return score;
}
```

### Piece Value Debates

Classic values: P=100, N=300, B=300, R=500, Q=900

Modern engines adjust:
- Knight ≈ 320 (slightly better than thought)
- Bishop ≈ 330 (bishop pair bonus)
- Rook ≈ 500-525 (endgame value higher)

These vary by position—a knight is worth more in closed positions.

## Incremental Material

Update material incrementally during make/unmake:

```c
struct Position {
    int material[2];  // [WHITE, BLACK]
    // ...
};

void make_move(Position* pos, Move move) {
    // On capture
    if (captured != EMPTY) {
        pos->material[them] -= PIECE_VALUE[piece_type(captured)];
    }

    // On promotion
    if (is_promotion(move)) {
        pos->material[us] -= PIECE_VALUE[PAWN];
        pos->material[us] += PIECE_VALUE[promotion_piece(move)];
    }
}
```

Then:

```c
int evaluate_material(Position* pos) {
    return pos->material[WHITE] - pos->material[BLACK];
}
```

## Evaluation Terms Overview

Beyond material, evaluation considers:

| Term | Importance | Chapter |
|------|------------|---------|
| Material | ~60-70% | This chapter |
| Piece-Square Tables | ~10-15% | Chapter 21 |
| Pawn Structure | ~10-15% | Chapter 22 |
| Piece Activity | ~5-10% | Chapter 23 |
| King Safety | ~5-10% | Chapter 24 |
| Endgame Specifics | Situational | Chapter 25 |

Percentages are rough—relative importance varies by position.

## Evaluation Symmetry

Evaluation should be symmetric: swapping colors should negate the score.

```c
// Test symmetry
int white_score = evaluate(pos);
flip_colors(pos);
int black_score = evaluate(pos);
flip_colors(pos);  // Restore

assert(white_score == -black_score);
```

Bugs often break symmetry—good to test.

## Lazy Evaluation

For pruning decisions, we often need only rough scores. **Lazy evaluation** computes material and PST only:

```c
int lazy_evaluate(Position* pos) {
    return pos->material[WHITE] - pos->material[BLACK] +
           pos->pst_score[WHITE] - pos->pst_score[BLACK];
}

// Use lazy eval for futility pruning
if (lazy_evaluate(pos) + margin < alpha) {
    // Probably can prune
}
```

Full evaluation runs only at leaf nodes in quiescence.

## Evaluation Caching

Some terms are expensive. Cache them:

```c
typedef struct {
    uint64 pawn_hash;
    int pawn_score;
    // Additional pawn structure info...
} PawnEntry;

PawnEntry pawn_table[PAWN_TABLE_SIZE];

int evaluate_pawns(Position* pos) {
    uint64 key = pos->pawn_hash;
    PawnEntry* entry = &pawn_table[key & PAWN_TABLE_MASK];

    if (entry->pawn_hash == key) {
        return entry->pawn_score;  // Cache hit
    }

    int score = compute_pawn_score(pos);

    entry->pawn_hash = key;
    entry->pawn_score = score;

    return score;
}
```

Pawn structure changes rarely, so hits are common.

## Tuning Evaluation

Evaluation weights need tuning:
- Hand-tuning: Adjust based on intuition and testing
- Texel tuning: Optimize against game outcomes (Chapter 30)
- SPSA: Stochastic optimization (Chapter 30)

Start with standard values, then tune for your engine.

## Complete Basic Evaluation

```c
int evaluate(Position* pos) {
    int score = 0;

    // Material
    score += pos->material[WHITE] - pos->material[BLACK];

    // Piece-square tables
    score += pos->pst_score[WHITE] - pos->pst_score[BLACK];

    // Pawn structure
    score += evaluate_pawns(pos);

    // Piece activity
    score += evaluate_mobility(pos);

    // King safety
    score += evaluate_king_safety(pos);

    // Side-to-move perspective
    return (pos->side_to_move == WHITE) ? score : -score;
}
```

## Debugging Evaluation

Print evaluation breakdown:

```c
void print_eval(Position* pos) {
    printf("Material: %d\n", pos->material[WHITE] - pos->material[BLACK]);
    printf("PST:      %d\n", pos->pst_score[WHITE] - pos->pst_score[BLACK]);
    printf("Pawns:    %d\n", evaluate_pawns(pos));
    printf("Mobility: %d\n", evaluate_mobility(pos));
    printf("King:     %d\n", evaluate_king_safety(pos));
    printf("Total:    %d\n", evaluate(pos));
}
```

This helps identify which terms are misbehaving.

## Summary

Evaluation basics:

- **Output**: Score in centipawns, side-to-move perspective
- **Material**: Most important term (~60-70% of evaluation)
- **Incremental updates**: Material and PST maintained during search
- **Lazy evaluation**: Fast approximation for pruning
- **Caching**: Store expensive terms (pawn structure)
- **Symmetry**: Flipping colors should negate score
- **Tuning**: Required for competitive strength

Material alone gives a playable engine. Each additional term adds strength but complexity.

Next: Piece-square tables—positional value of piece placement.
