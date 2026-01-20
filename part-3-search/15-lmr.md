# Chapter 15: Late Move Reductions (LMR)

Late Move Reductions is one of the most important modern search techniques. The idea: moves searched late (after the first few) are probably bad and can be searched with reduced depth. If they surprise us, we re-search at full depth.

## The Observation

With good move ordering, the best move is usually searched first. Moves searched later are likely refuted by alpha-beta. Why spend full depth on them?

Statistics from typical positions:
- Move 1 causes cutoff: ~65% of the time
- Move 2 causes cutoff: ~20% of the time
- Move 3+: Less than ~15% combined

The first few moves deserve full depth. Later moves can be reduced.

## Basic LMR

```c
int search(Position* pos, int depth, int alpha, int beta) {
    MoveList moves;
    generate_moves(pos, &moves);
    score_moves(&moves);

    int moves_searched = 0;

    for (int i = 0; i < moves.count; i++) {
        Move move = pick_best(&moves, i);
        make_move(pos, move);
        moves_searched++;

        int score;

        // Full search for first few moves
        if (moves_searched <= 3) {
            score = -search(pos, depth - 1, -beta, -alpha);
        }
        // Reduced search for later moves
        else {
            // Search with reduced depth
            int reduction = compute_reduction(depth, moves_searched);
            score = -search(pos, depth - 1 - reduction, -alpha - 1, -alpha);

            // If reduced search beats alpha, re-search at full depth
            if (score > alpha) {
                score = -search(pos, depth - 1, -beta, -alpha);
            }
        }

        unmake_move(pos, move);

        if (score > alpha) {
            alpha = score;
            // ... update best move ...
        }

        if (alpha >= beta) {
            return beta;  // Cutoff
        }
    }

    return alpha;
}
```

## Reduction Formula

The reduction amount depends on depth and move number:

```c
int LMR_TABLE[64][64];  // [depth][move_number]

void init_lmr_table() {
    for (int depth = 0; depth < 64; depth++) {
        for (int moves = 0; moves < 64; moves++) {
            if (depth == 0 || moves == 0) {
                LMR_TABLE[depth][moves] = 0;
            } else {
                // Logarithmic formula (common choice)
                double r = log(depth) * log(moves) / 2.0;
                LMR_TABLE[depth][moves] = (int)r;
            }
        }
    }
}

int compute_reduction(int depth, int moves_searched) {
    int r = LMR_TABLE[min(depth, 63)][min(moves_searched, 63)];
    return max(0, r);
}
```

Typical reductions:
- Depth 6, move 5: reduce by 1
- Depth 10, move 10: reduce by 2
- Depth 15, move 20: reduce by 3

## Conditions for LMR

Not all moves should be reduced:

```c
bool can_reduce(Move move, Position* pos, int depth, int moves_searched,
                bool in_check, bool gives_check) {
    // Don't reduce the first few moves
    if (moves_searched <= 3) return false;

    // Don't reduce at shallow depth
    if (depth < 3) return false;

    // Don't reduce while in check
    if (in_check) return false;

    // Don't reduce moves that give check
    if (gives_check) return false;

    // Don't reduce captures with good SEE
    if (is_capture(move) && see(pos, move) >= 0) return false;

    // Don't reduce promotions
    if (is_promotion(move)) return false;

    // Don't reduce killer moves
    if (is_killer(move, ply)) return false;

    return true;  // Ok to reduce
}
```

## Full Window Re-Search

The two-step process:

1. **Reduced search**: `score = -search(depth - 1 - R, -alpha - 1, -alpha)`
2. **If beats alpha**: `score = -search(depth - 1, -beta, -alpha)` (full depth, full window)

```c
int score;
int reduction = 0;

if (can_reduce(move, ...)) {
    reduction = LMR_TABLE[depth][moves_searched];

    // Additional reduction/reduction tweaks
    if (!is_pv_node) reduction++;
    if (history_score(move) < 0) reduction++;
    if (improving) reduction--;  // Position is getting better

    reduction = clamp(reduction, 0, depth - 2);
}

// Search with possible reduction
score = -search(pos, depth - 1 - reduction, -alpha - 1, -alpha);

// Re-search if needed
if (score > alpha && reduction > 0) {
    score = -search(pos, depth - 1, -alpha - 1, -alpha);
}
if (score > alpha && score < beta) {
    score = -search(pos, depth - 1, -beta, -alpha);
}
```

## History-Based Reductions

Adjust reduction based on move history:

```c
int reduction = LMR_TABLE[depth][moves_searched];

// Reduce more for moves with bad history
int hist = history_score(move, pos);
if (hist < -1000) reduction += 2;
else if (hist < 0) reduction += 1;

// Reduce less for moves with good history
if (hist > 5000) reduction -= 1;
```

Moves that have historically caused cutoffs deserve more depth.

## Improving Detection

Track whether static evaluation is improving:

```c
bool improving = false;
if (!in_check && ply >= 2) {
    int current_eval = ss->static_eval;
    int prev_eval = (ss - 2)->static_eval;  // 2 plies ago (our last move)
    improving = current_eval > prev_eval;
}

// Reduce less when improving
if (improving) {
    reduction--;
}
```

When our position is getting better, we should be more careful about reducing.

## LMR at PV Nodes

Some engines reduce less at PV nodes:

```c
if (is_pv_node) {
    reduction = reduction * 2 / 3;  // Smaller reduction
}
```

PV nodes are important—we want accurate scores there.

## Performance Impact

LMR typically:
- Reduces nodes by 70-85%
- Enables searching 3-4 plies deeper
- Small strength loss compensated by depth gain

The occasional re-search when reduction fails is cheap compared to always searching full depth.

## Complete Implementation

```c
int search(Position* pos, int depth, int alpha, int beta,
           bool is_pv, SearchStack* ss) {

    if (depth <= 0) return quiesce(pos, alpha, beta);

    bool in_check = is_in_check(pos);
    int best_score = -INFINITY;
    Move best_move = MOVE_NONE;

    // Static evaluation for pruning decisions
    int static_eval = in_check ? -INFINITY : evaluate(pos);
    ss->static_eval = static_eval;

    bool improving = !in_check && ply >= 2 &&
                     static_eval > (ss - 2)->static_eval;

    MoveList moves;
    generate_moves(pos, &moves);
    score_moves(&moves, ss);

    int moves_searched = 0;

    for (int i = 0; i < moves.count; i++) {
        Move move = pick_best(&moves, i);

        make_move(pos, move);
        if (!is_legal(pos)) {
            unmake_move(pos, move);
            continue;
        }

        moves_searched++;
        bool gives_check = is_in_check(pos);

        int score;
        int new_depth = depth - 1;

        // LMR
        int reduction = 0;
        if (moves_searched > 3 &&
            depth >= 3 &&
            !in_check &&
            !is_capture(move) &&
            !is_promotion(move))
        {
            reduction = LMR_TABLE[depth][moves_searched];

            if (!is_pv) reduction++;
            if (!improving) reduction++;
            if (gives_check) reduction--;
            if (is_killer(move, ply)) reduction--;

            int hist = history_score(move, pos);
            reduction -= hist / 5000;

            reduction = clamp(reduction, 0, new_depth - 1);
        }

        // PVS with possible reduction
        if (moves_searched == 1) {
            score = -search(pos, new_depth, -beta, -alpha, is_pv, ss + 1);
        } else {
            score = -search(pos, new_depth - reduction, -alpha - 1, -alpha,
                           false, ss + 1);

            if (score > alpha && reduction > 0) {
                score = -search(pos, new_depth, -alpha - 1, -alpha,
                               false, ss + 1);
            }

            if (score > alpha && score < beta) {
                score = -search(pos, new_depth, -beta, -alpha, true, ss + 1);
            }
        }

        unmake_move(pos, move);

        if (score > best_score) {
            best_score = score;
            best_move = move;

            if (score > alpha) {
                alpha = score;
                if (score >= beta) {
                    // Update killers, history
                    if (!is_capture(move)) {
                        store_killer(ply, move);
                        update_history(move, depth, pos);
                    }
                    break;
                }
            }
        }
    }

    // ... mate/stalemate detection, TT store ...

    return best_score;
}
```

## Summary

Late Move Reductions:

- **Concept**: Moves searched late are probably bad—reduce their depth
- **Re-search**: If reduced search beats alpha, search again at full depth
- **Conditions**: Don't reduce first moves, checks, captures, promotions, killers
- **Adjustment**: Use history scores and improving detection
- **Formula**: Logarithmic in depth and move number
- **Impact**: 70-85% node reduction, 3-4 extra plies

LMR is perhaps the single most impactful modern search technique. Combined with good move ordering, it enables depths that would be impossible with pure alpha-beta.

Next: Additional pruning techniques that complement LMR.
