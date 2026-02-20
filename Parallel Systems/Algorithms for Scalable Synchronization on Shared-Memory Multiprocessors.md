## 1) The core thesis: scalable busy-waiting = **local spinning** + **single remote write to wake you**

The paper’s central claim is that busy-wait synchronization *doesn’t inherently* have to create “hot spots” (contention on one shared word that everyone repeatedly reads/writes). 

> **Rule:** *Every waiter spins on a distinct flag that is locally accessible; some other processor ends the wait with one remote write to that flag.*

That “locally accessible” part matters:

* On **cache-coherent (CC)** machines: “local” often means *spinning on a cache line in your cache* (no repeated bus/network traffic).
* On **non-cache-coherent distributed shared memory (NCC NUMA)** machines: “local” means *spinning on memory physically located near your processor* (no interconnect traffic for the spin loop). 

From lecture: **contention is mostly coherence/interconnect traffic**, not the arithmetic of “waiting.” 

---

## 2) Spin locks: from “hot spot” locks to queue locks

### 2.1 Why the classic locks are noisy

**Test-and-Set (TAS)** lock:

* Everyone repeatedly does an atomic RMW on *one word* ⇒ every attempt causes coherence traffic / invalidations.

**Ticket lock**:

* Everyone does 1 RMW to get a ticket, then spins reading a *single* “now_serving” word.
* Better than TAS (fewer RMWs), but still a **hot read location**: when “now_serving” changes, many caches invalidate/update. That’s why exam answers often say ticket locks are “noisier” than queue locks. 

So the *scalability goal* is: **avoid many processors spinning on one location**.

---

## 3) The MCS queue lock (the paper’s big lock result)

### 3.1 Data structures

Each lock has **one shared pointer**:

* `tail`: points to the **last** node in the queue, or `nil` if free.

Each thread/processor has a **per-thread qnode** (typically on its stack or TLS, but must remain valid until unlock):

* `next`: pointer to successor node
* `locked`: boolean flag this thread spins on

This is exactly the paper’s “list-based queuing lock” design: FIFO, local spinning, constant space per lock. 

### 3.2 Acquire: “splice yourself at the tail; if someone was there, wait on *your* flag”

Mechanically:

1. Initialize your node:

   * `my.next = nil`
   * `my.locked = true` (i.e., assume you must wait)

2. Atomically append yourself:

   * `pred = swap(&L.tail, &my)`  (fetch-and-store / swap returns old tail)

3. If `pred == nil`:

   * Lock was free; you own it immediately (set `my.locked = false` or just proceed)

4. Else:

   * Tell predecessor you are next: `pred->next = &my`
   * Now **spin locally**: `while (my.locked) { }`
   * When predecessor releases, it will set `my.locked = false` (one remote write).

**Key invariant:** each waiter spins on **its own** `my.locked`, so the spin loop does not hammer a shared location. 

### 3.3 Release: “if no successor yet, try to reset tail; otherwise wake successor”

Release has a subtle race: your successor might be *in the middle of enqueuing* (it swapped into tail, but hasn’t yet linked itself via `pred->next`).

Release logic:

1. If `my.next == nil`:

   * No known successor *yet*.
   * Try `CAS(&L.tail, &my, nil)`:

     * If succeeds: queue was truly empty; lock becomes free.
     * If fails: someone is joining but hasn’t set `my.next` yet.

       * Spin (locally) until `my.next != nil`.

2. Once you have successor:

   * `my.next->locked = false` (one remote write to wake exactly one waiter)

The paper notes that if you **don’t** have compare-and-swap (CAS), you can still implement it, but you lose a clean “I am last” proof and introduce a small timing-window inefficiency; FIFO becomes “almost FIFO” and starvation is theoretically possible.  

### 3.4 Why MCS is scalable 

On *each lock acquisition* under contention, how much “shared fabric” traffic do you generate?

* 1 atomic `swap` on `L.tail`  (remote)
* 1 write to `pred->next`      (remote, but single)
* spinning: **local** only
* unlock: either a CAS on tail, or a write to successor’s flag (bounded constant work)

So: **O(1) remote references per acquire** independent of #waiters. That is the main theoretical win. 

### 3.5 When MCS wins / loses 

* **Beats ticket lock under high contention**: ticket lock’s `now_serving` invalidations hit *all* waiters; MCS wakes exactly *one* successor by writing one flag. 
* **May lose at very low contention**: a simple TAS/Ticket may have shorter fast-path latency (fewer pointer ops). The practice exam even hints at this kind of reasoning when comparing hybrid locks vs pure MCS vs ticket. 

---

## 4) Barriers: what “scalable” means for barriers

A barrier has two costs:

1. **Arrival/collect:** detect that everyone arrived
2. **Wakeup/release:** let everyone proceed

Bad barriers create hot spots (everyone pounding one counter or one sense flag).

The paper’s barrier strategy mirrors MCS lock strategy:

* **Use a tree** so communication is distributed.
* **Spin only on local flags**, and propagate progress with bounded remote writes. 

---

## 5) The paper’s scalable distributed tree barrier (Algorithm 7)

The paper gives a concrete “only local spinning” tree barrier (pseudocode shown in Algorithm 7). 

### 5.1 Per-processor node layout: *allocate barrier state “near” the processor*

They define `nodes[vpid]` as a per-processor `treenode`, and explicitly state it is allocated in shared memory **locally accessible to processor vpid**. 

Each node has (conceptually):

* `havechild[k]`: whether this processor actually has child k in the arrival tree
* `childnotready[k]`: flags that start as `havechild[k]` and are cleared by children when they arrive
* `parentpointer`: pointer to the parent’s `childnotready[...]` slot that *this node* clears when it arrives
* `parentsense`: a flag the parent toggles to wake this node (wake-up phase)
* `childpointers[...]`: pointers to children’s `parentsense` flags (to wake children)

Also each processor keeps a private `sense` (like sense-reversing barriers).

### 5.2 Arrival phase: bottom-up aggregation without atomic RMW

Core idea:

* Each parent waits until all of its children have reported arrival by clearing *their designated* `childnotready` slot.
* Those `childnotready` bits are in the parent’s node (hence “local” to parent).

Mechanics (from Algorithm 7):

1. `repeat until childnotready == {false,false,false,false}`
   (spin locally on your own node) 
2. Reset `childnotready = havechild` for the *next* barrier instance.
3. Notify parent: `parentpointer^ = false` (write to parent’s node to say “I arrived”).

**Why no RMW needed:** Each child writes a different `childnotready` slot in the parent’s node. No two children contend on the same word, so atomic decrement isn’t required.

### 5.3 Wakeup phase: top-down release using sense reversal

Non-root nodes wait until their parent toggles their `parentsense` to match their local `sense`:

* `repeat until parentsense == sense` (local spin)

Then they wake their own children by writing their children’s `parentsense` flags (remote writes, but only a constant number per node in a fixed-arity tree):

* `childpointers[0]^ = sense`
* `childpointers[1]^ = sense`
* `sense = !sense`

This is the same “local spinning + single remote write to wake” principle. 

### 5.4 Complexity

From the paper:

* **Space:** O(P) total (one node per processor), but **per barrier object** you can treat it as O(P) because barrier is a global object. (This differs from lock space, where they emphasize constant space *per lock*.)
* **Traffic:** O(1) remote writes per processor (constant children + parent notification).
* **Critical path:** O(log P) tree height for arrival + O(log P) for wakeup. 

---

## 6) Connecting to lecture barrier taxonomy: tree vs MCS-tree vs tournament vs dissemination

### 6.1 Why “static spin location” matters (NUMA reasoning)

A key critique in lecture: naive tree barriers can have **dynamic spinning locations**—who spins on which variable can depend on arrival timing, which can be disastrous on NCC NUMA if you end up spinning on remote memory. 

MCS-style barriers fix this by:

* pre-assigning which variables you spin on (your own node / your own flags),
* ensuring those variables are local.

That’s exactly why the MCS tree is “constructed this way” in lecture. 

### 6.2 Tournament barrier (lecture) vs tree barrier (paper)

Lecture summary of tournament barrier:

* “Rigged” bracket ⇒ deterministic who waits on whom, hence **static** spin variables.
* No RMW operations required.
* Communication complexity O(N) total but with parallel paths.
* Works even for clusters / NCC environments.
* Doesn’t exploit cache spatial locality as well as MCS-tree. 

“Which is better on NCC NUMA: dissemination vs tree barrier?”
* dissemination is O(P log P) messages but highly parallel and works without coherent caches,
* tree barrier is O(P) total traffic but longer critical path; might be better when network cannot handle all-to-all message patterns as well.  

### 6.3 Dissemination barrier and “knowledge” questions

dissemination-barrier reasoning question (“what does P6 know after each round?”).
* articulate the doubling knowledge property: after round k, each node knows 2^(k+1) arrivals (in a power-of-2-sized system), hence completion after ceil(log2 P) rounds.  

---

## 7) Common correctness pitfalls

### 7.1 Sense-reversing barrier races

The practice exam calls out two  bugs:

1. `count--` must be atomic (else multiple threads “think” they’re last).
2. The last thread must reset `count` **before** publishing the new `sense` (else next-iteration arrivals race). 

MCS-tree avoids these because:

* arrival uses **distinct per-child flags** instead of a shared counter decrement,
* wakeup uses per-node `parentsense` (not a single global sense variable).

### 7.2 MCS lock race windows

For MCS lock, you must be ready to explain the *successor linking window*:

* successor swaps into tail first,
* then later sets predecessor’s `next`.
  So unlock must handle “tail says someone exists, but my.next is still nil.” That’s why unlock uses CAS and/or a short spin on `my.next`. 

---

## 8) Exam Prep

### Q: “Why is ticket lock noisier than MCS under contention?”

* Ticket: all waiters poll `now_serving` ⇒ every unlock updates one shared word ⇒ coherence invalidations/updates to many caches.
* MCS: each waiter spins on private `locked` flag ⇒ unlock writes only successor’s flag ⇒ O(1) invalidations and no hot spot.  

### Q: “Why does MCS lock work well on NCC NUMA too?”

* Because “local spinning” doesn’t require caches; it requires that your spin variable be in memory local to you.
* Only bounded remote ops: swap tail + one write to wake successor. 

### Q: “Does MCS barrier need atomic RMW?”

* No, in the tree barrier each child clears a distinct parent flag; no shared counter is decremented concurrently.  

### Q: “Pick a barrier for CC SMP vs NCC NUMA”

A canonical mapping (consistent with the paper’s concluding guidance and lecture framing):

* CC SMP (broadcast-y, coherent caches): centralized/sense-reversing variants can be okay for modest P; tree-based for larger P.
* NCC NUMA: dissemination or tree-based with distributed flags; avoid centralized spinning on one location.  