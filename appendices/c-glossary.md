# Appendix C: Glossary

Common terms used in chess programming.

## A

**Alpha**: The lower bound in alpha-beta search. The best score the maximizing player is assured of.

**Alpha-Beta**: Pruning algorithm that eliminates branches that cannot affect the final decision.

**Aspiration Window**: A narrow alpha-beta window around the expected score to speed up search.

**Attack Table**: Precomputed table of squares attacked by a piece from each position.

## B

**Beta**: The upper bound in alpha-beta search. The best score the minimizing player is assured of.

**Best Move**: The move chosen as optimal after search completes.

**Bitboard**: A 64-bit integer where each bit represents a square on the board.

**Blocker**: A piece that stops a sliding piece's ray.

**Branch Factor**: Average number of children per node in the game tree. Chess averages ~35.

**Butterfly Board**: A 64×64 array indexed by [from][to] squares, used for history heuristic.

## C

**Castling**: Special king move involving the rook. Kingside (O-O) or queenside (O-O-O).

**Centipawn**: One hundredth of a pawn's value. Standard unit for evaluation scores.

**Check**: When the king is under attack.

**Check Extension**: Extending search depth when the position is in check.

**Checkmate**: When the king is in check and has no legal escape. Game over.

**Cutoff**: When alpha-beta can stop searching remaining moves (beta cutoff).

## D

**Depth**: How many plies (half-moves) the search looks ahead.

**Draw**: Game result where neither side wins. Caused by stalemate, repetition, 50-move rule, or insufficient material.

**Dynamic Evaluation**: Evaluation that considers search-derived features (threats, tactics).

## E

**Elo**: Rating system measuring relative playing strength. A 100-point difference means ~64% expected score.

**En Passant**: Special pawn capture of an adjacent pawn that just moved two squares.

**Endgame**: Late game phase with few pieces remaining.

**EPD**: Extended Position Description. FEN plus optional commands (best move, comments).

**Evaluation**: Function that estimates which side is winning and by how much.

**Extension**: Increasing search depth in certain positions (checks, singular moves, etc.).

## F

**FEN**: Forsyth-Edwards Notation. Standard format for describing chess positions.

**File**: A vertical column on the board (a through h).

**Futility Pruning**: Skipping moves that have no chance of raising alpha.

## G

**Game Phase**: Classification of position as opening, middlegame, or endgame.

**GUI**: Graphical User Interface. Programs like Arena, Cutechess, Fritz that display chess games.

## H

**Half-Move Clock**: Count of moves since last pawn move or capture. For 50-move rule.

**Hash**: Unique identifier for a position. See Zobrist hashing.

**History Heuristic**: Move ordering based on how often moves caused cutoffs in the past.

**Horizon Effect**: Missing important events because they occur beyond search depth.

## I

**Incremental Update**: Updating data structures partially rather than recomputing from scratch.

**Iterative Deepening**: Searching to depth 1, then 2, then 3, etc., using each result to improve the next.

## K

**Killer Move**: A move that caused a beta cutoff at the same depth in a sibling node.

**King Safety**: Evaluation of how exposed/protected the king is.

## L

**Late Move Reductions (LMR)**: Searching later moves (likely bad) to reduced depth.

**Legal Move**: A move that doesn't leave the king in check.

**LERF**: Little-Endian Rank-File mapping. Common square numbering (a1=0, h8=63).

## M

**Magic Bitboard**: Technique using perfect hashing to compute sliding piece attacks.

**Make Move**: Apply a move to the position, updating all state.

**Mate Distance Pruning**: Pruning when a mate has been found closer to the root.

**Material**: Piece count. Usually measured in centipawns.

**Middlegame**: Middle phase of the game with most pieces still on the board.

**Minimax**: Algorithm where one player maximizes score and the other minimizes.

**Mobility**: Number of legal moves a piece has. Often used in evaluation.

**Move Ordering**: Sorting moves to search likely-best moves first for better pruning.

**MVV-LVA**: Most Valuable Victim - Least Valuable Attacker. Capture ordering heuristic.

## N

**Negamax**: Minimax variant where score is always from current player's perspective.

**NNUE**: Efficiently Updatable Neural Network. Neural network evaluation with incremental updates.

**Node**: A position in the search tree.

**NPS**: Nodes Per Second. Measure of search speed.

**Null Move**: A "pass" move used for null move pruning.

**Null Move Pruning**: Giving opponent a free move; if still failing high, position is good enough to prune.

**Null Window**: Search window with beta = alpha + 1. Used in PVS.

## O

**Opening**: Early phase of the game. Often played from book.

**Opening Book**: Database of known good opening moves.

## P

**Passed Pawn**: A pawn with no enemy pawns blocking or attacking its advance.

**Perft**: Performance Test. Counts leaf nodes at given depth to verify move generation.

**Piece-Square Table (PST)**: Table giving bonus/penalty for each piece on each square.

**Pin**: A piece that cannot move without exposing a more valuable piece to attack.

**Ply**: One half-move. A move by one player.

**Pondering**: Thinking during opponent's time.

**Principal Variation (PV)**: The expected line of best play for both sides.

**Promotion**: A pawn reaching the 8th rank and becoming another piece.

**Pruning**: Eliminating branches of the search tree without fully searching them.

**Pseudo-Legal Move**: A move that follows piece movement rules but might leave the king in check.

**PVS**: Principal Variation Search. Searches first move fully, remaining moves with null window.

## Q

**Quiescence Search**: Search of captures/checks until position is "quiet."

## R

**Rank**: A horizontal row on the board (1 through 8).

**Razoring**: Pruning at low depths when static evaluation is far below alpha.

**Repetition**: Same position occurring multiple times. Three-fold repetition is a draw.

**Root**: The starting position of a search.

**Reduction**: Searching a move to lesser depth than normal.

## S

**Score**: Numerical evaluation of a position. Positive favors white, negative favors black.

**SEE**: Static Exchange Evaluation. Quick estimate of capture sequence outcome.

**Seldepth**: Selective depth. Maximum depth reached including extensions and quiescence.

**Singular Extension**: Extending search when one move is significantly better than alternatives.

**Sliding Piece**: Bishop, rook, or queen. Pieces that move along rays.

**SPRT**: Sequential Probability Ratio Test. Statistical test for engine testing.

**Stalemate**: When a player has no legal moves but is not in check. Draw.

**Static Evaluation**: Position evaluation without search.

## T

**Tablebase**: Database of perfect endgame analysis for positions with few pieces.

**Tapered Evaluation**: Blending middlegame and endgame evaluations based on game phase.

**TT**: Transposition Table. Hash table storing previously searched positions.

**Transposition**: Reaching the same position via different move sequences.

## U

**UCI**: Universal Chess Interface. Standard protocol for engine-GUI communication.

**Unmake Move**: Reverse a move, restoring the previous position.

## W

**Window**: The alpha-beta bounds within which the search operates.

## Z

**Zobrist Hashing**: Creating a unique hash by XORing random values for each piece-square.

**Zugzwang**: Position where any move worsens the position. Common in endgames.
