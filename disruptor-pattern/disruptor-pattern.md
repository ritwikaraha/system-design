# The Disruptor Pattern

One paragraph, then the full picture.

------------------------------------------------------------------------

## The One-Sentence Answer

The Disruptor is an **inter-thread communication pattern** --- a way for
one thread to hand data to other threads at maximum speed, without
locks, by pre-allocating a fixed circular buffer and using a single
monotonically increasing sequence number as the only coordination
primitive.

------------------------------------------------------------------------

## Why It Exists --- The Problem It Solves

Every program that processes a stream of events needs a queue between
the producer (thing generating events) and the consumer (thing
processing them). The standard answer is a `BlockingQueue` or
`LinkedBlockingQueue`.

Here is what happens inside a `LinkedBlockingQueue` under load:

``` text
Producer                    Queue                    Consumer
   │                          │                          │
   ├── lock(mutex) ──────────►│                          │
   ├── allocate Node ─────────┤                          │
   ├── enqueue ───────────────┤                          │
   ├── signal() ──────────────┤                          │
   ├── unlock(mutex) ─────────┤                          │
   │                          ├── lock(mutex) ──────────►│
   │                          ├── dequeue ───────────────┤
   │                          ├── unlock(mutex) ─────────┤
   │                          │                          │
```

Four problems:

1.  **Lock contention** --- producer and consumer fight for the same
    mutex. One always waits.
2.  **Heap allocation** --- every `new Node()` pressures the GC. GC
    pauses kill latency.
3.  **Cache line bouncing** --- the queue head/tail pointer is written
    by both threads; every write invalidates it on all CPU cores.
4.  **Context switching** --- `signal()` / `wait()` causes the OS to
    park and unpark threads. That alone costs 1--10 microseconds.

LMAX Exchange (a financial trading platform, 2011) needed to process 6
million orders per second with microsecond latency.
`LinkedBlockingQueue` couldn't get them there. They redesigned from
first principles. The result was the Disruptor.

------------------------------------------------------------------------

## The Core Idea --- Three Design Decisions

### Decision 1: Pre-Allocate Everything

Instead of creating and discarding objects (which triggers GC), allocate
all slots once at startup and reuse them forever.

``` text
Ring buffer: [ slot 0 | slot 1 | slot 2 | slot 3 | ... | slot N-1 ]
                  ↑ pre-allocated, never freed, mutated in place
```

No allocation in the hot path. No GC pressure. Predictable memory
layout.

### Decision 2: A Ring Buffer, Not a Linked List

A ring buffer is a fixed array. You never add or remove elements --- you
write into position `sequence % size` and read from the same position.
When you reach the end, you wrap around to position 0.

``` text
Sequence: 0  1  2  3  4  5  6  7  0  1  2  3 ...
Slot:     0  1  2  3  4  5  6  7  0  1  2  3 ...
               ↑ wraps around — infinite logical sequence,
                 finite physical storage
```

Because it is a plain array, access is O(1), cache-friendly (sequential
memory addresses), and the slot for sequence N is always at index
`N & (size - 1)` --- a bitmask, not a division.

### Decision 3: Sequence Numbers as the Coordination Primitive

No locks. No signals. No CAS on a shared queue pointer.

Every participant owns one number --- their current position in the
sequence:

``` text
Producer cursor:   how far the producer has written
Consumer cursor:   how far each consumer has read (one per consumer)
```

The producer asks one question before writing: *"Is any consumer more
than N slots behind me?"* If yes, wait. If no, write and advance.

The consumer asks one question before reading: *"Has the producer
written past my position?"* If yes, read and advance. If no, wait.

That is the entire protocol. No mutex. No atomic CAS loop. Just reading
a `volatile long`.

------------------------------------------------------------------------

## A Visual Walk-Through

Buffer size = 8. One producer, two consumers.

**State 1: Producer has written slots 0--4. Both consumers have read
slots 0--3.**

``` text
Slots:    [ 0 ][ 1 ][ 2 ][ 3 ][ 4 ][ 5 ][ 6 ][ 7 ]
                                ↑
                          Producer cursor = 4

Consumer A cursor = 3 ──────────┤
Consumer B cursor = 3 ──────────┘

Consumer A: "Producer is at 4, I am at 3 → slot 4 is available → read it"
Consumer B: "Producer is at 4, I am at 3 → slot 4 is available → read it"
```

Both consumers read slot 4 independently, simultaneously, with no
coordination between them.

**State 2: Producer wants to write slot 12 (= buffer position 4,
wrapping around).**

``` text
Can the producer advance to sequence 12?
  wrapPoint = 12 - 8 = 4
  min(consumer cursors) = 3
  4 > 3 → NO. Producer must wait — slot 4 is still being read.
```

The buffer is full. The producer spins (or yields, or parks) until at
least one consumer advances past slot 4.

This is the **backpressure mechanism** --- built in, automatic, zero
additional code.

------------------------------------------------------------------------

## What Makes It Fast --- The Physical Reason

Modern CPUs are fast. Main memory is slow. The gap is bridged by CPU
caches (L1/L2/L3). Cache performance determines real-world throughput
more than algorithm complexity.

Three things the Disruptor does that align with how CPU caches work:

**1. Sequential memory access** --- the ring buffer is a contiguous
array. Reading slot N, then N+1, then N+2 is a sequential scan. The
hardware prefetcher predicts this and loads data before you ask for it.
A linked list jumps randomly through memory --- the prefetcher cannot
help.

**2. Cache line padding** --- each sequence number lives on its own
64-byte cache line. When the producer updates its cursor, only the
producer's cache line is invalidated. Consumers see no disruption.
Without padding, one write invalidates the cache line for everyone ---
this is called false sharing.

**3. No lock acquisition** --- taking a lock in the JVM requires a
memory fence (a full CPU barrier that flushes the store buffer and
drains the load buffer). The Disruptor uses the cheapest barrier that is
still correct: a store-release on write, an acquire-load on read.

------------------------------------------------------------------------

## The Mental Model in Five Lines

``` text
1. Fixed circular buffer — pre-allocated, no GC, cache-friendly.
2. One sequence number per participant — the only shared state.
3. Producer checks: "Am I about to lap the slowest reader?" If yes, wait.
4. Consumer checks: "Has the producer advanced past me?" If yes, read.
5. Each sequence lives on its own cache line — writes never interfere.
```

------------------------------------------------------------------------

## What It Is Not

  -----------------------------------------------------------------------
  Misconception                       Reality
  ----------------------------------- -----------------------------------
  "A lock-free data structure"        It is a pattern --- a combination
                                      of ring buffer + sequence
                                      tracking + padding + wait strategy

  "Always faster than a queue"        Only faster under sustained high
                                      load with pre-allocated events. For
                                      sporadic, low-volume messages, a
                                      `BlockingQueue` is simpler and
                                      sufficient

  "Eliminates all coordination"       It eliminates locks. Coordination
                                      still happens --- via volatile
                                      reads of sequence numbers

  "Only for financial systems"        Used in: Netty (network I/O), Log4j
                                      2 (async logging), Apache Storm,
                                      Hazelcast, telemetry pipelines
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## The One-Line Summary

> The Disruptor replaces a lock-protected linked list with a
> pre-allocated circular array where threads coordinate by watching a
> single number --- structured so that number never shares a CPU cache
> line with anything else.
