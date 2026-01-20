# Chapter 7: Incremental Updates

Making a move updates the position. Unmaking restores it. These operations happen millions of times per second during search. This chapter covers efficient techniques for move making, including Zobrist hashing—the foundation for transposition tables.

## Make/Unmake Move

There are two fundamental approaches:

**Copy-make**: Copy the entire position before making a move. To unmake, discard the modified copy.

**Make-unmake**: Modify the position in place. To unmake, reverse the modifications using saved state.

### Copy-Make

```c
void search(Position* pos, int depth) {
    MoveList moves;
    generate_moves(pos, &moves);

    for (int i = 0; i < moves.count; i++) {
        Position child = *pos;  // Copy entire position
        make_move(&child, moves.moves[i]);
        search(&child, depth - 1);
        // No unmake needed - child is discarded
    }
}
```

Advantages:
- Simple to implement
- No bugs from incomplete unmake
- Easy to debug (original position unchanged)

Disadvantages:
- Copies ~200 bytes per node
- Cache pressure from copying

### Make-Unmake

```c
void search(Position* pos, int depth) {
    MoveList moves;
    generate_moves(pos, &moves);

    for (int i = 0; i < moves.count; i++) {
        UnmakeInfo info;
        make_move(pos, moves.moves[i], &info);
        search(pos, depth - 1);
        unmake_move(pos, moves.moves[i], &info);
    }
}
```

Advantages:
- No large copies
- Better cache utilization

Disadvantages:
- Must correctly reverse all changes
- Bugs are subtle and hard to find

### Which to Choose?

Modern engines typically use make-unmake for the main position but may use copy-make for specific situations (like null move search or generating positions for analysis).

With careful implementation, make-unmake is faster. But start with copy-make until your engine is working correctly, then optimize if profiling shows it matters.

## Implementing Make Move

A move affects multiple parts of the position:

1. **Piece bitboards**: Remove from origin, add to destination
2. **Board array** (if used): Update squares
3. **Captured piece**: Remove from enemy bitboards
4. **Castling rights**: May be lost
5. **En passant square**: Set or clear
6. **Half-move clock**: Reset on pawn move or capture
7. **Side to move**: Flip
8. **Zobrist hash**: Update incrementally

### Basic Make Move

```c
typedef struct {
    int captured_piece;
    int castling_rights;
    int en_passant_square;
    int halfmove_clock;
    uint64 hash;  // Previous hash for easy restore
} UnmakeInfo;

void make_move(Position* pos, Move move, UnmakeInfo* info) {
    // Save state for unmake
    info->captured_piece = EMPTY;
    info->castling_rights = pos->castling_rights;
    info->en_passant_square = pos->en_passant_square;
    info->halfmove_clock = pos->halfmove_clock;
    info->hash = pos->hash;

    int us = pos->side_to_move;
    int them = us ^ 1;
    int from = move_from(move);
    int to = move_to(move);
    int flags = move_flags(move);
    int piece = pos->board[from];
    int piece_type = piece_type_of(piece);

    // Remove en passant hash if set
    if (pos->en_passant_square != NO_SQUARE) {
        pos->hash ^= ZOBRIST_EP[file_of(pos->en_passant_square)];
    }
    pos->en_passant_square = NO_SQUARE;

    // Handle captures
    if (is_capture(move)) {
        int cap_sq = to;

        // En passant: captured pawn is not on 'to' square
        if (flags == EN_PASSANT) {
            cap_sq = to + (us == WHITE ? -8 : 8);
        }

        int captured = pos->board[cap_sq];
        info->captured_piece = captured;

        // Remove captured piece
        remove_piece(pos, cap_sq, them, captured);
        pos->halfmove_clock = 0;  // Reset on capture
    }

    // Move the piece
    remove_piece(pos, from, us, piece);

    // Handle promotion
    if (is_promotion(move)) {
        int promo_piece = promotion_piece_type(move);
        piece = make_piece(us, promo_piece);
    }

    add_piece(pos, to, us, piece);

    // Handle special moves
    switch (flags) {
        case DOUBLE_PAWN:
            pos->en_passant_square = from + (us == WHITE ? 8 : -8);
            pos->hash ^= ZOBRIST_EP[file_of(pos->en_passant_square)];
            pos->halfmove_clock = 0;
            break;

        case KING_CASTLE:
            // Move the rook
            if (us == WHITE) {
                remove_piece(pos, H1, WHITE, WROOK);
                add_piece(pos, F1, WHITE, WROOK);
            } else {
                remove_piece(pos, H8, BLACK, BROOK);
                add_piece(pos, F8, BLACK, BROOK);
            }
            break;

        case QUEEN_CASTLE:
            if (us == WHITE) {
                remove_piece(pos, A1, WHITE, WROOK);
                add_piece(pos, D1, WHITE, WROOK);
            } else {
                remove_piece(pos, A8, BLACK, BROOK);
                add_piece(pos, D8, BLACK, BROOK);
            }
            break;

        default:
            if (piece_type == PAWN) {
                pos->halfmove_clock = 0;
            } else {
                pos->halfmove_clock++;
            }
    }

    // Update castling rights
    update_castling_rights(pos, from, to);

    // Flip side to move
    pos->side_to_move = them;
    pos->hash ^= ZOBRIST_SIDE;

    pos->fullmove_number += (us == BLACK);
}
```

### Helper Functions

```c
void remove_piece(Position* pos, int sq, int color, int piece) {
    uint64 bb = 1ULL << sq;
    int piece_type = piece_type_of(piece);

    pos->pieces[color][piece_type] &= ~bb;
    pos->all_pieces[color] &= ~bb;
    pos->board[sq] = EMPTY;

    pos->hash ^= ZOBRIST_PIECE[piece][sq];
}

void add_piece(Position* pos, int sq, int color, int piece) {
    uint64 bb = 1ULL << sq;
    int piece_type = piece_type_of(piece);

    pos->pieces[color][piece_type] |= bb;
    pos->all_pieces[color] |= bb;
    pos->board[sq] = piece;

    pos->hash ^= ZOBRIST_PIECE[piece][sq];

    // Update king square if needed
    if (piece_type == KING) {
        pos->king_square[color] = sq;
    }
}

void update_castling_rights(Position* pos, int from, int to) {
    // Use a precomputed table for efficiency
    static const int castling_update[64] = {
        // Indexed by square; AND with current rights to get new rights
        // a1: lose white queenside, h1: lose white kingside, etc.
        13, 15, 15, 15, 12, 15, 15, 14,  // rank 1
        15, 15, 15, 15, 15, 15, 15, 15,  // rank 2-7
        // ...
         7, 15, 15, 15,  3, 15, 15, 11,  // rank 8
    };

    int old_rights = pos->castling_rights;
    pos->castling_rights &= castling_update[from];
    pos->castling_rights &= castling_update[to];

    // Update hash if rights changed
    if (old_rights != pos->castling_rights) {
        pos->hash ^= ZOBRIST_CASTLING[old_rights];
        pos->hash ^= ZOBRIST_CASTLING[pos->castling_rights];
    }
}
```

### Unmake Move

```c
void unmake_move(Position* pos, Move move, UnmakeInfo* info) {
    int them = pos->side_to_move;  // 'them' is now 'us' after unmake
    int us = them ^ 1;
    int from = move_from(move);
    int to = move_to(move);
    int flags = move_flags(move);

    // Flip side back
    pos->side_to_move = us;

    // Get the piece that's currently on 'to'
    int piece = pos->board[to];

    // Handle promotion: restore to pawn
    if (is_promotion(move)) {
        remove_piece(pos, to, us, piece);
        piece = make_piece(us, PAWN);
        add_piece(pos, from, us, piece);
    } else {
        // Move piece back
        remove_piece(pos, to, us, piece);
        add_piece(pos, from, us, piece);
    }

    // Restore captured piece
    if (info->captured_piece != EMPTY) {
        int cap_sq = to;
        if (flags == EN_PASSANT) {
            cap_sq = to + (us == WHITE ? -8 : 8);
        }
        add_piece(pos, cap_sq, them, info->captured_piece);
    }

    // Handle castling: move rook back
    if (flags == KING_CASTLE) {
        if (us == WHITE) {
            remove_piece(pos, F1, WHITE, WROOK);
            add_piece(pos, H1, WHITE, WROOK);
        } else {
            remove_piece(pos, F8, BLACK, BROOK);
            add_piece(pos, H8, BLACK, BROOK);
        }
    } else if (flags == QUEEN_CASTLE) {
        if (us == WHITE) {
            remove_piece(pos, D1, WHITE, WROOK);
            add_piece(pos, A1, WHITE, WROOK);
        } else {
            remove_piece(pos, D8, BLACK, BROOK);
            add_piece(pos, A8, BLACK, BROOK);
        }
    }

    // Restore state
    pos->castling_rights = info->castling_rights;
    pos->en_passant_square = info->en_passant_square;
    pos->halfmove_clock = info->halfmove_clock;
    pos->hash = info->hash;

    pos->fullmove_number -= (us == BLACK);
}
```

## Zobrist Hashing

Zobrist hashing is a technique for computing position hashes incrementally. It's named after Albert Zobrist, who introduced it in 1970.

### The Concept

Assign a random 64-bit number to every (piece, square) combination, plus side to move, castling rights, and en passant file.

The position hash is the XOR of all applicable values:

```
hash = piece_zobrist[white_pawn][a2]
     ^ piece_zobrist[white_pawn][b2]
     ^ ...
     ^ piece_zobrist[black_king][e8]
     ^ side_zobrist  // if black to move
     ^ castling_zobrist[castling_rights]
     ^ ep_zobrist[ep_file]  // if en passant available
```

### Why XOR?

XOR has magical properties for hashing:

1. **Self-inverse**: `a ^ a = 0`. This means:
   - To add a piece: `hash ^= zobrist[piece][square]`
   - To remove a piece: `hash ^= zobrist[piece][square]` (same operation!)
   - To move a piece: XOR out the old square, XOR in the new square

2. **Order-independent**: `a ^ b = b ^ a`. The same position always has the same hash regardless of move order.

3. **Incremental**: Update the hash in O(1) per move, not O(pieces).

### Initializing Zobrist Tables

```c
uint64 ZOBRIST_PIECE[12][64];    // [piece_type][square]
uint64 ZOBRIST_CASTLING[16];     // [castling_rights]
uint64 ZOBRIST_EP[8];            // [file]
uint64 ZOBRIST_SIDE;             // Black to move

void init_zobrist() {
    // Use a deterministic PRNG for reproducibility
    uint64 seed = 0x123456789ABCDEF0ULL;

    for (int piece = 0; piece < 12; piece++) {
        for (int sq = 0; sq < 64; sq++) {
            ZOBRIST_PIECE[piece][sq] = random_uint64(&seed);
        }
    }

    for (int i = 0; i < 16; i++) {
        ZOBRIST_CASTLING[i] = random_uint64(&seed);
    }

    for (int i = 0; i < 8; i++) {
        ZOBRIST_EP[i] = random_uint64(&seed);
    }

    ZOBRIST_SIDE = random_uint64(&seed);
}

uint64 random_uint64(uint64* seed) {
    // xorshift64 PRNG
    *seed ^= *seed >> 12;
    *seed ^= *seed << 25;
    *seed ^= *seed >> 27;
    return *seed * 0x2545F4914F6CDD1DULL;
}
```

### Computing Initial Hash

When setting up a position (from FEN):

```c
uint64 compute_hash(Position* pos) {
    uint64 hash = 0;

    // Pieces
    for (int color = 0; color < 2; color++) {
        for (int pt = 0; pt < 6; pt++) {
            uint64 pieces = pos->pieces[color][pt];
            while (pieces) {
                int sq = bitscan_forward(pieces);
                int piece = make_piece(color, pt);
                hash ^= ZOBRIST_PIECE[piece][sq];
                pieces &= pieces - 1;
            }
        }
    }

    // Side to move
    if (pos->side_to_move == BLACK) {
        hash ^= ZOBRIST_SIDE;
    }

    // Castling
    hash ^= ZOBRIST_CASTLING[pos->castling_rights];

    // En passant
    if (pos->en_passant_square != NO_SQUARE) {
        hash ^= ZOBRIST_EP[file_of(pos->en_passant_square)];
    }

    return hash;
}
```

### Incremental Updates (Already Shown)

As seen in make_move, we update the hash incrementally:

```c
// Moving a piece from 'from' to 'to':
pos->hash ^= ZOBRIST_PIECE[piece][from];  // Remove from origin
pos->hash ^= ZOBRIST_PIECE[piece][to];    // Add at destination

// Capturing a piece:
pos->hash ^= ZOBRIST_PIECE[captured][cap_sq];

// Changing side to move:
pos->hash ^= ZOBRIST_SIDE;

// Changing castling rights:
pos->hash ^= ZOBRIST_CASTLING[old_rights];
pos->hash ^= ZOBRIST_CASTLING[new_rights];

// Setting en passant:
pos->hash ^= ZOBRIST_EP[file];
```

### Hash Collisions

With 64-bit hashes, collisions are rare but possible. In a search of 10 billion nodes, expect about one collision. This occasionally causes search errors, but the impact on playing strength is negligible.

Some engines use additional verification (storing a few bytes of the position) to detect collisions. Most don't bother.

## Incremental Evaluation Updates

Beyond hashing, we can update evaluation components incrementally.

### Piece-Square Table Scores

If using piece-square tables (Chapter 21), update the score as pieces move:

```c
// In make_move:
pos->pst_score[us] -= PST[piece][from];
pos->pst_score[us] += PST[moved_piece][to];

if (captured != EMPTY) {
    pos->pst_score[them] -= PST[captured][cap_sq];
}
```

### Material Count

```c
// In make_move:
if (captured != EMPTY) {
    pos->material[them] -= piece_value(captured);
}

// Promotion
if (is_promotion(move)) {
    pos->material[us] -= piece_value(PAWN);
    pos->material[us] += piece_value(promotion_piece_type(move));
}
```

### Phase (for Tapered Evaluation)

```c
// Track game phase based on piece counts
// In make_move:
if (captured != EMPTY) {
    pos->phase -= phase_value(captured);
}
```

## Null Move Implementation

A null move skips the moving side's turn. It's used for null move pruning (Chapter 14).

```c
void make_null_move(Position* pos, UnmakeInfo* info) {
    // Save state
    info->en_passant_square = pos->en_passant_square;
    info->hash = pos->hash;

    // Clear en passant
    if (pos->en_passant_square != NO_SQUARE) {
        pos->hash ^= ZOBRIST_EP[file_of(pos->en_passant_square)];
        pos->en_passant_square = NO_SQUARE;
    }

    // Flip side
    pos->side_to_move ^= 1;
    pos->hash ^= ZOBRIST_SIDE;

    // Don't update halfmove_clock or fullmove_number
}

void unmake_null_move(Position* pos, UnmakeInfo* info) {
    pos->side_to_move ^= 1;
    pos->en_passant_square = info->en_passant_square;
    pos->hash = info->hash;
}
```

## Debugging Make/Unmake

Bugs in make/unmake are insidious. Strategies:

### Verify Hash Consistency

After every unmake, verify the hash matches recomputation:

```c
#ifdef DEBUG
void unmake_move(Position* pos, Move move, UnmakeInfo* info) {
    // ... unmake logic ...

    // Verify hash
    uint64 expected = compute_hash(pos);
    if (pos->hash != expected) {
        printf("Hash mismatch after unmake!\n");
        printf("Expected: %llx, Got: %llx\n", expected, pos->hash);
        abort();
    }
}
#endif
```

### Position Fingerprinting

Compute a complete position fingerprint before make and after unmake:

```c
uint64 position_fingerprint(Position* pos) {
    uint64 fp = compute_hash(pos);
    fp ^= pos->castling_rights * 0x12345678ULL;
    fp ^= pos->en_passant_square * 0x87654321ULL;
    fp ^= pos->halfmove_clock * 0xABCDEF00ULL;
    return fp;
}

// In search:
uint64 before = position_fingerprint(pos);
make_move(pos, move, &info);
// ... search ...
unmake_move(pos, move, &info);
uint64 after = position_fingerprint(pos);
assert(before == after);
```

### Perft Debugging

Perft (Performance Test) counts nodes at a given depth. Compare with known results to verify move generation and make/unmake:

```c
uint64 perft(Position* pos, int depth) {
    if (depth == 0) return 1;

    MoveList moves;
    generate_legal_moves(pos, &moves);

    uint64 nodes = 0;
    for (int i = 0; i < moves.count; i++) {
        UnmakeInfo info;
        make_move(pos, moves.moves[i], &info);
        nodes += perft(pos, depth - 1);
        unmake_move(pos, moves.moves[i], &info);
    }

    return nodes;
}

// Starting position, depth 5:
// Expected: 4,865,609 nodes
```

Perft is covered in detail in Chapter 28 (Debugging).

## Performance Considerations

### Minimize Branching

Use lookup tables instead of conditionals:

```c
// Bad: Branch on piece type
if (piece_type == PAWN) { ... }
else if (piece_type == KNIGHT) { ... }

// Good: Table lookup
pos->hash ^= ZOBRIST_PIECE[piece][square];  // Works for all pieces
```

### Precompute Update Masks

The castling rights update can use a precomputed table:

```c
static const int CASTLING_MASK[64] = {
    13, 15, 15, 15, 12, 15, 15, 14,  // Rank 1
    // ...
     7, 15, 15, 15,  3, 15, 15, 11,  // Rank 8
};

pos->castling_rights &= CASTLING_MASK[from] & CASTLING_MASK[to];
```

### Avoid Redundant Work

Don't update aggregates that can be recomputed cheaply:

```c
// Updating all_pieces every move
pos->all_pieces[us] = pos->pieces[us][PAWN] | pos->pieces[us][KNIGHT] | ...;

// Better: Update incrementally
pos->all_pieces[us] &= ~(1ULL << from);
pos->all_pieces[us] |= (1ULL << to);
```

## Summary

Efficient move making is crucial for search performance:

- **Copy-make**: Simple but copies ~200 bytes per move
- **Make-unmake**: Faster but requires careful state management
- **Zobrist hashing**: XOR-based incremental hash updates
- **Incremental updates**: PST scores, material, phase can all update incrementally
- **Null move**: Special case that just flips side and clears en passant
- **Debugging**: Hash verification and perft testing catch bugs

The hash computed by Zobrist hashing is the key to transposition tables—the topic of Chapter 11. With make/unmake working correctly, we're ready to build the search.

Part 2 complete. Next: The heart of the engine—search algorithms.
