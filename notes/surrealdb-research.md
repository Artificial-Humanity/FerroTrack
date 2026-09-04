# SurrealDB as FerroTrack's backing store — research

**Status: SCRATCHED (owner, 2026-09-03) — kept as evidence, not deleted.** Named front
runner that morning and scratched the same afternoon on the measurement below. This note
is retained deliberately: a candidate ruled out with its reasoning intact is what stops
the question being reopened from memory a month later. ⚠ **It was scratched on dependency
weight and an unavoidable C toolchain — NOT on its licence**, which the owner ruled
adequate the same day. What would reopen it is recorded in
[`AGENTS.md`](../AGENTS.md) item 2, which is the one copy of the ruling.
⚠ Nothing was promoted in its place at the time. **The store was settled on 2026-09-04 —
redb, with fjall secondary** — which does not revive this candidate; see AGENTS.md item 2.
[`AGENTS.md`](../AGENTS.md) item 2 is the one copy of that ruling and of the requirements
this note measures against — it is not restated here, and if the two disagree, AGENTS.md
wins.

Facts below were fetched 2026-09-03 from the project's own documentation, repository and
blog — sources at the bottom — **except the dependency-weight section, which is measured
locally on that date and is the first real measurement in this evaluation.** ⚠ Nothing
here has been verified against a *running* SurrealDB, and no binary has been built or
sized. Each "verify" marker names what remains to probe. This note follows the structure of [`redb-research.md`](redb-research.md), which
is the evaluation template, so the two candidates can be read side by side.

## What SurrealDB is

A multi-model database written in Rust — document, graph and relational access over one
store, with its own query language (SurrealQL), access control, vector index and a sync
protocol. It ships as a single binary and is designed to run **either** embedded
in-process as a Rust library **or** as a standalone server, including a distributed
cluster.

That dual mode is the first thing to notice, because it means "adopt SurrealDB" is not one
decision but two, and they have different answers on almost every line below.

**Licence: Business Source License 1.1 for the core** — converting to Apache-2.0 four
years per release (3.0 on 2030-01-01); SDKs and client libraries are Apache-2.0/MIT.
⚠ **Ruled adequate by the owner, 2026-09-03.** Recorded here only so the constraint it
still carries stays visible: BSL bars offering the database as a managed service without
an agreement, which is a limit on what an *adopter* of FerroTrack may do, not just on us.

## The storage engine choice — where "light install" is decided

SurrealDB does not have *a* storage engine; it has several, and the requirement for a
light, embeddable install lands entirely on which one is chosen.

- **RocksDB** — the default for single-node production. ⚠ **Written in C++, not Rust.**
  It is reported "very resource intensive to compile," takes a long time, and needs
  external toolchain dependencies on the development machine — awkward on Windows in
  particular. This is the engine SurrealDB blesses for production and it is the one most
  directly against a light install.
- **SurrealKV** — SurrealDB's own **pure-Rust**, embedded, ACID, versioned LSM engine,
  shipped in SurrealDB since 2.0 and aimed explicitly at "embedded and local-first
  in-process workloads where smaller resident memory and simple operational behaviour
  matter most." That is our shape almost exactly. ⚠ **It is beta** — the documentation
  says it "may require additional development and testing before it is ready for
  production use."
- **SurrealMX** — pure Rust, the default in-memory backend since 3.0, production-ready.
  ⚠ Not a persistence answer: all data must fit in RAM, and its snapshot/AOL persistence
  is documented as "a convenience to allow in-memory data to persist, but not a true
  primary solution for long-term storage."
- **TiKV** — distributed, non-Rust dependencies, out of scope for an embedded install.

⚠⚠ **The trilemma, stated plainly: pure-Rust, persistent, and production-blessed are not
currently available at the same time.** ⚠ And "pure-Rust" here describes **the storage
engine only** — the measured section below shows the surrounding crate still requires a C
toolchain regardless of which engine is chosen, so SurrealKV does not buy a C-free build. RocksDB gives up pure-Rust and light install;
SurrealKV gives up the production blessing; SurrealMX gives up persistence. This is the
single most important finding in this note and it is a *timing* fact — SurrealKV's beta
status is the kind of thing that changes, so **re-check it before the decision, do not
carry this paragraph forward as permanent.**

**Binary size — one datapoint, deliberately not treated as a measurement.** A published
example reports a 312 MB final binary, but that build enabled vector-database features we
would not. ⚠ **VERIFY: build a minimal-feature embedded SurrealDB against SurrealKV and
measure the artifact.** Until that exists, neither "SurrealDB is too heavy to embed" nor
"SurrealDB embeds lightly" is a claim this repo can make, and both have been asserted
casually elsewhere.

## Dependency weight — MEASURED 2026-09-03

Added by the owner as a fourth requirement that day: **a Rust-built database with minimal
dependencies is ideal.** "Rust-built" and "minimal dependencies" are two separate tests,
and SurrealDB passes the first easily. The second was measured rather than estimated —
`cargo add` into throwaway crates, then `cargo tree` over the resolved graph. No build was
run, so this measures *what must be compiled*, not how big the artifact is.

| candidate | version | crates (normal) | +build | native toolchain |
|---|---|---:|---:|---|
| redb | 4.2.0 | **1** | 1 | none |
| sled | 0.34.7 | 16 | 16 | none |
| fjall | 3.1.10 | 41 | 41 | none |
| rusqlite (bundled) | 0.40.2 | 9 | 14 | `cc`, `pkg-config` |
| libsql | 0.9.30 | 136 | 164 | `bindgen`, `cc`, `cmake` |
| **surrealdb** (`default-features = false`, `kv-surrealkv`) | 3.2.4 | **305** | 318 | `cc`, `cmake`, `pkg-config` |

redb's tree is literally one line: it has **zero** dependencies. SurrealDB, configured as
minimally as its feature flags allow and on its pure-Rust storage engine, resolves 305.

⚠⚠ **The C toolchain is NOT escapable by choosing SurrealKV, and this is the finding that
matters most.** Picking the pure-Rust storage engine to avoid RocksDB's C++ does not
produce a C-free build. The dependency enters through the **auth** path, not the storage
path:

```
aws-lc-sys (C, needs cc + cmake)  →  aws-lc-rs  →  jsonwebtoken  →  surrealdb-core  →  surrealdb
```

And it is **hard-coded, not feature-gated**. `surrealdb-core` selects the JWT crypto
backend by target family:

- non-wasm targets → `jsonwebtoken` with `aws_lc_rs` (C)
- wasm targets → `jsonwebtoken` with `rust_crypto` (pure Rust: ed25519-dalek, hmac, p256,
  p384, rand, rsa, sha2)

Because it is a `[target.'cfg(…)']` dependency rather than a feature, **a consumer on
Linux, macOS or Windows cannot opt into the pure-Rust path through feature flags at all.**
✅ **But the pure-Rust path demonstrably exists and works — SurrealDB ships it for wasm.**
So this is a plausible upstream request or a patch, not a fundamental limitation. Worth
knowing before it is filed as a hard blocker.

### ⚠ Read the 305-vs-1 fairly

The comparison is real but **not like-for-like, and quoting it without this paragraph
would be misleading.** redb is a key-value store; SurrealDB is a full multi-model database
with a query language, secondary indexes, access control and a server. Much of those 305
crates is machinery FerroTrack would otherwise pull in itself — a query layer, auth, HTTP,
serialisation. The honest comparison is *total system dependencies once FerroTrack is
built on each*, and nobody has that number for either candidate.

Two things survive that caveat, and they are what the measurement actually establishes:

1. **The C-toolchain requirement is binary, not a matter of scope.** It does not shrink
   when you account for what FerroTrack would have written anyway.
2. **Inherited versus chosen.** On a minimal store, every dependency FerroTrack adds is a
   decision someone makes and can decline. On SurrealDB, 305 arrive as a set. That is a
   difference in governance over the tree, not only in its size — and it is the one that
   bears on a product owing adopters upgrade guarantees.

⚠ `sled` at 0.34.7 appears in the table for measurement completeness only; **its
maintenance status is unchecked** and the version number is old enough that it should be
before anyone reads its 16 as attractive.

## What we get, mapped to what an agent tracker needs

1. **Change notification — native, and this is the standout.** `LIVE SELECT` keeps a
   session tracking changes to a table in real time; in the Rust SDK it is `.live()`
   appended to a `.select()`, and it returns a **stream of notifications** rather than a
   single result or a vector. It can be scoped to one record, to a range via `.range()`,
   or to a whole table.
   ✅ **Live queries are supported on the local/embedded engines** — the WebSocket engine
   too; the HTTP engine is the one that does not support them. Since our shape is
   embedded, this lands on the supported side.
   ⚠ **VERIFY: filtering.** Live queries have had open feature requests for arbitrary
   filters in the Rust SDK, so predicates beyond range restriction may be limited. Probe
   against the exact version pinned, not against the blog posts.
2. **Queries, secondary indexes, full-text and vector search — provided.** In the redb
   note these are all obligations that move into FerroTrack; here they arrive with the
   dependency.
3. **Access control — provided**, including record-level permissions. Relevant to the
   FerroStep contract's "bind an existing authenticated identity carrying a readable role"
   item, which redb neither helps nor hinders.
4. **A server — provided**, if we want it, which changes the shape of FerroTrack rather
   than just its store. See the open question below.
5. **ACID transactions.** ⚠ **VERIFY, and treat this as the load-bearing measurement:**
   the FerroStep contract needs a *conditional update evaluated inside the transaction*
   (compare-and-swap on the version token, with the compare on the inside). The referee's
   whole guarantee rests on it. Confirm at the call site in SurrealQL against the pinned
   version — this is exactly the item my memory of every other store says gets assumed and
   should not be.

## What it does NOT provide — obligations that move into FerroTrack

Notably fewer than redb, and that is not purely good news.

- **Inter-agent routing.** A live query reports *that a row changed*. It does not address a
  message to a named agent, does not define delivery semantics, and has no answer for a
  recipient that is not currently connected. See the open question below.
- **The referee's semantics.** FerroStep's state machine, counters and scope guards remain
  FerroTrack's to implement regardless of engine.
- ⚠⚠ **Control over its own behaviour.** The inverse of items 2–4 above: the more of the
  product's behaviour lives inside a dependency, the less of it FerroTrack decides. Two
  consequences worth naming before anyone gets attached:
  - **AGENTS.md item 1 gets harder to honour, not easier.** The rule is that when the
    contract is awkward for whichever store FerroTrack lands on, the move is never "bend
    the trait toward the native store." An opinionated store makes that pull stronger,
    because bending the trait will genuinely be the shorter path more often.
  - **Upgrade and breakage surface.** FerroTrack owes adopters upgrade guarantees as a
    product; a store that brings a query language, an auth model and a wire protocol
    brings all of their version churn into that promise.

## ⚠⚠ The open question this research turns on

✅ **CLOSED by the owner, 2026-09-03: FERROTRACK provides the pub-sub model, not the
store.** The reasoning that led there is left standing below because it is the argument
the ruling rests on, but the question itself is settled — see AGENTS.md item 2. The
practical effect: a store's lack of watch/subscribe stopped being a defect against it, and
SurrealDB's live queries — its single strongest card here — stopped being decisive.

The owner's own analogy is the strongest argument for the second reading. The stated shape
is "similar to SQLite in relation to PocketBase" — and **SQLite has no pub-sub whatsoever.**
PocketBase's realtime subscriptions are implemented by PocketBase, in the server layer,
over a store that knows nothing about them. If FerroTrack occupies PocketBase's seat, then
pub-sub is something FerroTrack can *own* rather than *inherit*.

Which reading holds decides how wide the candidate set is:

- **Engine must provide it** → SurrealDB is far ahead of every other name raised, and the
  set narrows sharply.
- **FerroTrack provides it** → the engine needs only a reliable change feed or write hook,
  the lighter candidates (redb, libSQL, SQLite, fjall) come back into play, and the
  decision returns to being about install weight and transaction semantics — where
  SurrealDB is weakest.

There is a further argument that FerroTrack ends up owning routing under either reading:
addressing, delivery semantics and offline recipients are not things a change feed
answers, so engine pub-sub would sit *underneath* a routing layer we write, rather than
being the feature. ⚠ Flagged as an observation, **not** a recommendation — it is stated
here so the choice is made deliberately rather than by default.

⚠⚠ **Boundary.** The inter-agent messaging system is the owner's design space and it spans
more than this repo. This note records the *store* consequence of the requirement and
deliberately stops there: it does not design the messaging system, name its addressing
scheme, or assume which component hosts it.

## The acceptance list this store must meet

The deployment brief ([`ferrostep-deployment-brief.md`](ferrostep-deployment-brief.md))
states the referee's six-item acceptance list, and
[`ferrostep-contract-fit.md`](ferrostep-contract-fit.md) carries the store-neutral bar.
Against SurrealDB, every item is currently **unmeasured** — this note is characterization,
not evaluation. The two that should be probed first, because they are the ones a
feature list will appear to answer and not actually answer:

- **Item 1, conditional update inside the transaction.** A store having "ACID
  transactions" is not the same claim. Measure at the call site.
- **Item 3, append-only history and against whom.** SurrealDB's versioning and SurrealKV's
  temporal queries look like they help; whether they are append-only *against a caller
  holding write access* is a different question and the one that matters.

## Sources

- https://github.com/surrealdb/surrealdb — repository, dual embedded/server design
- https://github.com/surrealdb/license — licensing (BSL 1.1, Apache-2.0 conversion)
- https://surrealdb.com/docs/surrealkv — SurrealKV, pure Rust, beta status
- https://github.com/surrealdb/surrealkv — SurrealKV: embedded, ACID, versioned
- https://surrealdb.com/docs/sdk/rust/concepts/live — live queries in the Rust SDK
- https://surrealdb.com/blog/live-queries-in-rust — live query engine support
- https://surrealdb.com/docs/reference/rust/embedding — embedding, RocksDB build cost
- https://surrealdb.com/blog/the-power-of-surrealdb-embedded — embedded mode
- https://surrealdb.com/learn/fundamentals/performance/deployment-storage — engine comparison
