# Appendix D: Resources

A curated collection of resources for chess programming.

## Websites

### Chess Programming Wiki

**https://www.chessprogramming.org/**

The definitive reference for chess programming. Covers every topic in depth with code examples and historical context.

Key pages:
- [Bitboards](https://www.chessprogramming.org/Bitboards)
- [Magic Bitboards](https://www.chessprogramming.org/Magic_Bitboards)
- [Alpha-Beta](https://www.chessprogramming.org/Alpha-Beta)
- [Transposition Table](https://www.chessprogramming.org/Transposition_Table)
- [Evaluation](https://www.chessprogramming.org/Evaluation)

### TalkChess Forum

**https://talkchess.com/**

Active community of chess programmers. Great for questions and discussions.

### Computer Chess Club Archives

Historical discussions from pioneering chess programmers.

## Open Source Engines

### Stockfish

**https://github.com/official-stockfish/Stockfish**

The strongest open-source engine. Excellent code quality and documentation.

Study for:
- Search techniques (LMR, null move, extensions)
- NNUE implementation
- Parallel search (Lazy SMP)
- Move generation

### Leela Chess Zero

**https://github.com/LeelaChessZero/lc0**

AlphaZero-style engine using Monte Carlo Tree Search and neural networks.

Study for:
- Neural network evaluation
- MCTS instead of alpha-beta
- Training infrastructure

### Ethereal

**https://github.com/AndyGrant/Ethereal**

Strong engine with clean, educational code.

Study for:
- Classical evaluation
- Tuning methodology (Texel tuning)
- Clean implementation patterns

### Rustic

**https://github.com/mvanthoor/rustic**

Educational chess engine in Rust with detailed documentation.

Study for:
- Step-by-step engine development
- Rust-specific patterns
- Accompanying tutorial series

### Vice

**https://github.com/bluefeversoft/vice**

Chess engine with video tutorial series by BlueFever Software.

Study for:
- Beginner-friendly explanations
- Video walkthrough of development

### Crafty

**https://craftychess.com/**

Historical importance—one of the first strong open-source engines.

## Books

### "Computer Chess Compendium" by David Levy

Classic collection of papers on computer chess history and techniques.

### "Chess Programming Theory" by Frantisek Franek

Academic treatment of chess programming algorithms.

### "How Computers Play Chess" by David Levy and Monty Newborn

Accessible introduction to the field.

## Test Suites

### Strategic Test Suite (STS)

**https://github.com/fsmosca/STS-Rating**

1500 positions testing various strategic concepts.

### Win at Chess (WAC)

300 tactical positions from Fred Reinfeld's book.

### Eigenmann Rapid Engine Test (ERET)

111 positions specifically for engine testing.

### Bratko-Kopec Test

24 positions covering various chess concepts.

## Tools

### Cutechess

**https://github.com/cutechess/cutechess**

GUI and CLI for running engine matches.

```bash
# Example match
cutechess-cli \
    -engine name=Engine1 cmd=./engine1 \
    -engine name=Engine2 cmd=./engine2 \
    -each tc=60+0.6 proto=uci \
    -games 1000 \
    -openings file=book.pgn format=pgn
```

### Arena

**http://www.playwitharena.de/**

Free chess GUI for Windows. Good for testing and debugging.

### Fritz / ChessBase

Commercial chess software with powerful analysis tools.

### python-chess

**https://github.com/niklasf/python-chess**

Python library for chess. Useful for:
- Generating training data
- Analysis scripts
- PGN processing

### Syzygy Tablebases

**https://syzygy-tables.info/**

Free endgame tablebases for up to 7 pieces.

## Datasets

### CCRL

**https://ccrl.chessdom.com/**

Computer Chess Rating Lists. Engine vs engine results.

### TCEC

**https://tcec-chess.com/**

Top Chess Engine Championship. Elite engine competition.

### Lichess Database

**https://database.lichess.org/**

Millions of games for training and analysis.

## Academic Papers

### "Deep Blue" (1997)

IBM's documentation on the Deep Blue project.

### "AlphaZero" (2017)

DeepMind's paper on mastering chess through self-play.

### "Efficiently Updatable Neural-Network-based Evaluation Functions" (2018)

NNUE paper by Yu Nasu.

## YouTube Channels

### Chess Programming

Sebastian Lague's chess programming video series—excellent visualization.

### BlueFever Software

Vice chess engine tutorial series with step-by-step development.

### Maksim Korzh

BBC chess engine series (simple bitboard engine).

## Tutorials

### Chess Programming Wiki Tutorials

Step-by-step guides on various topics.

### "Writing a Chess Engine" by Tom Kerrigan

Classic tutorial using the TSCP engine.

### "How to Write a Chess Engine"

Various blog series covering engine development.

## Communities

### r/chessprogramming

Reddit community for chess programming discussions.

### Discord Servers

- Stockfish Discord
- Chess Programming Discord

## Benchmarking

### Fishtest

**https://tests.stockfishchess.org/**

Distributed testing framework for Stockfish. Study for:
- SPRT implementation
- Patch testing methodology

### OpenBench

**https://github.com/AndyGrant/OpenBench**

Open-source distributed testing framework.

## Rating Lists

### CCRL (Computer Chess Rating Lists)

**https://ccrl.chessdom.com/**

Regular testing across thousands of engines.

### CEGT

**https://www.cegt.net/**

Chess Engines Grand Tournament.

### Stefan Pohl Computer Chess

**https://www.sp-cc.de/**

SPCC rating list with various time controls.

## Further Study

### Move Generation

- Study Stockfish's move generator
- Implement and test with perft

### Evaluation

- Read Stockfish evaluation code
- Study NNUE architecture
- Experiment with Texel tuning

### Search

- Implement basic alpha-beta
- Add iterative deepening
- Study LMR and pruning techniques

### Testing

- Set up automated testing
- Use SPRT for changes
- Join testing communities

## Getting Help

1. **Check the Chess Programming Wiki first**—most questions are answered there
2. **Search TalkChess archives**—your question has likely been asked
3. **Post minimal examples**—when asking questions, include code and test cases
4. **Be specific**—"my engine is slow" vs "my perft speed is 2M nodes/sec at depth 6"

## Contributing Back

Once you've learned:
- Contribute to open-source engines
- Improve the Chess Programming Wiki
- Answer questions on forums
- Share your engine and learnings

Good luck with your chess engine!
