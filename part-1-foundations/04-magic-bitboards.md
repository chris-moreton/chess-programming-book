# Chapter 4: Magic Bitboards

Magic bitboards are one of the most elegant algorithms in chess programming. They solve a fundamental problem: how do we quickly compute the attack squares of sliding pieces (bishops, rooks, queens)?

Unlike knights and kings, sliding pieces can move any number of squares along a ray until blocked. A rook on a1 attacks different squares depending on what pieces occupy the a-file and first rank. We need to compute these attacks billions of times during search, so efficiency is critical.

This chapter explains the problem, develops the solution step by step, and shows you how to implement magic bitboards in your engine.

## The Problem: Sliding Piece Attacks

Consider a rook on e4. In an empty board, it attacks the entire e-file and 4th rank—14 squares total:

```
    a  b  c  d  e  f  g  h
   -------------------------
8 |  .  .  .  .  X  .  .  . |
7 |  .  .  .  .  X  .  .  . |
6 |  .  .  .  .  X  .  .  . |
5 |  .  .  .  .  X  .  .  . |
4 |  X  X  X  X  R  X  X  X |
3 |  .  .  .  .  X  .  .  . |
2 |  .  .  .  .  X  .  .  . |
1 |  .  .  .  .  X  .  .  . |
   -------------------------
```

Now add a white pawn on e7 and a black pawn on b4:

```
    a  b  c  d  e  f  g  h
   -------------------------
8 |  .  .  .  .  .  .  .  . |
7 |  .  .  .  .  P  .  .  . |
6 |  .  .  .  .  X  .  .  . |
5 |  .  .  .  .  X  .  .  . |
4 |  .  X  X  X  R  X  X  X |
3 |  .  .  .  .  X  .  .  . |
2 |  .  .  .  .  X  .  .  . |
1 |  .  .  .  .  X  .  .  . |
   -------------------------
```

The rook is blocked:
- On the e-file: cannot see e8 (blocked by pawn on e7, but attacks e7)
- On the 4th rank: cannot see a4 (blocked by pawn on b4, but attacks b4)

The blocking piece's square is still attacked—the rook can capture there. But squares beyond the blocker are not attacked.

For knights and kings, we precomputed attack tables indexed solely by square. For sliding pieces, we also need to consider the **occupancy** of potential blocking squares.

## Relevant Occupancy

Here's a crucial insight: not all occupied squares matter for a sliding piece's attacks. Only pieces on the piece's **rays** can block it.

For a rook on e4:
- Pieces on f7 or a1 don't affect its attacks (not on its rays)
- Only pieces on the e-file (e1-e8) or 4th rank (a4-h4) matter

Furthermore, the **edge squares** of a ray don't matter for blocking:
- A rook on e4 attacks e8 regardless of whether e8 is occupied
- If e8 has a piece, the rook attacks it. If e8 is empty, the rook attacks it (the empty square)
- The rook cannot see beyond e8 either way

So the **relevant occupancy** for a rook on e4 excludes the edges:
- e-file: e2, e3, e5, e6, e7 (not e1 or e8)
- 4th rank: b4, c4, d4, f4, g4 (not a4 or h4)

That's 10 relevant squares. With 10 squares, there are 2^10 = 1024 possible occupancy configurations.

We could precompute all 1024 attack bitboards and store them in a table. Indexed by... what? This is where the magic comes in.

## From Occupancy to Index

We have:
- A **blocker mask**: the 10 relevant squares for our rook
- An **occupancy**: which of those 10 squares are actually occupied
- A desired **attack bitboard**: the squares the rook can reach

The occupancy is a subset of the blocker mask—a 10-bit pattern embedded in a 64-bit integer. We need to convert this sparse pattern into a dense index (0-1023) to look up in our table.

### The Direct Approach

One option: extract each relevant bit and pack them into a small integer:

```c
int occupancy_to_index(uint64 occupancy, uint64 blocker_mask) {
    int index = 0;
    int bit = 0;
    while (blocker_mask) {
        int sq = bitscan_forward(blocker_mask);
        if (occupancy & (1ULL << sq)) {
            index |= (1 << bit);
        }
        bit++;
        blocker_mask &= blocker_mask - 1;
    }
    return index;
}
```

This works but requires a loop. With 10+ iterations per lookup, it's too slow for the inner search loop.

### The PEXT Instruction

Modern CPUs (Intel Haswell and later, AMD Zen 3 and later) have a hardware instruction for exactly this: **PEXT** (Parallel Bits Extract).

```c
// Hardware instruction: extract bits from occupancy where blocker_mask has 1s
// and pack them into the low bits of the result
int index = _pext_u64(occupancy, blocker_mask);
```

PEXT is fast (1-3 cycles) and solves the indexing problem directly. If your target hardware supports it, PEXT-based sliding attacks are an excellent choice.

But what about older hardware, or when you want a portable solution? That's where **magic multiplication** comes in.

## The Magic Multiplication Insight

Here's the key observation: multiplication can scatter bits around in useful ways.

Consider multiplying the occupancy by a carefully chosen constant (the "magic number"):

```c
uint64 index = (occupancy * magic) >> shift;
```

If we choose the magic number correctly, different occupancy patterns produce different indices. We achieve a **perfect hash**—no collisions between different occupancies that would need different attack results.

### Why Multiplication Works

Think about what multiplication does in binary. Multiplying by a number with k set bits creates k shifted copies of the original, which are then XORed together (because addition in binary without carries is XOR).

For example, if the magic is `0b10100` (bits 2 and 4 set):
```
  occupancy × 0b10100
= occupancy << 4
+ occupancy << 2
```

With the right magic, the relevant bits from the occupancy get collected into adjacent positions in the upper bits of the product. We then shift right to extract just those bits as our index.

### What Makes a Magic "Magic"?

A magic number for a given square must satisfy:
1. For every possible occupancy pattern, `(occupancy * magic) >> shift` produces a unique index
2. The shift should be `64 - num_bits`, where `num_bits` is the number of relevant blocker squares
3. The resulting index must fit in our precomputed table

Finding such magic numbers is done offline (at program startup or compile time) through trial and random search.

## Finding Magic Numbers

The standard approach to finding magic numbers is trial and error with random candidates:

```c
uint64 find_magic(int square, bool is_bishop) {
    // Get the blocker mask for this square
    uint64 blocker_mask = is_bishop ? bishop_blocker_mask(square)
                                    : rook_blocker_mask(square);

    // Count relevant bits
    int num_bits = popcount(blocker_mask);
    int table_size = 1 << num_bits;

    // Generate all possible occupancy patterns
    uint64 occupancies[4096];
    uint64 attacks[4096];
    int num_occupancies = enumerate_occupancies(blocker_mask, occupancies);

    // Compute attack set for each occupancy
    for (int i = 0; i < num_occupancies; i++) {
        attacks[i] = compute_sliding_attacks(square, occupancies[i], is_bishop);
    }

    // Try random magic numbers until we find one that works
    while (true) {
        uint64 magic = random_sparse_uint64();  // Few bits set

        // Test if this magic produces no collisions
        uint64 used[4096] = {0};
        bool failed = false;

        for (int i = 0; i < num_occupancies; i++) {
            int index = (occupancies[i] * magic) >> (64 - num_bits);

            if (used[index] == 0) {
                used[index] = attacks[i];
            } else if (used[index] != attacks[i]) {
                // Collision with different attack set - magic failed
                failed = true;
                break;
            }
            // Note: collision with SAME attack set is fine (constructive collision)
        }

        if (!failed) {
            return magic;  // Found a working magic!
        }
    }
}
```

### Random Sparse Numbers

Magics with fewer set bits tend to work better. A common approach:

```c
uint64 random_sparse_uint64() {
    // AND three random numbers to get a sparse result
    return random_uint64() & random_uint64() & random_uint64();
}
```

This produces numbers with roughly 1/8 of the bits set on average.

### Constructive Collisions

Notice that two occupancies can map to the same index **if they produce the same attacks**. This happens when the difference in occupancy is beyond all blockers that matter.

For example, if blockers at e5 and e6 both block the rook from reaching e7, it doesn't matter which specific combination of e5/e6 is occupied—the attack set is the same. These can safely collide in our lookup table.

## Enumerating Occupancies

To test a magic, we need all possible occupancies for a blocker mask. This is a classic bit enumeration problem:

```c
// Enumerate all subsets of blocker_mask using Carry-Rippler trick
int enumerate_occupancies(uint64 blocker_mask, uint64* out) {
    int count = 0;
    uint64 occupancy = 0;

    do {
        out[count++] = occupancy;
        occupancy = (occupancy - blocker_mask) & blocker_mask;
    } while (occupancy);

    return count;  // Returns 2^popcount(blocker_mask)
}
```

The **Carry-Rippler** trick uses the observation that subtracting and masking produces the next subset. It's elegant and efficient.

### Computing Sliding Attacks Directly

For testing magics, we need the "ground truth" attacks for each occupancy. This is done the slow way—ray tracing:

```c
uint64 compute_rook_attacks(int square, uint64 occupancy) {
    uint64 attacks = 0;
    int rank = square / 8;
    int file = square % 8;

    // North
    for (int r = rank + 1; r <= 7; r++) {
        int sq = r * 8 + file;
        attacks |= (1ULL << sq);
        if (occupancy & (1ULL << sq)) break;  // Blocked
    }

    // South
    for (int r = rank - 1; r >= 0; r--) {
        int sq = r * 8 + file;
        attacks |= (1ULL << sq);
        if (occupancy & (1ULL << sq)) break;
    }

    // East
    for (int f = file + 1; f <= 7; f++) {
        int sq = rank * 8 + f;
        attacks |= (1ULL << sq);
        if (occupancy & (1ULL << sq)) break;
    }

    // West
    for (int f = file - 1; f >= 0; f--) {
        int sq = rank * 8 + f;
        attacks |= (1ULL << sq);
        if (occupancy & (1ULL << sq)) break;
    }

    return attacks;
}
```

This is only used during initialization, not during search.

## The Blocker Masks

For each square, we need the mask of relevant blocking squares.

### Rook Blocker Masks

For a rook, the relevant squares are on its file and rank, excluding edges:

```c
uint64 rook_blocker_mask(int square) {
    uint64 mask = 0;
    int rank = square / 8;
    int file = square % 8;

    // File (excluding top and bottom edges)
    for (int r = 1; r < 7; r++) {
        if (r != rank) {
            mask |= (1ULL << (r * 8 + file));
        }
    }

    // Rank (excluding left and right edges)
    for (int f = 1; f < 7; f++) {
        if (f != file) {
            mask |= (1ULL << (rank * 8 + f));
        }
    }

    return mask;
}
```

A rook in a corner (a1) has the fewest relevant squares: 12.
A rook in the center (e4) has the most: 10 (file) + 10 (rank) - overlap considerations... actually still around 10-11.

Wait, let me recalculate. For a rook on e4:
- e-file: e2, e3, e5, e6, e7 (5 squares, excluding e1, e4, e8)
- 4th rank: b4, c4, d4, f4, g4 (5 squares, excluding a4, e4, h4)
- Total: 10 relevant squares

For a rook on a1:
- a-file: a2, a3, a4, a5, a6, a7 (6 squares)
- 1st rank: b1, c1, d1, e1, f1, g1 (6 squares)
- Total: 12 relevant squares

So corner rooks actually have MORE relevant squares because they can't exclude an edge on two sides. Hmm, let me reconsider...

Actually, for corner squares, the edge exclusion still applies to the far edges:
- a-file: a2 through a7 (not a8, the far edge)
- 1st rank: b1 through g1 (not h1, the far edge)

So a rook on a1 has 6 + 6 = 12 relevant bits, requiring 2^12 = 4096 table entries.
A rook on e4 has 5 + 5 = 10 relevant bits, requiring 2^10 = 1024 table entries.

### Bishop Blocker Masks

Bishops move diagonally. Their blocker masks cover the diagonals excluding edges:

```c
uint64 bishop_blocker_mask(int square) {
    uint64 mask = 0;
    int rank = square / 8;
    int file = square % 8;

    // Northeast diagonal
    for (int r = rank + 1, f = file + 1; r < 7 && f < 7; r++, f++) {
        mask |= (1ULL << (r * 8 + f));
    }

    // Northwest diagonal
    for (int r = rank + 1, f = file - 1; r < 7 && f > 0; r++, f--) {
        mask |= (1ULL << (r * 8 + f));
    }

    // Southeast diagonal
    for (int r = rank - 1, f = file + 1; r > 0 && f < 7; r--, f++) {
        mask |= (1ULL << (r * 8 + f));
    }

    // Southwest diagonal
    for (int r = rank - 1, f = file - 1; r > 0 && f > 0; r--, f--) {
        mask |= (1ULL << (r * 8 + f));
    }

    return mask;
}
```

A bishop on a corner has very few relevant squares (5-6).
A bishop in the center has more (up to 9).

## Complete Implementation

Here's how the pieces fit together:

```c
// Global tables
uint64 ROOK_MAGICS[64];
uint64 BISHOP_MAGICS[64];
uint64 ROOK_MASKS[64];
uint64 BISHOP_MASKS[64];
int ROOK_SHIFTS[64];
int BISHOP_SHIFTS[64];
uint64 ROOK_ATTACKS[64][4096];    // [square][index]
uint64 BISHOP_ATTACKS[64][512];   // [square][index]

void init_magics() {
    // For each square, compute mask, find magic, populate table
    for (int sq = 0; sq < 64; sq++) {
        // Rooks
        ROOK_MASKS[sq] = rook_blocker_mask(sq);
        int rook_bits = popcount(ROOK_MASKS[sq]);
        ROOK_SHIFTS[sq] = 64 - rook_bits;
        ROOK_MAGICS[sq] = find_magic(sq, false);  // Or use precomputed magics

        // Populate rook attack table
        uint64 occupancy = 0;
        do {
            int index = (occupancy * ROOK_MAGICS[sq]) >> ROOK_SHIFTS[sq];
            ROOK_ATTACKS[sq][index] = compute_rook_attacks(sq, occupancy);
            occupancy = (occupancy - ROOK_MASKS[sq]) & ROOK_MASKS[sq];
        } while (occupancy);

        // Bishops (similar)
        BISHOP_MASKS[sq] = bishop_blocker_mask(sq);
        int bishop_bits = popcount(BISHOP_MASKS[sq]);
        BISHOP_SHIFTS[sq] = 64 - bishop_bits;
        BISHOP_MAGICS[sq] = find_magic(sq, true);

        uint64 occ = 0;
        do {
            int index = (occ * BISHOP_MAGICS[sq]) >> BISHOP_SHIFTS[sq];
            BISHOP_ATTACKS[sq][index] = compute_bishop_attacks(sq, occ);
            occ = (occ - BISHOP_MASKS[sq]) & BISHOP_MASKS[sq];
        } while (occ);
    }
}

// Runtime attack lookup - THIS IS WHAT WE USE DURING SEARCH
uint64 rook_attacks(int square, uint64 occupancy) {
    occupancy &= ROOK_MASKS[square];  // Keep only relevant bits
    int index = (occupancy * ROOK_MAGICS[square]) >> ROOK_SHIFTS[square];
    return ROOK_ATTACKS[square][index];
}

uint64 bishop_attacks(int square, uint64 occupancy) {
    occupancy &= BISHOP_MASKS[square];
    int index = (occupancy * BISHOP_MAGICS[square]) >> BISHOP_SHIFTS[square];
    return BISHOP_ATTACKS[square][index];
}

// Queen attacks = rook attacks | bishop attacks
uint64 queen_attacks(int square, uint64 occupancy) {
    return rook_attacks(square, occupancy) | bishop_attacks(square, occupancy);
}
```

The runtime lookup is just:
1. Mask the occupancy (AND)
2. Multiply by magic
3. Shift right
4. Index into table

Four operations, a few cycles, regardless of board complexity.

## Precomputed vs Generated Magics

Finding magic numbers takes time (seconds to minutes). You have two options:

**Generate at startup**: Run the magic finder during initialization. Simple but adds startup time.

**Precompute and hardcode**: Find magics once, store them in your source code:

```c
const uint64 ROOK_MAGICS[64] = {
    0x0080001020400080ULL,
    0x0040001000200040ULL,
    0x0080081000200080ULL,
    // ... 61 more values
};

const uint64 BISHOP_MAGICS[64] = {
    0x0002020202020200ULL,
    0x0002020202020000ULL,
    0x0004010202000000ULL,
    // ... 61 more values
};
```

Most engines use precomputed magics for fast startup. Many published magic sets are available online (Chess Programming Wiki has several).

## Fancy Magic Bitboards

The implementation above uses "plain" magics with fixed-size tables per square. **Fancy magic bitboards** use variable-size tables and more sophisticated indexing to reduce total memory usage.

The idea: instead of allocating 4096 entries for every rook square (even those needing only 1024), pack all tables together and use a pointer for each square:

```c
uint64 ROOK_TABLE[102400];  // Shared storage for all squares
uint64* ROOK_ATTACKS[64];   // Pointers into the shared table

void init_fancy_magics() {
    uint64* ptr = ROOK_TABLE;
    for (int sq = 0; sq < 64; sq++) {
        ROOK_ATTACKS[sq] = ptr;
        int bits = popcount(ROOK_MASKS[sq]);
        ptr += (1 << bits);  // Advance by this square's table size
    }
    // Now populate each subtable...
}
```

This reduces memory from 64 × 4096 × 8 = 2MB to roughly 800KB for rooks.

## PEXT Bitboards

On CPUs with the PEXT instruction (BMI2), we can skip magic multiplication entirely:

```c
uint64 rook_attacks_pext(int square, uint64 occupancy) {
    int index = _pext_u64(occupancy, ROOK_MASKS[square]);
    return ROOK_ATTACKS[square][index];
}
```

PEXT directly extracts the relevant bits into a dense index. It's faster and simpler than magic multiplication, but not available on all hardware.

Modern engines often support both, selecting at runtime based on CPU capabilities.

## Memory Layout Considerations

Cache efficiency matters. The attack table lookup pattern is:
1. Load magic and mask for this square (likely in L1 cache if recently used)
2. Compute index
3. Load attack bitboard from table (may be L2 or L3 cache miss)

Table layout affects performance:
- **Plain magics**: Each square has a separate 4096-entry table. Good locality for one square, but tables are spread across memory.
- **Fancy magics**: Packed tables may have better overall cache behavior.
- **Interleaved tables**: Some layouts interleave rook and bishop tables for squares that are used together.

In practice, the differences are small. Start with plain magics; optimize later if profiling shows cache issues.

## Putting It All Together: Sliding Move Generation

With magic bitboards, generating rook moves is elegant:

```c
void generate_rook_moves(Position* pos, int color, MoveList* moves) {
    uint64 rooks = (color == WHITE) ? pos->white_rooks : pos->black_rooks;
    uint64 own = (color == WHITE) ? pos->white_pieces : pos->black_pieces;
    uint64 enemy = (color == WHITE) ? pos->black_pieces : pos->white_pieces;

    while (rooks) {
        int from = bitscan_forward(rooks);

        // Get attacks using magic lookup
        uint64 attacks = rook_attacks(from, pos->all_pieces);

        // Cannot capture own pieces
        attacks &= ~own;

        // Split into captures and quiets
        uint64 captures = attacks & enemy;
        uint64 quiets = attacks & ~enemy;

        // Add capture moves
        while (captures) {
            int to = bitscan_forward(captures);
            add_move(moves, from, to, CAPTURE);
            captures &= captures - 1;
        }

        // Add quiet moves
        while (quiets) {
            int to = bitscan_forward(quiets);
            add_move(moves, from, to, QUIET);
            quiets &= quiets - 1;
        }

        rooks &= rooks - 1;  // Next rook
    }
}
```

The magic lookup is one line: `rook_attacks(from, pos->all_pieces)`. Everything else is standard bitboard iteration.

## Summary

Magic bitboards enable O(1) sliding piece attack computation:

1. **Blocker mask**: The squares that can block a piece from a given square (excluding edges)

2. **Occupancy**: The current blockers on the board, masked to relevant squares

3. **Magic multiplication**: `(occupancy * magic) >> shift` maps occupancy to a table index

4. **Precomputed table**: Stores the attack bitboard for each possible occupancy

The lookup takes ~4 operations and a memory access. This is fast enough for billions of lookups per second during search.

### Key Formulas

```c
// Rook attacks
occupancy &= ROOK_MASKS[square];
index = (occupancy * ROOK_MAGICS[square]) >> ROOK_SHIFTS[square];
attacks = ROOK_ATTACKS[square][index];

// Bishop attacks
occupancy &= BISHOP_MASKS[square];
index = (occupancy * BISHOP_MAGICS[square]) >> BISHOP_SHIFTS[square];
attacks = BISHOP_ATTACKS[square][index];

// Queen attacks
attacks = rook_attacks(sq, occ) | bishop_attacks(sq, occ);
```

### Implementation Checklist

1. Generate or hardcode magic numbers for all 64 squares (rooks and bishops)
2. Compute blocker masks for each square
3. Populate attack tables during initialization
4. Implement the lookup functions
5. Use attacks in move generation and evaluation

With magic bitboards working, you have everything needed for efficient move generation. The next chapters build on this foundation.
