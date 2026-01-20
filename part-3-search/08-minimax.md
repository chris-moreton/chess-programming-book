# Chapter 8: Minimax and Game Trees

Search is the heart of a chess engine. This chapter introduces the fundamental algorithm: minimax. Understanding minimax deeply is essential—every enhancement in subsequent chapters builds on this foundation.

## Game Trees and Chess

A **game tree** represents all possible continuations from a position. Each node is a position; each edge is a move.

```
                    Starting Position
                    /      |      \
                  e4       d4      Nf3   ... (20 moves)
                 /  \     /  \
               e5   c5   d5   Nf6   ...
              / \   ...
           Nf3  Bc4
           ...
```

The **root** is the current position. **Leaf nodes** (or terminal nodes) are positions where:
- The game ends (checkmate, stalemate, draw)
- We stop searching (reaching our depth limit)

### Branching Factor

The average number of legal moves per position is the **branching factor**. In chess, it's approximately 35.

At depth d, a complete tree has roughly 35^d nodes:
- Depth 1: 35 nodes
- Depth 2: 1,225 nodes
- Depth 4: ~1.5 million nodes
- Depth 6: ~1.8 billion nodes
- Depth 10: ~2.75 × 10^15 nodes

This exponential growth is why naive search fails. We need smart pruning.

### Ply vs Depth

A **ply** is a half-move (one player's move). A **depth** can mean different things:
- Some engines: depth = plies
- Some engines: depth = full moves (two plies)

This book uses depth = plies. "Searching to depth 6" means 6 half-moves (3 for each side).

## The Minimax Algorithm

Chess is a **zero-sum** game: what's good for white is bad for black. If a position scores +3 for white, it scores -3 for black.

**Minimax** exploits this: one player (maximizer) tries to maximize the score; the other (minimizer) tries to minimize it.

### The Key Insight

At each node:
- If it's the maximizer's turn: choose the move with the highest score
- If it's the minimizer's turn: choose the move with the lowest score

The scores "back up" from leaf nodes to the root.

### Minimax Pseudocode

```c
int minimax(Position* pos, int depth, bool is_maximizing) {
    // Terminal conditions
    if (depth == 0 || is_game_over(pos)) {
        return evaluate(pos);  // Static evaluation
    }

    MoveList moves;
    generate_moves(pos, &moves);

    if (is_maximizing) {
        int max_eval = -INFINITY;
        for (int i = 0; i < moves.count; i++) {
            make_move(pos, moves.moves[i]);
            int eval = minimax(pos, depth - 1, false);
            unmake_move(pos, moves.moves[i]);
            max_eval = max(max_eval, eval);
        }
        return max_eval;
    } else {
        int min_eval = +INFINITY;
        for (int i = 0; i < moves.count; i++) {
            make_move(pos, moves.moves[i]);
            int eval = minimax(pos, depth - 1, true);
            unmake_move(pos, moves.moves[i]);
            min_eval = min(min_eval, eval);
        }
        return min_eval;
    }
}
```

### Example: Simple Tree

Consider a tree with depth 2 (white moves, black responds):

```
              Root (White to move)
             /      |       \
           A        B        C
          /|\      /|\      /|\
         3 5 2   4 6 1    7 9 8
```

Black will minimize at the second level:
- Branch A: min(3, 5, 2) = 2
- Branch B: min(4, 6, 1) = 1
- Branch C: min(7, 9, 8) = 7

White will maximize at the root:
- max(2, 1, 7) = 7

So the best move is C with value 7.

### Evaluation Convention

There are two conventions for evaluation:
1. **Side-relative**: Positive = good for side to move
2. **Absolute**: Positive = good for white

This book uses **side-relative** evaluation. A score of +100 means the side to move is ahead by roughly one pawn.

With side-relative evaluation, we can simplify minimax...

## Negamax Formulation

**Negamax** is minimax rewritten using the identity:

```
max(a, b) = -min(-a, -b)
```

Instead of alternating between maximizing and minimizing, we always maximize—but negate the score when recursing.

```c
int negamax(Position* pos, int depth) {
    if (depth == 0 || is_game_over(pos)) {
        return evaluate(pos);  // Always from current player's perspective
    }

    MoveList moves;
    generate_moves(pos, &moves);

    int max_eval = -INFINITY;
    for (int i = 0; i < moves.count; i++) {
        make_move(pos, moves.moves[i]);
        int eval = -negamax(pos, depth - 1);  // Negate child's score
        unmake_move(pos, moves.moves[i]);
        max_eval = max(max_eval, eval);
    }

    return max_eval;
}
```

The negation on recursion flips the perspective. If the opponent's best achievable score is +50, from our view that's -50.

### Why Negamax?

1. **Simpler code**: No if/else for max vs min
2. **Fewer bugs**: One code path instead of two
3. **Same result**: Mathematically equivalent to minimax

All modern engines use negamax.

## Handling Terminal Nodes

### Checkmate

If the side to move is checkmated, return a very negative score (worst case):

```c
if (is_checkmate(pos)) {
    return -MATE_SCORE + ply;  // Adjusted for distance to mate
}
```

Adding `ply` ensures closer mates score better than distant ones. We prefer mate in 3 over mate in 10.

### Stalemate and Draws

```c
if (is_stalemate(pos)) {
    return 0;  // Draw
}

if (is_draw_by_repetition(pos) || is_draw_by_fifty_moves(pos)) {
    return 0;  // Draw
}
```

Draw scores can be more nuanced—slightly positive if you're losing, slightly negative if you're winning—but 0 is standard.

### Mate Score Handling

Define mate scores carefully:

```c
#define MATE_SCORE 32000
#define INFINITY   32767

// A mate at ply 0 scores -32000
// A mate at ply 1 scores -31999
// A mate at ply 10 scores -31990

bool is_mate_score(int score) {
    return abs(score) > MATE_SCORE - 100;
}

int mate_in(int score) {
    if (score > 0) {
        return (MATE_SCORE - score + 1) / 2;  // Mate in N moves
    } else {
        return (MATE_SCORE + score + 1) / 2;  // Getting mated in N moves
    }
}
```

## Complexity Analysis

### Time Complexity

For a tree with branching factor b and depth d:
- Minimax examines O(b^d) nodes

With b=35 and d=6, that's about 1.8 billion nodes.

### The Branching Factor Problem

The key challenge: b^d grows exponentially. Doubling the depth squares the number of nodes (approximately).

Solutions:
1. **Pruning**: Don't examine branches we can prove are useless (next chapter)
2. **Move ordering**: Examine promising moves first to maximize pruning
3. **Selective search**: Search some branches deeper than others

## Finding the Best Move

The root needs to track which move leads to the best score:

```c
Move find_best_move(Position* pos, int depth) {
    MoveList moves;
    generate_moves(pos, &moves);

    Move best_move = moves.moves[0];
    int best_score = -INFINITY;

    for (int i = 0; i < moves.count; i++) {
        make_move(pos, moves.moves[i]);
        int score = -negamax(pos, depth - 1);
        unmake_move(pos, moves.moves[i]);

        if (score > best_score) {
            best_score = score;
            best_move = moves.moves[i];
        }
    }

    return best_move;
}
```

## The Evaluation Function

At leaf nodes, we need a **static evaluation** function that estimates the position's value without further search.

A basic evaluation:

```c
int evaluate(Position* pos) {
    int score = 0;

    // Material
    score += 100 * (popcount(pos->white_pawns) - popcount(pos->black_pawns));
    score += 320 * (popcount(pos->white_knights) - popcount(pos->black_knights));
    score += 330 * (popcount(pos->white_bishops) - popcount(pos->black_bishops));
    score += 500 * (popcount(pos->white_rooks) - popcount(pos->black_rooks));
    score += 900 * (popcount(pos->white_queens) - popcount(pos->black_queens));

    // Return from side to move's perspective
    return (pos->side_to_move == WHITE) ? score : -score;
}
```

Part 4 covers evaluation in depth. For now, material counting is sufficient.

## Limitations of Pure Minimax

### Speed

At 35^6 nodes, even at 10 million nodes/second, searching depth 6 takes 3 minutes. That's too slow for practical play.

### The Horizon Effect

Minimax can only see `depth` plies ahead. If something bad happens at ply 7, a depth-6 search is blind to it.

Example: We're about to lose our queen, but we can delay it by giving a useless check. The engine keeps checking instead of minimizing damage.

### Static Evaluation Errors

The evaluation function is imperfect. Searching deeper reduces reliance on evaluation by pushing the "horizon" further out.

## Looking Ahead

The next chapter introduces **alpha-beta pruning**, which reduces the effective branching factor from 35 to about 6 with good move ordering. This is the most important enhancement to basic minimax.

Subsequent chapters add:
- Iterative deepening (Chapter 10)
- Transposition tables (Chapter 11)
- Move ordering techniques (Chapter 12)
- Quiescence search (Chapter 13)
- Null move pruning (Chapter 14)
- Late move reductions (Chapter 15)
- Various pruning techniques (Chapter 16)
- Search extensions (Chapter 17)
- Aspiration windows (Chapter 18)
- Principal variation search (Chapter 19)

Each technique addresses one or more limitations of basic minimax.

## Summary

- **Game trees** represent all possible continuations
- **Minimax** backs up scores: maximizer vs minimizer
- **Negamax** simplifies: always maximize, negate on recursion
- **Side-relative evaluation**: positive = good for side to move
- **Mate scores** include ply distance for preferring shorter mates
- **Branching factor** (~35 in chess) causes exponential growth
- **Horizon effect**: blindness beyond search depth

The foundation is laid. Next: making minimax practical with alpha-beta pruning.
