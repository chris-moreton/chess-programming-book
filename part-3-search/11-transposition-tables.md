# Chapter 11: Transposition Tables

A **transposition** occurs when the same position arises via different move orders. The opening sequence 1.e4 e5 2.Nf3 reaches the same position as 1.Nf3 e5 2.e4. Without memory, we'd analyze identical positions repeatedly.

A **transposition table** (TT) is a hash table storing previously searched positions. When we encounter a position again, we can reuse stored results instead of re-searching.

## Why Transpositions Matter

### Frequency of Transpositions

In typical middlegame positions:
- 20-30% of searched positions are transpositions
- In endgames, the percentage is higher

### Savings

Without a TT, searching depth 10 might examine 100 million nodes.
With a TT, maybe 30 million—a 70% reduction.

The TT also stores the best move, improving move ordering dramatically.

## Hash Table Structure

The TT is indexed by Zobrist hash (Chapter 7). Each entry stores:

```c
typedef struct {
    uint64_t key;      // Zobrist hash (for collision detection)
    int16_t score;     // Evaluation or search score
    Move best_move;    // Best move found
    uint8_t depth;     // Search depth
    uint8_t flag;      // Score type (exact, lower bound, upper bound)
    uint8_t age;       // For replacement policy
} TTEntry;
```

### Entry Flags

The flag indicates how to interpret the score:

```c
enum TTFlag {
    TT_EXACT,  // Score is exact (PV node)
    TT_LOWER,  // Score is lower bound (fail-high/cut node)
    TT_UPPER,  // Score is upper bound (fail-low/all node)
};
```

Why different flags?
- **Beta cutoff**: We know score ≥ beta, but not the exact score
- **All node (no improvement)**: We know score ≤ alpha, but not the exact score
- **PV node**: We searched all moves with alpha < score < beta

## Using the Transposition Table

### Probing

At the start of search, check if we've seen this position:

```c
int alpha_beta(Position* pos, int depth, int alpha, int beta) {
    // Probe TT
    TTEntry* entry = tt_probe(pos->hash);

    if (entry != NULL && entry->depth >= depth) {
        // Entry exists with sufficient depth
        switch (entry->flag) {
            case TT_EXACT:
                return entry->score;  // Exact score—use directly

            case TT_LOWER:
                // Score is at least entry->score
                if (entry->score >= beta) {
                    return entry->score;  // Fail-high
                }
                alpha = max(alpha, entry->score);
                break;

            case TT_UPPER:
                // Score is at most entry->score
                if (entry->score <= alpha) {
                    return entry->score;  // Fail-low
                }
                beta = min(beta, entry->score);
                break;
        }
    }

    // Get hash move for move ordering
    Move hash_move = (entry != NULL) ? entry->best_move : MOVE_NONE;

    // ... continue search using hash_move for ordering ...
}
```

### Storing

After searching a position, store the result:

```c
int alpha_beta(Position* pos, int depth, int alpha, int beta) {
    int original_alpha = alpha;

    // ... search ...

    // Determine flag based on score
    uint8_t flag;
    if (best_score <= original_alpha) {
        flag = TT_UPPER;  // Failed low—score is upper bound
    } else if (best_score >= beta) {
        flag = TT_LOWER;  // Failed high—score is lower bound
    } else {
        flag = TT_EXACT;  // Exact score
    }

    tt_store(pos->hash, depth, best_score, best_move, flag);

    return best_score;
}
```

## Table Indexing

The hash table size is typically a power of 2 for fast indexing:

```c
#define TT_SIZE (1 << 24)  // 16M entries = ~256MB for 16-byte entries
#define TT_MASK (TT_SIZE - 1)

TTEntry tt[TT_SIZE];

TTEntry* tt_probe(uint64_t hash) {
    int index = hash & TT_MASK;
    TTEntry* entry = &tt[index];

    // Verify key matches (detect collisions)
    if (entry->key == hash) {
        return entry;
    }

    return NULL;  // Miss or collision
}

void tt_store(uint64_t hash, int depth, int score, Move move, uint8_t flag) {
    int index = hash & TT_MASK;
    TTEntry* entry = &tt[index];

    // Replacement policy (see below)
    if (should_replace(entry, depth)) {
        entry->key = hash;
        entry->score = score;
        entry->best_move = move;
        entry->depth = depth;
        entry->flag = flag;
        entry->age = current_age;
    }
}
```

## Handling Collisions

### Type 1: Index Collision

Different positions map to the same table index. The key field detects this:

```c
if (entry->key != hash) {
    // Different position—don't use this entry
}
```

### Type 2: Key Collision

Different positions have the same 64-bit hash. Extremely rare (~1 in 2^64), but possible. Most engines accept this tiny error rate.

For more safety, store additional position data:

```c
typedef struct {
    uint32_t key_high;    // Upper 32 bits of hash
    // (lower 32 bits implicit from index)
    // ...
} TTEntry;

// At probe:
if (entry->key_high != (hash >> 32)) {
    return NULL;
}
```

## Replacement Policies

When storing to an occupied slot, which entry should we keep?

### Always Replace

Simplest policy: always overwrite.

```c
void tt_store(...) {
    int index = hash & TT_MASK;
    tt[index] = new_entry;  // Always overwrite
}
```

Problem: Loses valuable deep entries when replaced by shallow ones.

### Depth Preferred

Keep entries from deeper searches:

```c
if (depth >= entry->depth) {
    // Replace: new entry is from equal or deeper search
    *entry = new_entry;
}
```

Problem: Old deep entries can become stale across many positions.

### Age-Based

Track which search iteration created the entry:

```c
uint8_t current_search_age;  // Incremented each new search

bool should_replace(TTEntry* entry, int depth) {
    // Always replace old entries
    if (entry->age != current_search_age) {
        return true;
    }

    // For same-age entries, prefer deeper
    return depth >= entry->depth;
}
```

### Two-Tier (Bucket) Systems

Store multiple entries per index:

```c
#define BUCKET_SIZE 4

typedef struct {
    TTEntry entries[BUCKET_SIZE];
} TTBucket;

TTBucket tt[TT_SIZE / BUCKET_SIZE];
```

This allows keeping both a deep entry and a recent shallow entry.

Stockfish uses a 3-entry bucket with replacement based on depth, age, and bound type.

## Mate Score Adjustment

Mate scores include the distance from the root:

```
Mate in 5 at root = MATE_SCORE - 5
```

But stored positions might be reached at different depths. Adjust:

```c
// When storing:
if (is_mate_score(score)) {
    if (score > 0) {
        score += ply;  // Store distance from this node
    } else {
        score -= ply;
    }
}

// When retrieving:
if (is_mate_score(entry->score)) {
    if (entry->score > 0) {
        score = entry->score - ply;  // Adjust for current ply
    } else {
        score = entry->score + ply;
    }
}
```

This ensures "mate in 3" means 3 moves from the current position, not from some other position in a previous search.

## Extracting the Principal Variation

The TT contains the PV implicitly. Extract it by following hash moves:

```c
void extract_pv(Position* pos, Move* pv, int* pv_length) {
    Position temp = *pos;
    *pv_length = 0;

    for (int i = 0; i < MAX_PLY; i++) {
        TTEntry* entry = tt_probe(temp.hash);

        if (entry == NULL || entry->best_move == MOVE_NONE) {
            break;
        }

        // Verify move is legal (guards against hash collisions)
        if (!is_legal_move(&temp, entry->best_move)) {
            break;
        }

        pv[(*pv_length)++] = entry->best_move;
        make_move(&temp, entry->best_move);

        // Stop at repetition (prevents infinite loops)
        if (is_repetition(&temp)) {
            break;
        }
    }
}
```

This can be unreliable due to overwrites. Engines often maintain a separate PV array (triangular PV table) for display.

## TT in Quiescence Search

The TT is useful in quiescence search too:

```c
int quiesce(Position* pos, int alpha, int beta) {
    // Probe TT
    TTEntry* entry = tt_probe(pos->hash);
    if (entry && entry->flag == TT_EXACT) {
        return entry->score;  // Use stored score
    }

    // ... quiescence search ...

    // Store with depth 0 (or -1 for qsearch)
    tt_store(pos->hash, 0, best_score, best_move, flag);

    return best_score;
}
```

Since quiescence searches to variable depth, storing is less clear-cut. Some engines skip TT in qsearch.

## Memory Management

### Sizing

Typical sizes:
- Desktop: 256MB - 1GB
- Tournament: 1-4GB
- Analysis: 4-16GB

```c
// Calculate entries from megabytes
int entries = (size_mb * 1024 * 1024) / sizeof(TTEntry);
```

### Clearing

Clear the table between games or when setting a new position:

```c
void tt_clear() {
    memset(tt, 0, sizeof(tt));
}
```

Or use "generation" counting to invalidate old entries without clearing:

```c
void tt_new_search() {
    current_search_age++;
}
```

### Prefetching

Modern CPUs can prefetch memory. When making a move, prefetch the TT entry for the resulting position:

```c
make_move(pos, move);
__builtin_prefetch(&tt[pos->hash & TT_MASK]);
// By the time we probe, data may be in cache
```

## Debugging the TT

### Verify Key Uniqueness

After a long search, check for unexpected key collisions:

```c
// Count entries with same key (should be 0 or very rare)
```

### Hash Verification

Periodically verify stored hashes match recomputed hashes:

```c
uint64_t verify = compute_hash(pos);
assert(pos->hash == verify);
```

### Hit Rate Monitoring

Track TT effectiveness:

```c
int tt_probes = 0;
int tt_hits = 0;

// Report hit rate
printf("TT hit rate: %.1f%%\n", 100.0 * tt_hits / tt_probes);
```

A good hit rate is 30-60%. Lower suggests insufficient memory; higher is unusual.

## Summary

The transposition table is essential for efficient search:

- **Avoids re-searching** identical positions reached by different move orders
- **Provides move ordering** through stored best moves
- **Uses Zobrist hashing** for fast, incremental key computation
- **Stores bounds** (exact, lower, upper) based on node type
- **Requires replacement policy** to manage limited memory
- **Needs mate score adjustment** for correct mate distances

The TT typically reduces node count by 30-50% and dramatically improves move ordering through hash moves.

Next: How to order moves for maximum alpha-beta efficiency.
