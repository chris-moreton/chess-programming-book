# Chapter 14: Null Move Pruning

**Null move pruning** is a powerful technique that dramatically reduces search time. The idea: if we can skip our turn (make a "null move") and still achieve a good score, our position is probably winning, and we can prune.

## The Core Idea

In chess, having the move is almost always an advantage. If we could pass our turn and the opponent still can't improve their position enough, we must be doing well.

```
Position evaluation after null move:
- If opponent's best is still worse than beta
- Then our position is strong enough to prune
- No need to search actual moves
```

## Basic Implementation

```c
int alpha_beta(Position* pos, int depth, int alpha, int beta) {
    // ... early exits, TT probe ...

    bool in_check = is_in_check(pos);

    // Null move pruning
    if (!in_check &&
        depth >= 3 &&
        !is_pv_node &&
        has_non_pawn_material(pos))
    {
        int R = 3;  // Reduction depth

        make_null_move(pos);
        int score = -alpha_beta(pos, depth - 1 - R, -beta, -beta + 1);
        unmake_null_move(pos);

        if (score >= beta) {
            return beta;  // Null move cutoff
        }
    }

    // ... normal search ...
}
```

## Why It Works

### The Null Move Observation

If position P is good for us:
- Even giving opponent a free move shouldn't change that
- Searching with reduced depth confirms our advantage
- If we still beat beta, the position is clearly strong

### Risk: Zugzwang

**Zugzwang** is when being forced to move is a disadvantage. In zugzwang positions, passing would be optimal—but we can't pass in chess.

Null move pruning is unsafe in zugzwang because:
- We assume passing hurts us
- In zugzwang, passing would help us
- The null move search gets the wrong answer

Example: King and pawn endgame where any king move loses.

## Safety Conditions

### Don't Use When in Check

If we're in check, we must escape—can't pass:

```c
if (in_check) {
    // Skip null move
}
```

### Require Non-Pawn Material

Zugzwang rarely occurs with significant material:

```c
if (!has_non_pawn_material(pos, pos->side_to_move)) {
    // Skip null move—might be zugzwang
}

bool has_non_pawn_material(Position* pos, int color) {
    return pos->pieces[color][KNIGHT] ||
           pos->pieces[color][BISHOP] ||
           pos->pieces[color][ROOK] ||
           pos->pieces[color][QUEEN];
}
```

### Minimum Depth

Don't use at shallow depths:

```c
if (depth < 3) {
    // Skip null move—not enough benefit
}
```

### Not on PV Nodes

PV nodes should be searched thoroughly:

```c
if (is_pv_node) {
    // Skip null move—we want the full PV
}
```

## Reduction Depth (R)

The reduction R determines how much shallower we search the null move:

- R = 2: Conservative, fewer cutoffs
- R = 3: Standard
- R = 4+: Aggressive, more risk

### Dynamic R

Increase R with depth:

```c
int R = 3 + depth / 6;

// Or based on evaluation
int R = 3;
if (evaluation > 200) R = 4;  // Big advantage—can reduce more
```

## Verification Search

To reduce zugzwang errors, some engines add a **verification search**:

```c
if (score >= beta) {
    // Verify with reduced-depth normal search
    int verify_score = alpha_beta(pos, depth - R - 1, beta - 1, beta);

    if (verify_score >= beta) {
        return beta;  // Confirmed
    }
    // Verification failed—don't prune
}
```

This catches some zugzwang cases at the cost of extra search.

## Recursive Null Move

Prevent two consecutive null moves (nonsensical):

```c
int alpha_beta(Position* pos, int depth, int alpha, int beta, bool null_ok) {
    // ...

    if (null_ok && !in_check && depth >= 3) {
        make_null_move(pos);
        // Pass false to prevent recursive null move
        int score = -alpha_beta(pos, depth - 1 - R, -beta, -beta + 1, false);
        unmake_null_move(pos);

        if (score >= beta) return beta;
    }

    // Normal search passes true
    for (moves) {
        make_move(pos, move);
        score = -alpha_beta(pos, depth - 1, -beta, -alpha, true);
        unmake_move(pos, move);
    }
}
```

## Implementing Null Move

The null move doesn't move any pieces—just flips the side to move:

```c
void make_null_move(Position* pos) {
    pos->side_to_move ^= 1;
    pos->hash ^= ZOBRIST_SIDE;

    // Clear en passant
    if (pos->en_passant_square != NO_SQUARE) {
        pos->hash ^= ZOBRIST_EP[file_of(pos->en_passant_square)];
        pos->en_passant_square = NO_SQUARE;
    }

    pos->ply++;
}

void unmake_null_move(Position* pos) {
    pos->side_to_move ^= 1;
    pos->hash ^= ZOBRIST_SIDE;
    pos->ply--;
    // Note: en_passant restored from search stack, not here
}
```

## Performance Impact

Null move pruning typically:
- Reduces nodes by 50-80%
- Speeds up search by 2-4x
- Minimal playing strength loss

The occasional zugzwang error is far outweighed by the depth gained.

## Threat Detection

The null move search can reveal opponent threats:

```c
make_null_move(pos);
int null_score = -alpha_beta(pos, depth - 1 - R, -beta, -beta + 1);
Move threat = tt_probe_move(pos->hash);  // Get opponent's best
unmake_null_move(pos);

if (null_score < beta) {
    // Opponent has a strong threat; might want to extend
    // Store 'threat' for analysis
}
```

Some engines use this for singular extensions (Chapter 17).

## Full Implementation

```c
int search(Position* pos, int depth, int alpha, int beta,
           bool is_pv, bool null_ok, SearchStack* ss) {

    if (depth <= 0) {
        return quiesce(pos, alpha, beta);
    }

    bool in_check = is_in_check(pos);

    // Null move pruning
    if (null_ok &&
        !in_check &&
        !is_pv &&
        depth >= 3 &&
        has_non_pawn_material(pos, pos->side_to_move) &&
        evaluate(pos) >= beta)  // Static eval condition
    {
        int R = 3 + depth / 6;
        R = min(R, depth - 1);  // Don't reduce below 1

        make_null_move(pos);
        int score = -search(pos, depth - 1 - R, -beta, -beta + 1,
                           false, false, ss + 1);  // No recursive null
        unmake_null_move(pos);

        if (score >= beta) {
            // Don't return mate scores from null move search
            if (score >= MATE_THRESHOLD) {
                score = beta;
            }
            return score;
        }
    }

    // ... rest of search ...
}
```

## Common Bugs

### Returning Mate Scores

```c
// WRONG
if (score >= beta) return score;  // Might return -MATE

// CORRECT
if (score >= beta) {
    if (is_mate_score(score)) score = beta;
    return score;
}
```

Mate scores from null move search can be wrong since the position isn't real.

### Not Restoring En Passant

```c
// WRONG: Just flip side
void unmake_null_move(Position* pos) {
    pos->side_to_move ^= 1;  // Lost en_passant info!
}

// CORRECT: Restore from saved state
void unmake_null_move(Position* pos, NullMoveInfo* info) {
    pos->side_to_move ^= 1;
    pos->en_passant_square = info->en_passant;
    pos->hash = info->hash;
}
```

### Using in Quiescence

Don't use null move in quiescence search—it's too shallow and adds overhead.

## Summary

Null move pruning:

- **Concept**: If passing still beats beta, position is strong enough to prune
- **Reduction**: Search with R = 3-4 less depth
- **Safety**: Not in check, not PV, have non-pawn material
- **Zugzwang risk**: Minimized by conditions; occasional errors acceptable
- **Impact**: 2-4x speedup typical

Null move pruning is one of the most effective pruning techniques in chess programming. The occasional zugzwang mistake is vastly outweighed by the extra depth gained.

Next: Late move reductions—another powerful pruning technique based on move ordering.
