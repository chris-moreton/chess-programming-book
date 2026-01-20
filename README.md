# Fundamentals of Chess Programming

A comprehensive guide to building a chess engine from scratch.

## About This Book

This book takes you through the complete journey of building a competitive chess engine. Starting from basic board representation, we progress through move generation, search algorithms, position evaluation, and modern techniques like NNUE neural networks.

The focus is on practical implementation. Every concept is illustrated with C-style pseudocode that can be adapted to any programming language. We explain not just *what* to implement, but *why* each technique works and how the pieces fit together.

## Who This Book Is For

- Programmers curious about how chess engines work
- Developers wanting to build their own engine
- Anyone interested in game tree search, optimization, or AI techniques
- Chess players who want to understand what happens "under the hood"

**Prerequisites:**
- Comfortable reading C-style code (loops, arrays, structs, bitwise operations)
- Basic understanding of recursion
- No chess expertise required (rules are covered in Chapter 1)

## How to Read This Book

The chapters build on each other. Part 1 (Foundations) is essential reading. After that, you can explore Parts 2-4 in order, or jump to specific topics that interest you.

Each chapter includes:
- Conceptual explanation of the technique
- Pseudocode implementation
- Diagrams where helpful
- Performance considerations
- Common pitfalls to avoid

## Book Structure

### Part 1: Foundations
Core concepts: board representation, bitboards, and magic bitboard move generation.

### Part 2: Move Generation
Generating legal moves efficiently for all piece types.

### Part 3: Search
The heart of the engine: minimax, alpha-beta pruning, and the many techniques that make search fast.

### Part 4: Evaluation
Teaching the engine to assess positions: material, piece-square tables, pawn structure, king safety.

### Part 5: Engineering
Practical concerns: UCI protocol, time management, transposition tables, debugging.

### Part 6: Testing and Tuning
Ensuring correctness and optimizing parameters with techniques like SPSA.

### Part 7: Advanced Topics
Modern developments: NNUE neural networks, MCTS, and where the field is heading.

## Code Examples

All code in this book uses C-style pseudocode:

```c
// Example: counting set bits in a bitboard
int popcount(uint64 bb) {
    int count = 0;
    while (bb) {
        count++;
        bb &= bb - 1;  // Clear lowest set bit
    }
    return count;
}
```

The pseudocode prioritizes clarity over micro-optimization. Production implementations may differ in details but the algorithms remain the same.

## Companion Code

The author's chess engine, [Rusty Rival](https://github.com/chris-moreton/rusty-rival), implements all techniques described in this book. Written in Rust, it serves as a reference implementation you can study alongside the text.

## License

This work is licensed under [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) - you may share and adapt it for non-commercial purposes with attribution.

## Contributing

Found an error? Have a suggestion? Please open an issue or pull request on GitHub.
