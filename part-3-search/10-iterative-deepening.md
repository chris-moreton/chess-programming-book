# Chapter 10: Iterative Deepening

Iterative deepening searches progressively deeper: first depth 1, then depth 2, then depth 3, and so on. It seems wasteful—why search shallow depths when we want deep analysis? The answer: it's actually more efficient, and it solves critical practical problems.

## Why Search Multiple Times?

### Problem 1: Time Management

In a real game, we have limited time. How do we know when to stop searching?

Without iterative deepening:
- We start a depth-8 search
- It takes too long
- We have no result when time expires

With iterative deepening:
- We complete depth 6 in 2 seconds, save the result
- We start depth 7, and time runs out
- We use the depth-6 result—always have a valid move

### Problem 2: Move Ordering

Alpha-beta needs good move ordering. Where does ordering information come from?

- **Previous iteration**: The best move from depth (d-1) is likely best at depth d
- **Transposition table**: Filled during shallower searches

Without iterative deepening, we have no ordering information for the first search.

### Problem 3: Unpredictable Search Time

Search time varies wildly between positions:
- Quiet positions: Many cutoffs, fast search
- Tactical positions: Few cutoffs, slow search

Iterative deepening adapts naturally. We search until time runs out, regardless of how deep we got.

## The Algorithm

```c
Move iterative_deepening(Position* pos, int max_time_ms) {
    Move best_move = MOVE_NONE;
    int best_score = -INFINITY;
    Timer timer;
    timer_start(&timer);

    for (int depth = 1; depth <= MAX_DEPTH; depth++) {
        // Search at current depth
        int score = alpha_beta(pos, depth, -INFINITY, +INFINITY, &best_move);

        // Check time after each depth
        if (timer_elapsed_ms(&timer) > max_time_ms) {
            break;  // Use result from previous completed depth
        }

        best_score = score;
        printf("info depth %d score cp %d pv %s\n",
               depth, score, move_to_uci(best_move));
    }

    return best_move;
}
```

### Output During Search

Engines output "info" lines during search:

```
info depth 1 score cp 35 pv e2e4
info depth 2 score cp 0 pv e2e4 e7e5
info depth 3 score cp 30 pv e2e4 e7e5 g1f3
info depth 4 score cp 5 pv e2e4 e7e5 g1f3 b8c6
...
```

This shows progress and helps with debugging.

## Isn't It Wasteful?

Searching depth 1, then 2, then 3... seems to repeat work. But:

### Most Work Is at the Deepest Level

In a tree with branching factor b:
- Depth d has b^d leaf nodes
- Depths 1 through d-1 have b^(d-1) + b^(d-2) + ... + b^1 ≈ b^(d-1) × (b/(b-1))

For b=35:
- Depth 10 alone: 35^10 nodes
- Depths 1-9 combined: about 35^9 × 1.03 nodes

The shallower iterations add only ~3% overhead!

### Better Move Ordering Compensates

The ordering information from shallow iterations makes deeper searches much faster. The speedup from better ordering typically exceeds the overhead of repeated shallow searches.

### Empirical Reality

In practice, iterative deepening is faster than searching directly to the target depth, because:
1. Move ordering is dramatically better
2. Transposition table is populated
3. We can use aspiration windows (Chapter 18)

## Using Previous Iteration's PV

The best move from depth (d-1) should be searched first at depth d:

```c
Move iterative_deepening(Position* pos, SearchInfo* info) {
    Move best_move = MOVE_NONE;
    Move pv_move = MOVE_NONE;  // PV from previous iteration

    for (int depth = 1; depth <= MAX_DEPTH; depth++) {
        info->pv_move = pv_move;  // Make available to move ordering

        int score = alpha_beta(pos, depth, -INFINITY, +INFINITY);

        pv_move = info->best_move;  // Save for next iteration
        best_move = info->best_move;

        if (should_stop(info)) break;
    }

    return best_move;
}

// In move ordering (move_score function):
if (move == search_info->pv_move) {
    return 10000000;  // Highest priority
}
```

## Handling Incomplete Iterations

When time expires mid-search, we have partial results from the deepest iteration. Options:

### Option 1: Discard and Use Previous

```c
for (int depth = 1; depth <= MAX_DEPTH; depth++) {
    Move this_best = MOVE_NONE;
    int score = search(pos, depth, &this_best);

    if (time_exceeded()) {
        break;  // Don't update best_move
    }

    best_move = this_best;  // Only update if completed
}
```

Simple and safe. Wastes the partial work.

### Option 2: Use Partial Result

The partial search may have found a better move:

```c
int score = search(pos, depth, &this_best);

if (this_best != MOVE_NONE) {
    // Search found at least one move
    // If it found a better score, probably use it
    if (!time_exceeded() || score > best_score) {
        best_move = this_best;
    }
}
```

More complex but extracts value from partial searches.

### Option 3: Soft/Hard Time Limits

Use two time limits:
- **Soft limit**: Target time; try to complete current depth
- **Hard limit**: Absolute cutoff; stop immediately

```c
for (int depth = 1; depth <= MAX_DEPTH; depth++) {
    // Don't start a new depth if soft limit exceeded
    if (elapsed > soft_limit && depth > 1) break;

    // But allow completing current depth unless hard limit hit
    int score = search(pos, depth);

    if (elapsed > hard_limit) break;
}
```

## Integrating with Transposition Table

Each iteration populates the transposition table:

```c
// In alpha_beta:
TTEntry* entry = tt_probe(pos->hash);
if (entry && entry->depth >= depth) {
    // Use stored result
}

// After search:
tt_store(pos->hash, depth, score, best_move, flag);
```

Deeper iterations benefit from entries stored during shallower ones.

## Detecting Forced Moves

If there's only one legal move, return it immediately:

```c
Move iterative_deepening(Position* pos) {
    MoveList moves;
    generate_legal_moves(pos, &moves);

    if (moves.count == 1) {
        return moves.moves[0];  // Only move—don't think
    }

    // Normal iterative deepening...
}
```

Some engines also return quickly when one move is clearly best:

```c
// If best move has been stable for 5 iterations
// and score is at least 200cp better than second best
// consider stopping early
```

## Handling "Easy" Moves

If the current best move is clearly best:

```c
// After each iteration:
if (best_move == previous_best_move) {
    stability_count++;
} else {
    stability_count = 0;
}

// If move is stable and we've searched enough depth
if (stability_count >= 5 && depth >= 10) {
    // Consider using less time
    soft_limit /= 2;
}
```

## Handling "Hard" Positions

If the best move keeps changing:

```c
// Score dropped significantly from previous iteration
if (score < previous_score - 50) {
    // Position is tricky; allocate more time
    soft_limit *= 1.5;
}

// Best move changed
if (best_move != previous_best_move) {
    // Don't stop on this iteration
    min_depth = depth + 2;
}
```

## Search Instability

Sometimes the score oscillates between iterations:

```
depth 8: score +50, pv e4
depth 9: score -30, pv d4
depth 10: score +45, pv e4
```

This happens due to the horizon effect. Don't panic—trust deeper searches are generally more accurate. But be wary of time trouble decisions when scores are unstable.

## Implementation Structure

A clean implementation:

```c
typedef struct {
    Position* pos;
    Move best_move;
    int best_score;
    int depth;
    int64_t nodes;
    Timer timer;
    int soft_limit_ms;
    int hard_limit_ms;
    bool stopped;
} SearchInfo;

void search_position(SearchInfo* info) {
    info->best_move = MOVE_NONE;
    info->best_score = -INFINITY;
    info->nodes = 0;
    info->stopped = false;

    // Clear killer moves, history from previous search
    clear_search_tables();

    for (int depth = 1; depth <= MAX_DEPTH; depth++) {
        info->depth = depth;

        // Check soft limit before starting new depth
        if (elapsed_ms(&info->timer) > info->soft_limit_ms && depth > 1) {
            break;
        }

        int score = search_root(info);

        if (info->stopped) break;

        info->best_score = score;
        print_info(info);
    }
}

int search_root(SearchInfo* info) {
    MoveList moves;
    generate_moves(info->pos, &moves);
    score_moves(&moves, info);

    int alpha = -INFINITY;
    int beta = +INFINITY;

    for (int i = 0; i < moves.count; i++) {
        Move move = pick_best_move(&moves, i);

        make_move(info->pos, move);
        int score = -alpha_beta(info, info->depth - 1, -beta, -alpha);
        unmake_move(info->pos, move);

        if (info->stopped) break;

        if (score > alpha) {
            alpha = score;
            info->best_move = move;
        }
    }

    return alpha;
}

int alpha_beta(SearchInfo* info, int depth, int alpha, int beta) {
    // Check hard limit periodically
    if ((info->nodes & 2047) == 0) {  // Every 2048 nodes
        if (elapsed_ms(&info->timer) > info->hard_limit_ms) {
            info->stopped = true;
            return 0;
        }
    }

    info->nodes++;

    // ... rest of alpha-beta ...
}
```

## Parallel Considerations

When using multiple threads (not covered in detail here), iterative deepening helps:

- Threads share the transposition table
- Shallower iterations provide good starting moves for all threads
- "Lazy SMP" has threads search at different depths simultaneously

## Summary

Iterative deepening:

- **Solves time management**: Always have a valid move
- **Improves move ordering**: Previous PV searched first
- **Populates transposition table**: Deeper searches reuse shallow results
- **Overhead is minimal**: ~3% for repeated shallow searches
- **Enables aspiration windows**: Use previous score as window center
- **Handles instability**: Multiple iterations smooth out noise

Despite searching the same positions multiple times, iterative deepening is faster than direct deep search thanks to better move ordering. It's universally used in chess engines.

Next: The transposition table—remembering positions we've seen before.
