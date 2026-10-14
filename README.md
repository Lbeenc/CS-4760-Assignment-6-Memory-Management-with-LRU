# Memory Management Simulator (LRU Paging)

A simulation of how an operating system manages memory, written in C for Linux. A master process
(`oss`) plays the role of the OS: it launches user processes, receives their memory reads and writes
over a message queue, keeps a frame table and per-process page tables in shared memory, and handles
page faults with **least-recently-used (LRU) replacement**.

*Course project for CMP SCI 4760 (Operating Systems), University of Missouri–St. Louis.*

## How it works

```
 user_proc (x up to 18 at once)            oss (simulated OS)
 ┌──────────────────────────┐   request    ┌─────────────────────────────────────┐
 │ pick random page+offset  │ ───────────▶ │ page in a frame?  ── HIT  ─▶ reply  │
 │ 30% writes, 70% reads    │  msg queue   │        │                            │
 │ wait for reply           │ ◀─────────── │      FAULT ─▶ free frame or LRU     │
 └──────────────────────────┘    reply     │               victim; dirty? +I/O   │
                                           └─────────────────────────────────────┘
                     shared memory: simulated clock, frame table, page tables
```

- **Processes:** `oss` uses `fork()`/`execl()` to launch up to 100 user processes in total, with at most
  18 running at once. Each process makes 1,000–1,200 memory requests, then exits.
- **Memory model:** 256 frames of 1 KB each, and a 32-page table for each process.
- **Page faults:** the OS uses a free frame if one exists. Otherwise it evicts the least recently
  used frame, chosen by each frame's last-reference timestamp on the simulated clock.
- **Dirty bits:** writes mark a frame dirty. Evicting a dirty frame adds 14 ms of simulated disk I/O.
- **Inter-process communication:** System V shared memory (the clock and memory tables) and
  System V message queues (requests and replies addressed to each process's ID).
- **Logging:** every hit, fault, eviction, and process exit is written to `oss.log`, along with
  periodic snapshots of the frame table (dirty bit, last reference time, owner) and every page table.
- **Cleanup:** a `SIGINT` handler removes the shared memory and message queue, so Ctrl+C doesn't
  leave IPC resources behind.

## Build and run

Requires Linux (or WSL) with `gcc` and `make`.

```bash
make          # builds ./oss and ./user_proc
./oss         # runs the simulation; output goes to oss.log
make clean    # removes binaries and logs
```

If a run is interrupted abnormally, check for leftover IPC resources with `ipcs` and remove them
with `ipcrm`.

## Files

| File | Purpose |
| --- | --- |
| `oss.c` | The simulated OS: process launching, frame and page tables, LRU replacement, logging |
| `user_proc.c` | A user process that generates random reads and writes and waits for each reply |
| `Makefile` | Builds both programs |

## What I'd improve next

- Track simulated "blocked on disk I/O" time per process instead of only advancing the clock.
- Report summary statistics at the end (page-fault rate, hits per second, evictions).
- Replace the linear LRU scan with a structure that finds the victim faster.
