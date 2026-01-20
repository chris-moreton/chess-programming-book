# Chapter 1: Introduction

Chess has captivated humans for over a thousand years. It has also captivated computer scientists since the dawn of computing itself. Building a program that plays chess well requires solving fascinating problems in search, evaluation, and optimization. This book will teach you how.

## A Brief History of Computer Chess

The dream of a chess-playing machine predates electronic computers. In 1770, Wolfgang von Kempelen unveiled "The Turk," a mechanical automaton that appeared to play chess. It toured Europe for decades, defeating Napoleon Bonaparte and Benjamin Franklin. The secret? A human chess master hidden inside the cabinet. The Turk was a hoax, but it planted a seed: could a machine truly think?

### The Pioneers

In 1950, Claude Shannon published "Programming a Computer for Playing Chess," laying the theoretical foundation for computer chess. Shannon described two approaches:

**Type A (Brute Force):** Examine every possible move to a fixed depth, then evaluate the resulting positions. Simple but computationally expensive.

**Type B (Selective):** Use chess knowledge to examine only "interesting" moves, searching deeper in critical lines. More human-like but harder to implement correctly.

Shannon estimated that a typical chess position has about 30 legal moves, and a game lasts about 40 moves per side. To look just 10 moves ahead (5 for each player) requires examining 30^10 = 590 trillion positions. Even modern computers cannot do this in reasonable time. Shannon recognized that brute force alone would not suffice.

In 1951, Alan Turing wrote the first chess program, though no computer existed that could run it. He executed it by hand, taking about 30 minutes per move. The program lost to one of Turing's colleagues, but it played legal chess—a milestone.

### The First Programs

The first program to play a complete game on actual hardware was written at Los Alamos in 1956. It played on a 6x6 board (no bishops) due to memory constraints. In 1958, Alex Bernstein's program at IBM became the first to play standard chess on a computer.

Through the 1960s and 1970s, programs improved steadily. Key innovations included:

- **Alpha-beta pruning** (1958): A way to skip examining positions that cannot affect the final decision. This effectively doubled the search depth achievable in the same time.

- **Iterative deepening** (1960s): Search to depth 1, then depth 2, then depth 3, and so on. Sounds wasteful, but provides crucial time management and move ordering benefits.

- **Transposition tables** (1967): Remember positions you have seen before. Chess positions often repeat via different move orders.

- **Killer moves and history heuristic** (1970s-80s): Remember moves that worked well recently. They often work well again.

### The March Toward Mastery

In 1967, a program called Mac Hack VI became the first to defeat a human in tournament play. By the 1980s, the best programs could compete with strong club players.

The decisive moment came on May 11, 1997, when IBM's Deep Blue defeated World Champion Garry Kasparov in a six-game match. Deep Blue examined 200 million positions per second using specialized hardware. It was the first computer to defeat a reigning world champion under standard time controls.

Today, chess engines running on a smartphone are far stronger than Deep Blue. Stockfish, the leading open-source engine, is rated over 3500 Elo—hundreds of points above any human player. The journey from Shannon's paper to superhuman play took less than 50 years.

### The NNUE Revolution

For decades, engines evaluated positions using hand-crafted rules: count material, assess pawn structure, measure king safety. In 2018, a new approach emerged. Inspired by AlphaZero (DeepMind's self-taught chess system), developers began using neural networks for evaluation.

NNUE (Efficiently Updatable Neural Network) combined the pattern recognition of neural networks with the efficient incremental updates needed for fast search. Stockfish adopted NNUE in 2020, gaining approximately 80 Elo points—equivalent to years of traditional development.

This book covers both approaches: the classical hand-crafted evaluation that forms the foundation, and an introduction to neural network evaluation in the advanced chapters.

## How Chess Engines Work: The Big Picture

A chess engine has two main responsibilities:

1. **Generate legal moves** in any position
2. **Choose the best move** to play

Move generation is straightforward (though tricky to implement efficiently). Choosing the best move is the hard part.

### The Search-Evaluation Partnership

Engines use a technique called **game tree search**. From the current position, consider each legal move. For each move, consider the opponent's responses. For each response, consider your replies. Continue until reaching a certain depth, then evaluate the resulting positions.

```
Current Position
    |
    +-- Move e4
    |     |
    |     +-- Black plays e5
    |     |     |
    |     |     +-- ... evaluate ...
    |     |
    |     +-- Black plays c5
    |           |
    |           +-- ... evaluate ...
    |
    +-- Move d4
          |
          +-- Black plays d5
          |     |
          |     +-- ... evaluate ...
          ...
```

The **search** algorithm explores this tree efficiently. The **evaluation** function scores leaf positions, estimating which side is winning.

This division of labor is powerful:

- Search handles tactics: if I take his queen, he takes mine, then I fork his king and rook...
- Evaluation handles positional factors: pawn structure, piece activity, king safety

Together, they produce moves that are both tactically sound and positionally sensible.

### The Numbers

Consider searching 6 plies deep (3 moves for each side). With roughly 35 legal moves per position, a naive search examines 35^6 ≈ 1.8 billion positions. That is too slow.

Modern engines use **pruning** to skip branches that cannot affect the result. Alpha-beta pruning alone, with good move ordering, reduces the effective branching factor from 35 to about 6. Now 6^6 ≈ 47,000 positions—entirely manageable.

Add transposition tables, null move pruning, late move reductions, and other techniques, and engines routinely search 20-30 plies deep in middlegame positions. Each technique shaves off a bit more of the tree, and the improvements multiply.

## The Rules of Chess (Quick Reference)

This section provides a brief refresher on chess rules. If you are already familiar, skip ahead.

### The Board and Pieces

Chess is played on an 8x8 board. Each player starts with 16 pieces:

- 1 King
- 1 Queen
- 2 Rooks
- 2 Bishops
- 2 Knights
- 8 Pawns

White moves first. Players alternate turns. The goal is to checkmate the opponent's king (trap it with no escape).

### How Pieces Move

**King:** One square in any direction.

**Queen:** Any number of squares horizontally, vertically, or diagonally.

**Rook:** Any number of squares horizontally or vertically.

**Bishop:** Any number of squares diagonally.

**Knight:** An "L" shape: two squares in one direction, then one square perpendicular. Knights can jump over other pieces.

**Pawn:** Forward one square (or two from starting position). Captures diagonally forward. Cannot move backward.

### Special Moves

**Castling:** The king moves two squares toward a rook, and the rook jumps to the other side of the king. Conditions:
- Neither piece has moved before
- No pieces between them
- King is not in check
- King does not pass through or land on an attacked square

**En passant:** When a pawn advances two squares and lands beside an enemy pawn, the enemy pawn may capture it "in passing" as if it had moved only one square. This capture must be made immediately.

**Pawn promotion:** When a pawn reaches the far rank, it must promote to a queen, rook, bishop, or knight.

### Check, Checkmate, and Draws

**Check:** The king is under attack. The player must escape check on their next move.

**Checkmate:** The king is in check with no legal escape. The game ends; the attacking side wins.

**Stalemate:** The player to move has no legal moves but is not in check. The game is drawn.

**Other draws:**
- Threefold repetition (same position three times)
- Fifty-move rule (50 moves without a capture or pawn move)
- Insufficient material (e.g., king vs king)
- Agreement between players

## What Makes Chess Hard for Computers?

Chess is a **perfect information** game—both players see the entire board. Unlike poker, there is no hidden information. Unlike backgammon, there is no randomness. In principle, chess could be "solved" like tic-tac-toe: compute the optimal move in every position.

In practice, the game tree is too large. There are approximately 10^44 legal chess positions. If every atom in the observable universe were a computer, and each computer examined a billion positions per second since the Big Bang, we would have examined about 10^50 positions—barely scratching the surface.

### The Key Challenges

**Combinatorial explosion:** The number of positions grows exponentially with search depth. Searching one ply deeper multiplies the work by roughly 35.

**Evaluation difficulty:** How much is a bishop worth? It depends on the position. Is a passed pawn on the 6th rank worth a piece? Sometimes. Encoding chess knowledge into numbers is subtle.

**Tactical complexity:** Chess has forcing sequences—check, capture, threat—where one wrong move loses instantly. Missing a tactic 15 moves deep can mean losing a won game.

**Strategic depth:** Long-term plans may not pay off for many moves. Weakening your pawn structure might be fatal in the endgame but irrelevant if you checkmate first.

**The horizon effect:** Limited search depth means engines can be blind to events just beyond their horizon. If losing a piece is inevitable but can be delayed, a naive engine might keep delaying rather than finding the best defense.

These challenges have driven decades of research. The techniques in this book represent the accumulated wisdom of thousands of programmers and millions of games.

## Engine Architecture Overview

A complete chess engine consists of several interconnected components:

```
┌─────────────────────────────────────────────────────────────┐
│                         UCI Interface                        │
│                  (Communication with GUI)                    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                        Time Manager                          │
│               (Decides how long to search)                   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                          Search                              │
│     (Alpha-beta, iterative deepening, pruning, etc.)        │
└─────────────────────────────────────────────────────────────┘
           │                                    │
           ▼                                    ▼
┌──────────────────────┐          ┌────────────────────────────┐
│   Move Generator     │          │     Transposition Table    │
│ (Legal moves in any  │          │   (Cache of seen positions)│
│      position)       │          │                            │
└──────────────────────┘          └────────────────────────────┘
           │
           ▼
┌──────────────────────┐          ┌────────────────────────────┐
│  Board Representation│          │        Evaluation          │
│   (Stores position)  │◄────────►│   (Scores a position)      │
└──────────────────────┘          └────────────────────────────┘
```

### Board Representation

The foundation. We need an efficient way to represent:
- Which pieces are on which squares
- Whose turn it is
- Castling rights
- En passant target square
- Move counters for draw rules

Chapter 2 covers array-based representations. Chapters 3-4 cover bitboards, the modern standard.

### Move Generation

Given a position, generate all legal moves. This must be fast—engines generate moves millions of times per second.

Chapters 5-7 cover move generation, from basic algorithms to incremental update techniques.

### Search

The core algorithm that explores the game tree. Starting from Chapter 8, we build up from naive minimax to a fully-featured alpha-beta search with all the modern enhancements.

### Evaluation

When search reaches a leaf node, evaluation scores the position. Chapters 20-25 cover material, piece-square tables, pawn structure, and more.

### Transposition Table

A hash table storing previously evaluated positions. When we reach a position we have seen before (via a different move order), we can reuse the stored result. Chapter 11 covers implementation.

### UCI Interface

The Universal Chess Interface protocol lets engines communicate with chess GUIs. Chapter 26 covers the protocol and implementation.

### Time Management

In timed games, the engine must decide how long to think. Too fast and you miss good moves. Too slow and you run out of time. Chapter 27 covers time allocation strategies.

## What This Book Will Teach You

By the end of this book, you will understand:

- How to represent chess positions efficiently using bitboards
- How magic bitboards enable fast sliding piece move generation
- How to generate legal moves in any position
- How the minimax algorithm and alpha-beta pruning work
- How transposition tables, killer moves, and history heuristics speed up search
- How quiescence search prevents tactical blunders
- How null move pruning, LMR, and other techniques enable deeper search
- How to evaluate positions using material, piece-square tables, and structural features
- How to implement the UCI protocol
- How to test your engine for correctness
- How to tune engine parameters using SPSA
- How modern NNUE evaluation works at a high level

More importantly, you will understand *why* each technique works and how the pieces fit together. Chess engine development is a rich field with decades of accumulated wisdom. This book distills that wisdom into a coherent narrative.

## A Note on Implementation

This book uses C-style pseudocode for clarity. The algorithms translate readily to any language. The author's engine, Rusty Rival, is written in Rust. Other popular implementation languages include C, C++, and Java.

When starting out, prioritize correctness over performance. A slow engine that plays legal chess is infinitely better than a fast engine that makes illegal moves or crashes. Optimization comes later—and will make more sense once you understand what you are optimizing.

Let's begin.
