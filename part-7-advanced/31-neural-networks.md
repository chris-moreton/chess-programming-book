# Chapter 31: Neural Network Evaluation

Since AlphaZero (2017), neural networks have transformed chess engine evaluation. This chapter introduces NN-based evaluation for chess engines.

## Traditional vs Neural Evaluation

### Traditional (Hand-Crafted)

```c
int evaluate(Position* pos) {
    int score = 0;
    score += material(pos);
    score += pst_scores(pos);
    score += pawn_structure(pos);
    score += mobility(pos);
    score += king_safety(pos);
    // ... many more terms
    return score;
}
```

Features are designed by humans, weights are tuned.

### Neural Network

```c
int evaluate(Position* pos) {
    float input[INPUT_SIZE];
    encode_position(pos, input);

    float output = forward_pass(network, input);

    return (int)(output * 100);  // Scale to centipawns
}
```

Network learns features automatically from game data.

## Network Architectures

### Simple Feedforward

```
Input (768) → Hidden (256) → Hidden (32) → Output (1)
```

Input: 768 features (12 piece types × 64 squares)

```c
void encode_position(Position* pos, float* input) {
    memset(input, 0, 768 * sizeof(float));

    for (int sq = 0; sq < 64; sq++) {
        int piece = pos->board[sq];
        if (piece != EMPTY) {
            // Index = piece_type * 64 + square
            int idx = piece_index(piece) * 64 + sq;
            input[idx] = 1.0f;
        }
    }
}
```

### NNUE Architecture

**NNUE** (Efficiently Updatable Neural Network) revolutionized NN chess engines.

Key insight: Most moves only change a few pieces, so we can incrementally update the network instead of recomputing everything.

```
Accumulator Architecture:

                    ┌─────────────────┐
     White POV ────►│  Accumulator    │
     (HalfKP)       │  (256 neurons)  │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
     Black POV ────►│    Combine      │──► Output
     (HalfKP)       │  (hidden layers)│
                    └─────────────────┘
```

### HalfKP Features

Input features encode piece positions relative to the king:

```c
// HalfKP: King position × Piece position
// 64 king squares × 10 piece types × 64 squares = 40,960 features

int halfkp_index(int king_sq, int piece, int piece_sq) {
    // piece: 0-9 (P,N,B,R,Q for each color, excluding kings)
    return king_sq * 640 + piece * 64 + piece_sq;
}

void encode_halfkp(Position* pos, int color, int16_t* accumulator) {
    int king_sq = pos->king_square[color];

    for (int sq = 0; sq < 64; sq++) {
        int piece = pos->board[sq];
        if (piece != EMPTY && piece_type(piece) != KING) {
            int idx = halfkp_index(king_sq, piece_to_index(piece), sq);
            accumulator[idx] = 1;
        }
    }
}
```

### Incremental Updates

The key advantage of NNUE—update accumulator incrementally:

```c
void update_accumulator_add(int16_t* accumulator, int king_sq, int piece, int sq) {
    int idx = halfkp_index(king_sq, piece, sq);

    // Add feature weights to accumulator
    for (int i = 0; i < ACCUMULATOR_SIZE; i++) {
        accumulator[i] += weights_L1[idx][i];
    }
}

void update_accumulator_remove(int16_t* accumulator, int king_sq, int piece, int sq) {
    int idx = halfkp_index(king_sq, piece, sq);

    for (int i = 0; i < ACCUMULATOR_SIZE; i++) {
        accumulator[i] -= weights_L1[idx][i];
    }
}

void make_move_nnue(Position* pos, Move move, int16_t* white_acc, int16_t* black_acc) {
    int from = move_from(move);
    int to = move_to(move);
    int piece = pos->board[from];
    int captured = pos->board[to];

    // Remove piece from old square
    update_accumulator_remove(white_acc, pos->king_square[WHITE], piece, from);
    update_accumulator_remove(black_acc, pos->king_square[BLACK], piece, from ^ 56);

    // Add piece to new square
    update_accumulator_add(white_acc, pos->king_square[WHITE], piece, to);
    update_accumulator_add(black_acc, pos->king_square[BLACK], piece, to ^ 56);

    // Handle captures
    if (captured != EMPTY) {
        update_accumulator_remove(white_acc, pos->king_square[WHITE], captured, to);
        update_accumulator_remove(black_acc, pos->king_square[BLACK], captured, to ^ 56);
    }

    // Note: King moves require full recalculation
}
```

## Forward Pass

After accumulator, run through hidden layers:

```c
int32_t forward_pass(int16_t* white_acc, int16_t* black_acc, int color) {
    // Concatenate accumulators (perspective based on side to move)
    int16_t* first = (color == WHITE) ? white_acc : black_acc;
    int16_t* second = (color == WHITE) ? black_acc : white_acc;

    // Layer 1: Apply clipped ReLU to accumulators
    int8_t hidden1[512];
    for (int i = 0; i < 256; i++) {
        hidden1[i] = clipped_relu(first[i]);
        hidden1[i + 256] = clipped_relu(second[i]);
    }

    // Layer 2: 512 → 32
    int32_t hidden2[32] = {0};
    for (int i = 0; i < 32; i++) {
        for (int j = 0; j < 512; j++) {
            hidden2[i] += hidden1[j] * weights_L2[j][i];
        }
        hidden2[i] = clipped_relu(hidden2[i] >> SHIFT);
    }

    // Output layer: 32 → 1
    int32_t output = 0;
    for (int i = 0; i < 32; i++) {
        output += hidden2[i] * weights_out[i];
    }

    return output / OUTPUT_SCALE;
}

int8_t clipped_relu(int16_t x) {
    if (x < 0) return 0;
    if (x > 127) return 127;
    return (int8_t)x;
}
```

## SIMD Optimization

Neural networks benefit greatly from SIMD (Single Instruction Multiple Data):

```c
#include <immintrin.h>

void accumulator_add_avx2(int16_t* accumulator, const int16_t* weights) {
    for (int i = 0; i < ACCUMULATOR_SIZE; i += 16) {
        __m256i acc = _mm256_load_si256((__m256i*)&accumulator[i]);
        __m256i wgt = _mm256_load_si256((__m256i*)&weights[i]);
        acc = _mm256_add_epi16(acc, wgt);
        _mm256_store_si256((__m256i*)&accumulator[i], acc);
    }
}

void dot_product_avx2(const int8_t* input, const int8_t* weights,
                       int32_t* output, int input_size, int output_size) {
    // Vectorized matrix multiplication
    // ... SIMD implementation
}
```

## Network Training

### Data Collection

Generate training positions from:
- Self-play at various time controls
- Evaluations from strong engines
- Mixture of positions across game phases

```python
# Training data format (example)
# FEN, evaluation in centipawns
position_data = [
    ("rnbqkbnr/pppppppp/8/8/4P3/8/PPPP1PPP/RNBQKBNR b KQkq - 0 1", 25),
    ("rnbqkbnr/pppp1ppp/8/4p3/4P3/5N2/PPPP1PPP/RNBQKB1R b KQkq - 1 2", 15),
    # millions more...
]
```

### Loss Function

```python
def loss(predicted, target):
    # Convert to win probability
    pred_wp = sigmoid(predicted / 410)
    target_wp = sigmoid(target / 410)

    # Cross-entropy loss
    return -(target_wp * log(pred_wp) + (1 - target_wp) * log(1 - pred_wp))
```

### Training Loop

```python
def train_nnue(network, dataset, epochs=100):
    optimizer = Adam(network.parameters(), lr=0.001)

    for epoch in range(epochs):
        for batch in dataset.batches(batch_size=16384):
            positions, targets = batch

            # Forward pass
            predictions = network(positions)

            # Compute loss
            loss = compute_loss(predictions, targets)

            # Backward pass
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()

        print(f"Epoch {epoch}: Loss = {loss.item():.4f}")
```

## Stockfish NNUE

Stockfish's NNUE implementation:

```
Architecture: (HalfKP × 2) → 256 → 32 → 32 → 1

Input features: 40,960 (HalfKP)
First hidden: 256 neurons (per perspective)
Second hidden: 32 neurons
Third hidden: 32 neurons
Output: 1 (evaluation)
```

Network file is ~40MB, loaded at engine startup.

### Integration

```c
int nnue_evaluate(Position* pos) {
    // Check if accumulators need refresh
    if (accumulator_needs_refresh(pos)) {
        refresh_accumulators(pos);
    }

    // Forward pass
    int eval = forward_pass(pos->white_accumulator,
                            pos->black_accumulator,
                            pos->side_to_move);

    // Scale to centipawns
    return eval;
}
```

## Hybrid Evaluation

Combine NN evaluation with hand-crafted terms:

```c
int evaluate(Position* pos) {
    int nn_eval = nnue_evaluate(pos);

    // Add correction terms
    int correction = 0;

    // Material imbalance not well captured by NN
    if (opposite_color_bishops(pos)) {
        correction -= 20;  // Drawish
    }

    // Known theoretical draws
    if (is_kbnk(pos)) {
        return 0;  // Can't mate
    }

    return nn_eval + correction;
}
```

## Pros and Cons

### Advantages

- Stronger evaluation (learns complex patterns)
- Less human knowledge required
- Can capture subtle positional features

### Disadvantages

- Slower than simple hand-crafted evaluation
- Requires significant training infrastructure
- Network can have blind spots
- Harder to debug and understand

## Getting Started

### Using Existing Networks

Download pre-trained networks:
- Stockfish NNUE networks
- Leela Chess Zero weights
- Community-trained networks

### Implementing NNUE

1. Start with simple feedforward network
2. Add incremental accumulator updates
3. Optimize with SIMD
4. Train on self-play data

### Resources

- Stockfish source code
- nnue-pytorch training framework
- Bullet (NNUE training library)

## Summary

Neural network evaluation:

- **NNUE**: Efficiently Updatable Neural Network
- **HalfKP**: Features relative to king position
- **Incremental updates**: Only recompute changed features
- **SIMD**: Essential for performance
- **Training**: Self-play data with gradient descent

NNUE adds 50-100 Elo over traditional evaluation while maintaining reasonable speed.

Next: Parallel search—using multiple CPU cores.
