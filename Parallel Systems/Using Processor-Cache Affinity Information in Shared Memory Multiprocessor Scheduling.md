## 1) Problem the paper is solving: “cache warmth” is a scheduling resource

On a shared-memory multiprocessor (SMP), moving a runnable thread/process to a different CPU can be expensive because:

* It **throws away the benefit of the private cache state** it built up on its previous CPU (especially instruction cache + data cache working set).
* Even *rescheduling it on the same CPU* can be bad if **many other threads ran there in between**, because those intervening threads “pollute” (evict) its cache footprint. 

So the scheduling problem isn’t just “which runnable entity next?” but:

> **Which runnable entity on which CPU will minimize expected cache-miss penalty while still keeping CPUs busy and not creating severe load imbalance?**

That’s why the paper talks about using **processor-cache affinity information** as an explicit input to scheduling. 

---

## 2) Key modeling idea: quantify affinity using “intervening” work

### 2.1 What is “affinity” operationally?

Affinity means: *how likely is it that thread T will find useful cache contents if it runs on CPU P right now?*

The paper/lecture’s operational proxy is:

* **Intervening count** = number (or amount) of other work items that executed on P since T last executed on P.
* Low intervening count ⇒ T’s cache footprint is more likely still resident (“warm”).
* High intervening count ⇒ “warmth” decays ⇒ affinity benefit shrinks. 

This is a practical choice because OS can track it cheaply:

* For each CPU P: maintain a logical “execution stream” counter.
* For each thread T: record last CPU it ran on and the stream position at that time.
* Then for candidate (T,P): intervening ≈ stream(P) − last_stream_seen(T,P).

### 2.2 Why intervening threads, not time?

Time alone is a weaker predictor because cache eviction is mainly driven by *how much* and *what kind* of other work ran, not wall clock. Intervening work is a closer causal correlate of eviction.

---

## 3) The baseline policies 

### 3.1 FCFS (First Come First Served)

* **Mechanism:** run the oldest runnable item (global fairness).
* **Cache affinity:** basically ignored.
* **Load balancing:** typically good (keeps CPUs busy by taking whatever is next). 

### 3.2 FP (Fixed Processor)

* **Mechanism:** each thread has a “home” CPU; schedule it only there (or strongly prefer it).
* **Cache affinity:** excellent (maximizes same-CPU reuse).
* **Load balancing:** can be terrible (if some CPUs’ home threads block, others overload). 

### 3.3 LP (Last Processor)

* **Mechanism:** prefer to run a thread on the CPU it **last ran on**.
* **Cache affinity:** decent (but can still be polluted by interveners).
* **Load balancing:** better than FP because you can fall back when last CPU is busy/overloaded.

**practice exam info**

* At **low load**, LP behaves like FCFS because there aren’t many choices; the last CPU often isn’t available / there isn’t enough queue depth to find “good” last-CPU matches. 
* At **high load**, LP behaves more like FP because there *are* enough runnable threads that a CPU can usually find something with good affinity (someone ran here recently), so it tends to stick to “its” working set patterns. 

---

## 4) MI: Minimum Intervening — the core affinity heuristic

### 4.1 MI policy definition

**Minimum Intervening (MI):** when choosing where to run thread T, pick the processor P that minimizes the intervening work since T last ran on P. 

So MI is more refined than LP:

* LP only considers the *single* last processor.
* MI considers *all processors* and asks “where would T be warmest right now?”

### 4.2 Two axes: thread-centric vs processor-centric

* **Thread-centric routing:** when a thread becomes runnable, decide which CPU queue it should go to (based on affinity info).
* **Processor-centric selection:** when a CPU needs work, decide which thread to run next (based on affinity/age/etc).

MI is naturally *thread-centric*: place T into the “best” CPU’s queue based on affinity.

### 4.3 Why MI can be expensive

Naively, MI wants to compute affinity(T,P) for all P whenever T is enqueued. That’s O(#CPUs) per enqueue, potentially heavy at high scheduling rates.

So the paper introduces limited/approximate variants to reduce overhead while keeping most of the affinity win.

---

## 5) LMIR: Limited Minimum Intervening Router

Your lecture names **LMIR (Limited Minimum Intervening Router)** as the practically implementable MI variant. 

### 5.1 What LMIR is doing (conceptually)

When a thread T becomes runnable, LMIR decides which processor queue to place it into by balancing:

1. **Affinity benefit:** prefer CPUs where T has low intervening count (warm cache).
2. **Cache pollution cost:** avoid placing T onto a CPU where it will *harm* other threads’ cache locality too much (because T itself will pollute the cache of whatever runs later there).
3. **Practicality:** don’t scan everything; make the decision using limited queue visibility (hence “Limited”). 

The lecture’s “two-queue” picture is exactly this routing problem: you have processor queues Pu and Pv, an incoming thread Ty, and affinity scores to each queue’s processor, and you choose where Ty goes. 

### 5.2 How “limited” typically manifests

Implementation reality: you cannot afford global optimal matching.

So “limited” commonly means:

* Consider only a subset of processors (e.g., a neighborhood, or lightly loaded ones).
* Or consider only the first K runnable items (bounded lookahead).
* Or maintain compact per-thread state (last CPU + recency) rather than full matrix of (T,P).

From lecture: “consider tasks in the queue” and emphasizes policy-specific queue occupancy. 

---

## 6) Queue-based implementation model used in lecture/paper

The lecture gives the canonical implementation skeleton (and it matches how these policies are built). 

### 6.1 Structure

* Either a **global ready queue** or **per-CPU local ready queues**.
* Affinity-aware schedulers often use **affinity-based local queues** plus a routing step when threads wake up.

### 6.2 Priority function (important)

Lecture summary: each runnable thread Ti gets a dynamic priority like:

> **priority(Ti) = base_priority(Ti) + age(Ti) + affinity(Ti) − position_cost(Ti)**

The slide literally shows:
“Ti’s priority = BPi + agei + affinityi Position in the queue” (the last term is a positional/queue ordering effect). 

Interpretation:

* **base_priority:** user/system priority (nice level, class, etc.).
* **age:** increases with waiting time to avoid starvation (fairness).
* **affinity:** boost if we expect warm cache on this CPU.
* **position/occupancy:** prevents gaming, accounts for queue structure.

This priority model is how you reconcile “affinity vs fairness” on an exam.

---

## 7) “Procrastination”: intentionally idling to wait for a high-affinity thread

This is a classic (and counterintuitive) trick highlighted in the lecture. 

### 7.1 Why idling can help

If CPU P just ran thread A and A blocks briefly (e.g., short I/O wait), then:

* If P immediately runs some unrelated thread B,

  * B will evict A’s cache footprint.
  * When A wakes, A will pay cold/warmup misses.

* If P “procrastinates” (spins/idle-loops briefly),

  * A may wake quickly and resume on P with warm cache.
  * Total system throughput/latency can improve even though P was “idle” for a short time.

### 7.2 When it makes sense

Procrastination is only rational when:

* The expected wakeup time of a high-affinity thread is small.
* The expected cache-reload penalty avoided is large.
* There isn’t another equally-high-affinity runnable choice.

This is exactly the kind of tradeoff the paper is about: sometimes *utilization* is less important than *effective cycles* (cycles not wasted on cache misses).

---

## 8) Workload dependence: why no single policy dominates

* **FCFS**: fair, load-balanced, but high variance and can destroy cache locality.
* **MI variants (MI/LMIR)**: better at **light loads** because caches stay warm enough that affinity choices matter.
* **FP**: better at **heavy loads** because caches are heavily polluted anyway, and the overhead/complexity of affinity routing may not pay off; pinning/home-CPU behavior becomes more competitive.

---

## 9) Exam Prep

### 9.1 Compare FCFS vs FP on cache affinity and load balancing

Use the practice exam’s structure: 

* FCFS: ignores affinity, good fairness/load balance.
* FP: strong affinity, poor load balance risk.

### 9.2 “Why does LP ≈ FCFS at low load?”

Because the queue is short: you can’t usually find the “right” last-CPU thread; scheduling degenerates to “take what’s available.” 

### 9.3 “Why does LP ≈ FP at high load?”

Because with many runnable threads, each CPU can usually pick one that ran there recently, effectively creating a stable mapping like pinning. 

### 9.4 If asked “Why MI/LMIR helps?”

* MI/LMIR uses *intervening-work-based affinity* to reduce cache cold-starts.
* Reduced cache misses ⇒ improved throughput/response time, especially when caches are not constantly thrashed.
* LMIR keeps overhead bounded (doesn’t do global optimal search).

---

## 10) Mental model to keep straight

Think of scheduling as optimizing:

Effective CPU Time ~= Allocated cycles - (cache-miss stall cycles)

Affinity policies intentionally spend “scheduler effort” (and sometimes even brief idle time) to reduce stall cycles. That’s the core conceptual payload of the paper and the lecture.
