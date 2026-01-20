# Appendix A: FEN Notation

**FEN** (Forsyth-Edwards Notation) is the standard format for describing chess positions.

## Format

A FEN string has six space-separated fields:

```
rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w KQkq - 0 1
```

## Field 1: Piece Placement

Describes the board from rank 8 (top) to rank 1 (bottom), with ranks separated by `/`.

### Piece Characters

| Character | Piece |
|-----------|-------|
| K | White King |
| Q | White Queen |
| R | White Rook |
| B | White Bishop |
| N | White Knight |
| P | White Pawn |
| k | Black King |
| q | Black Queen |
| r | Black Rook |
| b | Black Bishop |
| n | Black Knight |
| p | Black Pawn |

### Empty Squares

Numbers 1-8 indicate consecutive empty squares.

### Examples

```
rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR
```

This is the starting position:
- Rank 8: r n b q k b n r (black pieces)
- Rank 7: p p p p p p p p (black pawns)
- Ranks 6-3: 8 8 8 8 (all empty)
- Rank 2: P P P P P P P P (white pawns)
- Rank 1: R N B Q K B N R (white pieces)

```
8/8/8/8/4P3/8/8/8
```

A lone white pawn on e4:
- Ranks 8-5: empty
- Rank 4: 4 empty, P on e4, 3 empty
- Ranks 3-1: empty

## Field 2: Active Color

- `w` = White to move
- `b` = Black to move

## Field 3: Castling Availability

Characters indicating which castling moves are still legal:

| Character | Meaning |
|-----------|---------|
| K | White can castle kingside |
| Q | White can castle queenside |
| k | Black can castle kingside |
| q | Black can castle queenside |
| - | Neither side can castle |

### Examples

- `KQkq` - All castling rights available
- `Kq` - White kingside and black queenside only
- `-` - No castling available

## Field 4: En Passant Target

The square behind a pawn that just moved two squares, or `-` if no en passant is possible.

### Examples

After 1.e4:
```
rnbqkbnr/pppppppp/8/8/4P3/8/PPPP1PPP/RNBQKBNR b KQkq e3 0 1
```

The en passant target is `e3` (the square a capturing pawn would land on).

After 1.e4 e5:
```
rnbqkbnr/pppp1ppp/8/4p3/4P3/8/PPPP1PPP/RNBQKBNR w KQkq e6 0 2
```

The en passant target is `e6`.

## Field 5: Halfmove Clock

Number of halfmoves (plies) since the last pawn move or capture. Used for the 50-move draw rule.

### Examples

- `0` - Pawn just moved or piece just captured
- `15` - 15 plies since last pawn move or capture

When this reaches 100, the game is drawn by the 50-move rule.

## Field 6: Fullmove Number

The current move number, starting at 1 and incrementing after Black's move.

### Examples

- `1` - First move
- `25` - 25th move

## Complete Examples

### Starting Position

```
rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w KQkq - 0 1
```

### After 1.e4 e5 2.Nf3 Nc6 3.Bb5

```
r1bqkbnr/pppp1ppp/2n5/1B2p3/4P3/5N2/PPPP1PPP/RNBQK2R b KQkq - 3 3
```

- White bishop on b5, knight on f3
- Black knight on c6
- Black to move
- All castling rights intact
- No en passant
- 3 halfmoves since last pawn move
- Move 3

### Complex Position

```
r3k2r/p1ppqpb1/bn2pnp1/3PN3/1p2P3/2N2Q1p/PPPBBPPP/R3K2R w KQkq - 0 1
```

"Kiwipete" - a position with many legal moves for testing.

### Endgame Position

```
8/8/4k3/8/8/4K3/4P3/8 w - - 0 1
```

King and pawn endgame: white king e3, white pawn e2, black king e6.

## Parsing FEN

```c
void parse_fen(Position* pos, const char* fen) {
    memset(pos, 0, sizeof(Position));

    int sq = 56;  // Start at a8

    // Field 1: Piece placement
    while (*fen && *fen != ' ') {
        if (*fen == '/') {
            sq -= 16;  // Next rank down
        } else if (*fen >= '1' && *fen <= '8') {
            sq += (*fen - '0');  // Skip empty squares
        } else {
            int piece = char_to_piece(*fen);
            pos->board[sq] = piece;
            set_bit(&pos->pieces[piece_color(piece)][piece_type(piece)], sq);
            sq++;
        }
        fen++;
    }
    fen++;  // Skip space

    // Field 2: Active color
    pos->side_to_move = (*fen == 'w') ? WHITE : BLACK;
    fen += 2;

    // Field 3: Castling
    pos->castling = 0;
    while (*fen && *fen != ' ') {
        switch (*fen) {
            case 'K': pos->castling |= WHITE_OO; break;
            case 'Q': pos->castling |= WHITE_OOO; break;
            case 'k': pos->castling |= BLACK_OO; break;
            case 'q': pos->castling |= BLACK_OOO; break;
        }
        fen++;
    }
    fen++;

    // Field 4: En passant
    if (*fen != '-') {
        int file = fen[0] - 'a';
        int rank = fen[1] - '1';
        pos->en_passant = rank * 8 + file;
        fen += 2;
    } else {
        pos->en_passant = NO_SQUARE;
        fen++;
    }
    fen++;

    // Field 5: Halfmove clock
    pos->halfmove = atoi(fen);
    while (*fen && *fen != ' ') fen++;
    fen++;

    // Field 6: Fullmove number
    pos->fullmove = atoi(fen);

    // Compute derived data
    compute_hash(pos);
    compute_checkers(pos);
}
```

## Generating FEN

```c
void position_to_fen(Position* pos, char* fen) {
    char* p = fen;

    // Field 1: Piece placement
    for (int rank = 7; rank >= 0; rank--) {
        int empty = 0;

        for (int file = 0; file < 8; file++) {
            int sq = rank * 8 + file;
            int piece = pos->board[sq];

            if (piece == EMPTY) {
                empty++;
            } else {
                if (empty > 0) {
                    *p++ = '0' + empty;
                    empty = 0;
                }
                *p++ = piece_to_char(piece);
            }
        }

        if (empty > 0) {
            *p++ = '0' + empty;
        }

        if (rank > 0) {
            *p++ = '/';
        }
    }

    // Field 2: Active color
    *p++ = ' ';
    *p++ = (pos->side_to_move == WHITE) ? 'w' : 'b';

    // Field 3: Castling
    *p++ = ' ';
    if (pos->castling == 0) {
        *p++ = '-';
    } else {
        if (pos->castling & WHITE_OO)  *p++ = 'K';
        if (pos->castling & WHITE_OOO) *p++ = 'Q';
        if (pos->castling & BLACK_OO)  *p++ = 'k';
        if (pos->castling & BLACK_OOO) *p++ = 'q';
    }

    // Field 4: En passant
    *p++ = ' ';
    if (pos->en_passant == NO_SQUARE) {
        *p++ = '-';
    } else {
        *p++ = 'a' + (pos->en_passant % 8);
        *p++ = '1' + (pos->en_passant / 8);
    }

    // Field 5: Halfmove clock
    p += sprintf(p, " %d", pos->halfmove);

    // Field 6: Fullmove number
    p += sprintf(p, " %d", pos->fullmove);

    *p = '\0';
}
```

## Special Positions

### Empty Board

```
8/8/8/8/8/8/8/8 w - - 0 1
```

### Pawn Promotion Test

```
8/P7/8/8/8/8/p7/8 w - - 0 1
```

### All Castling Options

```
r3k2r/pppppppp/8/8/8/8/PPPPPPPP/R3K2R w KQkq - 0 1
```

### En Passant Test

```
rnbqkbnr/ppp1p1pp/8/3pPp2/8/8/PPPP1PPP/RNBQKBNR w KQkq f6 0 3
```

White can capture en passant on f6.
