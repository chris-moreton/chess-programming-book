# Chapter 2: Board Representation

Before we can generate moves or evaluate positions, we need a way to represent the chess board in memory. This seemingly simple problem has several solutions, each with different tradeoffs.

The choice of board representation affects everything else: how we generate moves, how we detect checks, how we evaluate positions. Choose wisely, and your engine will be fast and clean. Choose poorly, and you will fight your data structures at every turn.

## Goals of a Board Representation

A good board representation should support:

1. **Answering "what piece is on square X?"** - needed for move generation, evaluation, display
2. **Answering "where are all pieces of type Y?"** - needed for generating moves of that piece type
3. **Making and unmaking moves efficiently** - done millions of times per search
4. **Incremental updates** - updating derived data (hash keys, piece lists) as moves are made

Let's examine several approaches.

## Array-Based Representations

The simplest approach: an array of 64 elements, one per square.

### The 64-Square Array

```c
// Piece values
#define EMPTY   0
#define WPAWN   1
#define WKNIGHT 2
#define WBISHOP 3
#define WROOK   4
#define WQUEEN  5
#define WKING   6
#define BPAWN   7
#define BKNIGHT 8
#define BBISHOP 9
#define BROOK   10
#define BQUEEN  11
#define BKING   12

// The board
int board[64];

// Square indexing (one option):
// a1=0, b1=1, ..., h1=7
// a2=8, b2=9, ..., h2=15
// ...
// a8=56, b8=57, ..., h8=63
```

To answer "what piece is on e4?", compute the index and look it up:

```c
int file = 4;  // e = file index 4 (a=0, b=1, ...)
int rank = 3;  // rank 4 = index 3 (rank 1 = index 0)
int square = rank * 8 + file;  // = 28
int piece = board[square];     // O(1) lookup
```

To answer "where are all white knights?", we must scan the entire board:

```c
for (int sq = 0; sq < 64; sq++) {
    if (board[sq] == WKNIGHT) {
        // Found a white knight on square sq
    }
}
```

This O(64) scan is inefficient. In move generation, we need to find all pieces of each type repeatedly. Scanning 64 squares every time adds up.

### Square Indexing Conventions

There are two common ways to number squares:

**Little-endian rank-file (LERF):**
- a1 = 0, h1 = 7
- a8 = 56, h8 = 63
- Index = rank × 8 + file

**Big-endian rank-file (BERF):**
- a8 = 0, h8 = 7
- a1 = 56, h1 = 63
- Index = (7 - rank) × 8 + file

Either works. This book uses LERF, matching most modern engines.

### The 0x88 Board

A clever variation uses a 128-element array (16×8) instead of 64 elements. Only half the array represents the actual board; the other half is "off-board."

```
  +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
7 | 70| 71| 72| 73| 74| 75| 76| 77| 78| 79| 7A| 7B| 7C| 7D| 7E| 7F|
  +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
6 | 60| 61| 62| 63| 64| 65| 66| 67| 68| 69| 6A| 6B| 6C| 6D| 6E| 6F|
  +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
5 | 50| 51| 52| 53| 54| 55| 56| 57| 58| 59| 5A| 5B| 5C| 5D| 5E| 5F|
  +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
4 | 40| 41| 42| 43| 44| 45| 46| 47| 48| 49| 4A| 4B| 4C| 4D| 4E| 4F|
  +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
3 | 30| 31| 32| 33| 34| 35| 36| 37| 38| 39| 3A| 3B| 3C| 3D| 3E| 3F|
  +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
2 | 20| 21| 22| 23| 24| 25| 26| 27| 28| 29| 2A| 2B| 2C| 2D| 2E| 2F|
  +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
1 | 10| 11| 12| 13| 14| 15| 16| 17| 18| 19| 1A| 1B| 1C| 1D| 1E| 1F|
  +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
0 | 00| 01| 02| 03| 04| 05| 06| 07| 08| 09| 0A| 0B| 0C| 0D| 0E| 0F|
  +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
    a    b    c    d    e    f    g    h   (off-board region)
```

The left 8 columns (0x00-0x07, 0x10-0x17, etc.) represent the real board. The right 8 columns are the off-board region.

The magic property: **a square is valid if and only if `(square & 0x88) == 0`**.

Consider the hexadecimal values:
- Valid squares: 0x00-0x07, 0x10-0x17, 0x20-0x27, etc.
- Invalid squares: 0x08-0x0F, 0x18-0x1F, 0x28-0x2F, etc.

The bit pattern 0x88 (binary: 10001000) has bit 3 and bit 7 set. Valid squares never have these bits set. Invalid squares always have at least one set.

Why is this useful? Move generation:

```c
// Direction offsets for a knight
int knight_offsets[] = {-33, -31, -18, -14, 14, 18, 31, 33};

// Generate knight moves from square sq
for (int i = 0; i < 8; i++) {
    int target = sq + knight_offsets[i];

    // Off-board check - single bitwise AND!
    if (target & 0x88) continue;

    // target is a valid square - check if capture or empty
    if (board[target] is empty or enemy) {
        add_move(sq, target);
    }
}
```

With a standard 64-square array, off-board detection requires range checking:

```c
// Messier off-board check
if (target < 0 || target > 63) continue;
// Also need to check wrap-around (e.g., h1 to a2 is not valid for knight)
```

The 0x88 trick eliminates boundary checks with a single bitwise AND. For sliding pieces (rooks, bishops, queens), this simplifies the inner loop considerably.

### Advantages of 0x88

1. **Simple off-board detection**: `(sq & 0x88) != 0`
2. **Easy rank/file extraction**: `rank = sq >> 4`, `file = sq & 7`
3. **Distance calculations**: The difference between two 0x88 squares uniquely identifies the relationship (same rank, same file, same diagonal)

### Disadvantages of 0x88

1. **Uses more memory**: 128 bytes vs 64 bytes per board
2. **Still O(64) to find all pieces of a type**
3. **Cannot use bitwise tricks that bitboards enable**

The 0x88 representation was popular in the 1990s and early 2000s. Today, bitboards have largely superseded it, but 0x88 remains a reasonable choice for simpler implementations.

## Piece Lists

To efficiently answer "where are all white knights?", maintain a list of piece locations:

```c
// Piece list for white knights
int white_knight_count = 2;
int white_knight_squares[10];  // Max 10 knights (after promotions)
// white_knight_squares[0] = b1 initially
// white_knight_squares[1] = g1 initially

// Find all white knight moves
for (int i = 0; i < white_knight_count; i++) {
    int sq = white_knight_squares[i];
    generate_knight_moves(sq);
}
```

Now finding all pieces of a type is O(number of those pieces) instead of O(64).

### Maintaining Piece Lists

When a move is made, the piece list must be updated:

- **Quiet move**: Update the moving piece's square in its list
- **Capture**: Update the moving piece; remove the captured piece from its list
- **Promotion**: Remove the pawn from pawns list; add the promoted piece to its list

Removals are tricky. If we remove a piece from the middle of the list, we need to fill the gap:

```c
// Remove piece at index idx from a piece list
void remove_from_list(int* list, int* count, int idx) {
    // Move last element to fill the gap
    list[idx] = list[*count - 1];
    (*count)--;
}
```

This requires knowing the index of the piece in its list. We can store a reverse mapping:

```c
// For each square, which index in which piece list?
int piece_list_index[64];

// When removing a piece from square sq:
int idx = piece_list_index[sq];
remove_from_list(white_knights, &white_knight_count, idx);

// Update the reverse mapping for the piece that moved to fill the gap
int moved_sq = white_knight_squares[idx];  // New occupant of idx
piece_list_index[moved_sq] = idx;
```

This bookkeeping is manageable but adds complexity.

## Hybrid Approaches

Most practical engines combine multiple representations:

```c
struct Position {
    // Board array for "what's on this square?"
    int board[64];

    // Piece lists for "where are pieces of type X?"
    int piece_squares[12][10];  // 12 piece types, up to 10 of each
    int piece_counts[12];

    // State
    int side_to_move;
    int castling_rights;  // Bit flags: K Q k q
    int en_passant_square;  // -1 if none
    int halfmove_clock;  // For 50-move rule
    int fullmove_number;

    // Zobrist hash (see Chapter 7)
    uint64 hash;
};
```

This redundancy requires keeping everything in sync during make/unmake, but provides O(1) access to both queries.

## Comparison of Approaches

| Approach | Piece on Square? | Find All Pieces? | Memory | Boundary Check |
|----------|-----------------|------------------|--------|----------------|
| 64-array | O(1) | O(64) | 64 bytes | Requires care |
| 0x88 | O(1) | O(64) | 128 bytes | `sq & 0x88` |
| Piece lists only | O(n pieces) | O(n pieces) | ~100 bytes | N/A |
| Hybrid | O(1) | O(n pieces) | ~200 bytes | Depends |
| Bitboards | O(1)* | O(popcount) | 96 bytes | Implicit |

*With bitboards, "piece on square" uses a different approach (bitwise AND).

## The Modern Choice: Bitboards

Today, nearly all competitive engines use **bitboards**. A bitboard is a 64-bit integer where each bit represents one square. Multiple bitboards represent different aspects of the position: one for white pawns, one for black pawns, one for white knights, and so on.

Bitboards support powerful operations:
- Find all pieces of a type: the bitboard itself
- Find all attacked squares: bitwise operations
- Check if a square is occupied: `(occupancy >> square) & 1`

The next two chapters cover bitboards in depth. If you are building a serious engine, bitboards are the way to go.

## Position State Beyond the Board

A complete position includes more than piece placement:

### Side to Move

```c
int side_to_move;  // WHITE = 0, BLACK = 1
```

The same piece arrangement with different side to move is a different position.

### Castling Rights

Four binary flags indicating whether each castling is still potentially legal:

```c
// Bit flags
#define WHITE_KINGSIDE   1  // 0001
#define WHITE_QUEENSIDE  2  // 0010
#define BLACK_KINGSIDE   4  // 0100
#define BLACK_QUEENSIDE  8  // 1000

int castling_rights;  // 0-15
```

Castling rights are lost when:
- The king moves (lose both sides)
- A rook moves (lose that side)
- A rook is captured (lose that side)

### En Passant Square

If a pawn just advanced two squares, the square "behind" it is the en passant target. Enemy pawns can capture to this square on the next move only.

```c
int en_passant_square;  // 0-63, or -1 if none
```

Note: The en passant square exists even if no enemy pawn can actually capture there. The position `1. e4` has en passant square e3, even though black has no pawn on d4 or f4. This affects position hashing.

### Move Counters

```c
int halfmove_clock;   // Moves since last capture or pawn move (for 50-move rule)
int fullmove_number;  // Increments after black's move
```

### Putting It Together: FEN Notation

FEN (Forsyth-Edwards Notation) encodes a complete position in a string. The starting position:

```
rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w KQkq - 0 1
```

The six fields:
1. **Piece placement** (rank 8 to rank 1, `/` separates ranks)
   - Lowercase = black, uppercase = white
   - Digits = consecutive empty squares
2. **Side to move**: `w` or `b`
3. **Castling rights**: `KQkq` or subset, `-` if none
4. **En passant target**: square in algebraic notation, `-` if none
5. **Halfmove clock**: for 50-move rule
6. **Fullmove number**: starts at 1, increments after black moves

Parsing FEN is an essential early exercise. See Appendix A for the full specification.

## Implementation Exercise

Before moving to bitboards, try implementing a basic board representation:

1. Define constants for piece types
2. Create a position structure with board array and state fields
3. Write a function to initialize the starting position
4. Write a function to parse a FEN string
5. Write a function to print the board to the console

Here's a skeleton:

```c
void init_start_position(Position* pos) {
    // Clear board
    for (int sq = 0; sq < 64; sq++) {
        pos->board[sq] = EMPTY;
    }

    // Place white pieces
    pos->board[A1] = WROOK;
    pos->board[B1] = WKNIGHT;
    // ... rest of back rank ...

    // White pawns
    for (int file = 0; file < 8; file++) {
        pos->board[A2 + file] = WPAWN;
    }

    // Black pieces (similar)
    // ...

    // Initial state
    pos->side_to_move = WHITE;
    pos->castling_rights = WHITE_KINGSIDE | WHITE_QUEENSIDE
                         | BLACK_KINGSIDE | BLACK_QUEENSIDE;
    pos->en_passant_square = -1;
    pos->halfmove_clock = 0;
    pos->fullmove_number = 1;
}

void print_board(Position* pos) {
    char piece_chars[] = ".PNBRQKpnbrqk";

    for (int rank = 7; rank >= 0; rank--) {
        printf("%d ", rank + 1);
        for (int file = 0; file < 8; file++) {
            int sq = rank * 8 + file;
            printf("%c ", piece_chars[pos->board[sq]]);
        }
        printf("\n");
    }
    printf("  a b c d e f g h\n");
}
```

## Summary

- Board representation is the foundation of your engine
- Simple arrays work but have performance limitations
- The 0x88 board simplifies boundary checking
- Piece lists enable fast iteration over pieces of a type
- Hybrid approaches combine strengths of multiple methods
- Modern engines use bitboards (next chapter)
- A complete position includes side to move, castling rights, en passant, and move counters
- FEN notation provides a standard serialization format

The next chapter introduces bitboards—the representation that enables the fastest engines.
