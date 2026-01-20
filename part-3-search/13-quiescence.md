# Chapter 13: Quiescence Search

Standard alpha-beta search stops at a fixed depth and evaluates. But what if that position is in the middle of a capture sequence? The evaluation might wildly misjudge the true position.

**Quiescence search** extends the search until the position is "quiet"—no captures or forcing moves pending. This eliminates the worst forms of the horizon effect.

## The Horizon Effect

Consider searching to depth 6:

```
Depth 6: White to move, material equal
         White can capture black's queen
         But black has Qxh2# (checkmate)

Static evaluation: +900 (white up a queen!)
Actual value: -MATE (black wins)
```

The problem: we stopped searching just as disaster struck. The evaluation is blind to what happens next.

### Without Quiescence

The engine sees: "I can take the queen, position is great!"
Reality: The opponent checkmates next move.

### With Quiescence

Quiescence search continues: "After QxQ, black plays Qxh2+, then..."
Now the engine sees the checkmate and avoids the blunder.

## Basic Quiescence Search

```c
int quiesce(Position* pos, int alpha, int beta) {
    // Stand pat: evaluate current position
    int stand_pat = evaluate(pos);

    // If we're already good enough, return
    if (stand_pat >= beta) {
        return beta;  // Fail high
    }

    // Update alpha if standing pat is better than current best
    if (stand_pat > alpha) {
        alpha = stand_pat;
    }

    // Generate and search captures only
    MoveList captures;
    generate_captures(pos, &captures);
    score_captures(&captures);

    for (int i = 0; i < captures.count; i++) {
        Move move = pick_best(&captures, i);

        make_move(pos, move);
        int score = -quiesce(pos, -beta, -alpha);
        unmake_move(pos, move);

        if (score >= beta) {
            return beta;  // Cutoff
        }

        if (score > alpha) {
            alpha = score;
        }
    }

    return alpha;
}
```

### Integration with Main Search

```c
int alpha_beta(Position* pos, int depth, int alpha, int beta) {
    // At depth 0, enter quiescence
    if (depth <= 0) {
        return quiesce(pos, alpha, beta);
    }

    // ... normal alpha-beta search ...
}
```

## Stand Pat

The **stand pat** score is the static evaluation at the start of quiescence. It represents "I don't have to capture anything."

```c
int stand_pat = evaluate(pos);
if (stand_pat >= beta) return beta;
```

If standing pat already beats beta, the position is good enough without any captures—we can cut off immediately.

This is sound because:
- Captures are optional (we can just make a quiet move)
- If the current position is already winning, we don't need to prove it further

## Delta Pruning

**Delta pruning** skips captures that can't possibly improve alpha, even with the best case capture:

```c
int DELTA_MARGIN = 200;  // About 2 pawns

int quiesce(Position* pos, int alpha, int beta) {
    int stand_pat = evaluate(pos);

    if (stand_pat >= beta) return beta;
    if (stand_pat > alpha) alpha = stand_pat;

    // Delta pruning: if even capturing queen doesn't help, prune
    if (stand_pat + QUEEN_VALUE + DELTA_MARGIN < alpha) {
        return alpha;  // Hopeless position
    }

    MoveList captures;
    generate_captures(pos, &captures);

    for (int i = 0; i < captures.count; i++) {
        Move move = pick_best(&captures, i);

        // Per-move delta pruning
        int captured_value = piece_value(captured_piece(move));
        if (stand_pat + captured_value + DELTA_MARGIN < alpha) {
            continue;  // This capture can't raise alpha
        }

        make_move(pos, move);
        int score = -quiesce(pos, -beta, -alpha);
        unmake_move(pos, move);

        if (score >= beta) return beta;
        if (score > alpha) alpha = score;
    }

    return alpha;
}
```

Delta pruning is safe because:
- We're comparing against the best possible outcome (capturing their best piece)
- If that's not enough, no capture will help

Exception: Don't delta prune in endgames where promotions matter significantly.

## SEE Pruning

Skip captures with negative SEE (losing captures):

```c
for (int i = 0; i < captures.count; i++) {
    Move move = captures.moves[i];

    // Skip losing captures
    if (see(pos, move) < 0) {
        continue;
    }

    make_move(pos, move);
    int score = -quiesce(pos, -beta, -alpha);
    unmake_move(pos, move);

    // ...
}
```

This dramatically reduces the number of captures searched.

## Quiescence Depth Limit

Without limits, quiescence can explode in tactical positions. Add a depth limit:

```c
int quiesce(Position* pos, int alpha, int beta, int qs_depth) {
    if (qs_depth <= 0) {
        return evaluate(pos);  // Force evaluation
    }

    int stand_pat = evaluate(pos);
    if (stand_pat >= beta) return beta;
    if (stand_pat > alpha) alpha = stand_pat;

    MoveList captures;
    generate_captures(pos, &captures);

    for (...) {
        make_move(pos, move);
        int score = -quiesce(pos, -beta, -alpha, qs_depth - 1);
        unmake_move(pos, move);
        // ...
    }

    return alpha;
}
```

Typical limit: 6-10 plies of quiescence.

## Checks in Quiescence

Some engines also search checks in quiescence:

```c
int quiesce(Position* pos, int alpha, int beta, int qs_depth) {
    bool in_check = is_in_check(pos);

    // If in check, can't stand pat—must escape
    if (!in_check) {
        int stand_pat = evaluate(pos);
        if (stand_pat >= beta) return beta;
        if (stand_pat > alpha) alpha = stand_pat;
    }

    MoveList moves;
    if (in_check) {
        generate_evasions(pos, &moves);  // All legal moves
    } else {
        generate_captures(pos, &moves);
        if (qs_depth > 0) {
            generate_checks(pos, &moves);  // Add checking moves
        }
    }

    // ...
}
```

Searching checks helps find tactical shots but increases tree size. Most engines limit check extensions to the first few qs plies.

## Handling Checkmate and Stalemate

In quiescence, we might be in check with no legal captures:

```c
int quiesce(Position* pos, int alpha, int beta) {
    bool in_check = is_in_check(pos);

    if (!in_check) {
        int stand_pat = evaluate(pos);
        if (stand_pat >= beta) return beta;
        if (stand_pat > alpha) alpha = stand_pat;
    }

    MoveList moves;
    if (in_check) {
        generate_all_moves(pos, &moves);  // Need all moves to escape
        if (moves.count == 0) {
            return -MATE_SCORE + ply;  // Checkmate
        }
    } else {
        generate_captures(pos, &moves);
        // If no captures and not in check, stand pat is our score
    }

    // Search moves...

    if (in_check && !found_legal_move) {
        return -MATE_SCORE + ply;
    }

    return alpha;
}
```

## Performance Impact

Quiescence search typically examines 50-80% of total nodes:

```
Total nodes: 10,000,000
Main search: 3,000,000 (30%)
Quiescence:  7,000,000 (70%)
```

This might seem inefficient, but without quiescence:
- The main search would need to go much deeper
- Evaluation errors would cause massive tactical mistakes

## Debugging Quiescence

Common bugs:

### Not Generating All Evasions When in Check

```c
// WRONG: Only generating captures while in check
if (in_check) {
    generate_captures(pos, &moves);  // Might miss non-capture escapes!
}

// CORRECT:
if (in_check) {
    generate_all_moves(pos, &moves);  // All legal evasions
}
```

### Wrong Mate Score

```c
// WRONG: Returning 0 when no moves
if (moves.count == 0) return 0;

// CORRECT: Check if we're in check
if (moves.count == 0 && in_check) {
    return -MATE_SCORE + ply;  // Checkmate
}
// If not in check and no captures, stand pat is correct
```

### Infinite Recursion

Quiescence can recurse indefinitely on perpetual captures:

```c
// Add depth limit
if (qs_depth <= 0) {
    return evaluate(pos);
}
```

## Full Implementation

```c
int quiesce(Position* pos, int alpha, int beta, int qs_depth, int ply) {
    nodes_searched++;

    if (qs_depth <= 0) {
        return evaluate(pos);
    }

    bool in_check = is_in_check(pos);
    int best_score;

    if (in_check) {
        best_score = -INFINITY;  // Must find escape
    } else {
        best_score = evaluate(pos);
        if (best_score >= beta) {
            return best_score;
        }
        if (best_score > alpha) {
            alpha = best_score;
        }

        // Delta pruning
        int big_delta = QUEEN_VALUE + 200;
        if (best_score + big_delta < alpha) {
            return alpha;
        }
    }

    MoveList moves;
    if (in_check) {
        generate_evasions(pos, &moves);
    } else {
        generate_captures(pos, &moves);
        // Optionally: generate_checks for first ply
    }

    score_moves_qs(&moves);

    int legal_moves = 0;

    for (int i = 0; i < moves.count; i++) {
        Move move = pick_best(&moves, i);

        // SEE pruning (not in check)
        if (!in_check && see(pos, move) < 0) {
            continue;
        }

        // Delta pruning per move
        if (!in_check && !is_promotion(move)) {
            int gain = piece_value(captured_piece(pos, move));
            if (best_score + gain + 200 < alpha) {
                continue;
            }
        }

        make_move(pos, move);

        if (!is_legal(pos)) {  // Pseudo-legal check
            unmake_move(pos, move);
            continue;
        }

        legal_moves++;
        int score = -quiesce(pos, -beta, -alpha, qs_depth - 1, ply + 1);
        unmake_move(pos, move);

        if (score > best_score) {
            best_score = score;
            if (score > alpha) {
                alpha = score;
                if (score >= beta) {
                    return score;
                }
            }
        }
    }

    if (in_check && legal_moves == 0) {
        return -MATE_SCORE + ply;
    }

    return best_score;
}
```

## Summary

Quiescence search:

- **Extends search until quiet**: No forcing moves pending
- **Stand pat**: Option to not capture—evaluation as baseline
- **Searches captures only** (plus evasions if in check)
- **Delta pruning**: Skip hopeless positions
- **SEE pruning**: Skip losing captures
- **Depth limited**: Prevent explosion in tactical positions
- **Handles check**: Must search all evasions, detect mate

Quiescence typically uses 50-80% of search time but prevents catastrophic tactical oversights. It's not optional for a practical chess engine.

Next: Null move pruning—a powerful technique for refuting weak positions.
