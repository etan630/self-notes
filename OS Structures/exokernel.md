## 1) The exokernel -- what it is trying to solve

Traditional OSes have **high-level abstractions** (processes, address spaces, files, IPC, TCP/IP stacks, etc.). These are:

* are **overly general**, so some apps pay for features they don’t need
* **hide low-level information** (faults, interrupts, raw device semantics), making specialization hard
* and make it hard to **replace or evolve** the abstractions (kernel change = huge trust and engineering burden)

The exokernel flips the relationship:

* The **kernel (exokernel)** is a *thin* protection/multiplexing layer over hardware resources.
* **Library OSes (LibOSes)** in user space implement the *abstractions/policies* (VM policy, IPC, filesystem, scheduling policy, networking stacks, etc.).
* Apps link against a LibOS (or multiple libraries) to get the exact semantics and performance profile they want.  

Concrete names from the paper/implementation:

* **Aegis** = prototype exokernel (kernel)
* **ExOS** = example untrusted library OS 

---

## 2) The core design rule: separate *protection* from *management*

> **Exokernel does protection + secure multiplexing; LibOS does management + policy.**

* **Protection** = ensure mutually distrustful LibOSes can’t violate isolation, steal resources, or corrupt each other.
* **Management** = decide *how* to use resources to implement OS abstractions (replacement policy, buffer cache policy, scheduling policy, etc.).  

So the exokernel must:

1. **export** hardware resources and “privileged” operations safely,
2. **allocate** and **revoke** resources (because ownership disputes need an arbiter),
3. keep itself small/simple so it can be fast. 

---

## 3) What the exokernel actually exports (the “low-level interface”)

The paper’s stance is: export **real hardware resources**, not heavyweight OS objects. Typical exported things include:

* **Physical memory pages**
* **CPU time slices**
* **TLB / address translation mechanisms**
* **Interrupts/exceptions (“discontinuities”)**
* **Disk blocks / device DMA capabilities**
* **“Addressing context identifiers”** (hardware tags/ASIDs if present)
* “Cross-domain calls” as a low-level primitive, not as an imposed RPC abstraction 

Lecture Emphasis: kernel is “resource sharing, no policies”; high-level stuff lives in LibOS. 

---

# 4) The three big mechanisms

The paper explicitly says exokernel securely exports resources via three techniques:

1. **Secure bindings**
2. **Visible revocation**
3. **Abort protocol**  

---

## 4.1 Secure bindings (the centerpiece)

### Definition

A **secure binding** *decouples authorization from use*:

* At **bind time**, the LibOS proves it is allowed to access a resource and establishes a binding.
* At **use time**, the resource can be used efficiently with **simple, fast checks** that do *not* require the kernel to understand high-level semantics.  

This gives two big wins (from the paper):

* **Fast checks** (simple primitives the kernel/hardware can do quickly)
* **Amortization**: you don’t redo expensive authorization/interpretation on every use. 

### What does “one-time authentication” mean?

“one-time authentication” is basically **bind-time authorization**: the exokernel validates a capability / validates downloaded code / validates the binding setup *once*, then later the “use” path is cheap.  

---

### 4.1.1 Secure bindings for memory: capabilities + TLB protection

### The problem

If the kernel lets LibOSes manage virtual memory policy, the LibOS must be able to establish mappings. But the kernel must ensure:

* you can’t map pages you don’t own,
* you can’t DMA into arbitrary memory,
* you can’t forge rights.

### Mechanism

The paper’s prototype uses **capabilities** to protect physical pages. A LibOS must present the right capability when it wants to access/map a page; the exokernel checks it. 

Also:

* if the hardware uses TLB loads as the privileged operation, the exokernel checks capabilities on **TLB entry installation** (bind time for that mapping).
* to reduce overhead, the exokernel may use a **software TLB (sTLB)** as a cache of translations/bindings beyond the hardware TLB.  

### Why this matches “protection vs management”

* Kernel: “You may map page frame P only if you hold capability C with the needed rights.”
* LibOS: decides page tables, replacement, paging layout, shared memory structure, etc. 

---

## 4.1.2 Secure bindings for networking: packet filters + code download

### The problem

Incoming packets arrive at the NIC—who gets them?

* Traditional kernel: networking stack + demux logic is in kernel.
* Exokernel: want to keep policy/semantics (TCP, sessions, retransmission, etc.) in LibOS or user-level libraries, but still demultiplex packets **securely and fast**.

### Mechanism

Use **packet filters** as secure bindings:

* **Bind time:** a LibOS installs a predicate/filter into the kernel (or a trusted demux mechanism).
* **Use time:** on every packet arrival, the kernel runs the filter to decide ownership/handler.  

The paper frames this explicitly as secure binding: download predicate at bind time, run it at access time, avoiding the kernel having to “ask everyone” which packet is whose. 

The paper also notes performance engineering: compiling filters to machine code at runtime (in their prototype) to avoid interpretation overhead. 

### “Downloaded code” generalization

Secure bindings can be implemented by **downloading code into the kernel**, invoked on events. Key advantage: it can run **even when the LibOS isn’t currently scheduled**, so the system doesn’t have to buffer indefinitely waiting for the LibOS to run. 

From Lecture: “Download code … functionally equivalent to SPIN” (in the sense that safe in-kernel extension code can avoid crossings), but exokernel uses it as a **resource binding / demux** tactic rather than a general “everything is an in-kernel extension” architecture. 

---

## 4.2 Visible revocation (resource reclamation as a dialogue)

### Why revocation matters

Even if you “bind” resources to LibOSes, the kernel still must handle:

* contention,
* fairness / shares / priority,
* reclaiming resources when needed.

### Key idea

**Revocation is visible**: the exokernel tells the LibOS it needs the resource back, and the LibOS participates (e.g., chooses which page to give up, writes it out, etc.).  

The paper frames it as a **dialogue**:

* Kernel: “Please return a page.”
* LibOS: picks a page, performs whatever high-level management is needed (writeback, rebind, update tables), frees it. 

From Lecture: “Expose Revocation … Polite first” and “two-level replacement” intuition. 

### Visible vs invisible revocation

* **Visible** is great for resources where the LibOS has the best knowledge of what to give up.
* But for some *very frequently revoked* resources, visibility can be too expensive; the paper notes cases where **invisible revocation** can be better (e.g., very frequent stateless identifiers). 

---

## 4.3 Abort protocol (when the LibOS won’t cooperate)

Visible revocation assumes a well-behaved LibOS. But LibOS is **untrusted** and may be buggy or slow.

So exokernel needs a “forceful” second stage:

* If LibOS doesn’t comply, the kernel **breaks secure bindings by force** (without killing the LibOS necessarily).
* The kernel records the forced loss in a **repossession vector** and delivers a **repossession exception** so the LibOS can repair/patch up its metadata (page tables, mappings, etc.). 

This is crucial because exokernels expose **physical names** (pages, blocks). If the kernel reclaims “page 5,” the LibOS must update any place it referred to that physical name. Abort protocol is the “you didn’t respond; I took it anyway” mechanism that still preserves system safety.

---

# 5) Exokernel design principles (lecture + paper)

From Lecture:

* **Expose allocation** (no implicit allocation; LibOS participates in allocations)
* **Expose names** (export physical names; also expose bookkeeping structures where possible)
* **Expose revocation** (visible protocol; polite then forceful)  

From an exam perspective, this is the “why exokernel is fundamentally different from a microkernel”:

* Microkernel still tends to define “standard” OS abstractions in privileged servers and has fixed kernel-mediated IPC interfaces.
* Exokernel tries to avoid “fixing” abstractions even in servers—abstractions are **libraries you link**; the kernel is a protection/mux layer. 

---

# 6) Discontinuities: interrupts, exceptions, faults (“who is this for?”)

A recurring systems issue: CPU receives discontinuities (interrupts, exceptions, faults). The kernel must deliver them to the right party.

From Lecture: Exokernel keeps **per-LibOS data structures** to know where to route these discontinuities, and there are cases where:

* you might buffer interrupts until the LibOS runs,
* OR (key!) if a handler is downloaded/installed, you can execute it immediately. 

That “execute it if the handler is already downloaded” idea is exactly what your practice exam’s networking question is probing. 

---

## 7) Walkthrough: the practice exam’s packet-receive / no-loss question

The practice exam question asks for “key interactions” between **LibOS** and **Exokernel** to receive packets minimizing loss, and to explicitly distinguish:

* **one-time authentication** vs **use**.

Explanation:

### Goal

Packets must be handled immediately even if the destination LibOS isn’t scheduled, so you don’t lose packets due to buffering limits.

### Mechanisms to use

* **secure bindings**
* **capabilities**
* **code download** (packet filter / handler installation)
* **pinned page** receive buffers (so DMA / kernel writes land safely)

### Interactions (bind time vs use time)

**Bind time (“one-time authentication”):**

1. **LibOS writes a packet receive handler** (or a filter+handler pair) that, when executed, copies or DMA-writes the packet into a specific buffer page owned by that LibOS. 
2. **LibOS presents a capability that authorizes downloading/installing that handler/filter** into the exokernel (“download code” mechanism). Exokernel validates this once and installs it.
3. **LibOS presents a capability for the receive buffer page(s)** and requests they be **pinned** (not paged/revoked unexpectedly during receive), so the handler has a safe target. Exokernel validates capability/ownership.

**Use time (“access”):**
4) When a packet arrives, **Exokernel executes the installed filter/handler** immediately (it’s already in-kernel), determines the destination securely, and writes the packet directly into the LibOS’s pinned buffer—**even if some other LibOS is currently running**.
5) **LibOS must keep buffers available** (e.g., manage a ring of pinned pages) so subsequent packets can be accepted without drops. That’s policy/management, so it’s on the LibOS. 

That is exactly the “secure binding: authenticate once, use fast many times” pattern, instantiated for network demux and interrupt-time execution.

---

## 8) Approaching Exokernel Exam Quetions

1. **What is the resource?** (pages, CPU slice, NIC receive path, disk blocks, TLB entries, interrupts)
2. **What is the binding object?**
   * Memory: capabilities + TLB install (and possibly sTLB caching) 
   * Network: downloaded packet filter/handler
3. **What happens at bind time vs use time?** (one-time authentication vs fast access)
4. **What happens under pressure?**

   * visible revocation dialogue (polite) 
   * abort protocol + repossession vector/exception (forceful) 
5. **What is kernel vs LibOS responsibility?**

   * kernel: protect/multiplex + revoke/allocate
   * LibOS: implement abstraction + policy + respond to revocation and exceptions

---