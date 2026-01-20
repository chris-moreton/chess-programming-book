# Chapter 26: UCI Protocol

The **Universal Chess Interface** (UCI) is the standard protocol for communication between chess engines and GUIs. Understanding UCI is essential for making your engine usable.

## Protocol Overview

UCI is text-based. The GUI sends commands; the engine responds:

```
GUI -> Engine: uci
Engine -> GUI: id name MyEngine
Engine -> GUI: id author MyName
Engine -> GUI: option name Hash type spin default 64 min 1 max 1024
Engine -> GUI: uciok

GUI -> Engine: isready
Engine -> GUI: readyok

GUI -> Engine: position startpos moves e2e4 e7e5
GUI -> Engine: go wtime 300000 btime 300000
Engine -> GUI: info depth 10 score cp 25 nodes 1234567 pv e2e4 e7e5 g1f3
Engine -> GUI: bestmove g1f3
```

## Required Commands

### uci

Engine identifies itself and lists options:

```c
void uci_command() {
    printf("id name MyEngine 1.0\n");
    printf("id author My Name\n");
    printf("option name Hash type spin default 64 min 1 max 4096\n");
    printf("option name Threads type spin default 1 min 1 max 64\n");
    printf("uciok\n");
    fflush(stdout);
}
```

### isready

GUI checks if engine is ready. Engine responds when initialization complete:

```c
void isready_command() {
    // Ensure any pending initialization is done
    printf("readyok\n");
    fflush(stdout);
}
```

### position

Set up a position:

```
position startpos
position startpos moves e2e4 e7e5 g1f3
position fen rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w KQkq - 0 1
position fen <fen> moves <move1> <move2> ...
```

```c
void position_command(char* line) {
    char* token = strtok(line, " ");  // "position"
    token = strtok(NULL, " ");

    if (strcmp(token, "startpos") == 0) {
        set_fen(&position, START_FEN);
        token = strtok(NULL, " ");  // "moves" or NULL
    } else if (strcmp(token, "fen") == 0) {
        // Collect FEN string (6 space-separated parts)
        char fen[256] = "";
        for (int i = 0; i < 6; i++) {
            token = strtok(NULL, " ");
            if (!token) break;
            strcat(fen, token);
            strcat(fen, " ");
        }
        set_fen(&position, fen);
        token = strtok(NULL, " ");  // "moves" or NULL
    }

    // Apply moves if present
    if (token && strcmp(token, "moves") == 0) {
        while ((token = strtok(NULL, " ")) != NULL) {
            Move move = parse_uci_move(&position, token);
            make_move(&position, move);
        }
    }
}
```

### go

Start searching:

```
go wtime 300000 btime 300000 winc 0 binc 0
go depth 10
go nodes 1000000
go infinite
go movetime 5000
```

```c
void go_command(char* line) {
    SearchLimits limits = {0};

    char* token = strtok(line, " ");  // "go"
    while ((token = strtok(NULL, " ")) != NULL) {
        if (strcmp(token, "wtime") == 0) {
            limits.wtime = atoi(strtok(NULL, " "));
        } else if (strcmp(token, "btime") == 0) {
            limits.btime = atoi(strtok(NULL, " "));
        } else if (strcmp(token, "winc") == 0) {
            limits.winc = atoi(strtok(NULL, " "));
        } else if (strcmp(token, "binc") == 0) {
            limits.binc = atoi(strtok(NULL, " "));
        } else if (strcmp(token, "movestogo") == 0) {
            limits.movestogo = atoi(strtok(NULL, " "));
        } else if (strcmp(token, "depth") == 0) {
            limits.depth = atoi(strtok(NULL, " "));
        } else if (strcmp(token, "nodes") == 0) {
            limits.nodes = atoll(strtok(NULL, " "));
        } else if (strcmp(token, "movetime") == 0) {
            limits.movetime = atoi(strtok(NULL, " "));
        } else if (strcmp(token, "infinite") == 0) {
            limits.infinite = true;
        }
    }

    start_search(&position, &limits);
}
```

### stop

Stop searching and return best move:

```c
void stop_command() {
    search_info.stop = true;
}
```

### quit

Exit the engine:

```c
void quit_command() {
    exit(0);
}
```

## Output: info

During search, output progress:

```
info depth 12 seldepth 18 score cp 35 nodes 2345678 nps 1234567 time 1900 pv e2e4 e7e5 g1f3 b8c6
```

```c
void print_info(SearchInfo* info) {
    printf("info depth %d", info->depth);
    printf(" seldepth %d", info->seldepth);

    if (is_mate_score(info->score)) {
        int mate_in = mate_distance(info->score);
        printf(" score mate %d", mate_in);
    } else {
        printf(" score cp %d", info->score);
    }

    printf(" nodes %lld", info->nodes);
    printf(" nps %lld", info->nodes * 1000 / max(info->time, 1));
    printf(" time %lld", info->time);

    printf(" pv");
    for (int i = 0; i < info->pv_length; i++) {
        printf(" %s", move_to_uci(info->pv[i]));
    }

    printf("\n");
    fflush(stdout);
}
```

## Output: bestmove

When search finishes:

```
bestmove e2e4
bestmove e2e4 ponder e7e5
```

```c
void print_bestmove(Move move, Move ponder) {
    printf("bestmove %s", move_to_uci(move));
    if (ponder != MOVE_NONE) {
        printf(" ponder %s", move_to_uci(ponder));
    }
    printf("\n");
    fflush(stdout);
}
```

## Engine Options

Define configurable options:

```c
void print_options() {
    printf("option name Hash type spin default 64 min 1 max 4096\n");
    printf("option name Threads type spin default 1 min 1 max 64\n");
    printf("option name Ponder type check default false\n");
    printf("option name UCI_Chess960 type check default false\n");
}

void setoption_command(char* line) {
    // setoption name Hash value 256
    char* name = strstr(line, "name ");
    char* value = strstr(line, "value ");

    if (name) name += 5;
    if (value) value += 6;

    if (strncmp(name, "Hash", 4) == 0) {
        int mb = atoi(value);
        resize_hash_table(mb);
    }
    // ... other options ...
}
```

## Main Loop

```c
int main() {
    init_engine();

    char line[4096];
    while (fgets(line, sizeof(line), stdin)) {
        // Remove newline
        line[strcspn(line, "\n")] = 0;

        if (strncmp(line, "uci", 3) == 0) {
            uci_command();
        } else if (strncmp(line, "isready", 7) == 0) {
            isready_command();
        } else if (strncmp(line, "position", 8) == 0) {
            position_command(line);
        } else if (strncmp(line, "go", 2) == 0) {
            go_command(line);
        } else if (strncmp(line, "stop", 4) == 0) {
            stop_command();
        } else if (strncmp(line, "setoption", 9) == 0) {
            setoption_command(line);
        } else if (strncmp(line, "quit", 4) == 0) {
            break;
        } else if (strncmp(line, "ucinewgame", 10) == 0) {
            clear_hash_table();
        }
    }

    return 0;
}
```

## Threading Considerations

The main thread handles UCI input. Search runs in a separate thread:

```c
void go_command(char* line) {
    // Parse limits...

    // Start search thread
    pthread_create(&search_thread, NULL, search_thread_func, &limits);
}

void stop_command() {
    search_info.stop = true;
    pthread_join(search_thread, NULL);
}
```

## Common Pitfalls

1. **Always flush stdout** after output
2. **Parse moves in position context** (need position to decode e7e8q)
3. **Handle "stop" during search** (check flag periodically)
4. **Respond to "isready" quickly** (don't block on search)

## Summary

UCI protocol:

- **Text-based**: Simple line-by-line communication
- **Required commands**: uci, isready, position, go, stop, quit
- **Output**: info (progress), bestmove (result)
- **Options**: Configurable via setoption
- **Threading**: Search in background, main thread handles input

UCI compatibility makes your engine work with any chess GUI.

Next: Managing the clock—time allocation strategies.
