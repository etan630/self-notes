## 1) What LRPC is fixing (the “why”)

### The setting: small-kernel / microkernel-ish OS structure

Small-kernel OSes push OS subsystems (file server, pager, network stack, etc.) into **separate protection domains** for:

* fault isolation
* modularity / maintainability
* least-privilege

But then those subsystems must communicate constantly. If that communication is expensive, designers “cheat” by merging subsystems into one big domain (faster, but less safe).

### Why “traditional RPC/message IPC” is too slow locally

Traditional RPC is designed for **cross-machine** calls:

* independent client/server threads
* message buffers, marshaling/unmarshaling
* multiple copies and dispatch layers

Even if you use it *on the same machine*, you still pay a lot of that machinery. The LRPC paper argues the common case is:

1. **cross-domain on the same machine** dominates, and
2. arguments are **simple/small** (handles + ints), not huge structures. 

So the design goal is: **isolate the common case** and make it almost as cheap as the hardware will allow.

---

## 2) LRPC’s core idea

>LRPC is “protected procedure call semantics + RPC-style interfaces,” implemented by temporarily moving the *client’s* thread into the server domain, and passing arguments via a shared, pairwise-mapped argument stack. 

This is the key conceptual shift:

### Conventional message IPC model

* client thread sends message
* server thread receives, runs, replies

### LRPC model

* client thread traps into kernel
* kernel validates + switches thread’s domain/address space
* that same thread executes the server procedure
* returns back via kernel

That collapses lots of overhead (thread handoff, queueing, dispatch, copying).

---

## 3) The main objects/mechanisms 

### 3.1 Protection Domains (PDs)

A **domain** is essentially an address-space protection boundary (server in its own, client in its own). LRPC is specifically optimized for *same-machine cross-domain calls*.

### 3.2 Binding Object (capability-like “ticket” for calling)

Before calling, a client must **bind** to a server interface. If the server allows the bind, it’s authorizing the client. After binding, the kernel returns a **Binding Object** to the client. 

Properties:

* Client must present it on each call.
* Kernel can detect forgeries → client can’t bypass bind. 
  This is LRPC’s “capability flavor”: authorization is explicit and checkable.

### 3.3 A-stacks (Argument stacks): fast data transfer

LRPC allocates **pairwise** shared memory regions between a client and a server called **A-stacks**.

* They are mapped into *both* domains.
* They are typically per-procedure (or per-interface entry) and managed as a small pool. 

Why they matter:

* Traditional RPC might copy an argument multiple times (client stack → message → kernel → server → server stack).
* LRPC often copies **once**: client stub → A-stack, then server reads directly. 

This is one of the biggest wins.

### 3.4 E-stacks (Execution stacks): where the thread actually runs

Even though arguments live on A-stack, the executing thread needs a normal call stack in the server domain: an **E-stack** allocated inside the server domain. LRPC switches the thread to an E-stack during the call. 

* **A-stack**: shared argument/results region (pairwise client-server)
* **E-stack**: private execution stack in the current domain

### 3.5 Linkage records + linkage stack (nested cross-domain calls)

When a thread crosses domains, LRPC needs to know how to get back.
On call:

* kernel records caller’s return PC + SP into a **linkage**
* pushes linkage onto a per-thread “linkage stack” in the thread control block 

This supports nested LRPC chains (client → serverA → serverB …) because returns follow the linkage stack.

---

## 4) The exact call/return fast path (step-by-step)

### 4.1 Call path

1. **Client code calls client stub** for procedure `p`.

2. Stub:

   * pops a free **A-stack** for `p` from a LIFO pool
   * writes/marshals arguments into the A-stack (often just memcpy according to calling convention)
   * puts (A-stack address, Binding Object, procedure ID) into registers
   * traps into kernel 

3. Kernel (still “on behalf of” the client thread):

   * verifies Binding Object + proc ID → finds server PD 
   * verifies A-stack belongs to that binding and not currently in use
   * records client return state into linkage, pushes onto thread linkage stack
   * finds/assigns an **E-stack** in server domain
   * switches thread’s user stack pointer to server E-stack
   * switches MMU context / address space to server domain
   * upcalls directly into the **server stub** entry point 

4. Server stub:

   * reads arguments from A-stack
   * (for pass-by-reference) recreates safe references (more below)
   * calls the actual server procedure

### 4.2 Return path

1. Server procedure returns to server stub.
2. Server stub traps to kernel to “return”.
3. Kernel:

   * does **not** need Binding verification (authorization happened on call; the linkage stack encodes where to go) 
   * switches address space back to caller domain (pops linkage)
   * resumes at caller’s return address
4. Client stub:

   * reads return values from A-stack
   * returns A-stack to pool

Key point: returns are cheap because much is implicit in per-thread linkage state.

---

## 5) Safety and correctness details 

### 5.1 “But the A-stack is shared—can someone tamper?”

Yes: client and server can theoretically mutate values asynchronously. LRPC’s stance is: don’t pay message-copy cost blindly; instead decide per-argument:

* copy when needed to ensure integrity
* leverage language calling conventions and semantics (value vs reference) 

So LRPC is “selectively copy” rather than “always copy 4 times.”

### 5.2 Pass-by-reference arguments: recreate references safely

The client can’t just hand the server a raw pointer (could be bogus or point to memory the server shouldn’t trust). LRPC does:

* client stub copies referent data onto A-stack
* server stub creates a reference to that A-stack region and places it on its own E-stack before invoking server code 

So: **data not re-copied**, but **reference is reconstituted** safely.

### 5.3 Domain termination and “captured thread” problem

Because the *client thread* runs in the server domain, a server can “hold” the client thread indefinitely (intentionally or by crash/bug).

LRPC includes “special case” mechanisms:

* If a server domain terminates while handling an LRPC, the kernel revokes bindings and restarts threads back in clients with a **call-failed** exception. 
* If a client domain terminates, an outstanding call must not return into a dead domain; invalid linkage records lead to call-failed or thread destruction. 
* If a client wants to recover from being captured, it can create a new thread that resumes as if the call returned **call-aborted**, while the original captured thread continues until released and then is destroyed. 

---

## 6) Scheduling + cache affinity: why LRPC also cares about the scheduler

The LRPC paper is very explicit that a big hidden cost in cross-domain calls is **domain switching overhead** (TLB/cache effects) and **scheduler interaction**. LRPC introduces optimizations that look a lot like “cache-affinity scheduling,” but for **domains** rather than just threads.

### 6.1 Handoff scheduling context (what LRPC builds on)

The paper notes other systems used **handoff scheduling** to avoid general scheduler overhead: if you know the two threads involved, you can context switch directly rather than enqueue/dequeue through global scheduler state. 

LRPC goes further because it doesn’t need a separate server thread in the common case—*it already has* the client thread.

### 6.2 Domain caching (LRPC’s big scheduling trick)

LRPC reduces domain switch overhead by **caching domains on idle processors**:

* If a call is made and there’s a CPU currently idling *in the server domain’s address space*, the kernel can **exchange processors** between:

  * the calling thread, and
  * that idling thread
* Result: the client thread starts running in the server domain on a CPU where the server domain context is already loaded (less TLB/cache disruption). 

On return, the same trick can be used if a CPU is idling in the client domain. 

**Think of it as “Last Processor / Fixed Processor” style affinity, but keyed on *domain context* rather than on the thread itself.**

This connects directly to the exam’s scheduling discussion: FP improves cache affinity but may hurt load balance; FCFS balances but ignores affinity.  LRPC’s domain caching is explicitly prioritizing *affinity wins* for the domain-switch-heavy workload.

### 6.3 “Prodding” idle CPUs toward hot domains

LRPC maintains per-domain counters for “wanted an idle CPU in this domain but didn’t find one.” The kernel uses this to nudge idle processors to spin in domains showing high LRPC activity. 

So LRPC is **actively shaping** idle CPU placement to reduce future cross-domain latency.

---

## 7) Why LRPC gets “near the hardware lower bound”

LRPC’s performance claims are essentially: we can’t beat:

* trap/return overhead
* address space switch overhead (TLB/cache)
* minimal bookkeeping

But we can remove:

* extra thread scheduling paths
* message queues and buffer allocators
* multi-copy marshaling paths
* multi-level dispatch

And we can *mitigate* address-space switch costs with domain caching. 

---

## 8) “Is LRPC useful in a microkernel OS?”

Yes—exactly because microkernels force lots of cross-protection-domain calls.

* subsystems above the microkernel need protected procedure call to talk
* use LRPC to optimize PPC across those subsystems (file server ↔ storage, VM manager ↔ storage, etc.) 

Also mention:

* microkernel decomposes services → IPC dominates
* LRPC makes IPC closer to procedure-call cost
* better modularity without forcing co-location (avoid “put everything in one server for performance”)

---

## 9) Mental model + “what to write on an exam” checklists

### If asked: “List the key techniques that make LRPC fast”

Use the paper’s four-bullet framing (and expand concretely):

1. **Simple control transfer:** client thread executes in server domain 
2. **Simple data transfer:** shared A-stack, minimal copies 
3. **Simple stubs:** predictable model → highly optimized stub generation (less runtime machinery) 
4. **Design for concurrency:** avoid global bottlenecks + exploit MP parallelism 

### “Walk through an LRPC call”

* bind → Binding Object
* choose A-stack → push args
* trap → validate binding/proc/A-stack
* linkage push
* switch to server E-stack + address space
* upcall to server stub → server proc
* return trap → pop linkage → back to client stub


### “What problems arise because the client thread runs in the server?”

* server can “capture” client thread
* domain termination / outstanding call return complexity
* need revocation + linkage invalidation + call-failed/call-aborted handling 

