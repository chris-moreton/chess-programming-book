# Table of Contents

## [Part 1: Foundations](part-1-foundations/)

- [Chapter 1: Introduction](part-1-foundations/01-introduction.md)
  - A Brief History of Computer Chess
  - How Chess Engines Work: The Big Picture
  - The Rules of Chess (Quick Reference)
  - What Makes Chess Hard for Computers?
  - Engine Architecture Overview

- [Chapter 2: Board Representation](part-1-foundations/02-board-representation.md)
  - Array-Based Representations
  - The 0x88 Board
  - Piece Lists
  - Hybrid Approaches
  - Choosing Your Representation

- [Chapter 3: Bitboard Basics](part-1-foundations/03-bitboard-basics.md)
  - What is a Bitboard?
  - Bitwise Operations for Chess
  - Representing the Board with Bitboards
  - Basic Bitboard Patterns
  - Non-Sliding Piece Attacks

- [Chapter 4: Magic Bitboards](part-1-foundations/04-magic-bitboards.md)
  - The Problem: Sliding Piece Attacks
  - Occupancy and Blocker Sets
  - Perfect Hashing with Magic Numbers
  - Finding Magic Numbers
  - Implementation Details
  - Fancy vs Plain Magic Bitboards

## [Part 2: Move Generation](part-2-move-generation/)

- [Chapter 5: Legal Move Generation](part-2-move-generation/05-legal-moves.md)
  - Pseudo-Legal vs Legal Moves
  - Generating Moves by Piece Type
  - Check Detection
  - Pin Detection
  - Castling Rules and Validation
  - En Passant Edge Cases

- [Chapter 6: Move Representation](part-2-move-generation/06-move-representation.md)
  - Encoding Moves Compactly
  - Move Flags (Castling, En Passant, Promotion)
  - Move Lists and Memory Management
  - Decoding and Displaying Moves

- [Chapter 7: Incremental Updates](part-2-move-generation/07-incremental-updates.md)
  - Make/Unmake Move
  - Zobrist Hashing
  - Incremental Hash Updates
  - Updating Attack Tables
  - Copy-Make vs Make-Unmake Tradeoffs

## [Part 3: Search](part-3-search/)

- [Chapter 8: Minimax and Game Trees](part-3-search/08-minimax.md)
  - Game Trees and Chess
  - The Minimax Algorithm
  - Negamax Formulation
  - Complexity and the Branching Factor

- [Chapter 9: Alpha-Beta Pruning](part-3-search/09-alpha-beta.md)
  - The Key Insight
  - Alpha-Beta Algorithm
  - Move Ordering Matters
  - Fail-Hard vs Fail-Soft
  - Theoretical Speedup

- [Chapter 10: Iterative Deepening](part-3-search/10-iterative-deepening.md)
  - Why Search the Same Position Multiple Times?
  - Implementation
  - Time Management Integration
  - Using Previous Iterations

- [Chapter 11: Transposition Tables](part-3-search/11-transposition-tables.md)
  - Why Positions Repeat
  - Hash Table Structure
  - Zobrist Keys
  - Handling Collisions
  - Replacement Strategies
  - Extracting the Principal Variation

- [Chapter 12: Move Ordering](part-3-search/12-move-ordering.md)
  - Why Move Order Matters
  - Hash Move
  - Captures and MVV-LVA
  - Killer Moves
  - History Heuristic
  - Countermove Heuristic
  - Staged Move Generation

- [Chapter 13: Quiescence Search](part-3-search/13-quiescence.md)
  - The Horizon Effect
  - Searching Only Captures
  - Standing Pat
  - Delta Pruning
  - SEE Pruning
  - Limiting Quiescence Depth

- [Chapter 14: Null Move Pruning](part-3-search/14-null-move.md)
  - The Idea: Passing Your Turn
  - Implementation
  - Reduction Depth
  - When Not to Use Null Move
  - Verification Search

- [Chapter 15: Late Move Reductions](part-3-search/15-lmr.md)
  - The Observation
  - Basic LMR
  - Reduction Formulas
  - Conditions and Exceptions
  - Re-Search on Failure

- [Chapter 16: Pruning Techniques](part-3-search/16-pruning.md)
  - Futility Pruning
  - Reverse Futility Pruning
  - Razoring
  - Late Move Pruning
  - SEE Pruning for Quiet Moves
  - Multi-Cut Pruning

- [Chapter 17: Extensions](part-3-search/17-extensions.md)
  - Check Extensions
  - Singular Extensions
  - Passed Pawn Extensions
  - Recapture Extensions
  - Extension Limits

- [Chapter 18: Aspiration Windows](part-3-search/18-aspiration-windows.md)
  - Narrowing the Search Window
  - Handling Failures
  - Window Sizing
  - Integration with Iterative Deepening

- [Chapter 19: Principal Variation Search](part-3-search/19-pvs.md)
  - The PVS Algorithm
  - Zero-Window Searches
  - When to Use Full Window
  - PVS vs Standard Alpha-Beta

## [Part 4: Evaluation](part-4-evaluation/)

- [Chapter 20: Evaluation Basics](part-4-evaluation/20-eval-basics.md)
  - What Are We Evaluating?
  - Material Counting
  - Piece Values
  - The Evaluation Function Interface

- [Chapter 21: Piece-Square Tables](part-4-evaluation/21-piece-square-tables.md)
  - Positional Value of Squares
  - Designing PSTs for Each Piece
  - Middlegame vs Endgame Tables
  - Tapered Evaluation
  - Incremental PST Updates

- [Chapter 22: Pawn Structure](part-4-evaluation/22-pawn-structure.md)
  - Doubled Pawns
  - Isolated Pawns
  - Backward Pawns
  - Passed Pawns
  - Pawn Chains
  - Caching Pawn Evaluation

- [Chapter 23: Piece Evaluation](part-4-evaluation/23-piece-evaluation.md)
  - Mobility
  - Outposts
  - Rooks on Open Files
  - Bishop Pair
  - Trapped Pieces
  - Coordination

- [Chapter 24: King Safety](part-4-evaluation/24-king-safety.md)
  - Pawn Shield
  - Attacking Units Near the King
  - King Exposure
  - Virtual Mobility
  - Castling Considerations

- [Chapter 25: Endgame Evaluation](part-4-evaluation/25-endgame.md)
  - When is it an Endgame?
  - King Centralization
  - Passed Pawn Advancement
  - Mating Patterns
  - Insufficient Material
  - Tablebase Integration

## [Part 5: Engineering](part-5-engineering/)

- [Chapter 26: UCI Protocol](part-5-engineering/26-uci.md)
  - Protocol Overview
  - Required Commands
  - Position and Move Parsing
  - Search Control
  - Engine Options
  - Common Pitfalls

- [Chapter 27: Time Management](part-5-engineering/27-time-management.md)
  - Time Controls
  - Allocating Time per Move
  - Soft and Hard Limits
  - Sudden Death vs Increment
  - When to Think Longer
  - Handling Time Pressure

- [Chapter 28: Debugging Techniques](part-5-engineering/28-debugging.md)
  - Perft Testing
  - Divide and Conquer
  - Assertion Strategies
  - Reproducible Bugs
  - Common Bug Patterns
  - Logging and Analysis

## [Part 6: Testing and Tuning](part-6-testing/)

- [Chapter 29: Testing Your Engine](part-6-testing/29-testing.md)
  - Unit Tests
  - Perft Suites
  - Test Positions (WAC, STS, etc.)
  - Engine vs Engine Matches
  - Statistical Significance
  - Regression Testing

- [Chapter 30: Parameter Tuning](part-6-testing/30-tuning.md)
  - What to Tune
  - Manual Tuning
  - Texel Tuning
  - SPSA (Simultaneous Perturbation Stochastic Approximation)
  - Distributed Tuning
  - Avoiding Overfitting

## [Part 7: Advanced Topics](part-7-advanced/)

- [Chapter 31: Neural Network Evaluation (NNUE)](part-7-advanced/31-nnue.md)
  - From Handcrafted to Learned
  - NNUE Architecture Overview
  - Efficiently Updatable Networks
  - Training Data Generation
  - Integration with Search
  - The Future of Evaluation

- [Chapter 32: Alternative Approaches](part-7-advanced/32-alternatives.md)
  - Monte Carlo Tree Search
  - AlphaZero and Self-Play
  - Hybrid Approaches
  - Hardware Considerations
  - Where the Field is Heading

## Appendices

- [Appendix A: FEN Notation](appendices/appendix-a-fen.md)
- [Appendix B: Algebraic Notation](appendices/appendix-b-notation.md)
- [Appendix C: Perft Results](appendices/appendix-c-perft.md)
- [Appendix D: Resources and Further Reading](appendices/appendix-d-resources.md)
