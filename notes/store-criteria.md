# What the store actually has to do — criteria derived from the product brief

**Status: research, 2026-09-03.** Derived from the owner's product brief
([`AGENTS.md`](../AGENTS.md) item 9) and the four stated requirements in item 2. AGENTS.md
is the one copy of both; this note derives *consequences* and restates neither. ⚠ **No
candidate has been measured against these criteria** — the only hard number in the whole
evaluation remains the dependency count.

## Why the earlier criteria no longer apply

The brief adds three workloads that are not issue tracking — a registry, a message bus,
and a scheduler — and it settles the shape they run in. Two conclusions from item 9 do most
of the work here:

- **FerroTrack is a server with an embedded store.** Requirements 3 and 4 cannot be met by
  any database (a store can notify a connected subscriber; it cannot start a stopped
  process or run anything at a scheduled time), and requirement 2's inter-brand goal rules
  out a Rust API as the interface, because agents in other vendors' runtimes cannot link a
  crate. So there is a resident process, and it speaks a network protocol.
- ✅ **Therefore the database question got SMALLER.** Everything the brief describes —
  registry semantics, routing, wake-up, scheduling — lives in the server, above the store.

## The four workloads, which are not alike

| workload | shape | hot access pattern | lifetime |
|---|---|---|---|
| **Issues** | read-dominated | lookup by id; filter by state/assignee/label/scope | long-lived |
| **Registry** | small, hot, read-heavy | lookup by agent address; enumerate live agents | ⚠ **entries expire** — agents register at startup and die without deregistering |
| **Messages** | **write-dominated**, append-shaped | per-recipient ordered drain; "anything for X?" | ⚠ **short** — delivered messages should not accumulate forever |
| **Schedule** | small | ⚠ **range scan by due-time**: "everything due before now" | until fired |

⚠ **The registry's hard problem is liveness, not storage.** An agent that crashes does not
deregister. Whatever the store, the server needs heartbeat-or-lease semantics and a sweep;
"is this agent reachable" is never answered by the presence of a row.

## What the store MUST provide

1. **Embedded, single-process** — item 2's requirements 2 and 4, and see the soundness
   argument in [`redb-research.md`](redb-research.md): single-writer is what makes the
   server's own pub/sub broadcast complete rather than best-effort.
2. **ACID, with a conditional update evaluated INSIDE the transaction.** The referee's
   whole guarantee. ⚠ Still unmeasured on every candidate.
3. **Ordered range scans.** Both the schedule ("due before now") and per-recipient message
   drain are range queries over an ordered keyspace. This is a real requirement, not a
   nicety, and it is the one most likely to be assumed rather than checked.
4. **Atomic writes across collections.** Delivering a message and recording its receipt,
   or moving an issue and appending its event, must commit together.
5. **Durability across crash.**

## What it need NOT provide — and this is the useful half

- **A query language.** The server owns filtering.
- **Native pub/sub.** Ruled 2026-09-03: FerroTrack provides it.
- **Full-text search.** See the open item below — but it is not a *store* requirement.
- **Server mode, multi-writer, clustering, replication.** Actively unwanted: a second
  writer would break the broadcast-completeness property above.

## The real axis: how much plumbing does FerroTrack handle?

Set by the owner, 2026-09-03: **every feature in the brief is surfaced by FerroTrack.** A
store may *facilitate* one — SurrealDB's open (live) queries facilitate pub/sub — but a
store without it does not prevent the feature, it only moves the work. So the question is
never "which features does the store have." It is how much FerroTrack has to build.

⚠ **This also disposes of the requirement-5 worry**: because the features are FerroTrack's,
a standalone FerroTrack has them by construction.

### Measured 2026-09-03: where the dependency budget actually goes

Same method as before — `cargo add` into throwaway crates, `cargo tree` over the resolved
graph, no build. Measuring a *plausible whole product* rather than the store alone, because
"minimal dependencies" is a property of the shipped binary.

| probe | crates | +build | native toolchain |
|---|---:|---:|---|
| redb (store alone) | 1 | 1 | none |
| fjall (store alone) | 41 | 41 | none |
| axum (server layer alone) | 48 | 48 | none |
| **redb + axum + tokio** (server, hand-built index) | **56** | 56 | **none** |
| tantivy (full-text search alone) | 109 | 115 | `cc`, `pkg-config` |
| redb + tantivy + axum + tokio | 145 | 151 | `cc`, `pkg-config` |
| fjall + tantivy + axum + tokio | 161 | 167 | `cc`, `pkg-config` |

Three findings, in order of how much they should change a decision:

1. ⚠⚠ **Full-text search is the single largest dependency decision in the product, and it
   is far larger than the store choice.** tantivy costs **+89 crates** over the no-FTS
   stack and **reintroduces the C toolchain** — via `zstd-sys`, in `tantivy-columnar`,
   which is core rather than an optional feature. It gives back precisely the property a
   minimal store was chosen for. Search is now explicitly FerroTrack's to support
   (AGENTS.md item 9), so this cost is on this repo's budget, not a dependency's.
2. ✅ **The store is NOT where the budget goes.** In whole-product terms redb versus fjall
   is 145 versus 161 — a 16-crate difference inside a stack of ~150. The server layer
   (axum + tokio, ~55) and search dwarf it. **Choosing the store on dependency count is
   optimising the smallest term.**
3. ✅ **It also settles the fairness caveat this evaluation has been carrying.** The
   305-vs-1 comparison was fairly criticised as store-alone versus whole-database. The
   honest version is now measured: **the entire FerroTrack product on redb is 56 crates
   without FTS, 145 with it — against 305 for SurrealDB's store alone.** The scratch holds
   up under the comparison I said nobody had done.

### What each candidate saves FerroTrack, concretely

Both give ordered key-value storage, ACID, cross-collection atomic writes and range scans —
criteria 1–5 above. Everything else in the brief is FerroTrack's either way: issue CRUD and
filtering, search, the registry, routing, wake-up, scheduling, and the referee's semantics.

- **redb** — 1 crate, zero dependencies; copy-on-write B+trees, read- and scan-favouring;
  typed tables; multimap tables. ⚠ **No expiry mechanism**: sweeping delivered messages and
  dead registry entries is hand-written.
- **fjall** — 41 crates; LSM, write-favouring, which is the message bus's actual shape.
  **Partitions** give cross-collection atomicity by construction, and ✅ **compaction
  filters** (3.1, 2026-03) are the natural mechanism for expiry — the one piece of plumbing
  it removes that redb does not.

⚠⚠ **Do not choose between them on throughput.** At the brief's scale — a lab's worth of
agents, messages in thousands per day — both are far beyond adequate. **The whole honest
difference is: redb costs 40 fewer crates; fjall saves writing an expiry mechanism.** That
is a small, specific trade, and it should be decided as one.

## Full-suite comparison, 2026-09-04: redb vs fjall inside a fixed stack

Requested by the owner: **redb + tantivy + axum + tokio (145 crates)** against
**fjall + tantivy + axum + tokio (161)**. ⚠ Three of the four components are identical, so
this is redb-versus-fjall in a fixed frame — but tantivy's presence changes what the
differences are *worth*, which is the part that is easy to miss.

### What putting tantivy in the stack neutralises

1. ⚠⚠ **redb's "no C toolchain" advantage disappears.** tantivy brings `cc` and
   `pkg-config` via `zstd-sys`. **Both stacks need a C toolchain**; redb's purity only pays
   if tantivy is dropped.
2. ⚠⚠ **The transactional-consistency argument stops differentiating.** Both stacks already
   run two stores with separate commit cycles — the KV store and tantivy's index — so
   "search can never disagree with the ledger" is lost in *both*. That argument separates
   tantivy from a hand-built index; it does not separate redb from fjall.
3. **The dependency gap shrinks to 16 crates in ~150 — about 11%**, and is the smallest
   term in the comparison.

### Where they are equal, and should not be argued about

Both are `MIT OR Apache-2.0`; both are pure-Rust *storage* (the C comes from tantivy either
way); both are embedded, single-process, ACID-capable, with ordered range scans and atomic
cross-collection writes — so **both satisfy criteria 1–5**. Both are far beyond adequate at
this scale. Both leave FerroTrack exactly the same plumbing.

### redb — pros

- **1 crate, zero transitive dependencies.** The smallest possible audit, upgrade and
  supply-chain surface.
- ✅ **ACID and single-writer are the DEFAULT, not an opt-in — there is no way to
  accidentally get weaker semantics.** Weigh this against fjall's first con.
- **Typed tables** (`TableDefinition<K, V>` + derive) move field-mapping risk to compile
  time — which field carries state, counters and version becomes a type the referee can
  see, rather than runtime config.
- **Multimap tables** fit the structured indexes (d) calls for, and postings lists if
  tantivy is ever dropped.
- Copy-on-write B+trees: read- and scan-favouring, which is the issue workload, and range
  scans are the scheduler's hot path.
- **One file, ~50 KiB minimum** — trivial to back up, copy, or ship as a project asset.
- File-format stability commitment **with a shipped precedent** (`Database::upgrade()`,
  v2→v3), plus `ReadOnlyDatabase` for a separate inspection or backup process.

### redb — cons

- ⚠ **No expiry/TTL.** Sweeping delivered messages and dead registry entries is a
  background job to design, schedule and test — including not sweeping what a slow consumer
  still needs.
- **No compression.** Issue bodies and messages sit on disk raw; fjall ships LZ4 by default.
- B+trees are the weaker shape for the write-heavy message bus (see the scale caveat —
  this is unlikely to matter here).
- **24 `unsafe {}` blocks, 18 `unsafe fn`, 5 `unsafe impl` in ~32k LOC** (measured
  2026-09-04).
- Soundness history worth respecting: 4.0 fixed data written out after transaction
  completion; 4.1 fixed an unsound `restore_savepoint()`. Pin 4.2 and track releases.
- ⚠ Maintainer concentration looks high — **unverified, and worth checking before adoption**
  rather than asserted here.

### fjall — pros

- ✅ **Compaction filters** (3.1, 2026-03): user logic during compaction, which is *the*
  mechanism for expiring delivered messages and dead registry entries. **The clearest
  functional win either engine has.**
- ✅ **Partitions with atomic cross-partition writes** through a single journal — the four
  workloads map to four partitions and criterion 4 is met by construction.
- **LSM shape suits the write-heavy message bus**; **LZ4 compression by default** for
  text-heavy bodies.
- **Two transaction models** — `SingleWriterTxDatabase` (serialised) and
  `OptimisticTxDatabase` (optimistic concurrency) — both supporting **RYOW and
  fetch-and-update**, so compare-and-swap is available.
- MVCC snapshot isolation; stated disk-format policy (breaking change ⇒ major version plus
  a migration path); 3.0's format explicitly built for longevity and forward compatibility.
- Fast, active cadence (3.0 in 2026-01, 3.1 in 2026-03); MSRV 1.90.0 stated.

### fjall — cons

- ⚠⚠ **Transactions are OPT-IN, and the non-transactional path silently does not give you
  read-modify-write.** fjall's own documentation says plain `WriteBatch` operations cannot
  perform read-modify-write reliably without a transactional wrapper. **For the referee
  that is a silent correctness hole rather than an error** — and it is precisely the hazard
  [`redb-research.md`](redb-research.md) already named: not a broken store, but *a correct
  store used through a wrong call site*. fjall offers two call sites and only one is right.
  ⚠ Mitigable, and cheaply: a single store-access module that only ever opens the
  transactional database, with a test that asserts it.
- ⚠⚠ **"100% safe & stable Rust" describes the WRAPPER, not the engine.** Measured
  2026-09-04: `fjall` carries `#![deny(unsafe_code)]` and is clean — but `lsm-tree`, the
  crate that actually stores the bytes, has **31 `unsafe {}` blocks and no such
  attribute**. Net across the stack this is comparable to redb, not better than it.
  ⚠ **Same shape as the SurrealKV finding**: the safety claim describes the layer you call,
  not the layer that persists. Check the layer that persists.
- 41 crates versus 1, across several crates in the trust boundary rather than one.
- LSM means **background compaction** — periodic work, write amplification, and more files
  on disk than redb's single file, which is slightly more to back up or ship as an asset.
- Younger API: 3.0 "fully revamped APIs" in 2026-01, so the surface has moved recently.

⚠ Raw `unsafe` counts are a **crude proxy, not an audit** — an `unsafe impl Send` and a
pointer-arithmetic block are not the same risk. They are reported because a marketing claim
was checkable, not because the number settles anything.

### The trade, reduced

The 16 crates are not the decision. **The decision is an inversion between two properties:**

- **redb makes the correctness-critical thing unmissable and the convenience manual** —
  ACID by default, expiry hand-written.
- **fjall provides the convenience and makes the correctness-critical thing opt-in** —
  compaction filters supplied, transactional semantics something you must remember to ask
  for.

⚠ **FerroTrack exists to be a referee, and the conditional update inside the transaction IS
the guarantee.** On that basis redb's failure mode ("we have to write a sweeper") is more
comfortable than fjall's ("the guarantee can be silently absent"). ⚠ **Recorded as
reasoning, not as a ruling — the store decision is the owner's and remains open**, and
fjall's con is mitigable by discipline if compaction filters are wanted.

## Roadmap: the same stacks WITHOUT tantivy — and where that leaves search

Measured 2026-09-04, same method.

| stack | crates | native toolchain |
|---|---:|---|
| redb + axum + tokio | **56** | **none** |
| fjall + axum + tokio | **84** | **none** |
| redb + axum + tokio + rust-stemmers | 58 | none |
| fjall + axum + tokio + rust-stemmers | 86 | none |
| *(with tantivy, for reference)* redb / fjall | 145 / 161 | `cc`, `pkg-config` |

⚠⚠ **This reverses a conclusion recorded earlier in this note.** "The store is not where the
budget goes" was measured **inside the tantivy stacks**, where redb-versus-fjall is 16
crates in ~150 (11%). **Without tantivy the gap WIDENS to 28 crates in 56–84 — about 50%
relative.** The reason: tantivy's tree overlaps heavily with fjall's, so tantivy was
*masking* fjall's marginal cost. Remove it and fjall's crates stand alone. **Drop tantivy
and the store choice becomes a major term again**, not a rounding error.

### What dropping tantivy buys — both stacks

1. ✅ **−89 crates, and the C toolchain goes away entirely.** The stack becomes **pure Rust
   end to end**, which is exactly what item 2's requirement 4 asked for and what tantivy
   took back.
2. ✅ **Transactional consistency is restored.** A hand-built index lives in the store's own
   tables and commits **in the same transaction** as the issue write. ⚠ **This is the
   decisive resolution of (c)** — the agent-files-then-immediately-searches-for-duplicates
   miss cannot happen, because there is no second store to lag. Dropping tantivy is what
   *fixes* (c); adopting it is what creates the problem.
3. ✅ **One store, not two** — no reconciliation path, no crash-recovery divergence, one
   backup artifact, and one thing to ship as a project asset (item 2, requirement 2).

### What it costs — what you give up

- **BM25-grade relevance.** Field-weighted TF-IDF is achievable by hand; genuinely tuned
  ranking is not a weekend.
- **Fuzzy / typo tolerance** — the hardest of these to hand-build well.
- **Phrase queries** — need positional postings, which roughly doubles index complexity.
- **Battle-testing.** tantivy is mature; a hand-built index is new code with its own bugs,
  in a component whose failures are quiet (a missing result looks like an absent issue).
- Stemming is **not** on this list: `rust-stemmers` costs **+2 crates** on either stack.

### ⚠ How the two engines differ for a HAND-BUILT index — the reversal

With tantivy the store does almost no search work. Remove it and the index moves *into* the
store, becoming a **churning, write-heavy workload** — and that is fjall's shape, not
redb's.

- **redb** — `MultimapTable<&str, u64>` *is* term → document-ids. The most direct API fit of
  the two; postings commit in the same `WriteTransaction`; deletion is a direct multimap
  remove.
- **fjall** — no multimap, so postings are composite keys (`term|doc_id`) read back by
  prefix range scan: the standard LSM shape, and a good one. ✅ **Compaction filters get a
  second use** — garbage-collecting postings for deleted documents, alongside message
  expiry. The index can live in its own partition, atomic with the issue write.

⚠⚠ **So the two engines' advantages move in opposite directions when tantivy is dropped:**
redb's dependency lead widens (56 vs 84), while fjall's structural fit for the workload
improves. That makes this a real trade rather than a formality — and it is the opposite of
what the tantivy-stack comparison suggested.

### The roadmap, in phases

**Phase 0 — needed regardless, zero extra dependencies.** Structured indexes over project,
author, assignee, state and labels, with exact and prefix lookup. This is (d)'s structured
half and the issue-list views need it anyway. Answers "issues assigned to X in project Y."

**Phase 1 — still zero extra dependencies.** Substring/prefix find across title and
description. At a few thousand issues a scan of short fields is fast. Answers a large share
of what an agent actually asks.

**Phase 2 — the point at which search genuinely exists.** Hand-built inverted index over
title / description / body with **field weighting** ((d) requires it: a title match must
outrank a body match), boolean AND/OR, and optional stemming at +2 crates. Stack stays at
**58 (redb) or 86 (fjall)**, pure Rust.

**Phase 3 — only if relevance quality becomes the binding constraint.** Adopt tantivy then,
paying +89 crates and the C toolchain **when it is justified rather than speculatively.**
✅ **Backfillable, because (d) retains the text.**

**Phase 3′ — the AI-native alternative, worth knowing exists.** Vector similarity for
duplicate detection, which is what agents actually need (they phrase the same issue
differently, which is where keyword search is weakest). A larger commitment on a different
axis; also backfillable from retained text.

⚠ **The asymmetry that makes this roadmap the low-risk order:** deferring tantivy is cheap
and reversible; adopting it now is expensive and **hard to reverse, because of (b)** — the
query surface it tempts you to expose becomes a contract other vendors' agents code
against. The cost of deferring is an index you might later discard; phases 0 and 1 you need
either way.

## The resident's recommendation, 2026-09-04 — asked for directly

✅ **ACCEPTED by the owner, 2026-09-04** — settled as `redb` + `axum` + `tokio`, with the
fjall equivalent kept as the secondary consideration. The reasoning below is preserved as
the basis of that ruling; AGENTS.md item 2 is the ruling itself.
⚠⚠ **It rests on characterization — the acceptance bar is STILL UNMEASURED on both
candidates**, and the "what would change this recommendation" section below did not stop
being live when the recommendation was accepted.

### Recommended: **redb + axum + tokio** (56 crates, pure Rust, no C toolchain)

**1. Referee integrity is the product's core promise, and the two engines treat it
differently.** redb's ACID single-writer semantics are the default and cannot be opted out
of. fjall's are opt-in, and its own documentation says the non-transactional path cannot do
read-modify-write reliably — a **silent** hole rather than an error. ⚠ The standard
mitigation (one store-access module that only opens the transactional database) is cheap
and real, but "don't call the wrong API" is a weak guarantee in a codebase this project
intends **agents to contribute to**, and the failure mode is a referee that quietly is not
one.

**2. ⚠⚠ fjall's headline advantage deflates on inspection — compaction filters save the
SWEEPER, not the LOGIC.** Worked through per workload:
- **Registry liveness:** "is this agent alive?" must be answered from a heartbeat timestamp
  **at read time**. Compaction runs on its own schedule and cannot answer a liveness
  question — it can only reclaim space afterwards. The read-side logic is needed either way.
- **Message expiry:** whether a message is deliverable is likewise a read-time question.
  Expiry is space reclamation, not correctness.
- **Postings GC:** postings are removed transactionally on document update or delete
  anyway; the filter is cleanup behind that.

⇒ The saving is a **background sweep job**, not the semantics. And ✅ **FerroTrack is
building durable scheduling machinery regardless** (item 9, requirement 4) — so running
periodic internal sweeps on it is nearly free. The one clear functional win fjall had is
largely already paid for by the brief.

**3. Dependencies.** 56 versus 84 — **28 fewer crates, ~50% relative**, against an
explicitly stated "minimal dependencies" ideal (item 2, requirement 4).

**4. One file on disk.** Item 2's requirement 2 asks for something embeddable "or just a
project asset." redb is a single file, ~50 KiB minimum: trivial to copy, back up or ship.
fjall is a directory with ongoing compaction churn.

**5. Direct API fit for what (d) needs.** `MultimapTable` serves both the structured indexes
(project / author / assignee) and phase-2 postings lists without an encoding convention.

### What would change this recommendation

- ⚠ **redb's maintainer concentration is UNVERIFIED and is the single thing to check before
  committing.** If it is genuinely a bus-factor-of-one and that is disqualifying, it is a
  real counterweight and fjall is the fallback.
- A message bus with sustained high write volume would favour LSM — **not in view at the
  brief's scale**, but it is the condition that would flip it.
- The acceptance-bar measurement, once run, outranks everything above. In particular the
  conditional-update-inside-the-transaction test at the call site, with a control.

## On building a new product from a permissive base (latitude 3)

⚠ **The owner's meaning, corrected 2026-09-03: creating a NEW PRODUCT using an existing
permissively-licensed product as a base** — not maintaining a divergent copy of upstream.
Conditional: *"only if the arguments for it are compelling enough."*

✅ **Permitted.** redb, sled and fjall are all `MIT OR Apache-2.0`.

**What the choice actually turns on is who owns crash-safety**, and that is the same
question under either reading of the word:

- **Depend on it** — the engine stays someone else's problem, upstream fixes keep arriving,
  and `FerroStore` is still a real product: our name, our API, our features, over an
  internal dependency. The brand does not require the code.
- **Derive from it** — from that commit forward, on-disk format stability and crash-safety
  are ours, and upstream fixes arrive only if someone merges them.

⚠⚠ **The reason to weight that heavily: in a storage engine the failure mode is SILENT
DATA LOSS.** Not speculation — redb 4.0 fixed `AccessGuardMut` allowing data to be "written
out after the transaction had already completed," and 4.1 fixed `restore_savepoint()` being
unsound without immediate durability. Upstream author, own code, full context.

**A compelling argument would name a capability that cannot be built above the store's
API.** As of 2026-09-03 none has been named, and every feature in the brief lives above it.
The obvious candidate — a change-feed hook inside the write path — is already answered
above the API, because FerroTrack is the only writer and so observes its own writes.

⚠ Recorded as the trigger to test future proposals against, not as a refusal: the owner set
the bar at "compelling," and this note's job is to say what would clear it.

## Open items

1. ✅ **Resolved 2026-09-03** — the requirement-5 conflict was an artefact of a misreading,
   corrected by the owner. FerroTrack carries the features; standalone works.
2. **Full-text search — decomposed, because "decide search" is four decisions of very
   different widths.** ⚠ **A sequencing claim made earlier in this note is withdrawn:** it
   said search should be decided *before* the store because it dominates the dependency
   budget. Dominating the budget is true; **gating the store choice is not.** A hand-built
   index needs ordered range scans, which criterion 3 already requires of every candidate;
   tantivy keeps its own files and involves the store not at all. Either way the store
   choice is unconstrained. The two decisions are independent and search does not have to
   go first.

   **(a) Which engine — NARROW and reversible.** Substring/prefix find over our own table;
   a hand-built inverted index (postings lists suit redb's multimap tables; keeps the stack
   at 56 crates, pure Rust); or tantivy (+89 crates, and a C toolchain via `zstd-sys` in
   core `tantivy-columnar`). This is the part that was measured, and it is the part that
   is *cheapest to change later* — provided (b) is designed so the engine does not show
   through.

   **(b) The query surface exposed in the protocol — WIDE, and effectively one-way.**
   FerroTrack is inter-brand and speaks over the network (item 9), so its query surface is
   a public contract that other vendors' agents will code against. Widening it later is
   easy; **narrowing it is a breaking change for clients this repo does not control.**
   ⚠ The hazard is specific: exposing an engine's capabilities directly — ranked relevance,
   boolean operators, fuzzy matching — commits FerroTrack to keeping them when the engine
   changes. Design the surface to the *weakest* engine considered acceptable, not the one
   chosen first.

   **(c) Index consistency with the ledger — WIDE, and the most under-appreciated of the
   four.** An inverted index in our own tables commits **in the same transaction** as the
   issue write, so search results can never disagree with the ledger. ⚠⚠ **tantivy is a
   second store with its own commit cycle and its own durability**, which makes search
   eventually-consistent and adds a crash-recovery reconciliation path that does not
   otherwise exist. This matters more for agents than for people: an agent that files an
   issue and immediately searches for duplicates **can miss its own write**, and that is a
   correctness-shaped bug, not a latency annoyance. Switching between these two models
   later changes observable behaviour that callers may already depend on.

   **(d) What text is retained and indexed — WIDE, a data-model commitment. ✅ ANSWERED by
   the owner, 2026-09-03**, and it is the only one of the four that is settled.

   For issues: **title, description** (a body of text sitting between the title and the
   full body — so the text model has three tiers, not two), **and the full body**; plus
   **project, author and assignee**, with the owner's own observation that those last three
   *"may be better off as id lookups rather than actual textual descriptions."*

   ✅ **That instinct is right and worth making explicit, because it splits the feature in
   two.** Project, author and assignee are **structured fields, not text.** They want
   equality filtering against an id, not text matching — and folding them into a text index
   actively misbehaves: a search for a person's name would match both issues *assigned to*
   them and issues merely *mentioning* them, with no way to rank the difference. The model
   that falls out:
   - **Text index** over title / description / body — ⚠ and because the tiers exist,
     ranking should weight them (a title match outranks a body match). Field weighting is
     a real requirement, not a nicety.
   - **Structured index** over project / author / assignee — plus state, labels and scope,
     which the issue-list views need anyway. These are the ordinary index tables criterion
     3 already implies; they are not search.
   ⇒ ⚠⚠ **Most of what the brief asks for is not full-text search at all.** Only three
   fields are text, and they are short. This should be weighed before (a) is priced as if
   the whole tracker needed an engine.

   ✅ **And (d) de-risks the rest: retaining that text makes every later index
   BACKFILLABLE.** A keyword index, a BM25 engine, or vector embeddings can all be built
   from retained text after the fact. The only unrecoverable decision is text never stored —
   which this answer avoids. ⚠ Worth knowing while (a) is open, because it means (a) is not
   a one-way door as long as (d) holds.

   ⇒ **What actually deserves early attention is (b), (c) and (d) — not (a).** The library
   is contained behind an interface; the contract, the consistency model and the retained
   data are not.
3. **Unmeasured on every candidate:** the conditional-update-inside-the-transaction
   guarantee at the call site, with a control so a conflict-free pass cannot be mistaken
   for a vacuous one; and the second-process-opens-the-file failure mode.
4. **The registry's liveness problem** is unaddressed by any store and needs a
   heartbeat-or-lease design in the server.
