# Chapter 30: Parameter Tuning

Chess engines have hundreds of parameters: piece values, PST values, search constants, pruning thresholds. Finding optimal values is crucial for strength.

## Types of Parameters

### Evaluation Parameters

```c
// Material values
int PAWN_VALUE = 100;
int KNIGHT_VALUE = 320;
int BISHOP_VALUE = 330;
int ROOK_VALUE = 500;
int QUEEN_VALUE = 900;

// Positional bonuses
int DOUBLED_PAWN_PENALTY = -15;
int ISOLATED_PAWN_PENALTY = -20;
int PASSED_PAWN_BONUS[8] = {0, 10, 20, 35, 60, 100, 150, 0};
int BISHOP_PAIR_BONUS = 30;
int ROOK_OPEN_FILE = 25;
int ROOK_SEMI_OPEN = 15;

// King safety
int PAWN_SHIELD_BONUS = 10;
int ATTACK_WEIGHT[20] = {0, 0, 5, 12, 25, ...};
```

### Search Parameters

```c
// Null move
int NULL_MOVE_REDUCTION = 3;
int NULL_MOVE_THRESHOLD = 3;  // Don't use below this depth

// LMR
int LMR_DEPTH_LIMIT = 3;
int LMR_MOVE_LIMIT = 4;
float LMR_BASE = 0.75;
float LMR_DIVISOR = 2.25;

// Futility pruning
int FUTILITY_MARGIN[4] = {0, 100, 200, 300};

// Aspiration windows
int ASP_WINDOW_INITIAL = 25;
int ASP_WINDOW_EXPANSION = 50;
```

## Manual Tuning

Start with established values from strong engines, then adjust based on testing.

### Piece Values

Classical starting point:
- Pawn: 100
- Knight: 320
- Bishop: 330
- Rook: 500
- Queen: 900

Test variations like Knight=300 vs Knight=325 with thousands of games.

### Iterative Refinement

1. Change one parameter
2. Run 1000+ games against baseline
3. If significant improvement, keep the change
4. Repeat

This is slow but straightforward.

## Texel Tuning

Named after the Texel chess engine. Uses gradient descent on game results.

### Principle

Minimize the error between:
- Engine's evaluation of positions
- Actual game results (1.0, 0.5, 0.0)

### Dataset

Extract millions of positions from games with known results:

```c
typedef struct {
    Position pos;
    double result;  // 1.0 = white win, 0.5 = draw, 0.0 = black win
} TrainingPosition;
```

Source games from:
- Self-play at long time controls
- CCRL/TCEC games
- Human master games (less ideal for engines)

### Sigmoid Function

Convert evaluation to expected result:

```c
double sigmoid(int eval, double K) {
    return 1.0 / (1.0 + pow(10.0, -K * eval / 400.0));
}
```

K is a scaling constant (typically around 1.0-1.5).

### Error Function

Mean squared error:

```c
double calculate_error(TrainingPosition* positions, int count, double K) {
    double total_error = 0.0;

    for (int i = 0; i < count; i++) {
        int eval = evaluate(&positions[i].pos);

        // Negate for black to move
        if (positions[i].pos.side_to_move == BLACK) {
            eval = -eval;
        }

        double predicted = sigmoid(eval, K);
        double actual = positions[i].result;

        double error = predicted - actual;
        total_error += error * error;
    }

    return total_error / count;
}
```

### Finding K

First, find optimal K for your current evaluation:

```c
double find_optimal_K(TrainingPosition* positions, int count) {
    double best_K = 1.0;
    double best_error = DBL_MAX;

    for (double K = 0.5; K <= 2.0; K += 0.01) {
        double error = calculate_error(positions, count, K);
        if (error < best_error) {
            best_error = error;
            best_K = K;
        }
    }

    return best_K;
}
```

### Gradient Descent

Adjust parameters to minimize error:

```c
void tune_parameters(TrainingPosition* positions, int count) {
    double K = find_optimal_K(positions, count);
    double learning_rate = 1.0;

    for (int iteration = 0; iteration < 10000; iteration++) {
        // For each tunable parameter
        for (int p = 0; p < NUM_PARAMS; p++) {
            double base_error = calculate_error(positions, count, K);

            // Try increasing parameter
            params[p] += 1;
            double error_up = calculate_error(positions, count, K);
            params[p] -= 1;

            // Try decreasing parameter
            params[p] -= 1;
            double error_down = calculate_error(positions, count, K);
            params[p] += 1;

            // Calculate gradient
            double gradient = (error_up - error_down) / 2.0;

            // Update parameter
            params[p] -= learning_rate * gradient;
        }

        // Decrease learning rate over time
        if (iteration % 1000 == 0) {
            learning_rate *= 0.9;
            printf("Iteration %d, Error: %.6f\n", iteration,
                   calculate_error(positions, count, K));
        }
    }
}
```

### Practical Implementation

```c
// Parameter definition
typedef struct {
    int* ptr;           // Pointer to actual parameter
    int min_val;        // Minimum allowed value
    int max_val;        // Maximum allowed value
    const char* name;   // For debugging
} TunableParam;

TunableParam tunable_params[] = {
    {&KNIGHT_VALUE, 250, 400, "Knight Value"},
    {&BISHOP_VALUE, 250, 400, "Bishop Value"},
    {&ROOK_VALUE, 400, 600, "Rook Value"},
    {&DOUBLED_PAWN_PENALTY, -50, 0, "Doubled Pawn"},
    {&ISOLATED_PAWN_PENALTY, -50, 0, "Isolated Pawn"},
    {&BISHOP_PAIR_BONUS, 0, 80, "Bishop Pair"},
    // ... hundreds more
};

// After tuning, print results
void print_tuned_params() {
    for (int i = 0; i < NUM_TUNABLE; i++) {
        printf("%s = %d\n", tunable_params[i].name, *tunable_params[i].ptr);
    }
}
```

## SPSA Tuning

**Simultaneous Perturbation Stochastic Approximation** - tunes by playing games.

### Principle

Instead of using static positions, SPSA measures parameter quality through actual game results.

### Algorithm

```c
void spsa_tune() {
    double a = 1.0;    // Step size
    double c = 2.0;    // Perturbation size
    double A = 100.0;  // Stability constant

    for (int k = 0; k < NUM_ITERATIONS; k++) {
        // Decay coefficients
        double ak = a / pow(k + 1 + A, 0.602);
        double ck = c / pow(k + 1, 0.101);

        // Generate random perturbation direction
        int delta[NUM_PARAMS];
        for (int i = 0; i < NUM_PARAMS; i++) {
            delta[i] = (rand() % 2) ? 1 : -1;
        }

        // Create perturbed parameter sets
        int params_plus[NUM_PARAMS];
        int params_minus[NUM_PARAMS];
        for (int i = 0; i < NUM_PARAMS; i++) {
            params_plus[i] = params[i] + ck * delta[i];
            params_minus[i] = params[i] - ck * delta[i];
        }

        // Play games with each parameter set
        double score_plus = play_match(params_plus, NUM_GAMES / 2);
        double score_minus = play_match(params_minus, NUM_GAMES / 2);

        // Estimate gradient and update
        double gradient_estimate = (score_plus - score_minus) / (2.0 * ck);
        for (int i = 0; i < NUM_PARAMS; i++) {
            params[i] -= ak * gradient_estimate * delta[i];
        }

        printf("Iteration %d: score+ = %.3f, score- = %.3f\n",
               k, score_plus, score_minus);
    }
}
```

### Advantages

- Measures actual playing strength
- Handles parameter interactions
- Works with any objective function

### Disadvantages

- Very slow (requires many games)
- High variance in results
- Requires careful tuning of SPSA parameters

## Local Search Methods

### Hill Climbing

```c
void hill_climb(int param_index) {
    int best_value = params[param_index];
    double best_score = play_match(params, 1000);

    for (int delta = -50; delta <= 50; delta += 5) {
        params[param_index] = best_value + delta;
        double score = play_match(params, 1000);

        if (score > best_score) {
            best_score = score;
            best_value = params[param_index];
        }
    }

    params[param_index] = best_value;
}
```

### Coordinate Descent

Optimize one parameter at a time:

```c
void coordinate_descent() {
    bool improved = true;

    while (improved) {
        improved = false;

        for (int i = 0; i < NUM_PARAMS; i++) {
            int old_value = params[i];
            hill_climb(i);

            if (params[i] != old_value) {
                improved = true;
            }
        }
    }
}
```

## Piece-Square Table Tuning

PSTs have 64 values per piece per phase. Use symmetry to reduce parameters:

```c
// For white pieces, only tune half the board (mirror for black)
// For symmetric pieces (rook, queen), only tune one side

int tunable_pst_knight[32];  // Only a1-h4, mirror for rest

void expand_pst() {
    for (int sq = 0; sq < 32; sq++) {
        // Original square
        PST_KNIGHT[WHITE][sq] = tunable_pst_knight[sq];

        // Horizontal mirror
        int mirror_sq = sq ^ 7;  // Flip file
        PST_KNIGHT[WHITE][mirror_sq] = tunable_pst_knight[sq];
    }

    // Black is vertically flipped and negated
    for (int sq = 0; sq < 64; sq++) {
        PST_KNIGHT[BLACK][sq ^ 56] = -PST_KNIGHT[WHITE][sq];
    }
}
```

## Search Parameter Tuning

Search parameters are harder to tune—they don't affect static evaluation.

### Approach 1: Playing Games

```c
// Test different LMR reduction values
for (int base = 50; base <= 100; base += 10) {
    for (int divisor = 150; divisor <= 300; divisor += 25) {
        LMR_BASE = base / 100.0;
        LMR_DIVISOR = divisor / 100.0;

        double score = play_match(1000);
        printf("Base=%.2f, Div=%.2f: %.1f%%\n",
               LMR_BASE, LMR_DIVISOR, score * 100);
    }
}
```

### Approach 2: Node Count

For pruning parameters, compare node counts at fixed depth:

```c
// Optimal pruning gives same result with fewer nodes
void test_futility_margin() {
    for (int margin = 50; margin <= 200; margin += 25) {
        FUTILITY_MARGIN = margin;

        uint64_t total_nodes = 0;
        int correct = 0;

        for (int i = 0; i < NUM_TEST_POSITIONS; i++) {
            Move found = search(&positions[i], 10);
            total_nodes += search_info.nodes;

            if (found == known_best[i]) correct++;
        }

        printf("Margin=%d: nodes=%llu, correct=%d/%d\n",
               margin, total_nodes, correct, NUM_TEST_POSITIONS);
    }
}
```

## Data Generation

Quality training data is crucial:

### From Self-Play

```c
void generate_training_data() {
    FILE* output = fopen("training.epd", "w");

    for (int game = 0; game < NUM_GAMES; game++) {
        Position pos;
        set_fen(&pos, START_FEN);

        Move history[500];
        int move_count = 0;
        int result;  // Game result

        // Play game
        while (!is_game_over(&pos, &result)) {
            Move move = search(&pos, TRAINING_DEPTH);
            history[move_count++] = move;
            make_move(&pos, move);
        }

        // Write positions from game (skip opening, avoid endgame tablebases)
        set_fen(&pos, START_FEN);
        for (int i = 0; i < move_count; i++) {
            if (i >= 8 && popcount(pos.all_pieces) > 6) {
                // Write position with result
                char fen[256];
                position_to_fen(&pos, fen);
                fprintf(output, "%s c9 \"%.1f\";\n", fen, result);
            }
            make_move(&pos, history[i]);
        }
    }

    fclose(output);
}
```

### Filtering Positions

Remove positions that confuse tuning:
- Tactical positions (score changes significantly with depth)
- Positions in check
- Endgame tablebase positions
- Opening book positions

```c
bool should_include_position(Position* pos) {
    // Skip positions in check
    if (in_check(pos)) return false;

    // Skip very early game
    if (pos->fullmove < 8) return false;

    // Skip if evaluation changes drastically with depth
    int shallow = evaluate(pos);
    int deep = quiesce(pos, -INF, INF);
    if (abs(shallow - deep) > 100) return false;

    return true;
}
```

## Putting It Together

### Texel Tuning Workflow

1. Generate 5-10 million positions from self-play
2. Filter to ~1-2 million quality positions
3. Find optimal K value
4. Run gradient descent for 10000+ iterations
5. Validate with engine matches

### SPSA Workflow

1. Define parameter ranges and step sizes
2. Set up game-playing infrastructure
3. Run SPSA for thousands of iterations
4. Periodically check Elo improvement vs baseline

### Hybrid Approach

1. Use Texel tuning for initial parameter estimates
2. Fine-tune with SPSA using game results
3. Validate final parameters with large match

## Summary

Parameter tuning methods:

- **Manual tuning**: Simple but slow
- **Texel tuning**: Fast gradient descent on position evaluations
- **SPSA**: Slower but measures actual playing strength
- **Local search**: Hill climbing, coordinate descent

Key considerations:

- Use quality training data
- Tune related parameters together
- Validate improvements with games
- Search parameters need different approaches than evaluation

Well-tuned parameters can add 50-100+ Elo over default values.

Part 6 complete. Next: Advanced topics—neural networks and parallel search.
