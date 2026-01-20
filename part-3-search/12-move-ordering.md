# Chapter 12: Move Ordering

Alpha-beta pruning's effectiveness depends critically on move order. Searching the best move first maximizes cutoffs; searching the worst move first eliminates all pruning. This chapter covers techniques to order moves effectively.

## Why Move Order Matters

With branching factor 35 and depth 10:
- **Perfect ordering**: ~35^5 = 52 million nodes
- **Random ordering**: ~35^10 = 2.7 × 10^15 nodes
- **Good practical ordering**: ~35^7 = 64 billion nodes

The difference between good and perfect ordering is 1000x. The difference between random and good is astronomical.

## Move Ordering Heuristics

We score each move and search in score order. Here's the typical priority:

1. **Hash move** (score: 10,000,000)
2. **Winning captures** by MVV-LVA (score: 1,000,000 + capture_value)
3. **Promotions** (score: 900,000)
4. **Killer moves** (score: 800,000 - 900,000)
5. **Counter moves** (score: 700,000)
6. **Quiet moves** by history (score: history_score)
7. **Losing captures** (score: negative MVV-LVA)

Let's examine each in detail.

## Hash Move

The hash move from the transposition table is almost always best:

```c
Move hash_move = tt_probe_move(pos->hash);

void score_moves(MoveList* moves, SearchInfo* info) {
    for (int i = 0; i < moves->count; i++) {
        if (moves->moves[i] == info->hash_move) {
            moves->scores[i] = 10000000;
        }
        // ... other scoring ...
    }
}
```

When valid, the hash move causes a beta cutoff ~90% of the time.

## MVV-LVA (Most Valuable Victim - Least Valuable Attacker)

For captures, prioritize capturing valuable pieces with less valuable attackers:

```
MVV-LVA score = victim_value × 10 - attacker_value
```

```c
// Piece values for MVV-LVA
int piece_value[7] = {0, 100, 320, 330, 500, 900, 10000};
// Index:            0    1    2    3    4    5     6
// Piece:         NONE PAWN KNIGHT BISHOP ROOK QUEEN KING

int mvv_lva_score(int attacker, int victim) {
    return piece_value[victim] * 10 - piece_value[attacker];
}

// Examples:
// PxQ: 900×10 - 100 = 8900
// QxQ: 900×10 - 900 = 8100
// PxP: 100×10 - 100 = 900
// QxP: 100×10 - 900 = 100
```

This orders captures sensibly: PxQ before QxQ before QxP.

### MVV-LVA Table

Pre-compute for speed:

```c
int MVV_LVA[7][7];  // [attacker][victim]

void init_mvv_lva() {
    for (int a = 0; a < 7; a++) {
        for (int v = 0; v < 7; v++) {
            MVV_LVA[a][v] = piece_value[v] * 10 - piece_value[a];
        }
    }
}

int score_capture(Move move, Position* pos) {
    int attacker = piece_type(pos->board[move_from(move)]);
    int victim = piece_type(pos->board[move_to(move)]);
    return MVV_LVA[attacker][victim] + 1000000;
}
```

## Static Exchange Evaluation (SEE)

MVV-LVA doesn't consider what happens after the capture. **SEE** simulates the full exchange:

```
Position: White Rook on d1, Black Queen on d7, Black Pawn on d6

Move RxQ:
1. White takes queen (+9)
2. Black pawn recaptures (-5)
SEE = +4 (winning exchange)

If no pawn guarded:
1. White takes queen (+9)
SEE = +9
```

SEE is useful for:
- Separating winning from losing captures
- Pruning bad captures in quiescence search
- Move ordering (winning captures first)

A full SEE implementation:

```c
int see(Position* pos, Move move) {
    int from = move_from(move);
    int to = move_to(move);
    int attacker = piece_type(pos->board[from]);
    int target = piece_type(pos->board[to]);

    // Initial gain
    int value = piece_value[target];

    // Simulate the exchange
    uint64 occupied = pos->all_pieces & ~(1ULL << from);
    uint64 attackers = get_attackers(pos, to, occupied);

    int gain[32];
    int d = 0;
    gain[d] = value;

    int side = pos->side_to_move;
    int piece = attacker;

    while (attackers & pos->pieces[side ^ 1]) {
        d++;
        side ^= 1;

        // Find least valuable attacker
        int lva = get_lva(pos, attackers, side);
        if (lva == NONE) break;

        gain[d] = piece_value[piece] - gain[d - 1];
        piece = lva;

        // Remove attacker, add x-ray attackers
        occupied &= ~(1ULL << lva_square);
        attackers = get_attackers(pos, to, occupied);

        // Stop if capturing the king
        if (piece == KING && (attackers & pos->pieces[side ^ 1])) {
            break;
        }
    }

    // Minimax the gains
    while (--d > 0) {
        gain[d - 1] = -max(-gain[d - 1], gain[d]);
    }

    return gain[0];
}
```

Use SEE to separate captures:

```c
int score_capture(Move move, Position* pos) {
    int see_value = see(pos, move);

    if (see_value >= 0) {
        // Winning or equal capture—search early
        return 1000000 + see_value;
    } else {
        // Losing capture—search late
        return see_value;  // Negative
    }
}
```

## Killer Moves

**Killer moves** are quiet moves that caused beta cutoffs at the same depth in sibling nodes.

The insight: if Nf3 refuted white's last move, it might refute white's current move too.

```c
#define MAX_PLY 128
#define NUM_KILLERS 2

Move killers[MAX_PLY][NUM_KILLERS];

void store_killer(int ply, Move move) {
    // Don't store captures (already ordered by MVV-LVA)
    if (is_capture(move)) return;

    // Don't store duplicates
    if (move == killers[ply][0]) return;

    // Shift and insert
    killers[ply][1] = killers[ply][0];
    killers[ply][0] = move;
}

int score_quiet(Move move, int ply, SearchInfo* info) {
    if (move == killers[ply][0]) return 900000;
    if (move == killers[ply][1]) return 800000;

    return history_score(move, info);
}
```

Killers are updated when a quiet move causes a beta cutoff:

```c
if (score >= beta) {
    if (!is_capture(move)) {
        store_killer(ply, move);
        update_history(move, depth, info);
    }
    return beta;
}
```

## History Heuristic

The **history heuristic** tracks which quiet moves have been successful across the entire search:

```c
int history[2][64][64];  // [color][from][to]

void update_history(Move move, int depth, Position* pos) {
    int from = move_from(move);
    int to = move_to(move);
    int color = pos->side_to_move;

    // Bonus proportional to depth squared
    int bonus = depth * depth;
    history[color][from][to] += bonus;

    // Cap to prevent overflow
    if (history[color][from][to] > 10000) {
        // Age all history values
        for (int c = 0; c < 2; c++) {
            for (int f = 0; f < 64; f++) {
                for (int t = 0; t < 64; t++) {
                    history[c][f][t] /= 2;
                }
            }
        }
    }
}

int history_score(Move move, Position* pos) {
    int from = move_from(move);
    int to = move_to(move);
    int color = pos->side_to_move;
    return history[color][from][to];
}
```

### History Malus

Penalize moves that didn't cause cutoffs:

```c
void update_history_searched(Move* moves, int count, Move best, int depth, Position* pos) {
    // Bonus for the best move
    update_history(best, depth, pos);

    // Malus for moves searched before best
    for (int i = 0; i < count; i++) {
        if (moves[i] == best) break;
        if (!is_capture(moves[i])) {
            int bonus = -(depth * depth);
            history[color][from][to] += bonus;
        }
    }
}
```

## Countermove Heuristic

Store the move that refutes each opponent move:

```c
Move countermoves[64][64];  // [prev_from][prev_to]

void store_countermove(Move prev_move, Move refutation) {
    int from = move_from(prev_move);
    int to = move_to(prev_move);
    countermoves[from][to] = refutation;
}

// In move scoring:
Move counter = countermoves[prev_from][prev_to];
if (move == counter && !is_capture(move)) {
    score += 700000;
}
```

## Staged Move Generation

Instead of generating all moves upfront, generate in stages:

```c
enum MoveGenStage {
    STAGE_HASH,
    STAGE_GEN_CAPTURES,
    STAGE_GOOD_CAPTURES,
    STAGE_KILLERS,
    STAGE_GEN_QUIETS,
    STAGE_QUIETS,
    STAGE_BAD_CAPTURES,
    STAGE_DONE
};

Move next_move(MoveGenerator* gen) {
    switch (gen->stage) {
        case STAGE_HASH:
            gen->stage = STAGE_GEN_CAPTURES;
            if (is_valid(gen->hash_move)) {
                return gen->hash_move;
            }
            // Fall through

        case STAGE_GEN_CAPTURES:
            generate_captures(gen->pos, &gen->captures);
            score_captures(&gen->captures);
            gen->stage = STAGE_GOOD_CAPTURES;
            // Fall through

        case STAGE_GOOD_CAPTURES:
            while (gen->cap_idx < gen->captures.count) {
                Move m = pick_best(&gen->captures, gen->cap_idx++);
                if (m == gen->hash_move) continue;  // Already tried
                if (see(gen->pos, m) >= 0) {
                    return m;
                }
                // Bad capture—save for later
                add_move(&gen->bad_captures, m);
            }
            gen->stage = STAGE_KILLERS;
            // Fall through

        case STAGE_KILLERS:
            while (gen->killer_idx < NUM_KILLERS) {
                Move m = killers[gen->ply][gen->killer_idx++];
                if (m != gen->hash_move && is_pseudo_legal(gen->pos, m)) {
                    return m;
                }
            }
            gen->stage = STAGE_GEN_QUIETS;
            // Fall through

        case STAGE_GEN_QUIETS:
            generate_quiets(gen->pos, &gen->quiets);
            score_quiets(&gen->quiets, gen);
            gen->stage = STAGE_QUIETS;
            // Fall through

        case STAGE_QUIETS:
            while (gen->quiet_idx < gen->quiets.count) {
                Move m = pick_best(&gen->quiets, gen->quiet_idx++);
                if (m == gen->hash_move) continue;
                if (m == killers[gen->ply][0]) continue;
                if (m == killers[gen->ply][1]) continue;
                return m;
            }
            gen->stage = STAGE_BAD_CAPTURES;
            // Fall through

        case STAGE_BAD_CAPTURES:
            while (gen->bad_idx < gen->bad_captures.count) {
                return gen->bad_captures.moves[gen->bad_idx++];
            }
            gen->stage = STAGE_DONE;
            // Fall through

        case STAGE_DONE:
            return MOVE_NONE;
    }
}
```

Benefits:
- Avoid generating all moves if cutoff occurs early
- Don't score moves that won't be searched
- Natural ordering by stage

## Move Picking

For unsorted lists, use partial sorting:

```c
Move pick_best(MoveList* list, int start) {
    int best_idx = start;
    int best_score = list->scores[start];

    for (int i = start + 1; i < list->count; i++) {
        if (list->scores[i] > best_score) {
            best_score = list->scores[i];
            best_idx = i;
        }
    }

    // Swap to front
    Move tmp = list->moves[start];
    list->moves[start] = list->moves[best_idx];
    list->moves[best_idx] = tmp;

    int tmp_s = list->scores[start];
    list->scores[start] = list->scores[best_idx];
    list->scores[best_idx] = tmp_s;

    return list->moves[start];
}
```

This is O(n²) but:
- We often cut off after 1-3 moves
- Small n (~30-50 moves max)
- Simpler than full sort

## Putting It Together

```c
void score_moves(MoveList* moves, SearchInfo* info, int ply) {
    Move hash_move = info->hash_move;
    Move prev_move = info->current_move[ply - 1];
    Move counter = countermoves[move_from(prev_move)][move_to(prev_move)];

    for (int i = 0; i < moves->count; i++) {
        Move m = moves->moves[i];

        if (m == hash_move) {
            moves->scores[i] = 10000000;
        }
        else if (is_capture(m)) {
            int see_val = see(info->pos, m);
            if (see_val >= 0) {
                moves->scores[i] = 1000000 + see_val;
            } else {
                moves->scores[i] = see_val - 1000000;  // Very negative
            }
        }
        else if (is_promotion(m)) {
            moves->scores[i] = 950000;
        }
        else if (m == killers[ply][0]) {
            moves->scores[i] = 900000;
        }
        else if (m == killers[ply][1]) {
            moves->scores[i] = 800000;
        }
        else if (m == counter) {
            moves->scores[i] = 700000;
        }
        else {
            // Quiet move—use history
            moves->scores[i] = history_score(m, info->pos);
        }
    }
}
```

## Measuring Move Ordering Quality

Track how often each stage produces cutoffs:

```c
int cutoff_at_move[64];  // How many cutoffs at move index 1, 2, 3...

// In search:
for (int i = 0; i < moves.count; i++) {
    Move m = pick_best(&moves, i);
    // ... search ...
    if (score >= beta) {
        cutoff_at_move[i]++;  // Cutoff at move i
        break;
    }
}

// Report:
// Move 1: 65% of cutoffs
// Move 2: 20% of cutoffs
// Move 3: 8% of cutoffs
// ...
```

Good ordering shows 60-70% of cutoffs on the first move.

## Summary

Move ordering techniques in priority order:

1. **Hash move**: Best move from transposition table (~90% cutoff rate)
2. **Winning captures**: SEE ≥ 0, ordered by MVV-LVA
3. **Promotions**: Almost always good
4. **Killer moves**: Quiet moves that worked at same depth
5. **Countermoves**: Refutations to opponent's last move
6. **History heuristic**: Track successful quiet moves
7. **Losing captures**: SEE < 0, search last

Implementation techniques:
- **Staged generation**: Generate moves on demand
- **Partial sorting**: Pick best without full sort
- **History aging**: Prevent unbounded growth

Good move ordering is essential for practical alpha-beta performance. The hash move alone provides most of the benefit; killers and history add incremental improvements.

Next: Extending search to avoid the horizon effect—quiescence search.
