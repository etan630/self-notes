## 1) What problem is Cellular Disco solving?

Large shared-memory multiprocessors (SMP/ccNUMA machines like SGI Origin 2000) have 3 recurring OS pain points:

1. **Scalability bottlenecks in “big” OS kernels** (global locks, centralized structures, non-NUMA-aware policies).
2. **NUMA effects**: local vs remote memory latency differences; naive placement hurts performance.
3. **Fault containment**: on very large machines, you don’t want *one* hardware/software fault to take down *everything*.

Historically, people tried:

* **Hardware partitioning**: split the machine into smaller partitions (“like a cluster”) → better fault containment, but **static**, poor global utilization, can strand resources.
* **New scalable OS designs** (Hive, etc.) → great properties but *massive engineering cost*.

**Cellular Disco’s core move**: put a **thin VMM layer** under *unmodified* commodity OS instances (like Disco), but extend it so the machine behaves like a **“virtual cluster”** with:

* **fault containment like a cluster**, AND
* **dynamic fine-grained sharing like shared memory** (CPU/memory can move across “cluster boundaries” when needed). 

---

## 2) What is the “virtual cluster” abstraction?

Cellular Disco runs **multiple VMs** on one ccNUMA shared-memory machine (like Disco), but internally it partitions the VMM itself into **cells**.

### Cells (fault-containment units)

A **cell** is a semi-independent slice of the VMM tied to a subset of hardware nodes (and their memory). The VMM is replicated across cells: *each cell has a full copy of the monitor text* and manages the machine pages of its nodes. 

**Fault containment goal**: if a hardware failure takes down a cell, only:

* that cell’s hardware, and
* the VMs that were using resources from that cell
  should be lost; everyone else keeps running. 

This is “cluster-like” fault behavior, but on a single shared-memory machine.

---

## 3) Hardware virtualization: how does a VM run “below” the OS?

Cellular Disco is a classic VMM design:

### Privilege-level trick

Only the monitor runs at the highest privilege. Guest OS kernels run at a lower privilege level, so when they try privileged instructions, they trap into the VMM which emulates them. 

### The key data structures: pmap and memmap

* **pmap**: maps *guest physical* pages → *machine* pages (real hardware pages). 
* **memmap**: reverse mapping machine → guest physical, used for page migration/replication and fault recovery. 

This extra indirection is the *enabler* for:

* NUMA-aware relocation (migrate/replicate “hot” pages),
* paging policies at the VMM layer,
* moving a VM’s “footprint” between cells, and
* fault recovery bookkeeping.

---

## 4) Trusted monitor assumption and why it matters

Cellular Disco explicitly assumes the monitor is **small enough to trust** (≈50K LOC), so the cell components can use shared memory efficiently without heavy distributed protocols.  

Why this matters:

* If you *don’t* trust the system layer (Hive-style), inter-cell updates require expensive distributed consensus-like protocols.
* Cellular Disco instead allows shared-memory updates for VM-specific structures, **but** restricts direct touching of other cells’ “critical survival” structures. 

So the philosophy is: **fast normal-case**, accept higher cost on failures (rare). 

---

## 5) Inter-cell communication: how do cells coordinate safely?

Two primitives:

### (A) Fast inter-processor RPC

A low-latency RPC for cross-cell/node requests; measured round-trip ~16 µs for cacheline-sized args/replies on their Origin prototype. 

### (B) “Message” primitive for per-VCPU serialization

For actions that must occur on the CPU that currently “owns” a VCPU. The big win: **avoid locking** by funneling per-VCPU operations through its current owner. 

This relies on:

* a **fault-tolerant distributed registry** to locate the current owner of a VCPU,
* rebuilt after failures,
* and it guarantees **exactly-once message semantics** even with contention, VCPU migration, and hardware faults. 

“how do they avoid global locks while allowing migration?”
* **owner-based serialization + registry + exactly-once messaging**.

---

## 6) Resource management “under constraints”: the hard part

Resource management is constrained by **fault containment**:

* Efficient utilization says: “use all resources in the system when needed.”
* Fault containment says: “keep each VM’s footprint confined to a small number of cells.”

If a VM starts using any resource from a cell, it becomes vulnerable to that cell’s failure. Therefore, load balancing and borrowing must be conservative and structured.  

That tension is *the* defining systems tradeoff of Cellular Disco.

---

## 7) CPU management in detail

Cellular Disco schedules **VCPUs** onto real CPUs. It must balance:

* cache affinity / NUMA locality (don’t bounce execution),
* fairness / utilization (don’t leave CPUs idle),
* fault containment (don’t spread VMs across too many cells),
* and **gang scheduling constraints** for multiprocessor VMs.

### 7.1 VCPU migration types and costs

They describe *three* migration “radii,” each with different costs:  

1. **Within the same node** (Origin node has 2 CPUs):

   * bookkeeping overhead ~37 µs
   * real cost is losing cache affinity (they estimate ~8 ms to “warm” a 4MB L2 scenario). 

2. **Across nodes within the same cell**:

   * must copy a software L2 TLB (“L2TLB”, ~32 KB) because it’s accessed frequently and kept with the VCPU’s node
   * cost ~520 µs
   * long-term cost: remote memory accesses (continuous penalty). 

3. **Across cells**:

   * cost ~1520 µs (includes L2TLB copy)
   * also increases fault vulnerability (now dependent on new cell).
   * Cellular Disco can later remove dependencies by moving VM data (paper points to Section 4.5). 

**Key takeaway**: migration cost isn’t just the one-time move; it’s *steady-state NUMA locality damage* unless you also move memory.

### 7.2 Gang scheduling constraint

Cellular Disco “gang schedules” multiprocessor VMs: it schedules a VM’s VCPU only when all its non-idle VCPUs are runnable. This prevents partial execution that can break performance expectations or fairness accounting. 

This becomes crucial for load balancing: you can’t steal *any* VCPU; you steal VCPUs that “unlock” the ability for the VM to run as a gang.

### 7.3 CPU balancing policies

Two balancers: 

* **Idle balancer** (primary workhorse): runs when a processor becomes idle, and tries to “steal” runnable VCPUs from nearby run queues (closest neighbor first) within the same cell. It selects VCPUs that satisfy gang constraints. 
* **Periodic balancer**: catches VCPUs the idle balancer doesn’t handle well.

The paper’s Figure 8 example: an idle CPU triggers a migration that enables a VM’s set of VCPUs to run together. 

### 7.4 Connection to lecture “cache affinity scheduling”

This maps directly to the lecture theme: scheduling policies trade off **affinity** vs **load balancing** . Cellular Disco is *explicitly* affinity-aware (migration tiers, “nearest neighbor” stealing), but can still shift load when needed.

---

## 8) Memory management in detail

Memory is where “virtual cluster” really shows: cells manage local machine memory but can **lend/borrow** and do NUMA-aware placement.

### 8.1 First-touch + hot page migration/replication

The VMM implements **first-touch allocation** and can dynamically **migrate or replicate** frequently accessed (“hot”) pages to reduce remote misses. 

This is the memory-side complement to CPU migration:

* if you migrate a VCPU and don’t migrate its working set, you pay remote memory penalties forever.
* so Cellular Disco uses page migration/replication to “follow” execution. 

### 8.2 Memory load balancing: borrowing across cells

Cells try to avoid global contention with local policies. One explicit heuristic: 

* If local free memory drops below **16MB**, the cell tries to ensure at least **4MB** free from each cell in its “allocation preferences list.”
* It borrows **4MB** from each such cell that doesn’t meet that availability.
* Other cells loan as long as they have more than **32MB** free. 

This is intentionally hysteretic/stable: tuned around how much memory could be allocated in ~10ms. 

The paper reports a database workload borrowing **596MB** across cells with **<1%** runtime increase. 

So the design is: **mostly local**, borrow in chunks, preserve containment by preferring cells already supplying memory to VMs.

---

## 9) Paging: avoiding classic VMM pitfalls (double paging)

If *all* cells are low on memory, you page to disk. But VMM paging introduces nasty interactions with guest OS paging.

Cellular Disco addresses three paging challenges: 

1. identify actively used pages,
2. handle pages shared by multiple VMs,
3. avoid redundant paging (“double paging”).

### 9.1 Replacement algorithm

They use a **second-chance FIFO** queue to approximate LRU (like VMS). Each VM has a resident set size that can be dynamically trimmed under pressure. 

### 9.2 Don’t page garbage: “non-intrusive” tracking of guest page use

A pure LRU approximation can’t distinguish:

* page with real data vs
* unallocated/free page that just contains junk.

Cellular Disco avoids writing unallocated garbage pages by **monitoring guest OS allocation/deallocation** via annotations on the OS memory allocator/free routines. 

### 9.3 Paging shared pages

A machine page may be shared by multiple VMs (shared memory regions or Copy-on-Write). The sharing metadata can’t stay in memory once the page is paged out and the machine page reused.

So Cellular Disco writes the **sharing info** to disk alongside the page data (in a contiguous following sector) so it can be written in the same I/O request (avoids extra seeks). 

### 9.4 Avoid redundant paging (classic problem)

The paper explicitly calls out “redundant paging” (historical VMM issue): VMM pages out a page, later guest OS pages it out again, causing extra disk I/O. 

They avoid this with a **virtual paging disk** technique (illustrated in the paper’s Figure 9): instead of triggering a real disk write in the guest, the guest’s “pageout” becomes a mapping update to the VMM-managed backing store, collapsing multiple disk operations into one. 

this answers: “how does Cellular Disco avoid double paging?”

---

## 10) Supporting large apps across VMs: shared memory registration

They also add support so large applications spanning VMs can communicate via **shared memory** by registering shared regions with the monitor—more efficient than distributed protocols, and transparent to the guest OS. 

This is part of the “shared-memory benefits preserved” story: you aren’t forced into a message-passing-only cluster model.

---

## 11) Fault recovery: what happens on hardware failure?

The system’s “virtual cluster” promise is that faults are contained and recovery is fast.

### 11.1 What’s guaranteed to survive?

If one cell fails:

* that cell and any VMs with dependencies on it may be lost,
* other cells/VMs continue. 

### 11.2 Evidence from fault injection experiments

They injected hardware faults and checked correctness of surviving VM workloads. An experiment is “successful” if:

* exactly **one** cell and VMs dependent on it are lost,
* surviving VMs produce correct results.
  They report **100% effectiveness over 1000 experiments** covering router/link/node/firmware failures. 

### 11.3 Recovery time

Recovery time is reported as **< 0.5 seconds** across tested configurations, with stronger dependence on **memory per node** than on number of nodes. The paper explains why:

* hardware must scan coherence directories to determine cache-line status after failure,
* firmware helps determine which memory pages contain inaccessible/incoherent lines,
* both require directory scans (expensive in firmware), could be faster with hardwired node controllers. 

Answers: “what dominates recovery time and why?”

---

## 12) Exam prep

### A) “Why cells? Why not just one global VMM?”

* Cells provide **hardware fault containment**; only VMs using that cell’s resources die. 
* Also improves scalability by avoiding global contention (local resource management).

### B) “What’s the key tradeoff?”

* **Fault containment vs resource utilization**: spreading a VM across more cells improves load balance but increases failure exposure.  

### C) “How do they avoid heavy distributed protocols between cells?”

* Trust the small monitor; use shared memory for VM-specific structures when safe. 
* Use RPC/message primitives + owner-based serialization for sensitive operations. 

### D) “Explain CPU load balancing + gang scheduling”

* Idle balancer steals *specific* VCPUs that make the VM runnable as a gang; periodic balancer fixes leftovers. 

### E) “Why migration isn’t just a one-time cost”

* Because remote memory access is a **continuous** penalty; hence dynamic page migration/replication of hot pages.  

### F) “How do they avoid redundant paging/double paging?”

* Virtual paging disk + mapping trick (Figure 9) collapses guest paging I/O into VMM-managed storage updates. 
