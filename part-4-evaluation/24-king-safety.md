# Chapter 24: King Safety

The king must be protected in the middlegame but becomes an active piece in endgames. King safety evaluation quantifies how exposed the king is to attack.

## Pawn Shield

The pawns in front of the castled king form a protective shield:

```c
int evaluate_pawn_shield(Position* pos) {
    int score = 0;

    // White king (assuming kingside castled)
    int wking_sq = pos->king_square[WHITE];
    int wking_file = wking_sq % 8;

    if (wking_file >= 5) {  // Kingside
        // Check f2, g2, h2 pawns
        if (pos->white_pawns & F2_BB) score += SHIELD_BONUS;
        else if (pos->white_pawns & F3_BB) score += SHIELD_BONUS / 2;

        if (pos->white_pawns & G2_BB) score += SHIELD_BONUS;
        else if (pos->white_pawns & G3_BB) score += SHIELD_BONUS / 2;

        if (pos->white_pawns & H2_BB) score += SHIELD_BONUS;
        else if (pos->white_pawns & H3_BB) score += SHIELD_BONUS / 2;
    }
    // Similar for queenside castling and black king...

    return score;
}
```

Advanced or missing shield pawns are weaknesses.

## Pawn Storm

Enemy pawns advancing toward our king are dangerous:

```c
int evaluate_pawn_storm(Position* pos) {
    int score = 0;

    int wking_file = pos->king_square[WHITE] % 8;

    // Check enemy pawns near white's king
    for (int f = max(0, wking_file - 1); f <= min(7, wking_file + 1); f++) {
        uint64 file_bb = FILE_A << f;
        uint64 enemy_pawns_on_file = pos->black_pawns & file_bb;

        if (enemy_pawns_on_file) {
            int pawn_sq = bitscan_forward(enemy_pawns_on_file);
            int pawn_rank = pawn_sq / 8;

            // More dangerous the closer to king
            score -= STORM_PENALTY[7 - pawn_rank];
        }
    }

    // Similar for black king vs white pawns...

    return score;
}

int STORM_PENALTY[8] = {0, 0, 5, 15, 30, 50, 70, 0};
```

## Attack Units

Count enemy pieces attacking the king zone:

```c
int evaluate_king_attackers(Position* pos) {
    int score = 0;

    // White king's zone (king + surrounding squares)
    int wking_sq = pos->king_square[WHITE];
    uint64 king_zone = KING_ATTACKS[wking_sq] | (1ULL << wking_sq);

    // Count attacking pieces and their weight
    int attack_units = 0;
    int attackers = 0;

    // Knights attacking king zone
    uint64 knights = pos->black_knights;
    while (knights) {
        int sq = bitscan_forward(knights);
        if (KNIGHT_ATTACKS[sq] & king_zone) {
            attack_units += 2;
            attackers++;
        }
        knights &= knights - 1;
    }

    // Bishops
    uint64 bishops = pos->black_bishops;
    uint64 occ = pos->all_pieces[WHITE] | pos->all_pieces[BLACK];
    while (bishops) {
        int sq = bitscan_forward(bishops);
        if (bishop_attacks(sq, occ) & king_zone) {
            attack_units += 2;
            attackers++;
        }
        bishops &= bishops - 1;
    }

    // Rooks
    uint64 rooks = pos->black_rooks;
    while (rooks) {
        int sq = bitscan_forward(rooks);
        if (rook_attacks(sq, occ) & king_zone) {
            attack_units += 3;
            attackers++;
        }
        rooks &= rooks - 1;
    }

    // Queen
    uint64 queens = pos->black_queens;
    while (queens) {
        int sq = bitscan_forward(queens);
        if (queen_attacks(sq, occ) & king_zone) {
            attack_units += 5;
            attackers++;
        }
        queens &= queens - 1;
    }

    // Scale by number of attackers
    // 1 attacker = not dangerous
    // 2+ attackers = increasingly dangerous
    if (attackers >= 2) {
        score -= ATTACK_WEIGHT[attack_units];
    }

    // Similar for black king...

    return score;
}

// Attack weight table (0-19 attack units)
int ATTACK_WEIGHT[20] = {
    0, 0, 5, 12, 25, 40, 60, 85, 115, 150,
    190, 235, 285, 340, 400, 465, 535, 610, 690, 775
};
```

The attack grows non-linearly—multiple attackers are disproportionately dangerous.

## King Exposure

Count squares around the king attacked by enemy but not defended:

```c
int evaluate_king_exposure(Position* pos) {
    int wking_sq = pos->king_square[WHITE];
    uint64 king_zone = KING_ATTACKS[wking_sq];

    uint64 our_attacks = all_attacks(pos, WHITE);
    uint64 their_attacks = all_attacks(pos, BLACK);

    // Weak squares: attacked by them, not defended by us
    uint64 weak_squares = king_zone & their_attacks & ~our_attacks;

    return -EXPOSURE_PENALTY * popcount(weak_squares);
}
```

## Open Lines Toward King

Open files and diagonals pointing at the king are dangerous:

```c
int evaluate_open_lines_to_king(Position* pos) {
    int score = 0;
    int wking_sq = pos->king_square[WHITE];
    int file = wking_sq % 8;

    // Open file toward king
    uint64 file_bb = FILE_A << file;
    if (!(file_bb & pos->white_pawns) && !(file_bb & pos->black_pawns)) {
        // Completely open file
        if (pos->black_rooks & file_bb || pos->black_queens & file_bb) {
            score -= OPEN_FILE_ATTACK;
        }
    }

    // Semi-open file
    if (!(file_bb & pos->white_pawns) && (file_bb & pos->black_pawns)) {
        if (pos->black_rooks & file_bb || pos->black_queens & file_bb) {
            score -= SEMI_OPEN_FILE_ATTACK;
        }
    }

    return score;
}
```

## Castling Status

Not having castled (and king in center) is dangerous:

```c
int evaluate_castling(Position* pos) {
    int score = 0;

    // White king not castled (still on e1 or nearby, queens present)
    if (pos->white_queens && pos->black_queens) {
        int wking_sq = pos->king_square[WHITE];
        if (rank_of(wking_sq) == 0 && file_of(wking_sq) >= 3 && file_of(wking_sq) <= 5) {
            // King still in center with queens on board
            score -= UNCASTLED_PENALTY;
        }
    }

    return score;
}
```

## Complete King Safety

```c
int evaluate_king_safety(Position* pos) {
    // Only evaluate in middlegame (when queens present)
    if (!pos->white_queens && !pos->black_queens) {
        return 0;  // Endgame—king safety less relevant
    }

    int score = 0;

    score += evaluate_pawn_shield(pos);
    score += evaluate_pawn_storm(pos);
    score += evaluate_king_attackers(pos);
    score += evaluate_king_exposure(pos);
    score += evaluate_open_lines_to_king(pos);
    score += evaluate_castling(pos);

    return score;
}
```

## Virtual Mobility

Some engines compute how many squares the king would have if attacked:

```c
int virtual_mobility(Position* pos, int color) {
    int king_sq = pos->king_square[color];
    uint64 king_moves = KING_ATTACKS[king_sq] & ~pos->all_pieces[color];
    uint64 enemy_attacks = all_attacks(pos, color ^ 1);

    // Safe squares the king could flee to
    uint64 safe_escapes = king_moves & ~enemy_attacks;

    return popcount(safe_escapes);
}
```

A king with few escape squares is in danger.

## Scaling by Game Phase

King safety matters most in the middlegame:

```c
int king_safety = evaluate_king_safety(pos);

// Scale down as pieces disappear
int phase = compute_phase(pos);  // 256 = full, 0 = endgame
king_safety = king_safety * phase / 256;
```

## Sample Values

| Term | Value (centipawns) |
|------|-------------------|
| Shield pawn present | +10 to +15 each |
| Shield pawn advanced | -5 to -10 |
| Storm pawn on 4th | -30 |
| Storm pawn on 5th | -50 |
| Each attacker after 1st | Non-linear (see table) |
| Open file to king | -30 to -50 |
| Uncastled with queens | -50 to -100 |

## Summary

King safety evaluation:

- **Pawn shield**: Bonus for intact shield pawns
- **Pawn storm**: Penalty for advancing enemy pawns
- **Attackers**: Count pieces attacking king zone, non-linear scaling
- **Exposure**: Penalty for undefended squares around king
- **Open lines**: Penalty for open files/diagonals to king
- **Castling**: Penalty for uncastled king when queens present

King safety is crucial in the middlegame and diminishes as pieces are exchanged.

Next: Endgame evaluation—when kings become fighters.
