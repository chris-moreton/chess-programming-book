# Chapter 18: Aspiration Windows

**Aspiration windows** narrow the alpha-beta search window based on the previous iteration's score. A narrower window means more cutoffs—faster search. If the true score falls outside the window, we re-search with a wider window.

## The Basic Idea

Instead of searching with the full window `(-INFINITY, +INFINITY)`, use a window centered on the expected score:

```c
int prev_score = 25;  // From previous iteration
int window = 25;      // Aspiration window size

int alpha = prev_score - window;  // 0
int beta = prev_score + window;   // 50

int score = search(pos, depth, alpha, beta);
```

If the score is between 0 and 50, we're done. If it falls outside, re-search.

## Why Narrower Windows Help

Alpha-beta prunes more with narrow windows:
- Wider window: Need score > alpha to improve, score >= beta to cutoff
- Narrow window: Easier to hit beta (more cutoffs)

With a 50-centipawn window:
- Many positions cutoff immediately
- Only unclear positions need full search

## Handling Window Failures

### Fail Low (score ≤ alpha)

The position is worse than expected. Widen the window downward:

```c
if (score <= alpha) {
    // Fail low—re-search with lower alpha
    alpha = -INFINITY;  // Or alpha - larger_margin
    score = search(pos, depth, alpha, beta);
}
```

### Fail High (score >= beta)

The position is better than expected. Widen the window upward:

```c
if (score >= beta) {
    // Fail high—re-search with higher beta
    beta = +INFINITY;  // Or beta + larger_margin
    score = search(pos, depth, alpha, beta);
}
```

## Implementation

```c
Move iterative_deepening(Position* pos) {
    int prev_score = 0;
    Move best_move = MOVE_NONE;

    for (int depth = 1; depth <= MAX_DEPTH; depth++) {
        int alpha, beta;

        if (depth <= 4) {
            // Full window for shallow depths
            alpha = -INFINITY;
            beta = +INFINITY;
        } else {
            // Aspiration window for deeper searches
            alpha = prev_score - ASPIRATION_WINDOW;
            beta = prev_score + ASPIRATION_WINDOW;
        }

        int score;

        while (true) {
            score = search(pos, depth, alpha, beta);

            if (score <= alpha) {
                // Fail low—widen downward
                beta = (alpha + beta) / 2;
                alpha = max(alpha - ASPIRATION_WINDOW * 4, -INFINITY);
            } else if (score >= beta) {
                // Fail high—widen upward
                beta = min(beta + ASPIRATION_WINDOW * 4, +INFINITY);
            } else {
                // Score is within window—done
                break;
            }
        }

        prev_score = score;
        best_move = tt_probe_move(pos->hash);

        if (should_stop()) break;
    }

    return best_move;
}
```

## Window Sizes

Common window sizes:
- Initial: 15-35 centipawns
- After fail: 2x-4x larger
- Final: Full window if needed

```c
#define ASPIRATION_WINDOW 25

int delta = ASPIRATION_WINDOW;
int alpha = prev_score - delta;
int beta = prev_score + delta;

while (true) {
    score = search(pos, depth, alpha, beta);

    if (score <= alpha) {
        beta = (alpha + beta) / 2;
        alpha = score - delta;
        delta *= 2;  // Exponential widening
    } else if (score >= beta) {
        beta = score + delta;
        delta *= 2;
    } else {
        break;
    }

    if (delta > 500) {
        // Give up and use full window
        alpha = -INFINITY;
        beta = +INFINITY;
    }
}
```

## When Aspiration Fails Often

If aspiration fails frequently:
- Position is tactically complex
- Previous iteration's score was unstable
- Consider using wider initial windows

Track failure rate:

```c
if (aspiration_failures > 2) {
    // Use full window next iteration
    use_aspiration = false;
}
```

## Aspiration and Time Management

Fail-high or fail-low may indicate unstable positions:

```c
if (score <= alpha && prev_score > alpha) {
    // Score dropped significantly—allocate more time
    time_limit *= 1.5;
}

if (score >= beta && best_move_changed) {
    // Found something good—allocate more time to verify
    time_limit *= 1.25;
}
```

## Implementation Details

### Always Store TT Results

Even on aspiration failure, store the result:

```c
score = search(pos, depth, alpha, beta);

// Store in TT (with appropriate bound type)
if (score <= alpha) {
    tt_store(pos->hash, depth, score, MOVE_NONE, TT_UPPER);
} else if (score >= beta) {
    tt_store(pos->hash, depth, score, best_move, TT_LOWER);
}
```

### PV Recovery After Failure

After aspiration failure, the PV might be incomplete. Extract from TT:

```c
// After successful search
extract_pv_from_tt(pos, &pv);
```

## Complete Example

```c
#define ASP_WINDOW 25
#define ASP_MAX_DELTA 500

int aspiration_search(Position* pos, int depth, int prev_score) {
    if (depth <= 4 || is_mate_score(prev_score)) {
        return search(pos, depth, -INFINITY, +INFINITY);
    }

    int delta = ASP_WINDOW;
    int alpha = max(prev_score - delta, -INFINITY);
    int beta = min(prev_score + delta, +INFINITY);

    while (true) {
        int score = search(pos, depth, alpha, beta);

        if (score > alpha && score < beta) {
            return score;  // Within window
        }

        if (score <= alpha) {
            beta = (alpha + beta) / 2;
            alpha = max(score - delta, -INFINITY);
        } else {
            beta = min(score + delta, +INFINITY);
        }

        delta += delta / 2;

        if (delta > ASP_MAX_DELTA) {
            return search(pos, depth, -INFINITY, +INFINITY);
        }
    }
}
```

## Summary

Aspiration windows:

- **Concept**: Narrow window centered on expected score
- **Benefit**: More cutoffs, faster search (10-20% speedup typical)
- **Cost**: May require re-searches on failure
- **Window size**: Start ~25cp, widen exponentially on failure
- **Skip**: For shallow depths, mate scores, highly tactical positions

Combined with iterative deepening (which provides the previous score), aspiration windows are a significant optimization with minimal risk.

Next: Principal Variation Search—combining narrow windows with move ordering insights.
