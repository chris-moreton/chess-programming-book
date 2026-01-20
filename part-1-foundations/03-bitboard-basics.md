# Chapter 3: Bitboard Basics

A **bitboard** is a 64-bit integer where each bit corresponds to one square of the chess board. This seemingly simple idea unlocks extraordinary power: we can answer complex questions about multiple squares simultaneously using bitwise operations.

This chapter introduces bitboards and the operations that make them useful. The next chapter tackles the hardest part: generating attacks for sliding pieces using magic bitboards.

## What is a Bitboard?

A chess board has 64 squares. A 64-bit integer has 64 bits. The correspondence is natural:

```
    a  b  c  d  e  f  g  h
   -------------------------
8 | 56 57 58 59 60 61 62 63 |
7 | 48 49 50 51 52 53 54 55 |
6 | 40 41 42 43 44 45 46 47 |
5 | 32 33 34 35 36 37 38 39 |
4 | 24 25 26 27 28 29 30 31 |
3 | 16 17 18 19 20 21 22 23 |
2 |  8  9 10 11 12 13 14 15 |
1 |  0  1  2  3  4  5  6  7 |
   -------------------------
```

Bit 0 represents a1, bit 7 represents h1, bit 56 represents a8, bit 63 represents h8.

A single bitboard can represent:
- All squares occupied by white pawns
- All squares attacked by black's knights
- All squares on the e-file
- All empty squares
- Any subset of the 64 squares

For example, the starting position's white pawns:

```
    a  b  c  d  e  f  g  h
   -------------------------
8 |  0  0  0  0  0  0  0  0 |
7 |  0  0  0  0  0  0  0  0 |
6 |  0  0  0  0  0  0  0  0 |
5 |  0  0  0  0  0  0  0  0 |
4 |  0  0  0  0  0  0  0  0 |
3 |  0  0  0  0  0  0  0  0 |
2 |  1  1  1  1  1  1  1  1 |
1 |  0  0  0  0  0  0  0  0 |
   -------------------------

Binary: 0000000000000000000000000000000000000000000000001111111100000000
Hex:    0x000000000000FF00
```

## Why Bitboards?

The power of bitboards comes from parallel operations. A single bitwise instruction can:

- **Find all white pieces on the d-file**: `white_pieces & FILE_D`
- **Find squares attacked by both bishops**: `bishop1_attacks & bishop2_attacks`
- **Check if the king is in check**: `(enemy_attacks & king_square) != 0`
- **Count pieces**: `popcount(piece_bitboard)`

These operations take just a few CPU cycles each, regardless of how many squares are involved. Without bitboards, each would require loops.

Modern CPUs are optimized for 64-bit operations. Bitboards exploit this perfectly.

## Representing the Position with Bitboards

A complete position uses multiple bitboards:

```c
struct Position {
    // Piece bitboards (one per piece type per color)
    uint64 white_pawns;
    uint64 white_knights;
    uint64 white_bishops;
    uint64 white_rooks;
    uint64 white_queens;
    uint64 white_king;

    uint64 black_pawns;
    uint64 black_knights;
    uint64 black_bishops;
    uint64 black_rooks;
    uint64 black_queens;
    uint64 black_king;

    // Aggregate bitboards (derived from above)
    uint64 white_pieces;   // All white pieces
    uint64 black_pieces;   // All black pieces
    uint64 all_pieces;     // All pieces (occupancy)

    // State
    int side_to_move;
    int castling_rights;
    int en_passant_square;
    int halfmove_clock;

    // ... hash, etc.
};
```

The aggregate bitboards are computed from the piece bitboards:

```c
void update_aggregates(Position* pos) {
    pos->white_pieces = pos->white_pawns | pos->white_knights |
                        pos->white_bishops | pos->white_rooks |
                        pos->white_queens | pos->white_king;

    pos->black_pieces = pos->black_pawns | pos->black_knights |
                        pos->black_bishops | pos->black_rooks |
                        pos->black_queens | pos->black_king;

    pos->all_pieces = pos->white_pieces | pos->black_pieces;
}
```

Now answering "what piece is on square X?" requires checking multiple bitboards:

```c
int piece_on_square(Position* pos, int sq) {
    uint64 mask = 1ULL << sq;

    if (pos->white_pawns & mask) return WPAWN;
    if (pos->white_knights & mask) return WKNIGHT;
    if (pos->white_bishops & mask) return WBISHOP;
    // ... etc ...
    if (pos->black_king & mask) return BKING;

    return EMPTY;
}
```

This is slightly slower than a board array lookup, but we rarely need to query individual squares. The parallel operations more than compensate.

## Bitwise Operations for Chess

Let's review the bitwise operations and see how they apply to chess.

### AND (&) - Intersection

Returns bits that are set in **both** operands.

```c
// Find white pieces on the fourth rank
uint64 white_on_rank4 = white_pieces & RANK_4;

// Find pawns that can capture to square sq
uint64 mask = 1ULL << sq;
uint64 pawn_attackers = pawn_attack_mask[sq] & enemy_pawns;
```

### OR (|) - Union

Returns bits that are set in **either** operand.

```c
// Combine all white pieces
uint64 white_pieces = pawns | knights | bishops | rooks | queens | king;

// All squares attacked by any piece
uint64 all_attacks = knight_attacks | bishop_attacks | rook_attacks;
```

### XOR (^) - Toggle

Returns bits that differ between operands. Flips specific bits.

```c
// Move a piece from sq1 to sq2 (toggle both squares)
uint64 move_mask = (1ULL << sq1) | (1ULL << sq2);
white_pieces ^= move_mask;

// Zobrist hashing uses XOR extensively (Chapter 7)
hash ^= piece_key[piece][from];
hash ^= piece_key[piece][to];
```

### NOT (~) - Complement

Flips all bits.

```c
// All empty squares
uint64 empty = ~all_pieces;

// All squares NOT on the a-file
uint64 not_a_file = ~FILE_A;
```

### Shift Left (<<) - Move Toward Higher Bits

Shifts all bits toward higher positions (toward rank 8).

```c
// Single white pawn push: move pawns up one rank
uint64 single_push = (white_pawns << 8) & empty;

// Double pawn push from rank 2
uint64 double_push = ((white_pawns & RANK_2) << 16) & empty & (empty << 8);
```

### Shift Right (>>) - Move Toward Lower Bits

Shifts all bits toward lower positions (toward rank 1).

```c
// Single black pawn push
uint64 single_push = (black_pawns >> 8) & empty;

// Find the rank a square is on (0-7)
int rank = sq >> 3;  // Equivalent to sq / 8
```

### Compound Operations

Combinations enable complex queries:

```c
// Squares attacked by white pawns (captures)
uint64 pawn_attacks = ((white_pawns & ~FILE_A) << 7) |  // Left captures
                      ((white_pawns & ~FILE_H) << 9);   // Right captures

// Isolated pawns (no friendly pawns on adjacent files)
uint64 isolated = pawns & ~(file_fill(pawns) >> 1) & ~(file_fill(pawns) << 1);
// (file_fill fills entire files where any pawn exists)
```

## Essential Bitboard Constants

Pre-computed constants simplify many operations:

```c
// Files (columns)
const uint64 FILE_A = 0x0101010101010101ULL;
const uint64 FILE_B = 0x0202020202020202ULL;
const uint64 FILE_C = 0x0404040404040404ULL;
const uint64 FILE_D = 0x0808080808080808ULL;
const uint64 FILE_E = 0x1010101010101010ULL;
const uint64 FILE_F = 0x2020202020202020ULL;
const uint64 FILE_G = 0x4040404040404040ULL;
const uint64 FILE_H = 0x8080808080808080ULL;

// Ranks (rows)
const uint64 RANK_1 = 0x00000000000000FFULL;
const uint64 RANK_2 = 0x000000000000FF00ULL;
const uint64 RANK_3 = 0x0000000000FF0000ULL;
const uint64 RANK_4 = 0x00000000FF000000ULL;
const uint64 RANK_5 = 0x000000FF00000000ULL;
const uint64 RANK_6 = 0x0000FF0000000000ULL;
const uint64 RANK_7 = 0x00FF000000000000ULL;
const uint64 RANK_8 = 0xFF00000000000000ULL;

// Diagonals
const uint64 DIAG_A1H8 = 0x8040201008040201ULL;  // Main diagonal
const uint64 DIAG_A8H1 = 0x0102040810204080ULL;  // Anti-diagonal

// Edges
const uint64 EDGES = FILE_A | FILE_H | RANK_1 | RANK_8;
```

These enable queries like:

```c
// Rooks on open files
for each rook on sq:
    uint64 file = file_of(sq);  // Get the file bitboard for sq's file
    if ((file & all_pawns) == 0) {
        // Rook is on an open file
    }
```

## Fundamental Bit Operations

Several low-level operations appear constantly in bitboard code.

### Setting, Clearing, and Testing Bits

```c
// Set bit at position n
bb |= (1ULL << n);

// Clear bit at position n
bb &= ~(1ULL << n);

// Toggle bit at position n
bb ^= (1ULL << n);

// Test if bit n is set
bool is_set = (bb >> n) & 1;
// Or equivalently:
bool is_set = (bb & (1ULL << n)) != 0;
```

### Population Count (popcount)

Counts the number of set bits. Essential for material counting and evaluation.

```c
// Software implementation
int popcount(uint64 bb) {
    int count = 0;
    while (bb) {
        count++;
        bb &= bb - 1;  // Clear lowest set bit
    }
    return count;
}

// Most CPUs have hardware support - use intrinsics:
// GCC/Clang: __builtin_popcountll(bb)
// MSVC: __popcnt64(bb)
```

The `bb &= bb - 1` trick clears the lowest set bit:
- If `bb = 01011000`, then `bb - 1 = 01010111`
- AND gives `01010000` - lowest 1 bit is gone

### Bit Scan (Finding Set Bits)

Find the index of the lowest (or highest) set bit.

```c
// Find lowest set bit index
int bitscan_forward(uint64 bb) {
    // Software: count trailing zeros
    // Hardware intrinsics:
    // GCC/Clang: __builtin_ctzll(bb)
    // MSVC: _BitScanForward64(&index, bb)
}

// Find highest set bit index
int bitscan_reverse(uint64 bb) {
    // Count leading zeros, then subtract from 63
    // GCC/Clang: 63 - __builtin_clzll(bb)
}
```

These are used to iterate over set bits:

```c
// Process each piece in a bitboard
uint64 pieces = white_knights;
while (pieces) {
    int sq = bitscan_forward(pieces);  // Get next knight square
    generate_knight_moves(sq);
    pieces &= pieces - 1;  // Remove this knight
}
```

### Isolate Lowest Bit

```c
uint64 lowest_bit = bb & -bb;  // Two's complement trick
```

If `bb = 01011000`, then `-bb = 10101000` (two's complement), and the AND gives `00001000` - just the lowest set bit.

## Non-Sliding Piece Attacks

Knights, kings, and pawns have fixed attack patterns that don't depend on other pieces. We precompute attack bitboards for each square.

### Knight Attacks

A knight on e4 attacks d2, f2, c3, g3, c5, g5, d6, f6. This pattern is constant regardless of what other pieces are on the board.

```c
// Precomputed knight attacks for all 64 squares
uint64 KNIGHT_ATTACKS[64];

void init_knight_attacks() {
    for (int sq = 0; sq < 64; sq++) {
        uint64 bb = 1ULL << sq;
        uint64 attacks = 0;

        // Each knight move is some combination of ±1 and ±2 in file/rank
        attacks |= (bb << 17) & ~FILE_A;  // Up 2, right 1
        attacks |= (bb << 15) & ~FILE_H;  // Up 2, left 1
        attacks |= (bb << 10) & ~FILE_AB; // Up 1, right 2
        attacks |= (bb <<  6) & ~FILE_GH; // Up 1, left 2
        attacks |= (bb >> 17) & ~FILE_H;  // Down 2, left 1
        attacks |= (bb >> 15) & ~FILE_A;  // Down 2, right 1
        attacks |= (bb >> 10) & ~FILE_GH; // Down 1, left 2
        attacks |= (bb >>  6) & ~FILE_AB; // Down 1, right 2

        KNIGHT_ATTACKS[sq] = attacks;
    }
}

// Usage: all squares attacked by knight on e4
uint64 attacks = KNIGHT_ATTACKS[E4];
```

The `& ~FILE_A` masks prevent wraparound. A knight on a-file cannot attack to the left.

### King Attacks

Similar approach for the king's 8-direction movement:

```c
uint64 KING_ATTACKS[64];

void init_king_attacks() {
    for (int sq = 0; sq < 64; sq++) {
        uint64 bb = 1ULL << sq;
        uint64 attacks = 0;

        attacks |= (bb << 8);                      // Up
        attacks |= (bb >> 8);                      // Down
        attacks |= (bb << 1) & ~FILE_A;            // Right
        attacks |= (bb >> 1) & ~FILE_H;            // Left
        attacks |= (bb << 9) & ~FILE_A;            // Up-right
        attacks |= (bb << 7) & ~FILE_H;            // Up-left
        attacks |= (bb >> 7) & ~FILE_A;            // Down-right
        attacks |= (bb >> 9) & ~FILE_H;            // Down-left

        KING_ATTACKS[sq] = attacks;
    }
}
```

### Pawn Attacks

Pawns attack diagonally forward. We need separate tables for white and black:

```c
uint64 PAWN_ATTACKS[2][64];  // [color][square]

void init_pawn_attacks() {
    for (int sq = 0; sq < 64; sq++) {
        uint64 bb = 1ULL << sq;

        // White pawns attack up-left and up-right
        PAWN_ATTACKS[WHITE][sq] = ((bb << 7) & ~FILE_H) |
                                   ((bb << 9) & ~FILE_A);

        // Black pawns attack down-left and down-right
        PAWN_ATTACKS[BLACK][sq] = ((bb >> 7) & ~FILE_A) |
                                   ((bb >> 9) & ~FILE_H);
    }
}
```

### Using Attack Tables

With precomputed attacks, many queries become trivial:

```c
// Is the white king in check from black knights?
uint64 king_sq_bb = 1ULL << white_king_square;
if (KNIGHT_ATTACKS[white_king_square] & black_knights) {
    // Yes, at least one black knight attacks the king
}

// All squares attacked by all white knights
uint64 all_knight_attacks = 0;
uint64 knights = white_knights;
while (knights) {
    int sq = bitscan_forward(knights);
    all_knight_attacks |= KNIGHT_ATTACKS[sq];
    knights &= knights - 1;
}

// Squares attacked by white pawns (all pawns simultaneously!)
uint64 pawn_attacks = ((white_pawns & ~FILE_A) << 7) |
                      ((white_pawns & ~FILE_H) << 9);
// This handles ALL white pawns in two shifts and three ANDs!
```

That last example demonstrates bitboard power: computing all pawn attacks simultaneously, regardless of how many pawns exist.

## Generating Pawn Moves

Pawn moves are more complex than attacks because they include non-capturing pushes, double pushes, and promotions.

```c
void generate_white_pawn_moves(Position* pos, MoveList* moves) {
    uint64 pawns = pos->white_pawns;
    uint64 empty = ~pos->all_pieces;
    uint64 enemy = pos->black_pieces;

    // Single pushes (not to promotion rank)
    uint64 single = (pawns << 8) & empty & ~RANK_8;
    while (single) {
        int to = bitscan_forward(single);
        add_move(moves, to - 8, to, QUIET);
        single &= single - 1;
    }

    // Double pushes (from rank 2)
    uint64 double_push = (pawns & RANK_2);
    double_push = (double_push << 8) & empty;
    double_push = (double_push << 8) & empty;
    while (double_push) {
        int to = bitscan_forward(double_push);
        add_move(moves, to - 16, to, DOUBLE_PAWN);
        double_push &= double_push - 1;
    }

    // Captures left (not to promotion rank)
    uint64 capture_left = ((pawns & ~FILE_A) << 7) & enemy & ~RANK_8;
    while (capture_left) {
        int to = bitscan_forward(capture_left);
        add_move(moves, to - 7, to, CAPTURE);
        capture_left &= capture_left - 1;
    }

    // Captures right (not to promotion rank)
    uint64 capture_right = ((pawns & ~FILE_H) << 9) & enemy & ~RANK_8;
    while (capture_right) {
        int to = bitscan_forward(capture_right);
        add_move(moves, to - 9, to, CAPTURE);
        capture_right &= capture_right - 1;
    }

    // Promotions (same logic but to RANK_8)
    // ... (4 promotion types per move: Q, R, B, N)

    // En passant (if available)
    if (pos->en_passant_square != -1) {
        uint64 ep_target = 1ULL << pos->en_passant_square;
        uint64 ep_attackers = PAWN_ATTACKS[BLACK][pos->en_passant_square] & pawns;
        while (ep_attackers) {
            int from = bitscan_forward(ep_attackers);
            add_move(moves, from, pos->en_passant_square, EN_PASSANT);
            ep_attackers &= ep_attackers - 1;
        }
    }
}
```

## What About Sliding Pieces?

Bishops, rooks, and queens are **sliding pieces**: they move along rays until blocked by another piece. Their attacks depend on the current position—a rook on a1 attacks different squares when a pawn is on a4 versus when a4 is empty.

We cannot simply precompute attack tables indexed by square. We also need to consider the **occupancy**—which squares are blocked.

This is the problem **magic bitboards** solve. The next chapter explains this technique in detail.

## Summary

- A bitboard is a 64-bit integer representing a subset of squares
- Bitwise operations enable parallel computation across all squares
- We precompute attack tables for non-sliding pieces (knight, king, pawn)
- Pawn move generation demonstrates bitboard parallelism
- Sliding piece attacks require knowing the occupancy—next chapter

### Key Operations Reference

| Operation | Code | Purpose |
|-----------|------|---------|
| Set bit n | `bb \|= (1ULL << n)` | Place a piece |
| Clear bit n | `bb &= ~(1ULL << n)` | Remove a piece |
| Test bit n | `(bb >> n) & 1` | Check occupancy |
| Clear lowest bit | `bb &= bb - 1` | Iterate over pieces |
| Isolate lowest bit | `bb & -bb` | Get one piece |
| Population count | `popcount(bb)` | Count pieces |
| Bit scan forward | `__builtin_ctzll(bb)` | Find first piece |

Next: Magic Bitboards—the elegant solution to sliding piece attack generation.
