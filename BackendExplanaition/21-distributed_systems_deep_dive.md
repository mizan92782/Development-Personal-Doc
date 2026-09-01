# Distributed Systems Deep Dive

## Why This Topic Exists

Everything up to this point (Levels 1–20) mostly assumed one server, or a few servers behaving
predictably. Distributed systems is what happens once you're honest about reality: **networks
fail, machines crash, messages get delayed or lost, and clocks on different machines don't
perfectly agree** — and your system still has to work. Almost every hard problem in this topic is
really the same question asked in different clothes: *when part of the system can't talk to
another part, what should happen?*

```
Single server:                            Distributed system:
┌─────────────┐                          ┌──────────┐        ┌──────────┐
│   Everything      │                          │  Node A     │◀──?──▶│  Node B     │
│   in one place      │                          └──────────┘        └──────────┘
│   (if it fails,      │                          What if the network between them drops?
│   everything fails,    │                          What if A thinks B is dead, but B is fine?
│   but there's no          │                          What if a message from A arrives at B
│   DISAGREEMENT to           │                          twice, or out of order, or never?
│   worry about)                │                          ← THIS is what distributed systems is about
└─────────────┘
```

---

## 1. Distributed Systems — The Core Definition

A distributed system is a collection of independent machines (nodes) that must coordinate to
appear, from the outside, as one coherent system — communicating only via network messages, which
are inherently unreliable (can be delayed, lost, duplicated, or arrive out of order).

```
┌──────────┐         messages over a network         ┌──────────┐
│  Node A     │ ◀──────────────────────────────────▶ │  Node B     │
└──────────┘   (can be delayed, dropped, duplicated,  └──────────┘
                or reordered — this unpredictability
                is the fundamental challenge)
```

**Three assumptions you can no longer safely make**, compared to a single machine:
1. A message you sent was received.
2. A message that was received was received in the order you sent it.
3. If a node stops responding, it has actually crashed (it might just be slow, or the network
   between you and it is the thing that failed — you often can't tell which from the outside).

---

## 2. Network Partitions

A partition is when a network failure splits the system into groups of nodes that **can still
talk to each other within their group, but not across groups** — from each side's perspective,
the other side has simply vanished.

```
Before partition:
┌────────┐     ┌────────┐     ┌────────┐
│  Node A   │◀───▶│  Node B   │◀───▶│  Node C   │
└────────┘     └────────┘     └────────┘

During partition (network link between {A,B} and {C} fails):
┌────────┐     ┌────────┐          ┌────────┐
│  Node A   │◀───▶│  Node B   │    ✗     │  Node C   │
└────────┘     └────────┘          └────────┘

From A and B's perspective: "C is unreachable — is it dead, or just cut off?"
From C's perspective:        "A and B are unreachable — same question, in reverse"
Nobody can be SURE which — this ambiguity is the entire crux of the CAP theorem below.
```

**Critically:** partitions are not a rare edge case to design around defensively — in any
real-world distributed system spanning multiple machines/data centers/regions, partitions *will*
happen (a switch fails, a fiber line gets cut, a data center loses power). The question isn't
"how do I prevent this," it's "what does my system do when it happens."

---

## 3. CAP Theorem

States that a distributed system can only guarantee **two out of three** of: Consistency,
Availability, and Partition tolerance — at the same time, during an actual partition.

```
                        Consistency
                         (C)
                          ╱╲
                         ╱  ╲
                        ╱    ╲
                       ╱      ╲
                      ╱  Pick   ╲
                     ╱   only     ╲
                    ╱   TWO          ╲
                   ╱                    ╲
      Availability ────────────────────── Partition
          (A)                              Tolerance (P)
```

**The key insight that makes this concrete, not abstract:** partition tolerance isn't really a
choice — in a real distributed system (multiple physical nodes, real networks), partitions *will*
happen eventually, so P is effectively mandatory. That means the actual choice CAP forces on you
is: **when a partition happens, do you sacrifice Consistency or Availability?**

```
Partition happens. A client asks Node A (cut off from the rest) to read/write data.

Choose CONSISTENCY (CP):                    Choose AVAILABILITY (AP):
Node A says "I can't guarantee my data       Node A says "here's my best current
is up to date with the other side of          answer, even though I can't confirm
the partition, so I'll REFUSE to respond      it matches what the other side has
(or only allow reads, not writes)             right now"
until the partition heals."
        │                                            │
        ▼                                            ▼
  System is CORRECT but UNAVAILABLE            System is AVAILABLE but might
  for some requests during the partition        return STALE or CONFLICTING data
```

| System type | Choice | Example |
|---|---|---|
| Traditional relational DB with synchronous replication | CP | Won't confirm a write until it's safely replicated; blocks during a partition |
| DNS | AP | Always answers with whatever it has, even if slightly stale |
| Most NoSQL stores (Cassandra, DynamoDB) | Configurable, often AP by default | Can be tuned toward C or A per operation |

**Why this matters practically:** CAP isn't a theorem you "apply" once — it's a lens for
understanding a design decision every distributed component actually makes, usually implicitly.
Knowing which side of C-vs-A a piece of your infrastructure falls on tells you exactly what kind
of bug to expect from it during a network blip.

---

## 4. Consistency (in the CAP sense)

Every node returns the **same, most-recent** data for the same query — as if there were really
only one copy of the data, no matter which node you ask.

```
Write X=5 to Node A
          │
Immediately read X from Node B
          │
          ▼
CONSISTENT system: Node B returns 5 (guaranteed, even though the write went to a different node)
```

## 5. Availability (in the CAP sense)

Every request to a non-failing node receives a response — not an error, not a hang — even if that
response can't be guaranteed to reflect the absolute latest write.

```
Node A is cut off by a partition, but still running
          │
Client sends a read request to Node A
          │
          ▼
AVAILABLE system: Node A responds with SOMETHING (possibly stale), rather than refusing to answer
```

## 6. Partition Tolerance

The system continues operating (in SOME form — per the C vs A choice above) despite network
partitions between nodes, rather than the whole system grinding to a complete halt.

---

## 7. Eventual Consistency

A weaker consistency model: after writes stop coming in, **given enough time**, all replicas will
converge to the same value — but at any given moment, different nodes might briefly disagree.

```
t=0    Write X=5 to Node A
t=0    Node B still has X=3 (hasn't received the update yet — replication is async)
t=0    Node C still has X=3
                │
                ▼  (replication propagates over the next few hundred ms)
t=0.3s Node B now has X=5
t=0.5s Node C now has X=5
                │
                ▼
t=1s   All nodes agree: X=5   ← "eventually" consistent
```

**Why accept this trade-off at all?** It usually buys much higher availability and lower latency
— nodes don't have to synchronously coordinate on every single write, they can just accept it
locally and propagate it in the background. Fine for data where a few hundred milliseconds of
staleness is invisible to users (like counters, feeds, view counts — this is the same idea from
Level 17's consistency section).

## 8. Strong Consistency

Every read reflects the most recent completed write, **guaranteed**, with no window of staleness
— usually achieved by requiring writes to be confirmed by a majority (or all) of replicas before
being considered "complete," and reads to check with enough replicas to be sure they're not
reading stale data.

```
Write X=5
    │
    ▼
Must be confirmed by a MAJORITY of replicas before the write is considered done
    │
    ▼
ANY subsequent read, from ANY node, is guaranteed to see X=5 (or a later value) — never X=3 again
```

**The cost:** every write (and often every read) now requires network round-trips to multiple
nodes before it can complete — inherently higher latency, and during a partition, a strongly
consistent system may have to refuse writes entirely if it can't reach enough replicas to
guarantee correctness (this is the "C over A" choice from CAP, made concrete).

---

## 9. Distributed Transactions

A transaction that must succeed or fail **atomically across multiple independent nodes/services**
— e.g., debiting an account on Database A and crediting one on Database B, where both must
succeed together or both must roll back, even though they're entirely separate systems.

```
Transfer $100 from Account A (on DB1) to Account B (on DB2)

┌───────────────────────────────────────────────────┐
│                  Coordinator                          │
└───────────┬───────────────────────┬───────────────┘
             │  Phase 1: PREPARE          │  Phase 1: PREPARE
             ▼                             ▼
      ┌───────────┐                 ┌───────────┐
      │    DB1        │                 │    DB2        │
      │ "can you        │                 │ "can you        │
      │  debit $100?"    │                 │  credit $100?"   │
      │  → YES, ready      │                 │  → YES, ready      │
      └───────────┘                 └───────────┘
             │                             │
             ▼  Phase 2: COMMIT            ▼  Phase 2: COMMIT
      (only if BOTH said "ready" in phase 1 — this is the "Two-Phase Commit" protocol)

If EITHER DB says "not ready" in Phase 1 (e.g., insufficient balance), the coordinator
tells BOTH to ROLL BACK instead — nothing partial ever happens.
```

**The catch:** this protocol (Two-Phase Commit) is itself vulnerable to the coordinator crashing
between phases — if it crashes after DB1 commits but before telling DB2, you're left in an
ambiguous, hard-to-resolve state. This is why distributed transactions are notoriously difficult,
and why many modern systems avoid them entirely in favor of patterns like the **Saga pattern**
(a sequence of local transactions, each with a defined compensating "undo" action if a later step
fails) which trades strict atomicity for resilience to partial failure.

---

## 10. Distributed Locks

Covered in depth in Level 10 (Redis) — a mechanism ensuring only one node/process performs a
given critical action at a time, across an entire distributed system, not just within one
process.

```
Worker A                     Lock service                 Worker B
   │  acquire("process_order_42") ──▶ granted                  │
   │                                                             │  acquire("process_order_42")
   │  ... processes order 42 ...                                │  ──▶ DENIED (already held)
   │  release("process_order_42")                                │
   │                                                             │  (Worker B backs off/retries)
```

**Why this is genuinely hard in a distributed setting:** you need the lock itself to survive node
crashes (via TTL/expiry, as in Level 10) without either (a) two nodes both believing they hold the
lock simultaneously (a correctness disaster) or (b) the lock never being released if its holder
crashes (a liveness disaster, everything stuck forever). Balancing these two failure modes safely
is the whole difficulty.

---

## 11. Leader Election

The process by which a group of nodes agree on **which single node** should act as the
coordinator/primary for some duty (accepting writes, scheduling jobs, etc.) — critical because
having zero leaders means nothing gets coordinated, and having two leaders simultaneously
("split-brain") can cause serious data corruption.

```
Before:  Node A (leader), Node B, Node C — all healthy, everyone agrees A is the leader

Node A crashes
        │
        ▼
Node B and Node C detect A is unresponsive (e.g., missed heartbeats)
        │
        ▼
An election happens: nodes vote / use a specific algorithm (e.g., Raft, see below)
to agree on a NEW leader
        │
        ▼
Node B is elected the new leader — B and C both agree; A (if it comes back later)
must recognize it's no longer the leader and step down
```

**The split-brain danger, made concrete:**

```
Node A is temporarily UNREACHABLE (network blip, not actually crashed) but still running
        │
        ▼
Node B and C, unable to reach A, elect Node B as the new leader
        │
        ▼
Network heals — Node A is back, but STILL THINKS IT'S THE LEADER (nobody told it otherwise)
        │
        ▼
Now BOTH A and B believe they're the leader — if both start accepting writes independently,
you can get conflicting, corrupted data. This is "split-brain," and preventing it is a
primary goal of consensus algorithms.
```

---

## 12. Consensus

The general problem of getting multiple distributed nodes to **agree on a single value or
decision**, even when some nodes might crash or messages might be lost/delayed — this is the
formal underpinning of leader election, distributed locks, and strongly-consistent replication
all at once. The two most well-known algorithms solving this are **Paxos** (older, notoriously
hard to understand/implement correctly) and **Raft** (designed specifically to be more
understandable, and now what most modern systems — etcd, Consul, CockroachDB — actually use).

### Raft, at a glance

```
Nodes can be in one of three states: Follower, Candidate, Leader

┌──────────┐  times out waiting   ┌──────────┐   wins majority    ┌──────────┐
│ Follower    │  for a leader's        │ Candidate   │   of votes            │ Leader      │
│              │  heartbeat ────────▶  │              │  ────────────────▶   │              │
└──────────┘                        └──────────┘                        └──────────┘
       ▲                                                                        │
       └────────────────────── steps down if a higher-term leader appears ──────┘
```

```
Normal operation:
   Leader sends periodic heartbeats to all Followers ("I'm still alive, still the leader")
   Followers reset their election timeout every time they receive one

Leader fails:
   Followers stop receiving heartbeats → election timeout fires → a Follower becomes
   a Candidate → requests votes from other nodes → if it gets a MAJORITY, it becomes
   the new Leader → resumes sending heartbeats
```

**Why "majority" specifically, not just "any votes"?** Requiring a strict majority guarantees
that even if the network partitions the cluster into two groups, **at most one** group can
possibly contain a majority — the other group, lacking a majority, cannot elect its own leader.
This is the actual mechanism that prevents split-brain: a minority partition simply can't make
progress (it sacrifices Availability to preserve Consistency — CAP theorem, made concrete again).

```
5-node cluster, partition splits it into a group of 3 and a group of 2:

Group of 3 (majority):                    Group of 2 (minority):
Can elect a leader (3 out of 5             CANNOT elect a leader (2 out of 5 is
is a majority) → continues operating         not a majority) → refuses to accept
normally, accepting writes                    writes, stays unavailable until the
                                              partition heals — CORRECTLY avoiding
                                              a second, conflicting leader
```

---

## 13. Replication

Covered in depth in Level 17 — keeping copies of data on multiple nodes, for both redundancy
(survive a node failure) and scalability (spread read load). Distributed systems theory is what
explains WHY replication has to make the consistency trade-offs it does (sync vs async
replication is literally the CAP/consistency trade-off applied to a database specifically).

```
Synchronous replication (favors Consistency):
   Write ──▶ Primary ──▶ waits for Replica to CONFIRM the write ──▶ THEN tells the client "done"
   (slower, but the replica is GUARANTEED to have the data if the primary dies right after)

Asynchronous replication (favors Availability/lower latency):
   Write ──▶ Primary ──▶ tells the client "done" IMMEDIATELY ──▶ replicates to Replica in the background
   (faster, but if the primary dies before replicating, that write can be LOST)
```

---

## Putting It All Together — The Example: What Happens When the Primary Crashes

```
             Database
             /      \
            /        \
       Replica 1    Replica 2
```

Let's make this concrete and walk through the full sequence of events.

### Step 0 — Normal operation

```
┌────────────┐
│   Primary       │  ◀── all WRITES go here
└──────┬─────┘
        │  replicates changes
   ┌────┴────┐
   ▼         ▼
┌───────┐ ┌───────┐
│Replica 1│ │Replica 2│  ◀── serve READS, stay in sync with the primary
└───────┘ └───────┘
```

### Step 1 — The Primary crashes

```
┌────────────┐
│   Primary   ✗   │  CRASHED — no longer responding
└────────────┘

┌───────┐ ┌───────┐
│Replica 1│ │Replica 2│  still alive, still have (mostly) up-to-date data
└───────┘ └───────┘
```

**Immediate consequences:**
- Any WRITE in flight to the primary at the moment of the crash is lost (if replication was
  asynchronous and hadn't caught up yet) — this is the async-replication trade-off from above,
  now actually happening.
- New write requests coming in have nowhere to go — **writes are unavailable** until a new
  primary is established. (Reads can often continue being served by the replicas.)

### Step 2 — Failure detection

```
Something (a monitoring/orchestration system, or the replicas themselves via heartbeats)
must first notice the primary is actually gone, not just momentarily slow.

┌───────┐         "I haven't heard from Primary       ┌───────┐
│Replica 1│  ◀────  in N heartbeat intervals..." ────▶ │Replica 2│
└───────┘                                              └───────┘
```

**This is genuinely tricky:** declare the primary dead too fast, and you might trigger a failover
for a primary that was just briefly slow (leading to a split-brain risk if the "dead" primary
comes back). Wait too long, and your system is unavailable for writes longer than necessary. Most
production systems use a timeout tuned to balance this.

### Step 3 — Leader election (choosing a new primary)

```
Replica 1 and Replica 2 (and/or an external coordinator like a consensus system) run
a leader election process (conceptually like the Raft example above):

┌───────┐                                    ┌───────┐
│Replica 1│  "I have the most up-to-date       │Replica 2│
│          │   data, I should be the new         │          │
│          │   primary" — or a formal vote           │          │
└───┬───┘   happens                            └───────┘
    │
    ▼
Replica 1 is promoted to be the NEW Primary
```

**A critical detail:** ideally, the replica with the **most up-to-date data** (least
replication lag) is chosen as the new leader — promoting a replica that was significantly behind
would mean silently losing more data than necessary.

### Step 4 — Reconfiguration

```
┌────────────┐
│  NEW Primary    │  (was Replica 1)
│  (was Replica 1) │
└──────┬─────┘
        │  replicates to
        ▼
┌────────────┐
│  Replica 2      │  (unchanged role, now follows the NEW primary)
└────────────┘

Application servers / connection routers must be updated to know
WHERE to send writes now — this is often handled by:
  - A proxy/router layer that tracks the current primary
  - DNS updates
  - The database driver itself querying "who is currently the primary?"
```

### Step 5 — The old Primary recovers

```
┌────────────┐
│  Old Primary     │  comes back online — but it must NOT resume acting
│  (recovering)      │  as primary, or you get split-brain (two primaries!)
└──────┬─────┘
        │
        ▼
It must recognize (via the same consensus/term mechanism from the Raft
example) that a NEWER primary already exists, and demote itself to
become a REPLICA of the new primary instead — catching up on any writes
it missed while it was down.
```

```
Final state:
┌────────────┐
│  New Primary    │  (was Replica 1) ◀── all writes now go here
└──────┬─────┘
   ┌────┴────┐
   ▼         ▼
┌───────┐ ┌───────┐
│Replica 2│ │Old       │  (demoted, now catching up / following)
│(unchanged)│ │Primary   │
└───────┘ └───────┘
```

### The Honest Summary of What Just Happened

```
1. Some in-flight writes may have been LOST (if replication was async and hadn't caught up)
   → this is the CAP/consistency trade-off, made real
2. Writes were UNAVAILABLE for a window of time (detection + election + reconfiguration)
   → this is the CAP/availability trade-off, made real
3. The system had to correctly avoid TWO primaries existing at once (split-brain)
   → this is exactly what consensus/leader-election protocols like Raft exist to guarantee
4. Reads may have continued from the surviving replicas the whole time (if your system
   allows serving reads from replicas even during a primary failover)
```

**The one-sentence takeaway:** every concept in this document — CAP, consensus, leader election,
replication strategy — exists to answer, precisely and in advance, the one question a primary
crash forces you to confront in real time: *how much data can we afford to lose, how long can we
afford to be unavailable, and how do we guarantee only one node ever thinks it's in charge?* A
system that hasn't explicitly decided these trade-offs ahead of time will discover its actual
(usually worse than intended) answer the hard way, during a real outage.
