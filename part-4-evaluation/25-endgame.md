# Chapter 25: Endgame Evaluation

Endgames have different priorities than middlegames. Kings become active fighters, passed pawns become critical, and specific material combinations have known outcomes.

## When Is It an Endgame?

Common definitions:
- No queens
- Few pieces remaining (phase calculation)
- Total material below threshold

```c
bool is_endgame(Position* pos) {
    // Simple: no queens
    return !pos->white_queens && !pos->black_queens;

    // Or: by material count
    int material = count_non_pawn_material(pos);
    return material < ENDGAME_THRESHOLD;
}
```

Use game phase for smooth transitions (tapered evaluation).

## King Centralization

In endgames, an active king is crucial:

```c
// King centralization table (endgame)
int KING_CENTRALIZATION[64] = {
    -30, -20, -10, -5, -5, -10, -20, -30,
    -20, -10,   0,  5,  5,   0, -10, -20,
    -10,   0,  10, 15, 15,  10,   0, -10,
     -5,   5,  15, 25, 25,  15,   5,  -5,
     -5,   5,  15, 25, 25,  15,   5,  -5,
    -10,   0,  10, 15, 15,  10,   0, -10,
    -20, -10,   0,  5,  5,   0, -10, -20,
    -30, -20, -10, -5, -5, -10, -20, -30,
};

int evaluate_king_endgame(Position* pos) {
    int score = 0;

    score += KING_CENTRALIZATION[pos->king_square[WHITE]];
    score -= KING_CENTRALIZATION[FLIP_VERTICAL(pos->king_square[BLACK])];

    return score;
}
```

## King-Pawn Distance

The king should approach passed pawns—both its own and the enemy's:

```c
int evaluate_king_pawn_distance(Position* pos) {
    int score = 0;

    // White king to white passed pawns (support them)
    uint64 white_passers = passed_pawns(pos, WHITE);
    while (white_passers) {
        int pawn_sq = bitscan_forward(white_passers);
        int dist = chebyshev_distance(pos->king_square[WHITE], pawn_sq);
        score += (7 - dist) * KING_PAWN_TROPISM;
        white_passers &= white_passers - 1;
    }

    // White king to black passed pawns (stop them)
    uint64 black_passers = passed_pawns(pos, BLACK);
    while (black_passers) {
        int pawn_sq = bitscan_forward(black_passers);
        int dist = chebyshev_distance(pos->king_square[WHITE], pawn_sq);
        score += (7 - dist) * KING_PAWN_DEFENSE;
        black_passers &= black_passers - 1;
    }

    // Similar for black king...

    return score;
}

int chebyshev_distance(int sq1, int sq2) {
    int r1 = sq1 / 8, f1 = sq1 % 8;
    int r2 = sq2 / 8, f2 = sq2 % 8;
    return max(abs(r1 - r2), abs(f1 - f2));
}
```

## Passed Pawn Advancement

In endgames, passed pawns become more valuable:

```c
int evaluate_passed_pawns_endgame(Position* pos) {
    int score = 0;

    // White passed pawns
    uint64 passers = passed_pawns(pos, WHITE);
    while (passers) {
        int sq = bitscan_forward(passers);
        int rank = sq / 8;

        // Base bonus increases with rank
        score += PASSED_BONUS_EG[rank];

        // Extra bonus if path to promotion is clear
        uint64 path = north_ray(sq);
        if (!(path & pos->all_pieces)) {
            score += CLEAR_PATH_BONUS;
        }

        // Bonus if enemy king is far
        int enemy_king_dist = chebyshev_distance(pos->king_square[BLACK], sq + 8);
        score += enemy_king_dist * 5;

        passers &= passers - 1;
    }

    // Similar for black...

    return score;
}

int PASSED_BONUS_EG[8] = {0, 10, 20, 35, 60, 100, 180, 0};  // Rank 7 pawn is huge
```

## Rule of the Square

Can the enemy king catch a passed pawn?

```c
bool king_can_catch_pawn(Position* pos, int pawn_sq, int enemy_king_sq, int color) {
    int promotion_sq = (color == WHITE) ? (pawn_sq % 8 + 56) : (pawn_sq % 8);
    int pawn_dist = abs(pawn_sq / 8 - promotion_sq / 8);
    int king_dist = chebyshev_distance(enemy_king_sq, promotion_sq);

    // If it's not pawn's side to move, add one
    if (pos->side_to_move != color) {
        king_dist--;
    }

    return king_dist <= pawn_dist;
}
```

If the king can't catch the pawn, it's essentially winning.

## Insufficient Material

Some endgames are known draws:

```c
bool is_insufficient_material(Position* pos) {
    // No pawns
    if (pos->white_pawns || pos->black_pawns) return false;

    // No queens or rooks
    if (pos->white_queens || pos->black_queens) return false;
    if (pos->white_rooks || pos->black_rooks) return false;

    int white_knights = popcount(pos->white_knights);
    int white_bishops = popcount(pos->white_bishops);
    int black_knights = popcount(pos->black_knights);
    int black_bishops = popcount(pos->black_bishops);

    // K vs K
    if (white_knights + white_bishops + black_knights + black_bishops == 0) {
        return true;
    }

    // K+B vs K or K+N vs K
    if (white_knights + white_bishops <= 1 && black_knights + black_bishops == 0) {
        return true;
    }
    if (black_knights + black_bishops <= 1 && white_knights + white_bishops == 0) {
        return true;
    }

    // K+N+N vs K (usually draw)
    // K+B vs K+B same color bishops
    // ... more cases ...

    return false;
}
```

## Scale Factor

Reduce winning score when winning is difficult:

```c
int scale_factor(Position* pos, int score) {
    // If ahead with no pawns, scale down
    if (score > 0 && !pos->white_pawns) {
        // Only minor pieces can't win without pawns (usually)
        if (!pos->white_rooks && !pos->white_queens) {
            return score / 4;  // Much harder to win
        }
    }

    // Opposite color bishops
    if (opposite_color_bishops(pos)) {
        return score * 3 / 4;  // Drawish
    }

    return score;  // No scaling
}
```

## Specific Endgames

Some endgames need special handling:

### KPK (King + Pawn vs King)

```c
int evaluate_kpk(Position* pos) {
    // Lookup in KPK tablebase or use rule of square
    // Key factors:
    // - Is pawn stoppable?
    // - Can defending king reach key squares?
    // - Rook pawn drawing zone
}
```

### KBNK (King + Bishop + Knight vs King)

```c
int evaluate_kbnk(Position* pos) {
    // Drive king to corner of bishop's color
    // ...
}
```

### Tablebase Integration

For perfect endgame play, use tablebases:

```c
int probe_tablebase(Position* pos) {
    if (popcount(all_pieces) <= TB_MEN) {
        int result = tb_probe_wdl(pos);
        if (result != TB_FAILED) {
            // Convert result to centipawn score
            return result * TB_WIN_VALUE;
        }
    }
    return SCORE_NONE;  // Not in tablebase
}
```

## Mating Patterns

Help the engine find mates:

### Lone King to Corner

```c
int evaluate_lone_king(Position* pos) {
    if (popcount(pos->black_pieces) == 1) {  // Just black king
        int bking = pos->king_square[BLACK];
        int wking = pos->king_square[WHITE];

        // Reward pushing black king to edge
        int score = 100 - distance_to_center[bking] * 10;

        // Reward white king near black king
        score += (14 - chebyshev_distance(wking, bking)) * 5;

        return score;
    }
    return 0;
}
```

### Mating with Bishop + Knight

Drive king to corner matching bishop color.

## Complete Endgame Evaluation

```c
int evaluate_endgame(Position* pos) {
    int score = 0;

    // Check for insufficient material
    if (is_insufficient_material(pos)) {
        return 0;  // Draw
    }

    // Probe tablebase if available
    int tb_score = probe_tablebase(pos);
    if (tb_score != SCORE_NONE) {
        return tb_score;
    }

    // King activity
    score += evaluate_king_endgame(pos);
    score += evaluate_king_pawn_distance(pos);

    // Passed pawns (increased importance)
    score += evaluate_passed_pawns_endgame(pos);

    // Mating patterns
    score += evaluate_mating_patterns(pos);

    // Scale for difficulty
    score = scale_factor(pos, score);

    return score;
}
```

## Summary

Endgame evaluation:

- **King centralization**: Central kings are powerful
- **King-pawn distance**: Kings must chase/protect passed pawns
- **Passed pawn value**: Increases dramatically as pawns advance
- **Insufficient material**: Detect known draws
- **Scale factor**: Reduce scores when winning is difficult
- **Tablebases**: Perfect play for low-piece positions
- **Mating patterns**: Guide king to edge/corner

Endgames require different thinking than middlegames—tapered evaluation blends both perspectives.

Part 4 complete. Next: Engineering the engine—UCI, time management, and debugging.
