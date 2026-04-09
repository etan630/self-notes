# Giant Comparison Table — Distributed Systems Reading List

## Giant comparison table

| Paper | Area | Core problem | Main abstraction / idea | How it works | What tradeoff it makes | Why it matters / likely exam angle |
|---|---|---|---|---|---|---|
| **Lamport — Time, Clocks, and the Ordering of Events** | Distributed systems | No obvious global time in a distributed system | **Happened-before** partial order; **logical clocks** | Each process maintains a logical clock; send includes timestamp; receive sets clock to `max(local, msg)+1`; total order via `(timestamp, process id)` | Gives causality-respecting order, not true physical time | Foundation of logical time, causality, event ordering, mutual exclusion; exam question is often “difference between partial and total order” |
| **Thekkath & Levy — Limits to Low-Latency Communication** | Distributed systems / networking | Why high-bandwidth networks still do not yield low RPC latency | System-level latency decomposition | Build a low-latency RPC and measure cost of controller, copying, interrupts, cache effects, OS path | Faster links expose host/controller/software overhead as bottlenecks | Classic “latency vs bandwidth” lesson; exam angle: why network speed alone is insufficient |
| **Schroeder & Burrows — Performance of the Firefly RPC** | Distributed systems / RPC | How to make RPC fast enough to be the default communication mechanism | Careful **fast-path RPC engineering** | Stub-based RPC; detailed end-to-end cost model across marshaling, checksums, scheduling, NIC, interrupts | Optimized implementation complexity for lower latency | Classic measurement paper; exam angle: where RPC time really goes |
| **Wetherall — Active Networks: Vision and Reality** | Distributed systems / networking | Whether programmable packets/routers are actually practical | **Capsules** executed at active nodes | Packets carry or reference code; nodes execute code with caching, safety checks, heterogeneous deployment | Gains flexibility, but raises performance/security/resource-control issues | Reality-check paper; exam angle: what worked in active networks and what did not |
| **Mitchell et al. — Spring System** | Distributed objects / middleware | How to build a distributed OS around objects | OS-wide **object model** | Services exposed as objects; references and invocation unify local/distributed services | Cleaner modularity, but distribution semantics become central system design issue | Exam angle: Spring as the distributed object OS vision |
| **Hamilton, Powell, Mitchell — Subcontract** | Distributed objects / middleware | One remote invocation policy does not fit all objects | **Pluggable invocation semantics** via subcontracts | A subcontract defines how references and calls behave for a class of objects | More flexibility, more complexity in runtime | Exam angle: why Spring separates object model from remote invocation policy |
| **Wollrath, Riggs, Waldo — Distributed Object Model for Java / RMI** | Distributed objects / middleware | How to support distributed objects naturally in Java | **Java-native remote method invocation** | Remote interfaces, stubs/skeletons, serialization, dynamic class loading, transport/ref layers | Homogeneous Java-only model rather than language-neutral interoperability | Exam angle: contrast Java RMI with CORBA; “preserve Java semantics where sensible” |
| **Feeley et al. — Global Memory Management in a Workstation Cluster** | Distributed subsystems | How to use idle RAM across a cluster to reduce paging to disk | **Cluster-wide memory management** | Pages can live in remote memory; global replacement balances local/global pages and fetches from peers before disk | Remote memory is faster than disk, but coordination/replacement policy is hard | Exam angle: remote memory as a cluster resource, not ad hoc remote paging |
| **Amza et al. — TreadMarks** | Distributed subsystems / DSM | How to provide shared-memory programming on a workstation cluster | **Lazy release consistency** with **twins/diffs** | Writes create twins; diffs computed later; updates propagated mainly at synchronization points | Weaker consistency for much less communication | Exam angle: why DSM becomes practical with lazy release consistency and diffs |
| **Anderson et al. — Serverless Network File System / xFS** | Distributed subsystems / file systems | Central file servers bottleneck performance and reliability | **Serverless file system** | Peers provide storage, cache, and control; uses distributed management, cooperative caching, log-based striped storage | More complex distributed control in exchange for scalability/availability | Exam angle: how xFS removes centralized bottlenecks |
| **Satyanarayanan et al. — Coda** | Distributed subsystems / file systems | How to keep a file system available despite server/network failures and disconnection | **Server replication + disconnected operation** | Whole-file caching, optimistic replication, callbacks, reintegration on reconnect, conflict detection | Gives up strong always-current consistency for availability and mobility | Exam angle: disconnected operation and optimistic replication |
| **Satyanarayanan et al. — Lightweight Recoverable Virtual Memory (RVM)** | Failures & recovery | How to provide simple transactional persistence without a full DBMS | **Recoverable virtual memory** as a user-level library | Map recoverable regions; log updates; support atomicity/durability with lightweight runtime | Less general than a full DBMS, but much simpler and more portable | Exam angle: why RVM was intentionally minimalist |
| **Lowell & Chen — Free Transactions With Rio Vista** | Failures & recovery | How to make transactions much cheaper using persistent memory ideas | Transactions on top of **recoverable main memory** | Treat persistent memory as durable state, reducing conventional logging costs | Depends on special assumptions/hardware support | Exam angle: how hardware changes the cost model of transactions |
| **Haskin et al. — Recovery Management in QuickSilver** | Failures & recovery | How to provide one recovery framework for a distributed client-server OS | **Transactions as unified recovery mechanism** | Servers use commit/log primitives; flexible one- or two-phase commit; custom server recovery policies | More infrastructure overhead, but uniform failure semantics across services | Exam angle: transactions beyond databases; why Quicksilver exposes primitives instead of hiding them |
| **Gray et al. — Recovery Manager of a Data Management System** | Failures & recovery / DB | How a DBMS guarantees atomicity/durability across crashes | **Write-ahead logging**, undo/redo, checkpoints | Log before page write; commit forces durable log state; restart redoes winners/undoes losers | Logging overhead for correctness and recoverability | Foundational recovery paper; exam angle: WAL and restart recovery logic |
| **Porter et al. — Operating System Transactions / TxOS** | Failures & recovery / OS | How to make multiple system calls atomic and isolated | **System transactions** over OS resources | Enclose system calls in transaction begin/end; kernel versions/isolate changes until commit | Adds OS complexity/overhead to eliminate races and partial-failure inconsistencies | Exam angle: TOCTTOU elimination, transactional install rollback |
| **Peng & Dabek — Percolator** | Failures & recovery / internet-scale processing | How to do incremental processing on huge datasets without rerunning full batch jobs | **Transactions + observers/notifications** | Built over Bigtable; snapshot-isolation transactions; column notifications trigger observer code incrementally | More coordination overhead than plain Bigtable, but much lower latency than batch recomputation | Exam angle: gap between DBMSs and MapReduce; why Percolator exists |
| **Brewer — Lessons from Giant-Scale Services** | Internet-scale computing | How to design services that stay up while growing and evolving rapidly | Operational principles: **graceful degradation, MTTR focus, disaster tolerance, online evolution** | Use replication, admission control, degraded modes, careful rollout/upgrade, availability-oriented metrics | Accept reduced functionality/freshness to preserve service continuity | Exam angle: operational design principles rather than one mechanism |
| **Dean & Ghemawat — MapReduce** | Internet-scale computing / data processing | How to make massive batch parallelism easy for programmers | **Map + Reduce** computation model | User writes map/reduce; runtime handles partitioning, scheduling, data locality, fault recovery, shuffle | Restricts programming model to get automatic scale and fault tolerance | Exam angle: why the abstraction is simple but powerful |
| **Barroso, Dean, Hölzle — Google Cluster Architecture** | Internet-scale computing | How to serve web search at planetary scale economically | **Huge cluster of commodity PCs + software reliability** | Global DNS/load balancing, query fan-out to index shards, result merge, doc servers, replication for capacity and fault tolerance | Use many cheap nodes instead of fewer expensive ones; reliability moved to software | Exam angle: throughput-oriented design and software-managed reliability |
| **Freedman, Freudenthal, Mazières — Coral** | Internet-scale computing / CDN | How to let small publishers survive flash crowds cheaply | Peer-to-peer CDN using **DSHT** | Coralized URL redirects via DNS to nearby proxy; proxies cache and advertise content through locality-aware distributed sloppy hash tables | Weaker consistency and more complex overlay in exchange for cheap scalable distribution | Exam angle: locality-aware lookup and hot-spot avoidance |
| **DeCandia et al. — Dynamo** | Internet-scale computing / storage | How to provide always-writable highly available storage for Amazon-like services | **Highly available key-value store** with eventual consistency | Consistent hashing, replication, vector clocks, quorum-style reads/writes, gossip, hinted handoff, app-level conflict resolution | Gives up strong consistency under some failures to maximize availability | Exam angle: availability-first design, eventual consistency, quorum + versioning |

## Fast cross-paper comparison by theme

| Theme | Papers | Main takeaway |
|---|---|---|
| **Logical time / ordering** | Lamport | In distributed systems, order comes from causality, not from a global clock |
| **Low-latency RPC / communication path** | Firefly RPC, Thekkath & Levy | Once networks get faster, software, NIC, and controller overhead dominate |
| **Programmable network / flexibility vs safety** | Active Networks, Coral | Network flexibility is useful, but performance/security/resource control become the hard part |
| **Distributed objects / middleware** | Spring, Subcontract, Java RMI | Distribution semantics should be explicit; remote invocation policy often needs flexibility |
| **Using the cluster as one machine** | GMS, TreadMarks, xFS | Cluster RAM/storage/cache can be treated as shared resources if communication is fast enough |
| **Availability in file systems** | Coda, xFS, Coral | Replication and decentralization improve availability, but consistency and management get harder |
| **Transactions / recovery** | RVM, Rio Vista, QuickSilver, Gray/System R, TxOS, Percolator | Transactions are the general tool for atomicity and failure recovery, but systems differ in scope and cost model |
| **Internet-scale batch vs incremental** | MapReduce, Percolator | Batch is simple and scalable; incremental lowers latency but needs more coordination |
| **Availability over consistency** | Dynamo, Coda, Brewer | Large services often prefer partial inconsistency over outage |
| **Commodity clusters + software fault tolerance** | Google cluster, MapReduce, Dynamo | Cheap machines plus software replication/failure handling beat expensive “big iron” in many settings |

## Fast “what should I compare on an exam?” sheet

| Compare | Key distinction |
|---|---|
| **Lamport vs real clocks** | Logical clocks preserve causality; physical clocks approximate real time with bounded skew |
| **Firefly RPC vs Thekkath & Levy** | Firefly dissects one RPC system; Thekkath/Levy generalize why system overhead dominates on fast networks |
| **Spring vs Java RMI** | Spring is an OS-wide distributed object system; RMI is a language/runtime-level distributed object model for Java |
| **Spring vs Subcontract** | Spring is the overall system vision; Subcontract is the pluggable substrate for invocation/reference semantics |
| **GMS vs TreadMarks** | GMS uses remote memory as backing/storage resource; TreadMarks exposes shared memory abstraction to programmers |
| **Coda vs xFS** | Coda emphasizes availability via replication/disconnection; xFS emphasizes scalability by eliminating the central server |
| **RVM vs full DB recovery systems** | RVM is lightweight/minimalist for persistent data structures; DB recovery systems are richer but heavier |
| **QuickSilver vs TxOS** | QuickSilver uses transactions as system-wide recovery for distributed servers; TxOS applies transactions directly to OS resources/system calls |
| **MapReduce vs Percolator** | MapReduce is bulk batch dataflow; Percolator is incremental transactional processing |
| **Google cluster vs Dynamo** | Google cluster paper is about query-serving architecture; Dynamo is about storage semantics under failure |
| **Coral vs CDN/server replication** | Coral decentralizes caching and lookup via peers/overlay instead of relying on managed central CDN infrastructure |
| **Dynamo vs traditional ACID store** | Dynamo chooses availability and eventual consistency; ACID systems choose stronger consistency/isolation |

## One-line memory hook for each paper

| Paper | Memory hook |
|---|---|
| Lamport | “Time in distributed systems is causality, not wall clock.” |
| Thekkath & Levy | “Fast networks don’t help if the host path is slow.” |
| Firefly RPC | “Measure every microsecond on the RPC fast path.” |
| Active Networks | “Programmable packets: appealing, but security/perf bite back.” |
| Spring | “Distributed OS built around objects.” |
| Subcontract | “Make remote invocation policy pluggable.” |
| Java RMI | “Remote objects, but Java-first.” |
| GMS | “Use idle RAM anywhere in the cluster.” |
| TreadMarks | “DSM with lazy release consistency and diffs.” |
| xFS | “No central file server.” |
| Coda | “Replicated files + disconnected operation.” |
| RVM | “Tiny transactional persistence library.” |
| Rio Vista | “Persistent memory can make transactions cheap.” |
| QuickSilver | “Transactions as OS recovery glue.” |
| System R recovery | “WAL, undo, redo, checkpoint.” |
| TxOS | “Transactional system calls.” |
| Percolator | “Incremental processing with transactions + notifications.” |
| Brewer | “At scale, graceful degradation beats purity.” |
| MapReduce | “Constrain the model, simplify the system.” |
| Google cluster | “Planet-scale search on commodity PCs.” |
| Coral | “Peer-to-peer CDN with locality-aware lookup.” |
| Dynamo | “Always writable, eventually consistent key-value store.” |
