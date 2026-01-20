# Chapter 6: Move Representation

A chess engine manipulates millions of moves per second. How we encode moves affects memory usage, cache efficiency, and code clarity. This chapter covers move encoding schemes and the data structures that hold them.

## What Information Does a Move Contain?

At minimum, a move needs:
- **From square**: Where the piece starts (0-63)
- **To square**: Where the piece ends (0-63)

But several move types need additional information:
- **Promotions**: Which piece type (queen, rook, bishop, knight)
- **Castling**: Distinguish from normal king move
- **En passant**: Distinguish from normal pawn capture
- **Captures**: What piece was captured (useful for unmaking moves)

## The 16-Bit Move Encoding

The most common encoding packs a move into 16 bits:

```
bits 0-5:   from square (0-63)
bits 6-11:  to square (0-63)
bits 12-15: flags (move type)
```

```c
typedef uint16_t Move;

// Encoding
Move encode_move(int from, int to, int flags) {
    return (Move)(from | (to << 6) | (flags << 12));
}

// Decoding
int move_from(Move m)  { return m & 0x3F; }
int move_to(Move m)    { return (m >> 6) & 0x3F; }
int move_flags(Move m) { return (m >> 12) & 0x0F; }
```

### Flag Values

With 4 bits, we have 16 possible flag values:

```c
enum MoveFlags {
    QUIET           = 0,   // Normal non-capture move
    DOUBLE_PAWN     = 1,   // Pawn double push
    KING_CASTLE     = 2,   // Kingside castling
    QUEEN_CASTLE    = 3,   // Queenside castling
    CAPTURE         = 4,   // Normal capture
    EN_PASSANT      = 5,   // En passant capture

    // Promotions (can combine with capture flag)
    PROMO_KNIGHT    = 8,   // 1000
    PROMO_BISHOP    = 9,   // 1001
    PROMO_ROOK      = 10,  // 1010
    PROMO_QUEEN     = 11,  // 1011
    PROMO_CAP_KNIGHT = 12, // 1100  (capture + promotion)
    PROMO_CAP_BISHOP = 13, // 1101
    PROMO_CAP_ROOK   = 14, // 1110
    PROMO_CAP_QUEEN  = 15, // 1111
};
```

The bit patterns are designed for easy testing:

```c
bool is_capture(Move m) {
    int flags = move_flags(m);
    return (flags & 4) || flags == EN_PASSANT;
    // Bit 2 set means capture, plus special case for en passant
}

bool is_promotion(Move m) {
    return move_flags(m) >= PROMO_KNIGHT;
    // Bit 3 set means promotion
}

int promotion_piece(Move m) {
    // Extract promotion piece type from flags
    // 8=knight, 9=bishop, 10=rook, 11=queen (and +4 for capture versions)
    return (move_flags(m) & 3) + KNIGHT;  // Assuming KNIGHT=2, BISHOP=3, etc.
}

bool is_castling(Move m) {
    int flags = move_flags(m);
    return flags == KING_CASTLE || flags == QUEEN_CASTLE;
}
```

### Why 16 Bits?

- **Memory efficiency**: Two moves fit in one 32-bit word
- **Cache friendly**: More moves per cache line
- **Fast comparison**: Single integer compare
- **Sufficient information**: All move types encodable

Some engines use 32 bits to include captured piece type or move score, but 16 bits is the standard.

## The Move List

Moves are collected in a list structure. The key requirement: fast append (generation) and iteration (search).

### Fixed-Size Array

The simplest approach—a fixed array with a count:

```c
#define MAX_MOVES 256  // Maximum legal moves in any position is ~218

typedef struct {
    Move moves[MAX_MOVES];
    int count;
} MoveList;

void movelist_init(MoveList* list) {
    list->count = 0;
}

void movelist_add(MoveList* list, Move move) {
    list->moves[list->count++] = move;
}

Move movelist_get(MoveList* list, int index) {
    return list->moves[index];
}
```

Advantages:
- No dynamic allocation
- Cache-friendly contiguous memory
- Simple implementation

### Stack-Based Allocation

During search, we need a new move list at each ply. Pre-allocate a stack:

```c
// Global move stack
Move move_stack[MAX_PLY * MAX_MOVES];
int move_stack_top = 0;

MoveList* get_move_list() {
    MoveList* list = (MoveList*)&move_stack[move_stack_top];
    list->count = 0;
    return list;
}

void release_move_list(MoveList* list) {
    // Reset stack pointer (called when returning from search depth)
    move_stack_top -= list->count;
}
```

Better approach: embed move list in the search stack (Chapter 10):

```c
typedef struct {
    Move moves[MAX_MOVES];
    int move_count;
    int current_move;
    // Other per-ply state...
} SearchStack;

SearchStack ss[MAX_PLY];
```

## Move Scores and Sorting

For move ordering (Chapter 12), we often need to sort moves by score. Two approaches:

### Separate Score Array

```c
typedef struct {
    Move moves[MAX_MOVES];
    int scores[MAX_MOVES];
    int count;
} ScoredMoveList;

void sort_moves(ScoredMoveList* list) {
    // Selection sort is often faster than quicksort for small arrays
    for (int i = 0; i < list->count - 1; i++) {
        int best_idx = i;
        for (int j = i + 1; j < list->count; j++) {
            if (list->scores[j] > list->scores[best_idx]) {
                best_idx = j;
            }
        }
        if (best_idx != i) {
            // Swap both move and score
            Move tmp_move = list->moves[i];
            int tmp_score = list->scores[i];
            list->moves[i] = list->moves[best_idx];
            list->scores[i] = list->scores[best_idx];
            list->moves[best_idx] = tmp_move;
            list->scores[best_idx] = tmp_score;
        }
    }
}
```

### Combined Move-Score Entry

```c
typedef struct {
    Move move;
    int16_t score;
} ScoredMove;

typedef struct {
    ScoredMove entries[MAX_MOVES];
    int count;
} MoveList;
```

This keeps move and score adjacent in memory, potentially better for cache.

### Lazy Sorting

We often don't need all moves sorted—just the best few. Pick the best move without sorting, then pick the next best from remaining, etc:

```c
Move pick_best_move(MoveList* list, int start_index) {
    int best_idx = start_index;
    int best_score = list->scores[start_index];

    for (int i = start_index + 1; i < list->count; i++) {
        if (list->scores[i] > best_score) {
            best_score = list->scores[i];
            best_idx = i;
        }
    }

    // Swap best to current position
    if (best_idx != start_index) {
        Move tmp = list->moves[start_index];
        list->moves[start_index] = list->moves[best_idx];
        list->moves[best_idx] = tmp;

        int tmp_score = list->scores[start_index];
        list->scores[start_index] = list->scores[best_idx];
        list->scores[best_idx] = tmp_score;
    }

    return list->moves[start_index];
}

// In search:
for (int i = 0; i < move_list.count; i++) {
    Move move = pick_best_move(&move_list, i);
    // Search 'move'...
    // If beta cutoff, we never looked at remaining moves
}
```

This is O(n²) worst case but typically much faster because alpha-beta cuts off early.

## Displaying Moves

For debugging and UCI communication, we need to convert moves to strings.

### Long Algebraic Notation (UCI)

UCI uses simple "from-to" notation: `e2e4`, `g1f3`, `e7e8q` (promotion).

```c
void move_to_string(Move move, char* buffer) {
    int from = move_from(move);
    int to = move_to(move);
    int flags = move_flags(move);

    // Convert square to algebraic (a1=0, h8=63)
    buffer[0] = 'a' + (from % 8);
    buffer[1] = '1' + (from / 8);
    buffer[2] = 'a' + (to % 8);
    buffer[3] = '1' + (to / 8);

    // Add promotion piece if applicable
    if (is_promotion(move)) {
        char promo_chars[] = "nbrq";
        buffer[4] = promo_chars[flags & 3];
        buffer[5] = '\0';
    } else {
        buffer[4] = '\0';
    }
}

// Output: "e2e4", "g1f3", "e7e8q"
```

### Standard Algebraic Notation (SAN)

Human-readable notation like "Nf3", "exd5", "O-O". More complex to generate:

```c
void move_to_san(Position* pos, Move move, char* buffer) {
    int from = move_from(move);
    int to = move_to(move);
    int flags = move_flags(move);
    int piece = pos->board[from];
    int piece_type = piece_type_of(piece);

    char* ptr = buffer;

    // Castling
    if (flags == KING_CASTLE) {
        strcpy(buffer, "O-O");
        return;
    }
    if (flags == QUEEN_CASTLE) {
        strcpy(buffer, "O-O-O");
        return;
    }

    // Piece letter (not for pawns)
    if (piece_type != PAWN) {
        *ptr++ = "PNBRQK"[piece_type];

        // Disambiguation if needed (multiple pieces can reach 'to')
        // Check for ambiguity with other pieces of same type
        // Add file, rank, or both as needed
        // ... disambiguation logic ...
    }

    // Capture
    if (is_capture(move)) {
        if (piece_type == PAWN) {
            *ptr++ = 'a' + (from % 8);  // Pawn captures include origin file
        }
        *ptr++ = 'x';
    }

    // Destination square
    *ptr++ = 'a' + (to % 8);
    *ptr++ = '1' + (to / 8);

    // Promotion
    if (is_promotion(move)) {
        *ptr++ = '=';
        *ptr++ = "NBRQ"[flags & 3];
    }

    // Check/checkmate indicator (requires making the move and checking)
    // *ptr++ = '+' or '#'

    *ptr = '\0';
}
```

SAN generation is complex because of disambiguation. When two knights can reach the same square, we need "Nbd2" or "N1d2" or "Nb1d2". Most engines only generate SAN for display purposes, using UCI internally.

## Parsing Moves

UCI input comes as strings. Parse them back to Move:

```c
Move parse_uci_move(Position* pos, const char* str) {
    // Parse from/to squares
    int from_file = str[0] - 'a';
    int from_rank = str[1] - '1';
    int to_file = str[2] - 'a';
    int to_rank = str[3] - '1';

    int from = from_rank * 8 + from_file;
    int to = to_rank * 8 + to_file;

    // Determine flags based on position and move
    int flags = QUIET;
    int piece = pos->board[from];
    int captured = pos->board[to];

    // Capture?
    if (captured != EMPTY) {
        flags = CAPTURE;
    }

    // Special pawn moves
    if (piece_type_of(piece) == PAWN) {
        // Double push
        if (abs(to - from) == 16) {
            flags = DOUBLE_PAWN;
        }
        // En passant
        else if (to == pos->en_passant_square) {
            flags = EN_PASSANT;
        }
        // Promotion
        else if (to_rank == 0 || to_rank == 7) {
            char promo = str[4];
            int promo_type;
            switch (promo) {
                case 'n': promo_type = 0; break;
                case 'b': promo_type = 1; break;
                case 'r': promo_type = 2; break;
                case 'q': default: promo_type = 3; break;
            }
            flags = (captured != EMPTY) ?
                    (PROMO_CAP_KNIGHT + promo_type) :
                    (PROMO_KNIGHT + promo_type);
        }
    }

    // Castling
    if (piece_type_of(piece) == KING && abs(to - from) == 2) {
        flags = (to > from) ? KING_CASTLE : QUEEN_CASTLE;
    }

    return encode_move(from, to, flags);
}
```

### Move Validation

After parsing, verify the move is legal:

```c
bool is_valid_move(Position* pos, Move move) {
    MoveList moves;
    generate_pseudo_legal_moves(pos, &moves);

    for (int i = 0; i < moves.count; i++) {
        if (moves.moves[i] == move) {
            // Found in pseudo-legal list; verify legality
            make_move(pos, move);
            bool legal = !is_in_check(pos, pos->side_to_move ^ 1);
            unmake_move(pos, move);
            return legal;
        }
    }

    return false;
}
```

## Null Move

A "null move" is a special move where the side to play passes their turn. It's not legal in chess but is used in search (Chapter 14).

```c
#define NULL_MOVE ((Move)0)

bool is_null_move(Move m) {
    return m == NULL_MOVE;
}
```

Using 0 is convenient because:
- Easy to test
- Default-initialized move lists start with null moves
- Clear semantic meaning

When making a null move:
- Flip side to move
- Clear en passant square
- Update Zobrist hash
- Do NOT move any pieces

## Principal Variation (PV)

The Principal Variation is the sequence of best moves from a position. We store it as an array of moves:

```c
#define MAX_PV_LENGTH 128

typedef struct {
    Move moves[MAX_PV_LENGTH];
    int length;
} PVLine;

void pv_clear(PVLine* pv) {
    pv->length = 0;
}

void pv_append(PVLine* pv, Move move) {
    if (pv->length < MAX_PV_LENGTH) {
        pv->moves[pv->length++] = move;
    }
}

// Copy child PV to parent, prepending the current move
void pv_copy(PVLine* parent, Move move, PVLine* child) {
    parent->moves[0] = move;
    int len = (child->length < MAX_PV_LENGTH - 1) ? child->length : MAX_PV_LENGTH - 1;
    for (int i = 0; i < len; i++) {
        parent->moves[i + 1] = child->moves[i];
    }
    parent->length = len + 1;
}
```

The PV is built bottom-up during search:
1. At leaf nodes, PV is empty
2. When a new best move is found, copy child's PV and prepend current move
3. At root, the PV contains the full best line

## Move Comparison and Hashing

Moves need to be compared for equality (hash move lookup, killer moves):

```c
bool moves_equal(Move a, Move b) {
    // 16-bit comparison - direct equality
    return a == b;
}
```

For hash tables that store moves, the 16-bit encoding is perfect—compact and directly comparable.

## Summary

Key points for move representation:

- **16-bit encoding**: 6 bits from, 6 bits to, 4 bits flags
- **Flag design**: Bit patterns enable fast type testing
- **Move lists**: Fixed-size arrays on the stack
- **Scoring**: Separate array or combined struct; lazy sorting
- **String conversion**: UCI (simple) vs SAN (complex with disambiguation)
- **Null move**: Special sentinel value (0)
- **PV line**: Array of moves built during search

The 16-bit move encoding is a de facto standard in chess programming. It balances compactness with expressiveness, fitting all the information needed for move making and display.

Next chapter: How to make and unmake moves efficiently with incremental updates.
