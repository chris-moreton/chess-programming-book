# Chapter 27: Time Management

Allocating thinking time per move is critical. Think too long early, and you'll run out of time. Think too briefly, and you'll miss good moves. This chapter covers time allocation strategies.

## Time Controls

Common time controls:

- **Sudden death**: Fixed time for entire game (e.g., 5 minutes)
- **Increment**: Time added per move (e.g., 3min + 2sec/move)
- **Moves-to-go**: Must complete N moves before time is added

```c
typedef struct {
    int wtime;      // White's remaining time (ms)
    int btime;      // Black's remaining time (ms)
    int winc;       // White's increment (ms)
    int binc;       // Black's increment (ms)
    int movestogo;  // Moves until next time control (0 = sudden death)
    int movetime;   // Fixed time for this move (overrides others)
    int depth;      // Search to fixed depth
    bool infinite;  // Search until "stop"
} SearchLimits;
```

## Basic Time Allocation

Divide remaining time by expected moves:

```c
void calculate_time(SearchLimits* limits, int side, int* soft_limit, int* hard_limit) {
    int time_left = (side == WHITE) ? limits->wtime : limits->btime;
    int increment = (side == WHITE) ? limits->winc : limits->binc;

    if (limits->movetime > 0) {
        // Fixed time per move
        *soft_limit = limits->movetime;
        *hard_limit = limits->movetime;
        return;
    }

    // Estimate moves remaining
    int moves_to_go = limits->movestogo;
    if (moves_to_go == 0) {
        moves_to_go = 30;  // Assume ~30 moves left in sudden death
    }

    // Base time per move
    int base_time = time_left / moves_to_go + increment;

    // Soft limit: target time
    *soft_limit = base_time * 60 / 100;  // 60% of base

    // Hard limit: absolute maximum
    *hard_limit = min(base_time * 200 / 100, time_left / 4);

    // Safety margin
    *soft_limit = max(*soft_limit - 50, 10);
    *hard_limit = max(*hard_limit - 50, 10);
}
```

## Soft and Hard Limits

- **Soft limit**: Target time; try to finish by this point
- **Hard limit**: Never exceed this; stop immediately

```c
void search_loop() {
    for (int depth = 1; depth <= MAX_DEPTH; depth++) {
        // Don't start new depth if soft limit exceeded
        if (elapsed_ms() > soft_limit && depth > 1) {
            break;
        }

        search(depth);

        // Stop if hard limit hit
        if (elapsed_ms() > hard_limit) {
            break;
        }
    }
}

// In search, check periodically:
if ((nodes & 2047) == 0 && elapsed_ms() > hard_limit) {
    stop_search();
}
```

## Dynamic Time Adjustment

Adjust time based on search behavior:

### Score Dropping

If the score drops significantly, think longer:

```c
int prev_score = search_info.score;
int new_score = search(depth);

if (new_score < prev_score - 50) {
    // Score dropped—position is tricky
    soft_limit = soft_limit * 150 / 100;
}
```

### Best Move Changing

If the best move keeps changing, think longer:

```c
if (best_move != prev_best_move) {
    move_changes++;
    if (move_changes > 2) {
        soft_limit = soft_limit * 130 / 100;
    }
}
```

### Best Move Stable

If the same move is best for many iterations, can stop early:

```c
if (best_move == prev_best_move) {
    stable_count++;
    if (stable_count >= 5 && depth >= 10) {
        soft_limit = soft_limit * 70 / 100;  // Can stop sooner
    }
} else {
    stable_count = 0;
}
```

### Easy Move

If one move is clearly best (score much higher than others):

```c
if (best_score > second_best_score + 150) {
    // Clear best move—don't need to think long
    soft_limit = soft_limit * 50 / 100;
}
```

## Time Trouble

Handle low time situations:

```c
if (time_left < 30000) {  // Under 30 seconds
    // Reduce thinking time drastically
    soft_limit = min(soft_limit, time_left / 15);
    hard_limit = min(hard_limit, time_left / 5);
}

if (time_left < 10000) {  // Under 10 seconds
    // Emergency mode—use increment only
    soft_limit = min(soft_limit, increment * 80 / 100);
    hard_limit = min(hard_limit, time_left / 3);
}
```

## Opening Book Time

Don't waste time on book moves:

```c
if (use_opening_book) {
    Move book_move = probe_book(&position);
    if (book_move != MOVE_NONE) {
        // No thinking needed
        return book_move;
    }
}
```

## Complete Time Management

```c
typedef struct {
    int64_t start_time;
    int soft_limit;
    int hard_limit;
    int optimal_time;
    int maximum_time;
    bool stopped;
} TimeManager;

void init_time_manager(TimeManager* tm, SearchLimits* limits, int side) {
    tm->start_time = current_time_ms();
    tm->stopped = false;

    if (limits->infinite) {
        tm->soft_limit = INT_MAX;
        tm->hard_limit = INT_MAX;
        return;
    }

    if (limits->movetime > 0) {
        tm->soft_limit = limits->movetime;
        tm->hard_limit = limits->movetime;
        return;
    }

    int time_left = (side == WHITE) ? limits->wtime : limits->btime;
    int inc = (side == WHITE) ? limits->winc : limits->binc;
    int mtg = limits->movestogo ? limits->movestogo : 40;

    // Optimal time = what we'd ideally use
    tm->optimal_time = time_left / mtg + inc - 50;
    tm->optimal_time = max(tm->optimal_time, 10);

    // Maximum time = never exceed this
    tm->maximum_time = min(time_left / 4, tm->optimal_time * 5);
    tm->maximum_time = max(tm->maximum_time, 10);

    tm->soft_limit = tm->optimal_time;
    tm->hard_limit = tm->maximum_time;
}

bool should_stop_soft(TimeManager* tm) {
    return elapsed_ms(tm) >= tm->soft_limit;
}

bool should_stop_hard(TimeManager* tm) {
    return elapsed_ms(tm) >= tm->hard_limit;
}

void adjust_time(TimeManager* tm, SearchInfo* info, int prev_score, Move prev_move) {
    // Score dropped
    if (info->score < prev_score - 30) {
        tm->soft_limit = tm->soft_limit * 140 / 100;
    }

    // Move changed
    if (info->best_move != prev_move) {
        tm->soft_limit = tm->soft_limit * 120 / 100;
    }

    // Clamp to maximum
    tm->soft_limit = min(tm->soft_limit, tm->hard_limit);
}
```

## Pondering

**Pondering** (thinking on opponent's time) extends effective thinking time:

```c
// After outputting bestmove:
if (pondering_enabled && ponder_move != MOVE_NONE) {
    // Make the expected opponent move
    make_move(&position, best_move);
    make_move(&position, ponder_move);

    // Search while waiting for opponent
    search_ponder(&position);

    // If opponent plays expected move ("ponderhit"):
    // Continue search with accumulated knowledge
}
```

Most modern engines benefit significantly from pondering.

## Summary

Time management:

- **Soft limit**: Target time; finish iteration if exceeded
- **Hard limit**: Absolute maximum; stop immediately
- **Dynamic adjustment**: Think longer if position is tricky
- **Time trouble**: Reduce time drastically when low on clock
- **Easy moves**: Stop early if best move is clear
- **Pondering**: Think on opponent's time for efficiency

Good time management can gain 50-100 Elo by avoiding time-pressure blunders.

Next: Debugging techniques—finding and fixing bugs.
