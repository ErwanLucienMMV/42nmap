# Architecture, ownership, and threads

This is a proposed starting architecture. Build it first in serial mode. Use
the same scanning function in serial and threaded modes so concurrency does not
create a second set of protocol rules.

## Suggested source tree

```text
Makefile
include/ft_nmap.h          shared public types and module interfaces
src/main.c                lifecycle and exit status
src/cli.c                 options, strict numbers, scan/port selections
src/targets.c             target files, AF_INET resolution, deduplication
src/route.c               local IPv4 and capture interface selection
src/packet.c              raw IPv4/TCP/UDP serialization and checksums
src/capture.c             pcap setup, supported link-layer decoding
src/match.c               direct/ICMP reply matching and event extraction
src/classify.c            pure event-to-state logic
src/scan.c                attempts, deadlines, per-job execution
src/workers.c             work distribution, worker startup/join
src/services.c            single-threaded service database lookups
src/report.c              stable tables, evidence, duration
tests/                    parser, packet, classifier, and fixture tests
```

These names are suggestions; the subject allows free organization. Avoid one
large shared header if it becomes difficult to understand ownership boundaries.

## Data model

Keep these concepts separate even if their C structures change:

| Object | Fields / responsibility | Owner |
| --- | --- | --- |
| Configuration | port array, scan set, speedup, target source | main, immutable during scans |
| Target | original name, IPv4, source IPv4, interface | main |
| Job | target index, port, transport, scan type, source-port reservation | job planner |
| Probe attempt | sequence/ACK tokens, attempt, send timestamp, deadline | assigned worker |
| Result | state, evidence, attempts, elapsed time, optional error | one assigned worker, then main |
| Worker context | raw socket, pcap handle, scratch buffers | that worker only |
| Work queue | immutable job array plus next index | shared; index protected by mutex |
| Shutdown/error state | stop request and first fatal failure | shared under synchronization |

For each target, allocate `port_count * scan_count` jobs and result slots with
overflow-checked arithmetic. Different jobs own different result slots. The
configuration and target data must stay alive until every worker has joined.

## Baseline thread model

Give each worker its own pcap handle and raw socket. A worker claims a job,
registers its matching context, sends, and waits for a matching reply or deadline
using its own handle. It executes only one job at a time. This is simpler to
reason about than a shared receiver delivering events to many workers.

- `--speedup 0`: main runs this loop, without calling `pthread_create`.
- `--speedup 1`: one worker thread; main coordinates and joins.
- `--speedup N`: create `min(N, job_count)` workers, at most 250.
- No additional capture, timer, or printing threads in this model.
- Process targets sequentially at first; jobs for each target still run in parallel.

The main thread is not counted as a created worker. Clearly report requested
and effective counts. Never create one thread per port or per packet.

Independent handles duplicate capture work and buffer memory at high worker
counts. Narrow BPF filters where practical, choose capture buffers deliberately,
and measure resource use at 250 workers. A later single-receiver design can
reduce duplication, but must retain bounded thread count and explicit ownership.

## Startup and job execution

Perform target route selection first. Each worker initializes its handles and
signals ready or failed through a condition-variable startup gate. Main waits
for all created workers to report their setup state before releasing the gate.
Use a custom mutex/condition gate if portability makes `pthread_barrier_t`
undesirable. A startup failure must release waiting threads before joining them.

```text
worker:
    initialize private context
    report setup result; wait for release or stop
    while not stopped and a job can be claimed:
        activate this job's tuple/token in private state
        for each allowed attempt:
            build bytes and set absolute deadline
            send probe; on send failure record operational error
            while deadline remains and not stopped:
                drain a bounded capture batch
                decode -> correlate -> classify matching events
                stop job if conclusive
                wait briefly without blocking past deadline
        if no matching evidence after all attempts:
            use scan-specific silence state
        write only this job's result slot
    release private context; return

main:
    parse -> resolve -> plan -> start/release workers -> join
    resolve service labels -> produce sorted report -> free shared data
```

Register correlation state before sending so a fast reply is recognized. If a
worker changes its BPF per job, install the filter before sending and still
reject stale buffered packets in software. Start with a stable target-level
filter to avoid frequent recompilation.

## Synchronization rules

Protect the queue index only while claiming a job. Do not hold that lock while
performing DNS, pcap calls, `sendto`, sleeping, or printing. A condition wait
must recheck its predicate in a loop, including during shutdown.

Immutable input and worker-owned result slots avoid many locks. Main reads
results only after joins. Shared error/stop flags need a mutex or C atomics;
`volatile` alone does not synchronize threads. Handle pthread return codes
directly; many pthread APIs return an error number rather than setting `errno`.

Perform `getservbyport` after joining, outside worker threads. Keep all regular
output in main. This prevents interleaved lines and shared static resolver data
from corrupting a report.

## Cleanup and failures

Initialize handles to NULL and FDs to -1. For each successfully acquired resource,
define one owner and one release operation. On a partial `pthread_create`
failure, request stop, release the startup gate, join every thread actually
created, and only then free shared state. Never join uninitialized thread IDs.

Each worker closes its own pcap handle and socket after it stops using them.
Free compiled BPF programs, interface lists, resolver results, line buffers,
port reservations, jobs, and results on all exit paths. Do not call `exit`
inside a worker; return an error to main. If capture fails, do not turn every
remaining port into `filtered`.

For Ctrl+C, a signal handler may set a `volatile sig_atomic_t` interruption flag.
Do not invoke stdio, malloc, pthread locks, or pcap cleanup from that handler.
Normal control flow translates the interruption into synchronized worker stop
requests, joins workers, and releases resources. Nonblocking bounded waits make
this feasible without unsafe cross-thread handle closure.
