# Appendix B: Square Naming Conventions

Different chess engines use different conventions for numbering squares. This appendix covers common approaches.

## Algebraic Notation

Standard human-readable notation:

```
   a  b  c  d  e  f  g  h
8  a8 b8 c8 d8 e8 f8 g8 h8  8
7  a7 b7 c7 d7 e7 f7 g7 h7  7
6  a6 b6 c6 d6 e6 f6 g6 h6  6
5  a5 b5 c5 d5 e5 f5 g5 h5  5
4  a4 b4 c4 d4 e4 f4 g4 h4  4
3  a3 b3 c3 d3 e3 f3 g3 h3  3
2  a2 b2 c2 d2 e2 f2 g2 h2  2
1  a1 b1 c1 d1 e1 f1 g1 h1  1
   a  b  c  d  e  f  g  h
```

- Files: a-h (columns, left to right from White's perspective)
- Ranks: 1-8 (rows, bottom to top from White's perspective)

## Little-Endian Rank-File Mapping (LERF)

Most common in bitboard engines. Square index = rank * 8 + file.

```
   a  b  c  d  e  f  g  h
8  56 57 58 59 60 61 62 63  8
7  48 49 50 51 52 53 54 55  7
6  40 41 42 43 44 45 46 47  6
5  32 33 34 35 36 37 38 39  5
4  24 25 26 27 28 29 30 31  4
3  16 17 18 19 20 21 22 23  3
2   8  9 10 11 12 13 14 15  2
1   0  1  2  3  4  5  6  7  1
   a  b  c  d  e  f  g  h
```

### Conversion

```c
int algebraic_to_square(const char* alg) {
    int file = alg[0] - 'a';  // 0-7
    int rank = alg[1] - '1';  // 0-7
    return rank * 8 + file;
}

void square_to_algebraic(int sq, char* alg) {
    alg[0] = 'a' + (sq % 8);
    alg[1] = '1' + (sq / 8);
    alg[2] = '\0';
}

int file_of(int sq) { return sq % 8; }  // 0=a, 7=h
int rank_of(int sq) { return sq / 8; }  // 0=1st, 7=8th
```

### Bitboard Correspondence

With LERF, the LSB (bit 0) corresponds to a1:

```c
uint64_t A1 = 1ULL << 0;   // = 0x0000000000000001
uint64_t H1 = 1ULL << 7;   // = 0x0000000000000080
uint64_t A8 = 1ULL << 56;  // = 0x0100000000000000
uint64_t H8 = 1ULL << 63;  // = 0x8000000000000000
```

## Big-Endian Rank-File Mapping (BERF)

Less common. Square index = (7 - rank) * 8 + file.

```
   a  b  c  d  e  f  g  h
8   0  1  2  3  4  5  6  7  8
7   8  9 10 11 12 13 14 15  7
6  16 17 18 19 20 21 22 23  6
5  24 25 26 27 28 29 30 31  5
4  32 33 34 35 36 37 38 39  4
3  40 41 42 43 44 45 46 47  3
2  48 49 50 51 52 53 54 55  2
1  56 57 58 59 60 61 62 63  1
   a  b  c  d  e  f  g  h
```

Note: a8 is 0, h1 is 63 (opposite of LERF).

## File-First Mapping

Some engines use file * 8 + rank instead:

```
   a  b  c  d  e  f  g  h
8   7 15 23 31 39 47 55 63  8
7   6 14 22 30 38 46 54 62  7
6   5 13 21 29 37 45 53 61  6
5   4 12 20 28 36 44 52 60  5
4   3 11 19 27 35 43 51 59  4
3   2 10 18 26 34 42 50 58  3
2   1  9 17 25 33 41 49 57  2
1   0  8 16 24 32 40 48 56  1
   a  b  c  d  e  f  g  h
```

## 0x88 Representation

Uses a 128-square array where valid squares have bit 0x88 clear:

```
   a   b   c   d   e   f   g   h
8  112 113 114 115 116 117 118 119  | 120-127 (invalid)
7   96  97  98  99 100 101 102 103  | 104-111 (invalid)
6   80  81  82  83  84  85  86  87  |  88-95  (invalid)
5   64  65  66  67  68  69  70  71  |  72-79  (invalid)
4   48  49  50  51  52  53  54  55  |  56-63  (invalid)
3   32  33  34  35  36  37  38  39  |  40-47  (invalid)
2   16  17  18  19  20  21  22  23  |  24-31  (invalid)
1    0   1   2   3   4   5   6   7  |   8-15  (invalid)
```

### Validity Check

```c
bool is_valid_square(int sq) {
    return (sq & 0x88) == 0;
}
```

### Conversion

```c
int lerf_to_0x88(int sq) {
    return (sq / 8) * 16 + (sq % 8);
}

int x88_to_lerf(int sq) {
    return (sq / 16) * 8 + (sq % 8);
}
```

## Directional Offsets

### LERF Offsets

```c
#define NORTH       8
#define SOUTH      -8
#define EAST        1
#define WEST       -1
#define NORTH_EAST  9
#define NORTH_WEST  7
#define SOUTH_EAST -7
#define SOUTH_WEST -9
```

### 0x88 Offsets

```c
#define NORTH       16
#define SOUTH      -16
#define EAST         1
#define WEST        -1
#define NORTH_EAST  17
#define NORTH_WEST  15
#define SOUTH_EAST -15
#define SOUTH_WEST -17
```

### Knight Offsets (LERF)

```c
int KNIGHT_OFFSETS[8] = {
    17, 15, 10, 6, -6, -10, -15, -17
};

// Or equivalently:
// +2 ranks, +1 file:  17
// +2 ranks, -1 file:  15
// +1 rank,  +2 files: 10
// +1 rank,  -2 files:  6
// -1 rank,  +2 files: -6
// -1 rank,  -2 files: -10
// -2 ranks, +1 file: -15
// -2 ranks, -1 file: -17
```

## Color Flipping

### Vertical Flip (LERF)

Mirror the board vertically (swap rank 1 with rank 8, etc.):

```c
int flip_vertical(int sq) {
    return sq ^ 56;  // XOR with 56 flips ranks
}

// Examples:
// a1 (0)  -> a8 (56)
// e4 (28) -> e5 (36)
// h8 (63) -> h1 (7)
```

### Horizontal Flip

Mirror the board horizontally (swap a-file with h-file):

```c
int flip_horizontal(int sq) {
    return sq ^ 7;  // XOR with 7 flips files
}
```

### Rotate 180 Degrees

```c
int rotate_180(int sq) {
    return 63 - sq;
    // Or equivalently: sq ^ 63
}
```

## File and Rank Constants

### File Masks (LERF)

```c
uint64_t FILE_A = 0x0101010101010101ULL;
uint64_t FILE_B = 0x0202020202020202ULL;
uint64_t FILE_C = 0x0404040404040404ULL;
uint64_t FILE_D = 0x0808080808080808ULL;
uint64_t FILE_E = 0x1010101010101010ULL;
uint64_t FILE_F = 0x2020202020202020ULL;
uint64_t FILE_G = 0x4040404040404040ULL;
uint64_t FILE_H = 0x8080808080808080ULL;

uint64_t FILE_MASK[8] = {
    FILE_A, FILE_B, FILE_C, FILE_D,
    FILE_E, FILE_F, FILE_G, FILE_H
};
```

### Rank Masks (LERF)

```c
uint64_t RANK_1 = 0x00000000000000FFULL;
uint64_t RANK_2 = 0x000000000000FF00ULL;
uint64_t RANK_3 = 0x0000000000FF0000ULL;
uint64_t RANK_4 = 0x00000000FF000000ULL;
uint64_t RANK_5 = 0x000000FF00000000ULL;
uint64_t RANK_6 = 0x0000FF0000000000ULL;
uint64_t RANK_7 = 0x00FF000000000000ULL;
uint64_t RANK_8 = 0xFF00000000000000ULL;

uint64_t RANK_MASK[8] = {
    RANK_1, RANK_2, RANK_3, RANK_4,
    RANK_5, RANK_6, RANK_7, RANK_8
};
```

## Common Square Constants

```c
enum Square {
    A1, B1, C1, D1, E1, F1, G1, H1,
    A2, B2, C2, D2, E2, F2, G2, H2,
    A3, B3, C3, D3, E3, F3, G3, H3,
    A4, B4, C4, D4, E4, F4, G4, H4,
    A5, B5, C5, D5, E5, F5, G5, H5,
    A6, B6, C6, D6, E6, F6, G6, H6,
    A7, B7, C7, D7, E7, F7, G7, H7,
    A8, B8, C8, D8, E8, F8, G8, H8,
    NO_SQUARE = 64
};
```

## Visual Representation

Useful for debugging:

```c
void print_bitboard(uint64_t bb) {
    printf("\n");
    for (int rank = 7; rank >= 0; rank--) {
        printf("%d  ", rank + 1);
        for (int file = 0; file < 8; file++) {
            int sq = rank * 8 + file;
            printf("%c ", (bb & (1ULL << sq)) ? '1' : '.');
        }
        printf("\n");
    }
    printf("\n   a b c d e f g h\n\n");
}
```

Output for the white pawns starting position:

```
8  . . . . . . . .
7  . . . . . . . .
6  . . . . . . . .
5  . . . . . . . .
4  . . . . . . . .
3  . . . . . . . .
2  1 1 1 1 1 1 1 1
1  . . . . . . . .

   a b c d e f g h
```
