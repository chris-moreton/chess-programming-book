# Chapter 32: Parallel Search

Modern CPUs have many cores. Parallel search uses them all to search deeper in the same time.

## The Challenge

Alpha-beta is inherently sequential—each node depends on results from previous nodes for pruning. Naive parallelization loses most cutoffs.

```
Sequential:
  Move 1 → Score 30 → alpha = 30
  Move 2 → Cutoff (score > 30 found quickly due to good alpha)
  Move 3 → Cutoff
  ...

Naive parallel:
  Move 1 → Score 30 (Thread 1)
  Move 2 → Full search (no alpha yet!) (Thread 2)
  Move 3 → Full search (no alpha yet!) (Thread 3)
```

## Lazy SMP

**Lazy SMP** (Symmetric Multi-Processing) is simple and effective:

### Concept

All threads search the same tree with slightly different parameters. They share a transposition table.

```c
typedef struct {
    int thread_id;
    Position pos;
    int depth;
    volatile bool* stop;
    SearchInfo* shared_info;
} ThreadData;

void* search_thread(void* arg) {
    ThreadData* td = (ThreadData*)arg;

    // Each thread searches with slight depth variation
    int depth = td->depth + (td->thread_id % 2);

    for (int d = 1; d <= depth; d++) {
        search(&td->pos, d, -INF, INF);

        if (*td->stop) break;
    }

    return NULL;
}

void parallel_search(Position* pos, int depth, int num_threads) {
    pthread_t threads[MAX_THREADS];
    ThreadData thread_data[MAX_THREADS];
    volatile bool stop = false;

    // Start helper threads
    for (int i = 1; i < num_threads; i++) {
        thread_data[i].thread_id = i;
        thread_data[i].pos = *pos;  // Copy position
        thread_data[i].depth = depth;
        thread_data[i].stop = &stop;

        pthread_create(&threads[i], NULL, search_thread, &thread_data[i]);
    }

    // Main thread searches
    thread_data[0].thread_id = 0;
    thread_data[0].pos = *pos;
    thread_data[0].depth = depth;
    thread_data[0].stop = &stop;
    search_thread(&thread_data[0]);

    // Signal stop and wait for threads
    stop = true;
    for (int i = 1; i < num_threads; i++) {
        pthread_join(threads[i], NULL);
    }
}
```

### Why It Works

- Threads diverge due to hash collisions and move ordering differences
- First thread to find good moves updates TT
- Other threads benefit from TT entries
- Natural load balancing

### Thread-Safe Transposition Table

```c
typedef struct {
    uint64_t key;
    int16_t score;
    int8_t depth;
    uint8_t flag;
    Move best_move;
} TTEntry;

// Lockless TT - multiple threads read/write
// Use XOR trick for data integrity
typedef struct {
    uint64_t data;   // Packed: score, depth, flag, move
    uint64_t key;    // XOR with data for verification
} TTEntryAtomic;

void tt_store(uint64_t hash, int score, int depth, int flag, Move move) {
    int idx = hash % TT_SIZE;

    uint64_t data = pack_entry(score, depth, flag, move);
    uint64_t key = hash ^ data;

    TT[idx].data = data;
    TT[idx].key = key;
}

bool tt_probe(uint64_t hash, TTEntry* entry) {
    int idx = hash % TT_SIZE;

    uint64_t data = TT[idx].data;
    uint64_t key = TT[idx].key;

    // Verify integrity
    if ((key ^ data) != hash) {
        return false;  // Corrupted or wrong entry
    }

    unpack_entry(data, entry);
    return true;
}
```

## Young Brothers Wait Concept (YBWC)

More sophisticated approach: only parallelize after first move is searched.

### Principle

The first move at a node (likely best due to move ordering) is searched sequentially. Remaining moves can be searched in parallel.

```c
int search_ybwc(Position* pos, int depth, int alpha, int beta) {
    MoveList moves;
    generate_moves(pos, &moves);
    order_moves(pos, &moves);

    if (moves.count == 0) return evaluate_terminal(pos);

    // Search first move sequentially
    make_move(pos, moves.moves[0]);
    int best = -search_ybwc(pos, depth - 1, -beta, -alpha);
    unmake_move(pos, moves.moves[0]);

    if (best >= beta) return best;
    if (best > alpha) alpha = best;

    // Search remaining moves in parallel (if deep enough)
    if (depth >= MIN_SPLIT_DEPTH) {
        parallel_search_remaining(&moves, 1, moves.count, pos, depth, alpha, beta);
    } else {
        // Sequential fallback
        for (int i = 1; i < moves.count; i++) {
            make_move(pos, moves.moves[i]);
            int score = -search_ybwc(pos, depth - 1, -beta, -alpha);
            unmake_move(pos, moves.moves[i]);

            if (score > best) {
                best = score;
                if (score > alpha) alpha = score;
                if (score >= beta) break;
            }
        }
    }

    return best;
}
```

### Work Splitting

```c
typedef struct {
    Position pos;
    MoveList* moves;
    int start_idx;
    int end_idx;
    int depth;
    int alpha;
    int beta;
    int* best_score;
    Move* best_move;
    pthread_mutex_t* lock;
} SplitPoint;

void* search_split_point(void* arg) {
    SplitPoint* sp = (SplitPoint*)arg;

    for (int i = sp->start_idx; i < sp->end_idx; i++) {
        make_move(&sp->pos, sp->moves->moves[i]);
        int score = -search(&sp->pos, sp->depth - 1, -sp->beta, -sp->alpha);
        unmake_move(&sp->pos, sp->moves->moves[i]);

        pthread_mutex_lock(sp->lock);
        if (score > *sp->best_score) {
            *sp->best_score = score;
            *sp->best_move = sp->moves->moves[i];
            if (score > sp->alpha) {
                sp->alpha = score;
            }
        }
        pthread_mutex_unlock(sp->lock);

        if (score >= sp->beta) break;  // Cutoff
    }

    return NULL;
}
```

## Principal Variation Splitting (PVS Parallel)

Split only at PV nodes for maximum efficiency:

```c
bool is_pv_node(int alpha, int beta) {
    return beta > alpha + 1;  // Not a null window
}

int search_pv_split(Position* pos, int depth, int alpha, int beta) {
    if (is_pv_node(alpha, beta) && depth >= SPLIT_DEPTH) {
        return search_with_splitting(pos, depth, alpha, beta);
    } else {
        return search_sequential(pos, depth, alpha, beta);
    }
}
```

## Thread Pool

Efficient thread management:

```c
typedef struct {
    pthread_t thread;
    int id;
    bool idle;
    Position pos;
    int depth;
    volatile bool searching;
} Worker;

typedef struct {
    Worker workers[MAX_THREADS];
    int num_workers;
    pthread_mutex_t lock;
    pthread_cond_t work_available;
    pthread_cond_t search_done;
} ThreadPool;

void init_thread_pool(ThreadPool* pool, int num_threads) {
    pool->num_workers = num_threads;
    pthread_mutex_init(&pool->lock, NULL);
    pthread_cond_init(&pool->work_available, NULL);
    pthread_cond_init(&pool->search_done, NULL);

    for (int i = 0; i < num_threads; i++) {
        pool->workers[i].id = i;
        pool->workers[i].idle = true;
        pool->workers[i].searching = false;

        pthread_create(&pool->workers[i].thread, NULL, worker_thread, &pool->workers[i]);
    }
}

void* worker_thread(void* arg) {
    Worker* worker = (Worker*)arg;

    while (true) {
        pthread_mutex_lock(&pool_lock);

        // Wait for work
        while (worker->idle && !shutdown) {
            pthread_cond_wait(&work_available, &pool_lock);
        }

        if (shutdown) {
            pthread_mutex_unlock(&pool_lock);
            break;
        }

        pthread_mutex_unlock(&pool_lock);

        // Do search
        worker->searching = true;
        search(&worker->pos, worker->depth);
        worker->searching = false;
        worker->idle = true;

        pthread_cond_signal(&search_done);
    }

    return NULL;
}
```

## Node Counting

Aggregate node counts across threads:

```c
typedef struct {
    atomic_uint64_t nodes;
    atomic_uint64_t tb_hits;
} SharedStats;

void increment_nodes(SharedStats* stats) {
    atomic_fetch_add(&stats->nodes, 1);
}

// Or batch updates for less contention:
void search_with_batched_stats(Position* pos, int depth) {
    uint64_t local_nodes = 0;

    for (/* each move */) {
        // search...
        local_nodes++;

        // Periodically sync
        if ((local_nodes & 1023) == 0) {
            atomic_fetch_add(&shared_stats.nodes, local_nodes);
            local_nodes = 0;
        }
    }

    // Final sync
    atomic_fetch_add(&shared_stats.nodes, local_nodes);
}
```

## ABDADA

**Alpha-Beta Distributed Search with Deferred Allowed**—prevents duplicate work.

### Concept

Before searching a node, check if another thread is already searching it. If so, defer it for later.

```c
typedef struct {
    uint64_t hash;
    atomic_int searching;  // Thread ID or -1
} ABDADAEntry;

ABDADAEntry abdada_table[ABDADA_SIZE];

bool try_start_search(uint64_t hash, int thread_id) {
    int idx = hash % ABDADA_SIZE;

    int expected = -1;
    return atomic_compare_exchange_strong(&abdada_table[idx].searching,
                                          &expected, thread_id);
}

void finish_search(uint64_t hash) {
    int idx = hash % ABDADA_SIZE;
    atomic_store(&abdada_table[idx].searching, -1);
}

int search_abdada(Position* pos, int depth, int alpha, int beta, bool defer_allowed) {
    uint64_t hash = pos->hash;

    // Check if another thread is searching this
    if (defer_allowed && !try_start_search(hash, thread_id)) {
        return DEFERRED;  // Try later
    }

    int result = search_internal(pos, depth, alpha, beta);

    finish_search(hash);
    return result;
}
```

## Stockfish's Approach

Stockfish uses Lazy SMP with refinements:

1. **Thread differentiation**: Different threads use different depths and search parameters
2. **Shared TT**: All threads share one transposition table
3. **Main thread decides**: Main thread controls time and reports bestmove
4. **Node counter**: Atomic aggregate of all thread nodes

```c
// Simplified Stockfish-style approach
void stockfish_parallel_search(Position* pos, int depth) {
    // Main thread starts iterative deepening
    for (int d = 1; d <= depth; d++) {
        // Helper threads search at d or d+1
        start_helpers(pos, d);

        // Main thread searches
        search_root(pos, d);

        // Check time
        if (should_stop()) break;
    }

    stop_helpers();
}
```

## Speedup Expectations

Theoretical vs practical speedup:

| Threads | Ideal | Typical |
|---------|-------|---------|
| 1 | 1.0x | 1.0x |
| 2 | 2.0x | 1.5-1.7x |
| 4 | 4.0x | 2.5-3.0x |
| 8 | 8.0x | 4.0-5.0x |
| 16 | 16.0x | 6.0-8.0x |

Diminishing returns due to:
- Search overhead (speculation)
- Synchronization costs
- Memory bandwidth limits
- Alpha-beta's sequential nature

## Elo Gain

Approximate Elo gain from threads:

| Threads | Elo vs 1 thread |
|---------|-----------------|
| 2 | +50-70 |
| 4 | +100-130 |
| 8 | +140-180 |
| 16 | +170-220 |

## Implementation Tips

### 1. Start Simple

Begin with Lazy SMP—it's simple and effective:

```c
// Each thread: search independently, share TT
// Main thread: report best result
```

### 2. Thread-Local State

Keep thread-local copies of:
- Position
- Move stack
- Killers
- History tables (can be shared with care)

### 3. Avoid Contention

Minimize shared mutable state:
- TT: Lockless with XOR verification
- Node count: Batch updates
- Best move: Only main thread reports

### 4. Test Thoroughly

Parallel bugs are subtle:
- Race conditions
- Deadlocks
- Corruption from simultaneous TT access

Use thread sanitizers:
```bash
gcc -fsanitize=thread -g -o engine engine.c
```

## Summary

Parallel search approaches:

- **Lazy SMP**: Simple, effective—threads search independently with shared TT
- **YBWC**: Search first move sequentially, then parallelize
- **PVS Parallel**: Split only at PV nodes
- **ABDADA**: Prevent duplicate work with deferred moves

Key considerations:

- Thread-safe transposition table
- Minimize synchronization
- Expect 5-8x speedup with 16 threads
- Lazy SMP is the practical choice for most engines

Multi-threading adds 150-200 Elo with 8+ cores.

Part 7 complete. Next: Appendices with reference material.
