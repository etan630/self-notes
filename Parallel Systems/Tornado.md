## 1) Why Tornado exists: the problem is no longer “just concurrency”

Classic SMP kernels evolved when:

* caches were smaller / coherence cheaper,
* memory latency “felt” uniform,
* systems were smaller.

Modern shared-memory multiprocessors (esp. NUMA) flip the bottleneck:

* **cache misses** and **write-sharing invalidations** dominate,
* **false sharing** becomes catastrophic,
* **remote memory access** is much slower than local.

Tornado’s core thesis: **map locality/independence in application requests onto locality/independence inside the OS**—not just “add more fine-grained locks.” 

---

## 2) The big architectural idea: everything is an object

Tornado is “microkernel-ish” in that many services can be in servers, but the more important structural point is:

> **Every virtual and physical resource is an object**, and requests are method invocations on those objects. 

This is an enforcement mechanism for:

* **encapsulation of state**
* **encapsulation of locking**
* **replication/partitioning of hot objects** (next section)

So instead of “global tables protected by global locks,” Tornado tries to make the *default* path:

* touch only local objects,
* acquire only local locks,
* avoid shared writable cache lines.

---

## 3) Clustered objects: the main mechanism for locality + scalability

### 3.1 What a clustered object is

A **clustered object** is one logical object that can have **multiple representatives (“reps”)**, typically one per processor (or per NUMA node / cluster), each handling calls from “its” processor(s). 

This attacks the classic SMP pathology:

* one “Process” structure (or one lock) becomes a serialization point,
* coherence traffic explodes because everyone writes the same lock/cache line.

Instead, Tornado **splits** the object:

* each processor mostly talks to its **local rep**,
* local rep has a **local lock**,
* coherence traffic becomes rare in the common case.

### 3.2 The key design tradeoff: replicate for the common case, pay on updates

Replication makes:

* **read-mostly / frequent operations** fast and parallel (e.g., page faults),
  but makes:
* **operations that must change shared meaning** more expensive (e.g., deleting a region), because you must keep reps consistent. 

The paper explicitly uses the **Process** clustered object to illustrate:

* Page fault handling benefits hugely from multi-rep concurrency,
* Region deletion becomes more expensive due to rep consistency. 

Common Misconception: you might think “reader-writer lock on one shared Process object” would achieve similar concurrency. Tornado argues RW locks are *still* bad here because:

* RW locks introduce their own sharing/overheads,
* the lock hold time is short, so RW overhead dominates, and
* coherence traffic from updating RW lock metadata is still expensive. 

### 3.3 How the OS finds your local rep: per-processor translation tables

This is the “one extra indirection” that makes Tornado work.

Mechanism:

* Each processor has a **local translation table**.
* A “reference” to a clustered object is basically a pointer into that table.
* The table entry points to the rep to use for *this processor*. 

Crucial trick:

* each processor’s copy of the table is mapped at the **same virtual address**,
* so the same “clustered object reference” resolves to a *different rep* on different processors. 

Cost model:

* a clustered object call is just one extra load/indirection in the hot path. 

### 3.4 Demand creation via miss handling

You *don’t* want to eagerly create reps everywhere.

So Tornado initializes table entries to a **global miss handler object**.
On first use on a processor:

1. call goes to global miss handler,
2. miss handler invokes the object’s **object-specific miss handler**,
3. miss handler creates/installs the local rep pointer in the local table entry,
4. restart the original call. 

This is exam-relevant because it’s the “OS-level equivalent” of lazy population, tuned for locality.

### 3.5 Migration: keep reps with the threads

If threads migrate, the corresponding reps migrate “with them” to preserve locality (the paper explicitly calls this out for Process reps). 

---

## 4) Memory management example: Process + Region objects and why exam questions look like they do

The practice exam’s Tornado questions are directly probing whether you understand **(a)** what’s replicated/partitioned and **(b)** how existence/concurrency is handled.

### 4.1 Address space is represented by objects

* Process object references Regions.
* Regions represent partitions of the address space (and often align with locality goals). 

When two threads fault on pages in the same Region:

* The question becomes: do they contend on shared kernel structures, or do their local reps allow concurrency?

### 4.2 “Will access be serialized?” (practice exam style)

In the practice exam answer key, simultaneous page faults in the same Region1 are treated as **read-only access** to Process object data, so not necessarily serialized by exclusive locking. 

But the deeper Tornado-specific reasoning is:

* If the relevant state is replicated per rep (e.g., a Region list copy), faults can proceed independently on each processor’s rep, each with its own lock.
* If some part is truly shared and requires update, that path must pay rep-consistency cost.

### 4.3 “Sudden spike: what should VMM do?”

The practice exam says: **split Region1 into smaller Regions** to distribute load across more objects and eliminate hot spots. 

That is Tornado’s design philosophy: *change object partitioning granularity* to match contention/locality patterns.

### 4.4 False sharing + malloc question (practice exam)

>two malloc(16) calls on distinct cores → false sharing?

Answer: **False**, because Tornado partitions heap / allocations so different nodes/cores allocate from different partitions, avoiding landing allocator metadata or returned blocks on the same cache line. 

To generalize:

* Tornado takes allocator and object placement seriously:

  * either per-processor/per-node arenas,
  * alignment/padding to cache line size where needed,
  * keep frequently-updated metadata local.

---

## 5) Synchronization: “locking” vs “existence guarantees”

Tornado separates two issues: 

### 5.1 Ordinary locking (mutual exclusion / concurrency control)

* Encapsulate locks *inside* objects (and inside reps).
* Clustered objects reduce lock contention by splitting into reps.
* They use efficient “spin-then-block” locks optimized for the uncontended case (very small instruction footprint). 

Tornado’s lock strategy is not just “fine-grained locks,” it’s “fine-grained locks + structure that ensures most locks are local and uncontended.”

### 5.2 Existence guarantees (harder, and very exam-likely)

The classic race:

* Thread A is about to use an object reference.
* Thread B deallocates the object concurrently.
  Traditional fix:
* hierarchical locks (“lock the pointer, then the object…”), leading to:

  * global lock ordering complexity,
  * long lock hold times,
  * deadlock risk,
  * terrible locality. 

Tornado’s approach:

* **semi-automatic garbage collection for clustered objects** so a clustered-object reference is safe to use even while deletion is in progress. 

How deletion safety is achieved (high-level):

* Track “active references / temporary references” per processor,
* Use a mechanism (the paper describes a **circulating token**) to ensure all processors that might hold temporary refs have quiesced (their local active count reaches zero) before final deallocation. 

This is exactly what the practice exam is getting at when it says:

* process object can’t be destroyed mid-page-fault because “reference count / existence guarantee.” 

say:

* Tornado avoids lock hierarchies by making object lifetime safe via GC-like protocols, so you don’t need to hold a Process lock for the full duration of a fault just to keep Regions alive. 

---

## 6) IPC done “the Tornado way”: Protected Procedure Call (PPC)

In SMP microkernels, IPC can destroy locality if:

* requests bounce to a single server thread,
* server data structures are shared hot spots,
* scheduling causes migrations and cache coldness.

Tornado uses **Protected Procedure Call (PPC)**:

* looks like a call-return across protection domains,
* but **preserves locality**: the client’s request is serviced on the client’s local processor,
* **handoff scheduling** style sharing of the CPU between client/server,
* and **server concurrency scales with the number of clients** (effectively “as many threads of control in the server as client requests”). 

This is huge:

* If each request is handled by a local server execution context, server-side per-client state stays local, avoiding coherence chatter. 

Implementation tie-in with clustered objects:

* Cross-domain calls use stubs generated from clustered object interfaces.
* PPC checks that object refs are valid (in translation table, aligned, security bits). 

The paper also contrasts with older LRPC-era hardware ideas:

* earlier systems sometimes migrated the caller to an idle server CPU to reduce overhead,
* Tornado argues modern caches/coherence make that prohibitively expensive (cache miss / invalidation costs). 

---

## 7) Exam Prep

### A) Is the resource an object? Is it clustered?

If clustered:

* **local rep** likely exists (or will be created on demand),
* local locking / local state is the common case.

### B) Is the operation read-mostly or update-heavy?

* Read-mostly → replication pays off → high concurrency.
* Update-heavy / global semantic change → must coordinate reps → slower path.

### C) false sharing or coherence noise?

Look for:

* shared cache lines,
* allocator metadata,
* shared counters/locks.
  Then answer with:
* partitioning,
* padding/alignment,
* per-rep/per-core structures.

### D) “can it be destroyed mid-operation?”

That’s **existence guarantee**:

* reference counting / GC protocol (token / quiescence) prevents freeing while active refs exist.

### E) cross-domain calls?

Answer with PPC properties:

* serviced locally,
* handoff scheduling,
* server concurrency scales with clients,
* avoids shared server bottlenecks. 
