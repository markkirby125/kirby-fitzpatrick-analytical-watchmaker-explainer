# Analytical Watchmaker Explainer — Technical Operational Dispatcher

**Framework Author**: William Fitzpatrick (*Writer Science*)  
**Source Lecture**: [20 Years of Writing Advice in 52 mins](https://www.youtube.com/watch?v=G-Sl0-PZv2Q)  
**Parent Collection**: [Master Collection](../../kirby-fitzpatrick-writers-collection/SKILL.md) | [Global Help](../../../kirby-help/SKILL.md)  

---

## 1. Cognitive Foundation: The Watchmaker's Stance — Gears Over Chronology

Every system has two orderings, and they are not the same ordering.

The **assembly order** is the sequence in which statements run: line 42 takes the token, line 44 checks the map, line 46 pushes onto the channel, line 51 takes the mutex. The **mechanism order** is the set of meshing relationships: which part consumes which artifact, what each part guarantees to the next, and which property the whole train is holding in place while it runs. Source files are written in assembly order. Trace logs print in assembly order. Stack frames unwind in assembly order. Nothing in the toolchain ever emits mechanism order — it exists only as an architectural decision inside the head of whoever built the subsystem.

That asymmetry is the entire problem. A reader given assembly order receives an unlabeled serial walk of a dependency graph and is required to reconstruct the graph themselves, from scratch, on every read. Line narration therefore charges the reader for knowledge the writer already paid for. Worse, the reconstruction is not mechanical: five named positions in a sequence tell the reader *where* things happen and nothing about *why the arrangement holds*, so the reader cannot predict what the system does when an input changes, cannot tell which edit is safe, and cannot localise a fault to a part. They can recite the code and cannot reason about it.

The watchmaker's stance inverts the delivery order. A watchmaker does not explain a watch by narrating its assembly line. They describe the train: the mainspring delivers torque; the gear train scales that torque through a known ratio; the escapement converts continuous torque into discrete, evenly spaced ticks; the balance wheel holds the period; the hands are the output. Then — only then — the individual teeth matter, because the reader now has a slot to place each one in. The mechanism is **topology plus invariants**, and the parts are placeable detail inside it.

Four registers carry the whole explanation, plus the coupling that binds them:

1. **Input** — the artifact entering the subsystem (a token stream, a batch of records, a config revision, a request) *and its precondition*. Stating the precondition first is what makes the mechanism conditional rather than universal: "given UTF-8 without BOM" is the difference between a description and a claim.
2. **Intermediate Transform (gear)** — a named component that performs *exactly one* transformation and holds its own contract. "The lexer" is a gear. "The logic" is not a gear; it is the housing of an unexamined design. A gear that cannot be given a single responsibility in one clause is really two gears welded together, and naming it honestly is usually the first defect the framework surfaces.
3. **Invariant (the escapement)** — the property that every mesh point preserves while the train runs: ordering per key, single-writer access, monotonicity, conservation of records, at-most-once execution, termination. The invariant is the load-bearing part of the explanation, because it is what separates *what the code happens to do on this input* from *what it must do for any input*. It is also what makes the explanation falsifiable — a stated invariant can be tested, and an invariant that no test enforces is a fact the reader needs.
4. **Output** — the artifact leaving the subsystem *plus its guarantee and its cost*. Every gear train pays for its ratio in friction; an output claim without a cost axis is a brochure.
5. **Mesh point** — the boundary where one gear's output becomes another's input. This is where assumptions cross, where a contract is either honoured or quietly broken, and therefore where concurrency bugs, ordering bugs, and impedance mismatches live. Mesh points are the reviewable unit of the whole explanation.

Why software is uniquely prone to getting this wrong: the artifact's own register is assembly order. The author who just wrote the code has the mechanism in working memory and habitually emits the cheapest encoding of it, which is the walk they just performed. Two degraded shapes result, and both are systematic rather than accidental. The **exploded view** is a parts list with no torque path — every class named, no coupling described, so the reader receives a box of components and cannot assemble them. The **black box** is the inverse — a gear named at high altitude (*"the scheduler handles it"*, *"the executor drains the queue"*) and never meshed, so a load-bearing component is asserted and its contract is never stated. Exploded view leaves the reader with parts; black box leaves them with a word. Neither leaves them with a mechanism.

The framework also has a diagnostic edge that pure style advice does not. Because gears are *claimed* rather than observed, a writer who attempts the decomposition on a genuine hairball is forced to say so: the parts only mesh through a shared mutable blob, or there is no invariant the train preserves, or the topology is a cycle with no termination condition. Naming gears is a design audit that produces findings, not just prose. If the gear train cannot be drawn, that is the most important thing the document can report.

```text
[ANTI-PATTERN: Chronological Line Narration — the exploded view]

  "First line 42 takes the token. Then line 44 checks the map. Line 46 pushes
   onto the channel. Then line 51 locks the mutex. Finally the consumer wakes up."

   trace:  L42 ──▶ L44 ──▶ L46 ──▶ L51 ──▶ L58 ──▶ ?
             │      │      │      │      │
         (five edges described separately, none of them named, none of them
          given a contract, no shared property identified anywhere)

  Reader holds 5 unlabeled positions in working memory and still cannot answer:
    - what happens if L44 blocks?      - which part owns the ordering guarantee?
    - what is L46 allowed to assume?   - what breaks if the consumer dies?
  Reader leaves with: a sequence they can recite, no part they can reason about.
  Symptoms: re-derives the mechanism in every read; comments that narrate lines;
  cannot predict the effect of changing one component; asks "but why?" forever.
```

```text
[PROTOCOL: The Gear Train — torque path, mesh contracts, escapement]

   INPUT                    INTERMEDIATE TRANSFORMS                 OUTPUT
  ┌────────┐   mesh     ┌──────────────┐  mesh   ┌──────────────┐  ┌───────────┐
  │ token  │───────────▶│  lexer gear  │────────▶│ parser gear  │─▶│ typed AST │
  │ stream │            │ chars→tokens │         │ tokens→tree  │  │ + spans   │
  └────────┘            └──────────────┘         └──────────────┘  └───────────┘
   precondition              contract                  contract         guarantee
   UTF-8, no BOM        longest match wins       every token is either   AST is a
   (else: reject at     (maximal munch)          consumed or reported    total function
   entry, never mid-                            with a span — never     of the input
   stream)                                      silently dropped        + error list

  INVARIANT (the escapement — enforced at every mesh point):
    "no character is both consumed as input and reported as an error"
    enforced at: parser.go:118 (error path re-reads the same span it stopped on)

  MESH MODES: sync-blocking (lexer→parser, pull)  ● feedback: none, the train is acyclic

  Reader leaves with: the torque path, what each part may assume, the property the
  train preserves, the output guarantee and its error contract — and can therefore
  swap the parser for a different one and predict the effect on the output.
```

---

## 2. Core Transformation Protocols

1. **Open with the input and its precondition — never with line 1.** The first sentence of any walkthrough, module docstring, or PR body names what enters the subsystem and what must be true of it before the train moves. A reader who meets an identifier before knowing the artifact it consumes has no slot to place the detail in and will read the whole explanation twice.
2. **Every actor must be a named gear with exactly one transformation.** "The lexer converts character runs into tokens with spans" is a gear. "The code", "the logic", "it", "we then", and "the handling" are housings. A component that needs two clauses to state its transformation is two gears; splitting them is where the real explanation appears.
3. **State each contract at the mesh point, not inside the gear.** The reader needs to know what a part may *assume about its input* and what it *guarantees to its consumer* — not how it is implemented internally. Internal detail is placed only after the mesh is described, and only where it is load-bearing for a contract.
4. **Draw the torque path before describing any tooth.** Topology precedes detail: the reader must hold the train before meeting the components. A document that describes component A completely, then component B completely, has published an exploded view regardless of how accurate each section is.
5. **Never narrate a line the reader can read.** The explanation budget is spent entirely on what the source does not say: why the mesh holds, what the ordering assumption is, which alternative arrangement was rejected. `Line 4 initialises the client. Line 5 sends the request.` is zero-value prose; it restates syntax at the cost of attention.
6. **Declare exactly one invariant per subsystem, with its enforcement site.** One sentence, in the system's own vocabulary (ordering, single-writer, monotonic offset, at-most-once, conservation, termination), plus the `path:line` where it is enforced and the test that would fail if it broke. An explanation with four "invariants" is a list of hopes; an explanation with none is a description of the current input only.
7. **Name the coupling mode at every mesh.** Synchronous or asynchronous, pull or push, rendezvous or buffered (with the buffer width), ordering-preserving or reordering, blocking or non-blocking. "A → B" is not a mesh; "A pushes to B through a capacity-0 channel, so A blocks until B receives" is, because it tells the reader what happens under load.
8. **Declare feedback as feedback, with its termination condition.** A cycle in the train (retry loops, re-entrant callbacks, fixpoint passes, gossip gossip) is legitimate — but it must be drawn as a cycle, labelled with the invariant it maintains across iterations, and bounded by a stated termination argument. An undeclared loop is the black box wearing a gear costume.
9. **Explain failure as gear slip, per gear.** For each gear name the three slip modes — stall (produces nothing), double-fire (produces twice), reorder (produces out of sequence) — and which invariant breaks first in each case. This is what converts an explanation into an operational artifact: the on-call engineer reads the failure section to route the pager, not the happy path.
10. **Separate the trigger from the guard in stateful and concurrent gears.** A transition is two gears, not one: the *event* that supplies torque and the *condition* that decides whether the teeth engage. Describing them as a single arrow ("`Pending` → `Active`") hides the guard, and the guard is where every concurrency defect lives.
11. **Give the ratio, not the instance.** "The batcher flushes on a watermark" describes the gear; "at 4k rps it flushed three times" is evidence of one input. Mechanism claims belong in the prose; measured quantities belong in the evidence section with the command that produced them. Mixing the two produces numbers that cannot be re-derived and mechanisms that scale to exactly one workload.
12. **Close on the output guarantee and its cost; state the ratio-change condition.** The last rung names what the output guarantees, what it costs (latency budget spent, transactions per row added, memory bounded, throughput exchanged for ordering), and the condition under which the current ratio stops being the right one. *"More scalable"* names no property, so nothing below it can discharge the claim.
13. **Label every gear you cannot mesh.** When a component is out of scope, third-party, or unknown, mark it at the boundary with what you *do* know about its contract, and never write pseudocode in the register of real implementation. A declared boundary is information; an invented gear is a defect that survives review because it reads confidently.

### 2.1 The gear train reference — registers and failure modes

| Register | Question it answers | Software artifact | Failure mode if omitted |
|---|---|---|---|
| Input + precondition | *What enters, and what must already be true?* | Call contract, config schema, message envelope, IR properties | Reader assumes universal validity; every downstream claim becomes unfalsifiable |
| Gear + contract | *Which part runs, and what may it assume / what does it promise?* | Module boundary, interface, pass, actor, pool | Exploded view — parts named, couplings unstated, no safe edit |
| Mesh point | *How does torque cross, and what happens under load?* | Channel/buffer, lock, RPC, callback, iterator pull | Blocking and ordering bugs are invisible until production |
| Invariant (escapement) | *What property is preserved while the train runs?* | Ordering per key, single-writer, monotonicity, at-most-once, termination | Explanation describes one input; no prediction, no test target |
| Output + guarantee + cost | *What leaves, what does it promise, what did it cost?* | Return contract, error semantics, SLO delta, budget spend | Gears described, purpose absent; no basis for accepting the design |
| Ratio-change condition | *When does this arrangement stop being correct?* | Threshold, expiry, load ceiling, p99 bound | Every reader re-litigates the decision; no signal to revisit it |

### 2.2 Transformation table: anti-patterns and clean replacements

| Anti-Pattern | Diagnosis | Clean Replacement |
|---|---|---|
| *"First we read the file, then we parse it, then we write it back."* | Serialized assembly order; no gears, no contracts, no invariant | *Input:* a config file whose entries are keyed and versioned → *gears:* reader (I/O, no interpretation), parser (entry → typed record, unknown keys are errors), applier (record → mutation, applied in key order) → *invariant:* a partially-applied config is never observable (`config.go:212` applies into a scratch copy and swaps) → *output:* a new immutable config snapshot, costing one full copy per write |
| `// retry logic` above a retry block | Zero-value tooth narration; the reader can already see the block | `// Bounds the caller's deadline, not the downstream service: 3 attempts x 400ms backoff`<br>`// stays inside the 2s budget. Reversed by setting attempts from config.` |
| *"Line 4 initialises the client. Line 5 sends the request."* | Line narration — assembly order at zero information cost | *The client is built once per worker and reused; the request path only mutates headers. The mesh is one pool per worker, which is what bounds concurrent connections.* |
| *"The mutex protects the map."* | Gear (lock) asserted, mesh and invariant unnamed | *Invariant: at most one writer mutates the session map, so every read observes a whole entry. Mesh: every writer takes the lock; readers take it too, because the map is not copy-on-write — the cost is read contention under fan-in.* |
| *"We use a channel so there are no races."* | Mesh mode omitted; the interesting failure (blocking, reordering) is hidden | *Mesh: producers push onto a capacity-0 channel, so a producer blocks until the consumer receives — backpressure is by construction, throughput is capped by the consumer, and ordering is FIFO per channel but not across channels.* |
| *"The scheduler handles the rest."* | Black box — a gear named and never meshed | Name the gear and its contract, or fence it: *out of scope — assumed to be work-conserving; if it is not, the invariant at `scheduler.go:88` does not hold.* |
| State enumeration: *"States are Pending, Active, Draining, Closed."* | Parts list without triggers or guards | Four gears: `Pending → Active` on `admit` **if** capacity remains; `Active → Draining` on `shutdown`; `Draining → Closed` when the last in-flight request returns. *Invariant: no state observes a partially-written session; guard lives at `session.go:301`.* |
| Compiler walkthrough: *"We visit the AST, then emit code."* | Chronological visit order; pass contracts and IR invariants absent | Pass pipeline with the IR property at each boundary: *the fold pass may assume no unrealized casts; it guarantees every constant subexpression is evaluated; the bounded-`const` invariant is what makes the later SSA build total.* |
| PR body that lists changed files | The diff is already the assembly order; the body adds nothing | Body carries input/precondition → gear contracts changed → invariant → output guarantee + cost → evidence table with the command that produced it |
| *"Optimised with memoisation."* | Output claim with no gear, no invariant, no mesh | *Gear:* memo keyed on the immutable config hash → *invariant:* recomputation is never observable, so the cache is transparent → *output guarantee:* 1 recompute per 4,000 calls, *cost:* one hash per call, *invalidated if* the hash ever covers mutable state |
| Pseudocode fenced as if it were source | Fabricated gear | Label it: *"shape of the mesh, not the implementation"*, then cite the real `path:line` |

### 2.3 Failure diagnostics

| Symptom in review | Diagnosis | Fix |
|---|---|---|
| *"I still don't know what this does."* | Exploded view — every part named, no torque path | Draw the train: input → gears → output, one line, above the parts |
| *"Wait, does this run before or after that?"* | Assembly order supplied where mechanism order was asked for | Declare mesh modes; state ordering as an invariant, not as line order |
| *"What stops two writers doing this?"* | Invariant absent or unnamed | State it, cite the enforcement site, name the failing test |
| *"Is this safe under load?"* | Mesh described topologically but not operationally | Add buffer width, blocking mode, and the stall behaviour of each gear |
| Reviewer argues about style, not the mechanism | No gear boundaries exist to argue about | Introduce gear names and contracts; move the argument to the mesh point that decides behaviour |
| Explanation is re-derived in every new PR | No written train to inherit | Persist the train in the artifact — module docstring, ADR, or PR body |
| Component is "the one we don't touch" | Black box accepted as knowledge | Fence it with its assumed contract and the consequence if the assumption fails |
| Doc reads fine, but nobody changes the code confidently | Missing ratio-change condition; the reader cannot tell what may be altered | Add the condition that reverses the design and mark the gears that are free to swap |

**Related dispatchers.** Set the abstraction ladder above the train with [Uneven U Explainer](../../kirby-fitzpatrick-uneven-u-explainer/SKILL.md); prevent gear vocabulary from arriving before the reader has primitives by announcing structure with [Cathedral Taxonomy](../../kirby-fitzpatrick-cathedral-taxonomy/SKILL.md) and grounding it via [Bilbo Simple-to-Complex](../../kirby-fitzpatrick-bilbo-simple-to-complex/SKILL.md); chain gears explicitly with [Linear Relay Linking](../../kirby-fitzpatrick-linear-relay-linking/SKILL.md); make mesh points the reviewable unit via **3D Architectural Grounding**; hunt unhandled gear slip with **Semantic Gap Hunter**; verify the train survives without its author via the [Silent Author Test](../../kirby-fitzpatrick-silent-author-test/SKILL.md).

---

## 3. Engineering Application Scenarios

### 3.1 Code Reviews — Review the Mesh, Not the Lines

Three rules govern review under this framework. First, **each comment is itself a gear claim**: a reviewer asserting a defect must name the gear that owns it and the mesh point where the contract fails, or the author can only guess which part the reviewer was reading. Second, **the highest-severity comment class is a missing or unenforced invariant**, above any boundary or tooth detail — an unnamed invariant means no test can exist for it and no future edit can be judged safe. Third, **severity follows the register order**: invariant > mesh contract > gear boundary > tooth detail. A comment about a literal is a tooth comment and should be one line, because the budget rule applies to reviews too.

**Before — line narration and an unlabeled objection.**

```go
// retry logic
client := newClient(cfg)
resp, err := client.send(req)
if attempts > 3 {
    return err
}
```

> I don't follow this. Line 1 initialises the client and line 2 sends the request. Why is the client created here? This retry logic is confusing and hardcoded. Please add clarity.

Diagnostics: the comment is a serial walk of the diff at zero information cost; *"confusing"* names no gear, no mesh, and no invariant; the author's only available response is to guess which register was wanted, which typically produces a comment-explaining-the-code fix — more assembly order.

**After — gears named, mesh point localised, missing invariant escalated first.**

```markdown
**Gear identification**
- Gear 1 — client construction (creates a connection pool). Called from inside the loop,
  so its output (a live pool) is discarded on every iteration: the component exists, but
  nothing meshes to it.
- Gear 2 — request path. Pure header mutation; fine as written.
- Gear 3 — attempt counter / retry gear.

**The real question (highest severity, asked not asserted)**
The invariant "at most one live pool per worker" is neither stated nor enforced. Is the
pool meant to be per-worker or per-call? Every other comment depends on the answer:
per-worker makes Gear 1 a worker-scoped gear and the mesh becomes borrow-a-connection;
per-call makes the current shape correct and the retry budget the only defect.

**Then, at the mesh point**
Gear 3's contract is unstated: how does the backoff schedule compose with a slow attempt —
does a 1.9s attempt consume the 2s budget, or is it abandoned mid-flight? Warm-up of 400ms
plus a slow attempt means the caller's deadline is the actual bound, not `attempts > 3`.

**Teeth (one line each, deferrable)**
- `attempts > 3` should come from config; the literal is the only thing here that is genuinely hardcoded.
- `retry_test.go` covers the attempt cap but not the deadline-crossing case (`:214` is the gap).
```

**Acceptance rule for review comments.** A comment that requests clarity must name the missing register — input precondition, gear contract, mesh mode, invariant, or output guarantee — and the artifact that would close it. A comment that supplies a fix must supply the mechanism, not a corrected line, or it hands the author the same defect one register up. For concurrency and state-machine changes, an approval without a stated invariant is not a completed review.

### 3.2 PR Descriptions — The Gear Train in the Description Layer

The diff is the assembly order, produced by the machine. The body's job is the other register: what enters, which gears changed and what they now promise, what invariant is being established or protected, what the output guarantees, and what it costs. A body that lists files or narrates hunks has moved the diff one level up the stack.

```markdown
## Input + precondition
Compilation of a 4,000-unit pass graph on a 64-core host. Precondition: pass dependencies
form a DAG (already asserted at graph construction, `deps.go:44`). 41.2s wall, 38% of which
is worker idle time waiting on dependencies that are ready but unclaimed.

## Gear train (mechanism, as changed)
```
   ┌───────────┐  push-ready   ┌──────────────┐  steal  ┌──────────────────┐
   │ dep graph │──────────────▶│ ready deque  │◀────────│ worker gears x64 │
   │ (DAG)     │  (in-degree   │ per-worker,  │  (LIFO  │ execute pass unit│
   └───────────┘   reaches 0)  │ + global FIFO│  tail)  └──────────────────┘
   └───────────┘               └──────────────┘         └──────────────────┘
   mesh: push is non-blocking; steal is non-blocking and retries from the global deque
   INVARIANT: a unit runs exactly once, and only after every dependency has completed
   enforcement: completion counter CAS at `driver.go:206`; readiness gate at `deps.go:88`
```
- **Dispatch gear (`driver.go:180`)** — was a single global lock-protected queue; now one
  deque per worker plus a global FIFO fallback. Contract: never blocks a worker that has
  a ready unit of its own; borrowing only when the local deque is empty.
- **Steal gear (`driver.go:214`)** — victim's tail, own head (LIFO-local/ FIFO-remote),
  so the steal rate stays low and locality stays high. Contract: no unit is visible to two
  workers at once — stolen units are removed under the victim's tail CAS before handoff.
- **Completion gear (`driver.go:206`)** — unchanged in contract; now the *only* serialisation
  point, because dependency release happens on the counter transition to zero.

## Invariant protected
"Exactly-once execution, dependencies-before-consumers." Unchanged from before this PR —
the change is that the invariant is now enforced by one CAS on the completion counter
rather than by a global queue whose ordering happened to imply it. Diagnostics remain
deterministic because results are sorted at the join, not by execution order (`report.go:61`);
that is why the steal order is free to be unfair.

## Evidence
`make bench-compile UNITS=4000 HOST=c64r5`
| Metric | main @ 7c1a9d2 | PR head | Δ |
|---|---|---|---|
| wall time | 41.2 s | 6.8 s | −83% |
| worker idle (dependency-wait) | 38% | 3% | −35 pp |
| units stolen / dispatched | — | 1.2% | locality preserved |
| per-unit dispatch overhead | 0.9 µs | 1.0 µs | +11% (cost of the invariant) |

## Output guarantee + cost + ratio-change condition
*Guarantee:* every pass unit executes exactly once, after its dependencies, with
deterministic diagnostic ordering. *Cost:* +11% per-unit dispatch overhead and one extra
CAS on the completion path — the deque makes load balancing cheaper, not units cheaper.
*Ratio changes if:* the graph stops being a DAG (then readiness gating must become
cycle detection), or units drop below ~1 µs work (then dispatch dominates and the
64-deque split costs more than the idle time it removes).
```

**Binding rules.** Heading text carries the register, so a 15-second skim still crosses the train. The invariant states its enforcement site and the test that would fail without it. The evidence table names the command that produced it, and the prose never supplies a number the table cannot re-derive. Every claim about a gear is a contract claim, not a line claim — and if a register has no artifact (no test for the invariant, no measurement of the cost), it is written as an open question with an owner rather than in the confident register.

### 3.3 Architecture RFCs / ADRs — The Gear Train at Document Scale

At document scale the train becomes a **section map**: each section carries one register, and the order of sections is the torque path. Two flat documents are equally defective — the parts-catalogue RFC (every component described, no coupling, nothing falsifiable) and the invariant-only decision log (a property asserted, no gears shown enforcing it, so no reader can tell whether it still holds after the next refactor).

| ADR section | Gear register | Obligation |
|---|---|---|
| Context | Input + precondition | The artifact entering and what must already be true of it, in the system's vocabulary |
| Mechanism | Gears + contracts | One transformation per gear, with the contract it holds at each mesh |
| Invariants | Invariant (escapement) | The property preserved, its enforcement site, its test — one per subsystem |
| Failure modes | Gear slip | Stall / double-fire / reorder per gear, and which invariant breaks first |
| Consequences | Output + cost | The guarantee, the price of the ratio, and the condition that reverses the decision |

**Rules for the section map.**

- The Mechanism section must be drawable. If the reviewer cannot sketch input → gears → output from the prose, the RFC is an exploded view with headings on it.
- Invariants live in their own section with enforcement sites, never inside a gear description. An invariant mentioned in passing inside a component is indistinguishable from an aspiration.
- Failure modes are mandatory and per-gear. An RFC for a concurrency engine without a stall/double-fire/reorder analysis has described only the version of the system that is not having its worst day.
- Rejected alternatives live at the mechanism register with their invariants, or the next reader re-litigates them from scratch — which is exactly the cost this framework exists to remove.
- Consequences must not ascend to generality. *"This makes the platform more scalable"* names no property and cannot be falsified; state the exchange — ordered commits bought with a throughput ceiling per shard.
- Every RFC closes with the ratio-change condition. A design decision with no expiry condition is a decision nobody is allowed to revisit.

```markdown
# ADR-034 — Single-writer commit path per shard

## Context (input + precondition)
Commit records enter the storage node from N application goroutines; the WAL is per shard.
Precondition: every record carries a shard key, and the shard count is fixed at deploy time.

## Mechanism (gears)
- **Batcher gear (`wal.go:64`)** — coalesces records into blocks of `min(interval, 512 rec)`.
  Contract: never emits a block that exceeds the 1 MiB page budget.
- **Fencer gear (`shard.go:112`)** — serialises block assignment per shard, monotonic LSN.
  Contract: an LSN is issued to exactly one block; re-issuing is a hard error, not a retry.
- **Committer gear (`wal.go:151`)** — fsync of the assigned prefix, in LSN order.
  Contract: returns only after the prefix is durable.
- **Publisher gear (`read.go:88`)** — advances the readable offset.
  Contract: advances strictly after the committer returns for every LSN below it.

## Invariants (escapement)
1. **A shard's readable offset only advances through a durably-committed prefix.** Enforced
   at `read.go:88`; the offset is only mutated by the publisher gear on committer success.
   Test: `TestPublisherNeverOutrunsCommitter` (fault-injects fsync failure at LSN N+3).
2. **The readable offset never regresses.** Enforced by the monotonic LSN assignment in the
   fencer; regression is impossible by construction rather than by check.

## Failure modes (gear slip)
| Gear | Stall | Double-fire | Reorder |
|---|---|---|---|
| Batcher | write path backpressures; readable offset pins — correct, not a bug | duplicate LSN → fencer rejects (invariant 1 holds) | none, per-shard serial |
| Fencer | new commits queue; readers unaffected | second issue is a hard error, surfaced as a shard fault | n/a |
| Committer | write path stalls inside its deadline → surfaces as caller timeout | fsync replay is idempotent by LSN | guarded: committer drains in LSN order |
| Publisher | readers stall at the pinned offset — the safety-favouring degradation | idempotent (offset is a max, not an increment) | would violate invariant 2; prevented structurally |

## Consequences (output + cost + reversal)
*Guarantee:* readers never observe an offset that later changes, and never observe an
uncommitted prefix. *Cost:* commit throughput per shard is capped at one fsync batch per
~2 ms — parallelism is bought by sharding, not by writers. *Reverses if* fsync p99 exceeds
8 ms sustained for an hour: at that point shard the log rather than add writers, because
adding writers breaks invariant 1 and recovering it requires consensus, not a lock.
Rollback is a config flag; no format migration.
```

---

## 4. Verification Checklist

- [ ] **Every actor is a named gear with exactly one transformation and a stated contract.** No line narration (`Line 4 does X`), no anonymous actors (`it`, `the logic`, `the code`), and no component described both as a gear and as a housing without being split — each gear's input assumption and output promise appears at its mesh, not buried in its internals.
- [ ] **The torque path precedes the teeth, and every mesh declares its coupling mode.** The document draws input → gears → output before any component detail; each mesh names sync/async, pull/push, buffered width, and whether ordering is preserved; every feedback edge is drawn as a cycle with a termination argument, and no gear is named without being meshed.
- [ ] **Exactly one invariant per subsystem, stated in the system's vocabulary with its enforcement site and its test.** No invariant is left implicit; none is asserted without a `path:line` enforcement point; and the failure section identifies, per gear, the stall / double-fire / reorder modes and the invariant that breaks first in each.
- [ ] **The output carries a guarantee, a cost axis, and a ratio-change condition.** The close names what the output promises (including error semantics), what the arrangement paid for it (latency budget, transactions per row, memory bound, throughput ceiling), and the condition under which the ratio is no longer correct — with no ascent to unearned generality (*"more scalable"*, *"cleaner"*, *"more robust"*).
- [ ] **The substitution test passes and every unmeshed boundary is labelled.** A reader who has finished the document can replace one gear and predict the effect on the output and on the invariant; quantities appear only in the evidence section and are re-derivable from the cited command, test name, or trace; and any gear that could not be reached is fenced at its boundary with its assumed contract rather than invented.