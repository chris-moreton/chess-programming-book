# Chapter 19: Principal Variation Search (PVS)

**Principal Variation Search** (also called NegaScout) is an enhancement to alpha-beta that assumes the first move is best. It searches the first move with a full window, then uses **null-window** searches for remaining moves. If a later move beats the first, it re-searches with a full window.

## The Core Insight

With good move ordering, the first move is usually best. If it is:
- We can verify remaining moves are worse using null-window search
- Null-window search is faster (more cutoffs)
- No re-search needed

If a later move is better (rare with good ordering):
- The null-window search tells us immediately
- We re-search with a full window
- Re-search cost is acceptable because it's rare

## Null-Window Search

A **null-window** or **zero-window** search uses `(alpha, alpha + 1)`:

```c
int score = -search(pos, depth - 1, -alpha - 1, -alpha);
```

This answers: "Is this move better than alpha?"
- If `score > alpha`: Yes, it's better (needs full re-search)
- If `score <= alpha`: No, it's worse or equal (done with this move)

Null-window searches are very fast because the tight window causes immediate cutoffs.

## PVS Algorithm

```c
int pvs(Position* pos, int depth, int alpha, int beta) {
    if (depth <= 0) return quiesce(pos, alpha, beta);

    MoveList moves;
    generate_moves(pos, &moves);
    score_moves(&moves);

    int best_score = -INFINITY;
    bool first_move = true;

    for (int i = 0; i < moves.count; i++) {
        Move move = pick_best(&moves, i);

        make_move(pos, move);

        int score;

        if (first_move) {
            // Full window search for first move
            score = -pvs(pos, depth - 1, -beta, -alpha);
            first_move = false;
        } else {
            // Null-window search for remaining moves
            score = -pvs(pos, depth - 1, -alpha - 1, -alpha);

            // If it beats alpha, re-search with full window
            if (score > alpha && score < beta) {
                score = -pvs(pos, depth - 1, -beta, -alpha);
            }
        }

        unmake_move(pos, move);

        if (score > best_score) {
            best_score = score;

            if (score > alpha) {
                alpha = score;

                if (score >= beta) {
                    return score;  // Beta cutoff
                }
            }
        }
    }

    return best_score;
}
```

## Why PVS Works

Consider searching 30 moves at depth 10:

**Without PVS (standard alpha-beta)**:
- All 30 moves searched with same window
- Each search does full work

**With PVS (assuming first move is best)**:
- Move 1: Full-window search (same as before)
- Moves 2-30: Null-window search each (much faster)
- No re-searches needed

**With PVS (if move 5 is actually best)**:
- Move 1: Full-window search (wasted but unavoidable)
- Moves 2-4: Null-window search (fast, confirm worse than move 1)
- Move 5: Null-window search (discovers it's better!)
- Move 5: Re-search with full window (to get exact score)
- Moves 6-30: Null-window search (fast)

The re-search on move 5 is extra work, but it's just one move. The savings from moves 6-30 far outweigh it.

## Combining PVS with LMR

PVS and LMR work together naturally:

```c
int search(Position* pos, int depth, int alpha, int beta, bool is_pv) {
    // ...

    for (int i = 0; i < moves.count; i++) {
        Move move = pick_best(&moves, i);
        make_move(pos, move);

        int score;
        int reduction = 0;

        // First move: full window, no reduction
        if (i == 0) {
            score = -search(pos, depth - 1, -beta, -alpha, true);
        }
        // PV nodes: null-window with possible reduction
        else if (is_pv) {
            // Calculate LMR reduction
            if (can_reduce(move, ...)) {
                reduction = lmr_reduction(depth, i);
            }

            // Null-window search with possible reduction
            score = -search(pos, depth - 1 - reduction, -alpha - 1, -alpha, false);

            // Re-search if needed
            if (score > alpha && reduction > 0) {
                score = -search(pos, depth - 1, -alpha - 1, -alpha, false);
            }
            if (score > alpha && score < beta) {
                score = -search(pos, depth - 1, -beta, -alpha, true);
            }
        }
        // Non-PV nodes: already using null-window from parent
        else {
            if (can_reduce(move, ...)) {
                reduction = lmr_reduction(depth, i);
            }

            score = -search(pos, depth - 1 - reduction, -beta, -alpha, false);

            if (score > alpha && reduction > 0) {
                score = -search(pos, depth - 1, -beta, -alpha, false);
            }
        }

        unmake_move(pos, move);
        // ... update best ...
    }
}
```

## PV Nodes vs Non-PV Nodes

PVS creates two types of nodes:

### PV Nodes

Nodes where we expect `alpha < score < beta`:
- First child searched with full window
- Other children with null-window
- We care about the exact score
- These form the Principal Variation

### Non-PV Nodes (Cut/All Nodes)

Nodes where we expect fail-high or fail-low:
- All children searched with null-window (inherited from parent)
- We only care if score beats a bound
- Don't need exact score

Track node type:

```c
int search(Position* pos, int depth, int alpha, int beta, bool is_pv) {
    // is_pv affects:
    // - Whether to do certain pruning
    // - Whether children are PV nodes
    // - PV collection
}
```

## PV Collection in PVS

Collect the PV using a triangular table:

```c
typedef struct {
    Move pv[MAX_PLY + 1];
    int pv_length;
} SearchStack;

int search(Position* pos, int depth, int alpha, int beta,
           bool is_pv, SearchStack* ss) {
    ss->pv_length = 0;

    // ... search ...

    for (int i = 0; i < moves.count; i++) {
        Move move = pick_best(&moves, i);
        make_move(pos, move);

        int score;
        // ... search call ...

        unmake_move(pos, move);

        if (score > alpha) {
            alpha = score;

            // Update PV
            ss->pv[0] = move;
            for (int j = 0; j < (ss + 1)->pv_length; j++) {
                ss->pv[j + 1] = (ss + 1)->pv[j];
            }
            ss->pv_length = (ss + 1)->pv_length + 1;

            if (score >= beta) break;
        }
    }

    return best_score;
}
```

## Complete Implementation

```c
int search(Position* pos, int depth, int alpha, int beta,
           bool is_pv, SearchStack* ss) {

    if (depth <= 0) {
        return quiesce(pos, alpha, beta);
    }

    // TT probe
    TTEntry* entry = tt_probe(pos->hash);
    // ... TT cutoff logic ...

    bool in_check = is_in_check(pos);

    // Pruning (skip on PV nodes for most pruning)
    if (!is_pv && !in_check) {
        // Null move, reverse futility, razoring...
    }

    MoveList moves;
    generate_moves(pos, &moves);
    score_moves(&moves, ss);

    int best_score = -INFINITY;
    Move best_move = MOVE_NONE;
    int moves_searched = 0;

    for (int i = 0; i < moves.count; i++) {
        Move move = pick_best(&moves, i);

        make_move(pos, move);
        bool gives_check = is_in_check(pos);

        // Extensions
        int extension = 0;
        if (gives_check) extension = 1;

        int new_depth = depth - 1 + extension;

        // LMR reduction
        int reduction = 0;
        if (moves_searched >= (is_pv ? 5 : 3) && depth >= 3 &&
            !in_check && !gives_check && !is_capture(move))
        {
            reduction = lmr_table[min(depth, 63)][min(moves_searched, 63)];
            if (!is_pv) reduction++;
        }

        int score;

        // PVS: first move full window, rest null-window
        if (moves_searched == 0) {
            score = -search(pos, new_depth, -beta, -alpha, is_pv, ss + 1);
        } else {
            // Null-window search with reduction
            score = -search(pos, new_depth - reduction, -alpha - 1, -alpha,
                           false, ss + 1);

            // Re-search if reduced search beats alpha
            if (score > alpha && reduction > 0) {
                score = -search(pos, new_depth, -alpha - 1, -alpha,
                               false, ss + 1);
            }

            // Full window re-search if beats alpha on PV node
            if (score > alpha && score < beta) {
                score = -search(pos, new_depth, -beta, -alpha, true, ss + 1);
            }
        }

        unmake_move(pos, move);
        moves_searched++;

        if (score > best_score) {
            best_score = score;
            best_move = move;

            if (score > alpha) {
                alpha = score;

                // Update PV
                if (is_pv) {
                    ss->pv[0] = move;
                    memcpy(ss->pv + 1, (ss + 1)->pv,
                           (ss + 1)->pv_length * sizeof(Move));
                    ss->pv_length = (ss + 1)->pv_length + 1;
                }

                if (score >= beta) {
                    // Update killers, history
                    if (!is_capture(move)) {
                        store_killer(ss->ply, move);
                        update_history(move, depth);
                    }
                    break;
                }
            }
        }
    }

    // Mate/stalemate detection
    if (best_score == -INFINITY) {
        if (in_check) {
            return -MATE_SCORE + ss->ply;
        }
        return 0;  // Stalemate
    }

    // TT store
    uint8_t flag = (best_score >= beta) ? TT_LOWER :
                   (best_score > original_alpha) ? TT_EXACT : TT_UPPER;
    tt_store(pos->hash, depth, best_score, best_move, flag);

    return best_score;
}
```

## Performance

PVS typically provides:
- 10-20% speedup over standard alpha-beta
- Better with good move ordering
- Minimal overhead (just comparison logic)

Combined with LMR and aspiration windows, PVS is standard in modern engines.

## Summary

Principal Variation Search:

- **First move**: Full-window search (establishes alpha)
- **Later moves**: Null-window search (verify they're worse)
- **Re-search**: Only if null-window suggests move is better
- **PV nodes**: Track principal variation, less aggressive pruning
- **Non-PV nodes**: Can prune more aggressively

PVS exploits move ordering: if the first move really is best, we save time on all other moves. The rare re-searches when we're wrong are acceptable overhead.

Part 3 complete. Next: Teaching the engine to evaluate positions.
