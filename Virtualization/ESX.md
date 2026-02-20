## 1) What ESX is trying to achieve 

ESX Server is a **bare-metal VMM** that runs **unmodified guest OSes** while still giving the operator control over **isolation guarantees** and **high utilization** under **overcommit** (sum of VM “physical” sizes > machine RAM). 

The paper’s theme is: *use low-level indirection (extra translation) to implement powerful policies without guest changes.*

Two tensions drive all mechanisms:

1. **Performance isolation**: “important” VMs shouldn’t be harmed disproportionately.
2. **Efficiency**: idle VMs shouldn’t hoard RAM while active VMs thrash.

A pure “partition by shares” scheme gives isolation but can waste memory; ESX adds **working-set awareness** via an **idle tax** (details later).

---

## 2) Low-level memory virtualization: VPN → PPN → MPN

### 2.1 The three “address spaces” to must keep straight

ESX introduces a standard naming split (borrowed from Disco):

* **VPN (virtual page number)**: what processes generate.
* **PPN (“physical” page number)**: what the *guest OS thinks* is physical memory (a contiguous, zero-based “hardware RAM” illusion).
* **MPN (machine page number)**: actual host machine RAM page.

### 2.2 Data structures: pmap + shadow page tables (SPT)

ESX maintains per-VM:

1. **pmap**: maps **PPN → MPN**.
2. **Shadow Page Tables (SPTs)**: hardware-used page tables that map **VPN → MPN** directly.

The key design move: **guest page tables are not what the hardware walks**. The hardware walks the **SPT** maintained by ESX.

### 2.3 How ESX keeps SPTs consistent with the guest (trap-and-emulate for PT/TLB writes)

Guests try to:

* write page table entries,
* load CR3 / change page table base,
* modify TLB state, etc.

ESX **intercepts** those actions, so the guest cannot directly alter real MMU state. ESX then updates:

* pmap (PPN→MPN) bookkeeping as needed
* SPT entries (VPN→MPN) so that normal loads/stores run fast via the hardware TLB caching VPN→MPN translations.

**Practice-exam tie-in:** the exam explicitly calls out the “glaring inefficiency” of a naive two-step translation and the fix: trap guest PT/TLB updates, then directly populate the SPT with VPN→MPN mappings (using stored PPN→MPN).

### 2.4 Why this indirection is “extremely powerful”

Once ESX owns PPN→MPN, it can **remap** a guest-“physical” page by just changing the pmap entry—**transparent to the guest**. That’s the enabling trick for:

* **ballooning “pin these PPNs”**
* **hypervisor paging “this PPN now lives on disk”**
* **page sharing “this PPN now points at a shared MPN (CoW)”**
* **I/O remapping “this PPN now lives in low memory for DMA/copy reasons”**

---

## 3) Reclaiming memory under overcommit: ballooning vs paging

When ESX is overcommitted, it must reduce some VM(s)’ machine-memory usage.

### 3.1 Why “just swap from the hypervisor” is tricky (meta-page replacement + double paging)

If ESX pages out guest-“physical” pages to a host swap area, ESX must decide *which guest pages to evict*—but the guest OS has the best information about which pages are least valuable (its own replacement policy, file cache semantics, etc.). This is the classic **meta-level page replacement** problem.

Even worse, **double paging** can happen: ESX pages out a PPN; later the guest (under memory pressure) decides to page out that same page to its own virtual swap, causing extra I/O churn.

### 3.2 Ballooning: “coax the guest OS to give up the pages it least wants”

**Ballooning** is ESX’s preferred reclamation mechanism because it delegates “which pages?” to the guest OS.

Mechanism:

* ESX has a **balloon driver/module** inside the guest (a pseudo-device driver / kernel service).
* To reclaim memory, ESX tells it to **inflate**:

  * the driver allocates guest-“physical” pages via normal OS mechanisms,
  * then **pins** them so the guest can’t just reuse them,
  * those pinned pages become “unavailable” to the guest → guest OS experiences memory pressure → it runs its native reclamation/page replacement to free pages it considers least valuable.

To give memory back, ESX tells it to **deflate** (free/unpin those pages).

Why ballooning is so effective conceptually:

* ESX chooses *how much* memory to reclaim from a VM.
* The guest chooses *which pages* to sacrifice.
* So ESX avoids needing a sophisticated meta-page policy and avoids many double-paging pathologies.

### 3.3 When ballooning isn’t enough → ESX paging (forced reclamation)

Ballooning requires:

* the balloon driver to be running,
* the guest to be able to free something (otherwise it may start paging internally).

If ballooning cannot reclaim enough (or isn’t available), ESX uses **hypervisor paging** to forcibly reclaim memory. The paper describes state-based control (soft→hard→low) where paging becomes the tool of last resort.

---

## 4) VM-oblivious page sharing (content-based sharing) and its costs

### 4.1 Goal: eliminate redundancy across VMs without guest cooperation

ESX runs a background scanner that looks for **identical page contents** across VMs (and within a VM), then merges them to a single machine page, using **copy-on-write** to preserve correctness.

Conceptually:

1. Find candidate pages (via hashing/content signatures).
2. Verify equality (full compare) to avoid hash collisions.
3. Remap multiple PPNs (from possibly different VMs) to point to the same MPN.
4. Mark mappings **CoW**.
5. On a write, trap, allocate a private copy, update that VM’s pmap/SPT mapping.

### 4.2 Overheads (exactly what your practice exam asks)

The exam’s “VM oblivious page sharing overheads” answer:

* **Hashing + comparison overhead**: CPU cycles and memory bandwidth for scanning, hashing, then comparing full pages when hashes match.
* **CoW overhead**: write to a shared page triggers a trap and a copy+remap, which is on the application’s critical path.

That’s the tradeoff: spend background CPU to save RAM; pay occasional CoW faults to keep isolation.

### 4.3 “Share before swap” intuition

The paper’s dynamic reallocation experiment highlights that sharing (especially many zero pages) can dramatically reduce how much actual disk paging is needed during pressure—i.e., reclaim memory by *deduplicating* before you *swap*.

---

## 5) Estimating working sets: statistical active-memory sampling

To tax “idle” memory, ESX needs an estimate of how much of each VM’s allocation is actively used.

Key technique:

* ESX **samples** a small number of pages periodically (default: **100 pages per 30 seconds**).
* It induces **minor page faults** on those pages to observe whether they get touched again, producing a statistical estimate of active use with negligible overhead.

Then ESX smooths estimates using multiple exponentially-weighted moving averages:

* a **slow** average for stability,
* a **fast** average for agility,
* and an “incremental fast” variant to respond within a period,
* finally taking a max to react quickly to increases and slowly to decreases.

Interpretation:

* “Active fraction” ≈ working set / allocation.
* “Idle memory” ≈ allocation − working set.

---

## 6) Allocation policy core: shares + min/max + idle memory tax

### 6.1 Administrative knobs: min, max, shares

ESX exposes three main per-VM controls:

* **max**: VM’s configured “physical” size (guest boots thinking it has this much).
* **min**: guaranteed lower bound even under overcommit.
* **shares**: proportional entitlement (relative importance).

Unless overcommitted, VMs get their max. Under overcommit, ESX computes a **target allocation** per VM based on entitlement and working set.

### 6.2 Share-based allocation: min-funding revocation

In proportional-share allocation, each VM has shares; its “fair” fraction depends on total shares in the system. When someone needs memory, ESX can reclaim from a “victim” VM that has the **fewest shares per allocated page** (economic interpretation: lowest “price” paid per page).

### 6.3 The big fix: idle memory tax (idle-adjusted shares)

Pure shares can be inefficient: an idle VM with many shares hoards memory while an active VM with few shares suffers.

ESX solves this by charging a higher “cost” for idle pages than active pages: an **idle memory tax**. Intuition:

* As memory gets scarce, ESX preferentially reclaims from VMs with lots of idle memory.
* If that VM later becomes active, it can “buy back” memory up to share entitlement.

Formally (paper’s description): ESX extends shares-per-page with an *adjusted* ratio that depends on the fraction of pages that are active vs idle and a tax rate τ.

**Practice-exam (idle-tax step-by-step):** the exam’s reclamation policy “tax 25% of idle memory each pass” is basically a discretized version of “reclaim from idle first.” The worked solution: (reclaim 100MB from the VM with 400MB idle, then 75MB, then 50MB, then 75MB…)

---

## 7) High-level coordination: admission control + reclamation states

### 7.1 Admission control: ensure mins and swap safety

Before powering on a VM, ESX checks:

* reserve machine memory for **min + overhead** (overhead includes structures like pmap and SPTs, plus frame buffer, etc.).
* reserve disk swap for **max − min** (so ESX can preserve VM memory under worst-case conditions, though typically it doesn’t use all of it).

This is “don’t admit what you can’t backstop.”

### 7.2 Reclamation state machine: high / soft / hard / low

ESX uses free-memory thresholds to decide how aggressive to be:

* **high**: no reclamation
* **soft**: reclaim via **ballooning**; page only if ballooning not possible
* **hard**: rely on **paging** to forcibly reclaim
* **low**: page and **block** VMs above their targets until free memory rises

Default thresholds are given as percentages (e.g., 6%, 4%, 2%, 1% for high/soft/hard/low). There’s hysteresis to avoid oscillation.

“what does ESX do under pressure?”: *it tries cooperative mechanisms first, then forced mechanisms, then throttling.*

---

## 8) Hot I/O page remapping (reducing copy overhead)

On some hardware/OS combinations, I/O buffers in “high memory” can require expensive copying/bounce buffering. ESX monitors “hot” pages involved in repeated I/O copies (e.g., network transmit buffers), and when a page’s copy count crosses a threshold, ESX **remaps** that guest PPN to a low-memory MPN transparently, reducing copies by orders of magnitude for some workloads.

This is another example of “extra indirection lets the hypervisor fix a performance problem without guest changes.”

---

## 9) On exam

### A) “Explain ballooning and why it’s better than hypervisor paging”



* balloon driver inflates → pins guest pages → guest reclaims internally using its own policy (choosing least valuable pages).
* avoids meta-page replacement and reduces double-paging risk.
* paging is fallback (hard state) when ballooning can’t work.

### B) “VM-oblivious page sharing: what overheads?”

* hashing/comparison overhead
* CoW trap+copy+remap overhead

### C) “Given an idle-tax policy, show step-by-step reclamation”

each pass taxes a fraction of current idle, re-sort as required, iterate until request met.

### D) “Explain VPN→PPN→MPN and how to avoid the inefficiency”

* guest maintains VPN→PPN; ESX maintains PPN→MPN and installs VPN→MPN in SPT by trapping guest PT updates.

---
