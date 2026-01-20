# Chapter 9: Alpha-Beta Pruning

Alpha-beta pruning is the most important enhancement to minimax. It proves that many branches cannot affect the final decision and skips them entirely. With good move ordering, alpha-beta searches the same tree with the same result as minimax, but examines far fewer nodes.

## The Key Insight

Consider this scenario during search:

1. We search move A and find it leads to a score of 5
2. We start searching move B
3. Within B's subtree, we find the opponent has a response that guarantees a score of 3

Should we continue searching B? No. We already have a move (A) that scores 5. Move B can score at most 3 (the opponent will ensure this). Move A is better regardless of B's other variations.

This is **pruning**: cutting off branches we can prove are useless.

## Alpha and Beta Bounds

Alpha-beta tracks two values:

- **Alpha**: The best score the maximizer can guarantee so far
- **Beta**: The best score the minimizer can guarantee so far

During search:
- The maximizer improves alpha (raises the floor)
- The minimizer improves beta (lowers the ceiling)
- If alpha ≥ beta, we **cut off**—no further search needed

### Why Cutoff at Alpha ≥ Beta?

If alpha ≥ beta, it means:
- The maximizer is guaranteed at least alpha
- The minimizer is guaranteed at most beta
- But alpha ≥ beta means the maximizer's guarantee exceeds the minimizer's guarantee

This is contradictory—the minimizer would never let this position be reached. So we can stop searching.

## Alpha-Beta Algorithm

```c
int alpha_beta(Position* pos, int depth, int alpha, int beta) {
    // Terminal conditions
    if (depth == 0) {
        return evaluate(pos);
    }

    MoveList moves;
    generate_moves(pos, &moves);

    if (moves.count == 0) {
        if (is_in_check(pos)) {
            return -MATE_SCORE + ply;  // Checkmate
        }
        return 0;  // Stalemate
    }

    for (int i = 0; i < moves.count; i++) {
        make_move(pos, moves.moves[i]);
        int score = -alpha_beta(pos, depth - 1, -beta, -alpha);
        unmake_move(pos, moves.moves[i]);

        if (score >= beta) {
            return beta;  // Beta cutoff (fail-high)
        }

        if (score > alpha) {
            alpha = score;  // New best move
        }
    }

    return alpha;
}

// Initial call from root
int score = alpha_beta(pos, depth, -INFINITY, +INFINITY);
```

### The Negamax Transformation

Notice how we pass `-beta, -alpha` when recursing. In negamax:
- Our alpha becomes the opponent's beta (negated)
- Our beta becomes the opponent's alpha (negated)

The negation flips everything correctly.

## A Worked Example

Consider this tree (maximizer at root):

```
                Root (to maximize)
               /         \
              A           B
            / | \       / | \
           3  5  2     4  1  ?
```

Search without pruning evaluates all 6 leaves.

Search with alpha-beta:

1. **Start at root**: alpha = -∞, beta = +∞
2. **Search A**:
   - At A (minimizer), alpha = -∞, beta = +∞
   - Evaluate 3: update beta to 3
   - Evaluate 5: 5 > beta? No. Keep beta = 3
   - Evaluate 2: 2 < 3, update beta to 2
   - A returns 2
3. **Back at root**:
   - Score 2 > alpha? Yes. alpha = 2
4. **Search B**:
   - At B (minimizer), alpha = -2, beta = -∞ wait... In negamax, we pass `-beta, -alpha` = `-∞, -2`
   - Actually in negamax form, at B: alpha = -∞, beta = 2 (after negation flip)
   - Evaluate 4: 4 ≥ beta (2)? Yes! **Beta cutoff**
   - We don't need to evaluate positions 1 and ?

We examined 4 leaves instead of 6. The savings grow dramatically in larger trees.

## Types of Nodes

Alpha-beta creates three types of nodes:

### PV Nodes (Principal Variation)

Nodes where we find a new best move (score improves alpha and is less than beta).

```
alpha < score < beta
```

These lie on the "principal variation"—the expected best play for both sides.

### Cut Nodes (Fail-High)

Nodes where beta cutoff occurs. The opponent has a refutation, so we prune.

```
score >= beta
```

### All Nodes (Fail-Low)

Nodes where no move improves alpha. Every move is worse than what we already have.

```
score <= alpha for all moves
```

Understanding node types helps with move ordering and transposition table design.

## Move Ordering Matters

Alpha-beta's effectiveness depends critically on the order we try moves.

### Best Case

If we always try the best move first, alpha-beta examines only O(b^(d/2)) nodes instead of O(b^d). This effectively reduces the branching factor from b to √b—from 35 to about 6!

### Worst Case

If we always try the worst move first, alpha-beta examines the same O(b^d) nodes as minimax. No pruning occurs.

### Practical Move Ordering

Good move ordering strategies (detailed in Chapter 12):

1. **Hash move**: Move from transposition table
2. **Captures**: Especially winning captures
3. **Killer moves**: Moves that caused cutoffs at this depth
4. **History heuristic**: Moves that historically score well
5. **Piece-square improvements**: Moves to better squares

With good ordering, real engines achieve effective branching factors of 3-6.

## Fail-Hard vs Fail-Soft

The algorithm above is **fail-hard**: it returns exactly alpha or beta on bounds failures.

**Fail-soft** returns the actual score even outside the window:

```c
int alpha_beta_fail_soft(Position* pos, int depth, int alpha, int beta) {
    if (depth == 0) return evaluate(pos);

    int best_score = -INFINITY;

    MoveList moves;
    generate_moves(pos, &moves);

    for (int i = 0; i < moves.count; i++) {
        make_move(pos, moves.moves[i]);
        int score = -alpha_beta_fail_soft(pos, depth - 1, -beta, -alpha);
        unmake_move(pos, moves.moves[i]);

        if (score > best_score) {
            best_score = score;

            if (score > alpha) {
                alpha = score;
            }
        }

        if (score >= beta) {
            return best_score;  // Return actual score, not beta
        }
    }

    return best_score;  // May be <= original alpha
}
```

### Fail-Soft Advantages

- Returns more accurate scores for transposition table storage
- Slightly better move ordering information

### Fail-Hard Advantages

- Simpler implementation
- Slightly faster (fewer comparisons)

Most modern engines use fail-soft or a hybrid.

## Tracking the Best Move

At the root, we need the actual best move, not just its score:

```c
Move root_search(Position* pos, int depth) {
    MoveList moves;
    generate_moves(pos, &moves);

    Move best_move = moves.moves[0];
    int alpha = -INFINITY;
    int beta = +INFINITY;

    for (int i = 0; i < moves.count; i++) {
        make_move(pos, moves.moves[i]);
        int score = -alpha_beta(pos, depth - 1, -beta, -alpha);
        unmake_move(pos, moves.moves[i]);

        if (score > alpha) {
            alpha = score;
            best_move = moves.moves[i];
        }
    }

    return best_move;
}
```

## Collecting the Principal Variation

The PV is the sequence of best moves from root to leaf:

```c
void alpha_beta_pv(Position* pos, int depth, int alpha, int beta, PVLine* pv) {
    PVLine child_pv;
    pv->length = 0;

    if (depth == 0) {
        return;  // No PV at leaf
    }

    MoveList moves;
    generate_moves(pos, &moves);

    for (int i = 0; i < moves.count; i++) {
        make_move(pos, moves.moves[i]);
        int score = -alpha_beta_pv(pos, depth - 1, -beta, -alpha, &child_pv);
        unmake_move(pos, moves.moves[i]);

        if (score >= beta) {
            return;  // Beta cutoff - PV not reliable here
        }

        if (score > alpha) {
            alpha = score;

            // Update PV: current move + child's PV
            pv->moves[0] = moves.moves[i];
            for (int j = 0; j < child_pv.length; j++) {
                pv->moves[j + 1] = child_pv.moves[j];
            }
            pv->length = child_pv.length + 1;
        }
    }
}
```

## Search Windows

The initial call uses the full window `(-INFINITY, +INFINITY)`. But we can narrow the window for efficiency (see Chapter 18: Aspiration Windows).

### Zero-Window Search

A **zero-window** or **null-window** search uses `(alpha, alpha + 1)`:

```c
int score = -alpha_beta(pos, depth - 1, -alpha - 1, -alpha);
```

This answers a yes/no question: "Is this move better than alpha?"

- If score > alpha: Yes, it's better (fail-high)
- If score ≤ alpha: No, it's worse or equal (fail-low)

Zero-window searches are faster because the narrow window maximizes cutoffs. Principal Variation Search (Chapter 19) uses this extensively.

## Theoretical Speedup

### Best Case (Perfect Ordering)

With perfect move ordering:
- Odd depths: (b^((d+1)/2)) + (b^((d-1)/2)) - 1 nodes
- Even depths: 2 × b^(d/2) - 1 nodes

For d=6, b=35:
- Minimax: 35^6 ≈ 1.8 billion
- Alpha-beta (perfect): 2 × 35^3 - 1 ≈ 85,750

That's a 21,000x speedup!

### Practical Performance

In practice, move ordering isn't perfect. Real engines examine roughly b^(d×0.7) nodes—still dramatically better than minimax.

With b=35, d=6:
- Theoretical: 35^6 = 1.8B
- Alpha-beta (good ordering): 35^4.2 ≈ 3 million
- Speedup: ~600x

## Common Bugs

### Forgetting to Negate

```c
// WRONG
int score = alpha_beta(pos, depth - 1, -beta, -alpha);

// CORRECT
int score = -alpha_beta(pos, depth - 1, -beta, -alpha);
```

The negation makes scores side-relative.

### Swapping Alpha and Beta

```c
// WRONG
int score = -alpha_beta(pos, depth - 1, -alpha, -beta);

// CORRECT
int score = -alpha_beta(pos, depth - 1, -beta, -alpha);
```

Our beta becomes their alpha, our alpha becomes their beta.

### Returning Wrong Bound

```c
// Fail-hard should return beta on cutoff, not score
if (score >= beta) {
    return beta;  // Correct for fail-hard
    // return score;  // This is fail-soft
}
```

## Summary

Alpha-beta pruning:

- **Maintains bounds**: alpha (floor) and beta (ceiling)
- **Prunes when alpha ≥ beta**: Opponent would avoid this line
- **Best-case speedup**: b^d → b^(d/2), effectively taking the square root of node count
- **Requires good move ordering**: Bad ordering loses most benefit
- **Node types**: PV (improved alpha), Cut (beta cutoff), All (no improvement)

With alpha-beta, searching 6 ply in a second becomes feasible. Combined with techniques in upcoming chapters, modern engines search 20+ ply.

Next: How to search progressively deeper and manage time with iterative deepening.
