# Chapter 17: Search Extensions

While pruning reduces search in less important positions, **extensions** increase search in critical positions. The goal: see further in lines that matter most.

## The Extension Philosophy

Not all positions are equal. Some deserve deeper analysis:
- Checks and escaping check
- Single-reply positions
- Promotions approaching
- Recaptures
- Moves that are clearly singular (much better than alternatives)

Extensions ensure we don't miss critical tactical lines.

## Check Extensions

When we give check, extend by one ply:

```c
bool gives_check = is_in_check(pos);  // After make_move

int extension = 0;
if (gives_check) {
    extension = 1;
}

int score = -search(pos, depth - 1 + extension, -beta, -alpha);
```

Why extend checks?
- Checks are forcing—opponent has limited responses
- Tactical sequences often involve multiple checks
- Missing a check can miss a checkmate

### Check Extension Limits

Unlimited check extensions can cause search explosion:

```c
// Limit total extensions
if (gives_check && ss->ply < 2 * root_depth) {
    extension = 1;
}

// Or limit check extensions per line
if (gives_check && ss->check_extensions < 10) {
    extension = 1;
    ss->check_extensions++;
}
```

## Singular Extensions

**Singular extensions** extend when one move is much better than all alternatives—it's the only good move:

```c
// Get hash move and its score
Move hash_move = tt_probe_move(pos->hash);
int hash_score = tt_probe_score(pos->hash);

if (hash_move != MOVE_NONE && depth >= 8 && hash_score > alpha) {
    // Search without the hash move at reduced depth
    int singular_beta = hash_score - depth * 2;

    int score = search_excluding_move(pos, depth / 2, singular_beta - 1,
                                      singular_beta, hash_move);

    // If nothing else reaches singular_beta, hash move is singular
    if (score < singular_beta) {
        extension = 1;  // Extend the singular move
    }
}
```

The search excludes the hash move. If no other move comes close to the hash move's score, the hash move is "singular"—the only good choice.

### Negative Extensions (Reductions)

If the hash move fails low, maybe reduce instead:

```c
if (score < singular_beta) {
    extension = 1;
} else if (hash_score >= beta) {
    // Hash move isn't special; maybe reduce
    extension = -1;
}
```

## Passed Pawn Extensions

Extend when a pawn is close to promotion:

```c
if (piece_type(moved_piece) == PAWN) {
    int to_rank = rank_of(move_to(move));

    // White pawn on 7th rank, or black pawn on 2nd rank
    if ((color == WHITE && to_rank == 6) ||
        (color == BLACK && to_rank == 1)) {
        extension = 1;  // One step from promotion
    }
}
```

Some engines also extend passed pawn pushes earlier.

## Recapture Extensions

Extend when recapturing on a square where we just lost material:

```c
if (is_capture(move) && move_to(move) == move_to(prev_move)) {
    // Recapturing on the same square
    extension = 1;
}
```

This helps see complete exchanges.

## One-Reply Extensions

When there's only one legal move, extend:

```c
if (moves.count == 1) {
    extension = 1;
}
```

With only one choice, we should see what happens next.

## Extension Limits

Extensions can cause search explosion. Implement limits:

### Maximum Extensions Per Node

```c
extension = min(extension, 1);  // Never extend more than 1 ply
```

### Maximum Total Extensions

Track cumulative extensions in the search stack:

```c
if (ss->ply > 2 * root_depth) {
    extension = 0;  // Too deep already
}
```

### Fractional Extensions

Instead of always extending by 1, use fractional extensions:

```c
int frac_extension = 0;  // In 1/16ths of a ply

if (gives_check) frac_extension += 16;  // Full ply for check
if (is_passed_push) frac_extension += 8;  // Half ply for passed pawn

int extension = (ss->frac_extension + frac_extension) / 16;
ss->frac_extension = (ss->frac_extension + frac_extension) % 16;
```

This allows multiple partial extensions to combine.

## Complete Implementation

```c
int search(Position* pos, int depth, int alpha, int beta, SearchStack* ss) {
    // ... pruning, TT ...

    MoveList moves;
    generate_moves(pos, &moves);

    Move best_move = MOVE_NONE;
    int best_score = -INFINITY;
    int moves_searched = 0;

    // Singular extension search
    Move singular_move = MOVE_NONE;
    if (depth >= 8 && tt_entry && tt_entry->flag != TT_UPPER) {
        int singular_beta = tt_entry->score - depth * 2;
        int score = search_excluding(pos, depth / 2, singular_beta - 1,
                                     singular_beta, tt_entry->best_move);
        if (score < singular_beta) {
            singular_move = tt_entry->best_move;
        }
    }

    for (int i = 0; i < moves.count; i++) {
        Move move = pick_best(&moves, i);

        make_move(pos, move);
        bool gives_check = is_in_check(pos);

        // Calculate extension
        int extension = 0;

        if (move == singular_move) {
            extension = 1;
        } else if (gives_check && ss->ply < 2 * root_depth) {
            extension = 1;
        } else if (is_passed_pawn_push(pos, move, ss->side_to_move)) {
            extension = 1;
        }

        extension = min(extension, 1);
        int new_depth = depth - 1 + extension;

        // ... LMR, search ...

        int score = -search(pos, new_depth, -beta, -alpha, ss + 1);

        unmake_move(pos, move);

        // ... update best ...
    }

    return best_score;
}
```

## Balancing Extensions

Extensions vs pruning must be balanced:
- Too many extensions → search explosion
- Too few extensions → miss tactics

Test on large game sets:
- Self-play for Elo impact
- Tactical test suites for correctness
- Node count monitoring

## Summary

| Extension | Condition | Benefit |
|-----------|-----------|---------|
| Check | Move gives check | See forcing lines |
| Singular | One move much better | Don't miss critical moves |
| Passed Pawn | Pawn near promotion | See promotion tactics |
| Recapture | Capture on same square | Complete exchanges |
| One-Reply | Only one legal move | No choice—see further |

Extensions ensure critical lines are searched deeply. Combined with pruning, they focus search effort where it matters most.

Next: Aspiration windows—narrowing the search window for efficiency.
