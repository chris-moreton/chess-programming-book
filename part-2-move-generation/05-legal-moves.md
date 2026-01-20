# Chapter 5: Legal Move Generation

Move generation is the engine's most frequently called function. Every node in the search tree requires generating moves. This chapter covers how to generate all legal moves in any position, handling the many edge cases that make chess interesting.

## Pseudo-Legal vs Legal Moves

There are two approaches to move generation:

**Pseudo-legal generation**: Generate all moves that follow piece movement rules, ignoring whether they leave the king in check. Then verify legality when making the move.

**Fully legal generation**: Only generate moves that are actually legal—never leaving the king in check.

Most engines use pseudo-legal generation because:
1. It's simpler to implement
2. Many moves are pruned before being played (alpha-beta cutoffs)
3. Legality check during make-move is fast

However, understanding fully legal generation teaches important concepts about check and pin detection.

### The Pseudo-Legal Approach

```c
void generate_pseudo_legal_moves(Position* pos, MoveList* moves) {
    generate_pawn_moves(pos, moves);
    generate_knight_moves(pos, moves);
    generate_bishop_moves(pos, moves);
    generate_rook_moves(pos, moves);
    generate_queen_moves(pos, moves);
    generate_king_moves(pos, moves);
    generate_castling_moves(pos, moves);
}

bool make_move(Position* pos, Move move) {
    // Make the move
    apply_move(pos, move);

    // Check if our king is now in check
    if (is_king_in_check(pos, pos->side_to_move ^ 1)) {
        // Illegal - undo and return false
        unmake_move(pos, move);
        return false;
    }

    return true;
}
```

### The Fully Legal Approach

```c
void generate_legal_moves(Position* pos, MoveList* moves) {
    if (is_in_check(pos)) {
        generate_evasions(pos, moves);  // Only moves that escape check
    } else {
        generate_non_evasions(pos, moves);  // Normal moves, respecting pins
    }
}
```

This chapter covers both approaches, as the concepts apply regardless of which you choose.

## Generating Moves by Piece Type

Let's implement move generation for each piece type using bitboards.

### Knight Moves

Knights are simplest—their attacks don't depend on occupancy:

```c
void generate_knight_moves(Position* pos, MoveList* moves) {
    int us = pos->side_to_move;
    uint64 knights = pos->pieces[us][KNIGHT];
    uint64 own_pieces = pos->all_pieces[us];

    while (knights) {
        int from = bitscan_forward(knights);
        uint64 attacks = KNIGHT_ATTACKS[from] & ~own_pieces;

        // Split into captures and quiets for move ordering
        uint64 captures = attacks & pos->all_pieces[us ^ 1];
        uint64 quiets = attacks & ~pos->all_pieces[us ^ 1];

        while (captures) {
            int to = bitscan_forward(captures);
            add_move(moves, from, to, CAPTURE);
            captures &= captures - 1;
        }

        while (quiets) {
            int to = bitscan_forward(quiets);
            add_move(moves, from, to, QUIET);
            quiets &= quiets - 1;
        }

        knights &= knights - 1;
    }
}
```

### Bishop and Rook Moves

Using magic bitboards from Chapter 4:

```c
void generate_bishop_moves(Position* pos, MoveList* moves) {
    int us = pos->side_to_move;
    uint64 bishops = pos->pieces[us][BISHOP];
    uint64 own_pieces = pos->all_pieces[us];
    uint64 occupancy = pos->all_pieces[WHITE] | pos->all_pieces[BLACK];

    while (bishops) {
        int from = bitscan_forward(bishops);
        uint64 attacks = bishop_attacks(from, occupancy) & ~own_pieces;

        uint64 captures = attacks & pos->all_pieces[us ^ 1];
        uint64 quiets = attacks & ~pos->all_pieces[us ^ 1];

        while (captures) {
            int to = bitscan_forward(captures);
            add_move(moves, from, to, CAPTURE);
            captures &= captures - 1;
        }

        while (quiets) {
            int to = bitscan_forward(quiets);
            add_move(moves, from, to, QUIET);
            quiets &= quiets - 1;
        }

        bishops &= bishops - 1;
    }
}

void generate_rook_moves(Position* pos, MoveList* moves) {
    // Same structure as bishop, using rook_attacks()
    int us = pos->side_to_move;
    uint64 rooks = pos->pieces[us][ROOK];
    uint64 own_pieces = pos->all_pieces[us];
    uint64 occupancy = pos->all_pieces[WHITE] | pos->all_pieces[BLACK];

    while (rooks) {
        int from = bitscan_forward(rooks);
        uint64 attacks = rook_attacks(from, occupancy) & ~own_pieces;

        uint64 captures = attacks & pos->all_pieces[us ^ 1];
        uint64 quiets = attacks & ~pos->all_pieces[us ^ 1];

        while (captures) {
            int to = bitscan_forward(captures);
            add_move(moves, from, to, CAPTURE);
            captures &= captures - 1;
        }

        while (quiets) {
            int to = bitscan_forward(quiets);
            add_move(moves, from, to, QUIET);
            quiets &= quiets - 1;
        }

        rooks &= rooks - 1;
    }
}

void generate_queen_moves(Position* pos, MoveList* moves) {
    int us = pos->side_to_move;
    uint64 queens = pos->pieces[us][QUEEN];
    uint64 own_pieces = pos->all_pieces[us];
    uint64 occupancy = pos->all_pieces[WHITE] | pos->all_pieces[BLACK];

    while (queens) {
        int from = bitscan_forward(queens);
        // Queen = bishop + rook attacks
        uint64 attacks = (bishop_attacks(from, occupancy) |
                          rook_attacks(from, occupancy)) & ~own_pieces;

        uint64 captures = attacks & pos->all_pieces[us ^ 1];
        uint64 quiets = attacks & ~pos->all_pieces[us ^ 1];

        while (captures) {
            int to = bitscan_forward(captures);
            add_move(moves, from, to, CAPTURE);
            captures &= captures - 1;
        }

        while (quiets) {
            int to = bitscan_forward(quiets);
            add_move(moves, from, to, QUIET);
            quiets &= quiets - 1;
        }

        queens &= queens - 1;
    }
}
```

### King Moves (Non-Castling)

```c
void generate_king_moves(Position* pos, MoveList* moves) {
    int us = pos->side_to_move;
    int from = pos->king_square[us];
    uint64 own_pieces = pos->all_pieces[us];

    uint64 attacks = KING_ATTACKS[from] & ~own_pieces;

    // Note: We don't check if destination is attacked here (pseudo-legal)
    // That check happens in make_move or in fully-legal generation

    uint64 captures = attacks & pos->all_pieces[us ^ 1];
    uint64 quiets = attacks & ~pos->all_pieces[us ^ 1];

    while (captures) {
        int to = bitscan_forward(captures);
        add_move(moves, from, to, CAPTURE);
        captures &= captures - 1;
    }

    while (quiets) {
        int to = bitscan_forward(quiets);
        add_move(moves, from, to, QUIET);
        quiets &= quiets - 1;
    }
}
```

### Pawn Moves

Pawns are the most complex due to their directional movement, double pushes, promotions, and en passant:

```c
void generate_pawn_moves(Position* pos, MoveList* moves) {
    int us = pos->side_to_move;
    int them = us ^ 1;
    uint64 pawns = pos->pieces[us][PAWN];
    uint64 empty = ~(pos->all_pieces[WHITE] | pos->all_pieces[BLACK]);
    uint64 enemy = pos->all_pieces[them];

    // Direction depends on color
    int push_dir = (us == WHITE) ? 8 : -8;
    uint64 start_rank = (us == WHITE) ? RANK_2 : RANK_7;
    uint64 promo_rank = (us == WHITE) ? RANK_8 : RANK_1;
    uint64 pre_promo = (us == WHITE) ? RANK_7 : RANK_2;

    // Single pushes (non-promotion)
    uint64 single = shift_forward(pawns & ~pre_promo, us) & empty;
    while (single) {
        int to = bitscan_forward(single);
        add_move(moves, to - push_dir, to, QUIET);
        single &= single - 1;
    }

    // Double pushes
    uint64 double_push = shift_forward(pawns & start_rank, us) & empty;
    double_push = shift_forward(double_push, us) & empty;
    while (double_push) {
        int to = bitscan_forward(double_push);
        add_move(moves, to - 2 * push_dir, to, DOUBLE_PAWN_PUSH);
        double_push &= double_push - 1;
    }

    // Captures (non-promotion)
    uint64 capture_left = shift_attack_left(pawns & ~pre_promo, us) & enemy;
    uint64 capture_right = shift_attack_right(pawns & ~pre_promo, us) & enemy;

    while (capture_left) {
        int to = bitscan_forward(capture_left);
        int from = to - ((us == WHITE) ? 7 : -9);
        add_move(moves, from, to, CAPTURE);
        capture_left &= capture_left - 1;
    }

    while (capture_right) {
        int to = bitscan_forward(capture_right);
        int from = to - ((us == WHITE) ? 9 : -7);
        add_move(moves, from, to, CAPTURE);
        capture_right &= capture_right - 1;
    }

    // Promotions (push)
    uint64 promo_push = shift_forward(pawns & pre_promo, us) & empty;
    while (promo_push) {
        int to = bitscan_forward(promo_push);
        int from = to - push_dir;
        add_move(moves, from, to, PROMO_QUEEN);
        add_move(moves, from, to, PROMO_ROOK);
        add_move(moves, from, to, PROMO_BISHOP);
        add_move(moves, from, to, PROMO_KNIGHT);
        promo_push &= promo_push - 1;
    }

    // Promotions (capture)
    uint64 promo_cap_left = shift_attack_left(pawns & pre_promo, us) & enemy;
    uint64 promo_cap_right = shift_attack_right(pawns & pre_promo, us) & enemy;

    while (promo_cap_left) {
        int to = bitscan_forward(promo_cap_left);
        int from = to - ((us == WHITE) ? 7 : -9);
        add_move(moves, from, to, PROMO_CAPTURE_QUEEN);
        add_move(moves, from, to, PROMO_CAPTURE_ROOK);
        add_move(moves, from, to, PROMO_CAPTURE_BISHOP);
        add_move(moves, from, to, PROMO_CAPTURE_KNIGHT);
        promo_cap_left &= promo_cap_left - 1;
    }

    while (promo_cap_right) {
        int to = bitscan_forward(promo_cap_right);
        int from = to - ((us == WHITE) ? 9 : -7);
        add_move(moves, from, to, PROMO_CAPTURE_QUEEN);
        add_move(moves, from, to, PROMO_CAPTURE_ROOK);
        add_move(moves, from, to, PROMO_CAPTURE_BISHOP);
        add_move(moves, from, to, PROMO_CAPTURE_KNIGHT);
        promo_cap_right &= promo_cap_right - 1;
    }

    // En passant
    if (pos->en_passant_square != NO_SQUARE) {
        uint64 ep_bb = 1ULL << pos->en_passant_square;
        uint64 ep_attackers = PAWN_ATTACKS[them][pos->en_passant_square] & pawns;

        while (ep_attackers) {
            int from = bitscan_forward(ep_attackers);
            add_move(moves, from, pos->en_passant_square, EN_PASSANT);
            ep_attackers &= ep_attackers - 1;
        }
    }
}

// Helper functions for pawn shifts
uint64 shift_forward(uint64 bb, int color) {
    return (color == WHITE) ? (bb << 8) : (bb >> 8);
}

uint64 shift_attack_left(uint64 bb, int color) {
    return (color == WHITE) ? ((bb & ~FILE_A) << 7) : ((bb & ~FILE_H) >> 7);
}

uint64 shift_attack_right(uint64 bb, int color) {
    return (color == WHITE) ? ((bb & ~FILE_H) << 9) : ((bb & ~FILE_A) >> 9);
}
```

## Check Detection

Detecting whether the king is in check is fundamental. We need it for:
- Validating pseudo-legal moves
- Generating check evasions
- Search extensions
- Evaluation

### Is the King Attacked?

```c
bool is_square_attacked(Position* pos, int square, int by_color) {
    uint64 occupancy = pos->all_pieces[WHITE] | pos->all_pieces[BLACK];

    // Knight attacks
    if (KNIGHT_ATTACKS[square] & pos->pieces[by_color][KNIGHT])
        return true;

    // King attacks (for adjacent king check detection)
    if (KING_ATTACKS[square] & pos->pieces[by_color][KING])
        return true;

    // Pawn attacks
    if (PAWN_ATTACKS[by_color ^ 1][square] & pos->pieces[by_color][PAWN])
        return true;

    // Bishop/Queen attacks (diagonal)
    uint64 bishop_attackers = pos->pieces[by_color][BISHOP] |
                              pos->pieces[by_color][QUEEN];
    if (bishop_attacks(square, occupancy) & bishop_attackers)
        return true;

    // Rook/Queen attacks (orthogonal)
    uint64 rook_attackers = pos->pieces[by_color][ROOK] |
                            pos->pieces[by_color][QUEEN];
    if (rook_attacks(square, occupancy) & rook_attackers)
        return true;

    return false;
}

bool is_in_check(Position* pos) {
    int us = pos->side_to_move;
    int king_sq = pos->king_square[us];
    return is_square_attacked(pos, king_sq, us ^ 1);
}
```

### Finding All Attackers

Sometimes we need to know which pieces give check:

```c
uint64 attackers_to(Position* pos, int square, int by_color) {
    uint64 attackers = 0;
    uint64 occupancy = pos->all_pieces[WHITE] | pos->all_pieces[BLACK];

    attackers |= KNIGHT_ATTACKS[square] & pos->pieces[by_color][KNIGHT];
    attackers |= KING_ATTACKS[square] & pos->pieces[by_color][KING];
    attackers |= PAWN_ATTACKS[by_color ^ 1][square] & pos->pieces[by_color][PAWN];
    attackers |= bishop_attacks(square, occupancy) &
                 (pos->pieces[by_color][BISHOP] | pos->pieces[by_color][QUEEN]);
    attackers |= rook_attacks(square, occupancy) &
                 (pos->pieces[by_color][ROOK] | pos->pieces[by_color][QUEEN]);

    return attackers;
}
```

### Double Check

When two pieces give check simultaneously (discovered + direct), only king moves are legal:

```c
int count_checkers(Position* pos) {
    int us = pos->side_to_move;
    uint64 checkers = attackers_to(pos, pos->king_square[us], us ^ 1);
    return popcount(checkers);
}
```

## Pin Detection

A piece is **pinned** if moving it would expose the king to check. Pinned pieces have restricted movement.

### Finding Pinned Pieces

```c
uint64 get_pinned_pieces(Position* pos, int color) {
    uint64 pinned = 0;
    int king_sq = pos->king_square[color];
    int them = color ^ 1;
    uint64 occupancy = pos->all_pieces[WHITE] | pos->all_pieces[BLACK];

    // Check for pins by bishops/queens (diagonal)
    uint64 diagonal_pinners = (pos->pieces[them][BISHOP] |
                               pos->pieces[them][QUEEN]) &
                              bishop_attacks(king_sq, 0);  // X-ray through all pieces

    while (diagonal_pinners) {
        int pinner_sq = bitscan_forward(diagonal_pinners);

        // Get squares between king and potential pinner
        uint64 between = between_bb(king_sq, pinner_sq) & occupancy;

        // If exactly one piece between, it's pinned
        if (popcount(between) == 1) {
            pinned |= between & pos->all_pieces[color];  // Only our pieces can be pinned
        }

        diagonal_pinners &= diagonal_pinners - 1;
    }

    // Check for pins by rooks/queens (orthogonal)
    uint64 orthogonal_pinners = (pos->pieces[them][ROOK] |
                                 pos->pieces[them][QUEEN]) &
                                rook_attacks(king_sq, 0);

    while (orthogonal_pinners) {
        int pinner_sq = bitscan_forward(orthogonal_pinners);
        uint64 between = between_bb(king_sq, pinner_sq) & occupancy;

        if (popcount(between) == 1) {
            pinned |= between & pos->all_pieces[color];
        }

        orthogonal_pinners &= orthogonal_pinners - 1;
    }

    return pinned;
}
```

The `between_bb(sq1, sq2)` function returns the squares strictly between two squares on the same line. This is typically precomputed:

```c
uint64 BETWEEN_BB[64][64];  // Precomputed

void init_between_bb() {
    for (int sq1 = 0; sq1 < 64; sq1++) {
        for (int sq2 = 0; sq2 < 64; sq2++) {
            BETWEEN_BB[sq1][sq2] = compute_between(sq1, sq2);
        }
    }
}
```

### Using Pin Information

For fully legal generation, pinned pieces can only move along the pin ray:

```c
void generate_legal_knight_moves(Position* pos, MoveList* moves) {
    uint64 pinned = get_pinned_pieces(pos, pos->side_to_move);
    uint64 knights = pos->pieces[pos->side_to_move][KNIGHT] & ~pinned;

    // Pinned knights cannot move at all (knight moves never stay on pin ray)
    // Generate moves only for unpinned knights
    // ... same as before but using 'knights' which excludes pinned ...
}

void generate_legal_bishop_moves(Position* pos, MoveList* moves) {
    uint64 pinned = get_pinned_pieces(pos, pos->side_to_move);
    int us = pos->side_to_move;
    int king_sq = pos->king_square[us];

    uint64 bishops = pos->pieces[us][BISHOP];

    while (bishops) {
        int from = bitscan_forward(bishops);
        uint64 attacks = bishop_attacks(from, occupancy) & ~own_pieces;

        // If pinned, restrict to pin ray
        if ((1ULL << from) & pinned) {
            attacks &= LINE_BB[king_sq][from];  // Only moves along the pin line
        }

        // Add moves from 'attacks'...
        bishops &= bishops - 1;
    }
}
```

The `LINE_BB[sq1][sq2]` table contains all squares on the line through both squares (the full ray, not just between).

## Castling Rules and Validation

Castling has several requirements:

1. King and rook have not moved (tracked by castling rights flags)
2. Squares between king and rook are empty
3. King is not in check
4. King does not pass through or land on an attacked square

```c
void generate_castling_moves(Position* pos, MoveList* moves) {
    int us = pos->side_to_move;
    int them = us ^ 1;
    int king_sq = pos->king_square[us];

    // Can't castle out of check
    if (is_square_attacked(pos, king_sq, them)) return;

    uint64 occupancy = pos->all_pieces[WHITE] | pos->all_pieces[BLACK];

    if (us == WHITE) {
        // Kingside (O-O)
        if (pos->castling_rights & WHITE_OO) {
            // Squares f1, g1 must be empty
            if (!(occupancy & (F1_BB | G1_BB))) {
                // King passes through f1, lands on g1 - neither can be attacked
                if (!is_square_attacked(pos, F1, them) &&
                    !is_square_attacked(pos, G1, them)) {
                    add_move(moves, E1, G1, CASTLING);
                }
            }
        }

        // Queenside (O-O-O)
        if (pos->castling_rights & WHITE_OOO) {
            // Squares b1, c1, d1 must be empty
            if (!(occupancy & (B1_BB | C1_BB | D1_BB))) {
                // King passes through d1, lands on c1 - neither can be attacked
                // Note: b1 can be attacked (rook passes through, not king)
                if (!is_square_attacked(pos, D1, them) &&
                    !is_square_attacked(pos, C1, them)) {
                    add_move(moves, E1, C1, CASTLING);
                }
            }
        }
    } else {
        // Black castling (similar, with e8, f8, g8 / b8, c8, d8)
        if (pos->castling_rights & BLACK_OO) {
            if (!(occupancy & (F8_BB | G8_BB))) {
                if (!is_square_attacked(pos, F8, them) &&
                    !is_square_attacked(pos, G8, them)) {
                    add_move(moves, E8, G8, CASTLING);
                }
            }
        }

        if (pos->castling_rights & BLACK_OOO) {
            if (!(occupancy & (B8_BB | C8_BB | D8_BB))) {
                if (!is_square_attacked(pos, D8, them) &&
                    !is_square_attacked(pos, C8, them)) {
                    add_move(moves, E8, C8, CASTLING);
                }
            }
        }
    }
}
```

### Updating Castling Rights

When making a move, castling rights may be lost:

```c
void update_castling_rights(Position* pos, Move move) {
    int from = move_from(move);
    int to = move_to(move);

    // If king moves, lose both castling rights
    if (pos->board[from] == WKING) {
        pos->castling_rights &= ~(WHITE_OO | WHITE_OOO);
    } else if (pos->board[from] == BKING) {
        pos->castling_rights &= ~(BLACK_OO | BLACK_OOO);
    }

    // If rook moves from original square, lose that side
    if (from == A1 || to == A1) pos->castling_rights &= ~WHITE_OOO;
    if (from == H1 || to == H1) pos->castling_rights &= ~WHITE_OO;
    if (from == A8 || to == A8) pos->castling_rights &= ~BLACK_OOO;
    if (from == H8 || to == H8) pos->castling_rights &= ~BLACK_OO;
}
```

Note that `to` is also checked—capturing an unmoved rook removes that castling right.

## En Passant Edge Cases

En passant has subtle legality issues.

### The Horizontal Pin

Consider: White king on e5, white pawn on d5, black pawn on c5 (just moved c7-c5), black rook on a5.

If the white pawn captures en passant (dxc6), both the white pawn AND the captured black pawn leave the 5th rank, exposing the white king to the rook!

```
Before en passant:
8 |  .  .  .  .  .  .  .  . |
7 |  .  .  .  .  .  .  .  . |
6 |  .  .  .  .  .  .  .  . |
5 |  r  .  p  P  K  .  .  . |    r=black rook, p=black pawn, P=white pawn, K=white king
4 |  .  .  .  .  .  .  .  . |
...

After en passant dxc6:
8 |  .  .  .  .  .  .  .  . |
7 |  .  .  .  .  .  .  .  . |
6 |  .  .  P  .  .  .  .  . |
5 |  r  .  .  .  K  .  .  . |    King exposed to rook!
4 |  .  .  .  .  .  .  .  . |
```

The en passant capture is illegal, but this isn't caught by normal pin detection (the pawn isn't pinned by the rook before the move).

### Handling En Passant Legality

The simplest approach: always make the en passant move and check legality:

```c
// In pseudo-legal generation, generate all en passant moves
// In make_move, verify the king isn't exposed:

bool make_move(Position* pos, Move move) {
    apply_move(pos, move);

    // For en passant, we must check for this special pin
    // (Standard check detection handles it - the king will be in check)
    if (is_in_check(pos, pos->side_to_move ^ 1)) {
        unmake_move(pos, move);
        return false;
    }

    return true;
}
```

For fully legal generation, you can detect this case explicitly:

```c
bool is_en_passant_legal(Position* pos, int from, int ep_square) {
    int us = pos->side_to_move;
    int king_sq = pos->king_square[us];
    int captured_sq = ep_square + ((us == WHITE) ? -8 : 8);

    // Temporarily remove both pawns
    uint64 occupancy = pos->all_pieces[WHITE] | pos->all_pieces[BLACK];
    occupancy &= ~(1ULL << from);           // Remove our pawn
    occupancy &= ~(1ULL << captured_sq);    // Remove captured pawn

    // Check if king is attacked with this occupancy
    int them = us ^ 1;

    // Only need to check rook/queen attacks (the horizontal pin scenario)
    uint64 rook_attackers = pos->pieces[them][ROOK] | pos->pieces[them][QUEEN];
    if (rook_attacks(king_sq, occupancy) & rook_attackers) {
        return false;
    }

    return true;
}
```

## Generating Check Evasions

When in check, only specific moves are legal:

1. **King moves**: Move the king to a non-attacked square
2. **Block**: Place a piece between the attacker and king (only for single check by slider)
3. **Capture**: Capture the attacking piece (only for single check)

Double check only allows king moves.

```c
void generate_evasions(Position* pos, MoveList* moves) {
    int us = pos->side_to_move;
    int them = us ^ 1;
    int king_sq = pos->king_square[us];

    uint64 checkers = attackers_to(pos, king_sq, them);
    int num_checkers = popcount(checkers);

    // Always generate king moves
    generate_king_evasions(pos, moves, checkers);

    // Double check: only king moves are legal
    if (num_checkers > 1) return;

    // Single check: can also block or capture
    int checker_sq = bitscan_forward(checkers);
    uint64 target;

    // Determine target squares (capture checker or block)
    if (is_slider(pos->board[checker_sq])) {
        // Can capture checker or block the ray
        target = (1ULL << checker_sq) | BETWEEN_BB[king_sq][checker_sq];
    } else {
        // Knight or pawn check: can only capture
        target = 1ULL << checker_sq;
    }

    // Generate non-king moves that land on target squares
    generate_pawn_moves_to(pos, moves, target);
    generate_knight_moves_to(pos, moves, target);
    generate_bishop_moves_to(pos, moves, target);
    generate_rook_moves_to(pos, moves, target);
    generate_queen_moves_to(pos, moves, target);
}

void generate_king_evasions(Position* pos, MoveList* moves, uint64 checkers) {
    int us = pos->side_to_move;
    int them = us ^ 1;
    int king_sq = pos->king_square[us];

    uint64 king_attacks = KING_ATTACKS[king_sq] & ~pos->all_pieces[us];

    // Remove squares attacked by enemy (including X-ray through king)
    uint64 occupancy = (pos->all_pieces[WHITE] | pos->all_pieces[BLACK])
                       & ~(1ULL << king_sq);  // Remove king for X-ray

    while (king_attacks) {
        int to = bitscan_forward(king_attacks);

        if (!is_square_attacked_occ(pos, to, them, occupancy)) {
            if (pos->all_pieces[them] & (1ULL << to)) {
                add_move(moves, king_sq, to, CAPTURE);
            } else {
                add_move(moves, king_sq, to, QUIET);
            }
        }

        king_attacks &= king_attacks - 1;
    }
}
```

## Performance Considerations

Move generation is called billions of times. Small optimizations matter:

### Avoid Redundant Computation

```c
// Bad: Recomputing inside loop
while (pieces) {
    int sq = bitscan_forward(pieces);
    uint64 occ = pos->all_pieces[WHITE] | pos->all_pieces[BLACK];  // Redundant!
    // ...
}

// Good: Compute once outside loop
uint64 occupancy = pos->all_pieces[WHITE] | pos->all_pieces[BLACK];
while (pieces) {
    int sq = bitscan_forward(pieces);
    // Use 'occupancy'
}
```

### Use Bulk Operations

```c
// All pawn attacks at once (no loop over individual pawns)
uint64 pawn_attacks_left = ((white_pawns & ~FILE_A) << 7);
uint64 pawn_attacks_right = ((white_pawns & ~FILE_H) << 9);
uint64 all_pawn_attacks = pawn_attacks_left | pawn_attacks_right;
```

### Staged Move Generation

Generate moves in stages, stopping early if a beta cutoff occurs:

1. Hash move (from transposition table)
2. Winning captures (SEE > 0)
3. Killer moves
4. Quiet moves
5. Losing captures

This is covered in detail in Chapter 12 (Move Ordering).

## Summary

Move generation requires handling many cases:

- **Piece-specific movement**: Knights, bishops, rooks, queens, kings each have unique patterns
- **Pawns**: Pushes, double pushes, captures, en passant, promotions
- **Castling**: Check all five conditions
- **Check detection**: Essential for legality validation
- **Pin detection**: Restricts pinned piece movement
- **Evasions**: Special handling when in check

For pseudo-legal generation:
1. Generate all moves following movement rules
2. Verify legality when making the move (check if king is left in check)

For fully legal generation:
1. Detect pins and check status upfront
2. Only generate moves that are actually legal

Most engines use pseudo-legal generation for simplicity, with legality verified in the make-move function.

Next chapter: How to efficiently encode and decode moves.
