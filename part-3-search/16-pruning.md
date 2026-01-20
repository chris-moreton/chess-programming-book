# Chapter 16: Pruning Techniques

Beyond null move and LMR, several pruning techniques can safely skip searching moves or subtrees. This chapter covers the major ones.

## Futility Pruning

**Futility pruning** skips quiet moves that cannot possibly raise alpha, even with optimistic assumptions.

At shallow depths, if static evaluation plus a margin is below alpha, quiet moves are unlikely to help:

```c
// At depth 1-2, before searching quiet moves
int futility_margin = 200 * depth;  // ~2 pawns per depth
int static_eval = evaluate(pos);

if (!in_check && depth <= 2 && static_eval + futility_margin <= alpha) {
    // Don't search quiet moves—they can't raise alpha
    // Still search captures and promotions
    skip_quiet_moves = true;
}
```

Or per-move:

```c
if (!in_check && depth <= 2 && !is_capture(move) && !is_promotion(move)) {
    if (static_eval + futility_margin <= alpha) {
        continue;  // Skip this quiet move
    }
}
```

## Reverse Futility Pruning (Static Null Move)

If static evaluation minus a margin exceeds beta, the position is so good we can return beta without searching:

```c
if (!in_check && depth <= 6 && !is_pv_node) {
    int margin = 100 * depth;

    if (static_eval - margin >= beta) {
        return static_eval - margin;  // Position is clearly winning
    }
}
```

This is like null move pruning but uses static evaluation instead of search.

## Razoring

**Razoring** drops into quiescence search at low depths when evaluation is far below alpha:

```c
if (!in_check && depth <= 3) {
    int razor_margin = 300 + 200 * depth;

    if (static_eval + razor_margin <= alpha) {
        // Position is bad; verify with quiescence
        int qs_score = quiesce(pos, alpha, beta);

        if (qs_score <= alpha) {
            return qs_score;  // Confirmed bad
        }
    }
}
```

The quiescence search confirms the position is really as bad as it looks.

## Late Move Pruning (Move Count Pruning)

After searching many moves without improvement, remaining moves are unlikely to help:

```c
int lmp_threshold[] = {0, 5, 9, 14, 21, 30};  // Index by depth

for (int i = 0; i < moves.count; i++) {
    Move move = pick_best(&moves, i);

    // Late move pruning
    if (!in_check &&
        depth <= 5 &&
        moves_searched > lmp_threshold[depth] &&
        !is_capture(move) &&
        !is_promotion(move) &&
        !gives_check)
    {
        continue;  // Skip remaining quiet moves
    }

    // ... search move ...
    moves_searched++;
}
```

## SEE Pruning for Quiet Moves

Use SEE to prune quiet moves with bad move quality:

```c
// For quiet moves at low depth
if (depth <= 4 && !in_check) {
    int see_threshold = -20 * depth * depth;

    if (see(pos, move) < see_threshold) {
        continue;  // Move loses material; skip
    }
}
```

This catches moves that hang pieces.

## History Pruning

Prune quiet moves with very negative history scores:

```c
int hist = history_score(move, pos);
int hist_threshold = -2000 * depth;

if (!in_check && depth <= 4 && hist < hist_threshold) {
    continue;  // Historically bad move; skip
}
```

## Multi-Cut Pruning

If multiple moves at a cut node fail high with shallow search, the node is probably a real cut node:

```c
if (!is_pv_node && depth >= 5 && !in_check) {
    int cut_count = 0;
    int cut_threshold = 3;

    for (int i = 0; i < min(moves.count, 6); i++) {
        Move move = pick_best(&moves, i);
        make_move(pos, move);

        int score = -search(pos, depth - 4, -beta, -beta + 1);
        unmake_move(pos, move);

        if (score >= beta) {
            cut_count++;
            if (cut_count >= cut_threshold) {
                return beta;  // Multiple moves beat beta
            }
        }
    }
}
```

Rarely used in modern engines; LMR serves a similar purpose more elegantly.

## Singular Extensions Basis

Not a pruning technique, but related: if one move is much better than alternatives, extend its search (Chapter 17).

## Combining Pruning Techniques

The pruning techniques interact and are applied in sequence:

```c
int search(Position* pos, int depth, int alpha, int beta, bool is_pv) {
    // ... TT probe ...

    bool in_check = is_in_check(pos);
    int static_eval = in_check ? -INFINITY : evaluate(pos);

    if (!in_check && !is_pv) {
        // Reverse futility pruning
        if (depth <= 6 && static_eval - 100 * depth >= beta) {
            return static_eval - 100 * depth;
        }

        // Razoring
        if (depth <= 3 && static_eval + 300 + 200 * depth <= alpha) {
            int qs = quiesce(pos, alpha, beta);
            if (qs <= alpha) return qs;
        }

        // Null move pruning
        if (depth >= 3 && has_non_pawn_material(pos)) {
            // ... null move ...
        }
    }

    MoveList moves;
    generate_moves(pos, &moves);
    score_moves(&moves);

    int moves_searched = 0;

    for (int i = 0; i < moves.count; i++) {
        Move move = pick_best(&moves, i);

        make_move(pos, move);
        if (!is_legal(pos)) {
            unmake_move(pos, move);
            continue;
        }

        bool gives_check = is_in_check(pos);

        // Futility pruning
        if (!in_check && depth <= 2 && moves_searched > 0 &&
            !is_capture(move) && !is_promotion(move) && !gives_check)
        {
            if (static_eval + 200 * depth <= alpha) {
                unmake_move(pos, move);
                continue;
            }
        }

        // Late move pruning
        if (!in_check && depth <= 5 && moves_searched > lmp_threshold[depth] &&
            !is_capture(move) && !gives_check)
        {
            unmake_move(pos, move);
            continue;
        }

        // SEE pruning
        if (!in_check && depth <= 4 && !is_promotion(move) &&
            see(pos, move) < -20 * depth * depth)
        {
            unmake_move(pos, move);
            continue;
        }

        moves_searched++;

        // ... LMR and search ...

        unmake_move(pos, move);
    }

    return best_score;
}
```

## Safety and Testing

Pruning is risky—each technique can occasionally prune good moves. Test each addition:

1. **Self-play matches**: Does it improve Elo?
2. **Test positions**: Does it pass tactical suites?
3. **Regression testing**: Does it break previously working positions?

Conservative parameters are safer than aggressive ones.

## Summary

| Technique | When | What |
|-----------|------|------|
| Futility | Depth ≤ 2, quiet moves | Skip if eval + margin ≤ alpha |
| Reverse Futility | Depth ≤ 6, non-PV | Return if eval - margin ≥ beta |
| Razoring | Depth ≤ 3, low eval | Drop to quiescence |
| Late Move Pruning | Many moves searched | Skip remaining quiet moves |
| SEE Pruning | Low depth, bad SEE | Skip moves that hang material |
| History Pruning | Low depth, bad history | Skip historically weak moves |

These techniques combine to reduce node count by 80-95% at low depths. The challenge is calibrating margins and conditions to maximize pruning while minimizing errors.

Next: Extending the search in critical positions.
