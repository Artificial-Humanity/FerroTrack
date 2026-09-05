# The ideal datastore, and the case for eventually building it

**Status: research, opened 2026-09-05 at the owner's direction.** The store ruling stands —
[`AGENTS.md`](../AGENTS.md) item 2 is the one copy of it — and this note does not reopen it.
It records *why a derivative is on the ladder at all*, *what we would actually want*, and
*where the two live candidates fall short*, so that if the question returns it is answered
from a record rather than from a mood.

⚠ **Attribution, because this file mixes two kinds of statement.** §1 and §5 are the
owner's, dated. §2, §3 and §4 are the resident's analysis, offered as such — if a line from
them is ever cited as a constraint, it has been promoted.

---

## 1. The ranking (owner, 2026-09-05)

1. **redb — we proceed with this.**
2. **fjall — backup.**
3. **Our own derivative work — third.**

⚠⚠ **What changed: a derivative is now ON THE LADDER, not a hypothetical.** It was
previously an authorised escape hatch (item 9, latitude 3) with no position. It is now
ranked, which means it is a direction the project may take rather than only a mitigation it
may be forced into.

## 2. Why building our own is worth considering — the owner's reasons, recorded

1. ⚠⚠ **The cost has moved, and the earlier "don't build a storage engine" advice was
   priced on the old cost.** The owner's point, twice: studying and writing this class of
   code has become materially more reachable in recent years. **The resident's advice
   against it did not examine that premise** — and cited evidence against itself without
   noticing: **redb's own 4.1 release credits a large batch of bug fixes to AI coding
   agents.** The same shift that lowers our cost is already demonstrably effective on this
   exact codebase.
2. **redb's origins predate generative AI** (owner). ⚠ Read precisely: this is not an
   indictment of its core, because durability, atomicity and ordering are workload-agnostic
   and a 2018 B-tree is not worse at them because agents arrived. It is a claim about its
   **feature surface**, which was chosen without agent workloads in view — and §4 is where
   that shows.
3. **Two high-quality reference implementations exist, covering both major designs** — redb
   (copy-on-write B+tree) and fjall/`lsm-tree` (LSM) — and **both are `MIT OR Apache-2.0`**,
   so studying and deriving is licence-clean and compatible with standing rule 2.
   ⚠ Contrast `persy`, characterized in [`store-criteria.md`](store-criteria.md): MPL-2.0,
   which would forfeit exactly this.
4. **Every viable candidate is single-maintainer** (measured 2026-09-04). Depending on one
   is not the conservative choice it appears to be, so a derivative is a **mitigation as
   much as an ambition**.
5. **We will already own a database's worth of machinery above the store** — indexes, query
   surface, search, referee semantics, history. The distance between "FerroTrack over redb"
   and "FerroStore" is shorter than it looks from here.

## 3. What we would want in an ideal datastore for *this* product

Derived from the four workloads and the brief. Marked with what the two candidates do.

| want | why this product needs it | redb | fjall |
|---|---|---|---|
| **CAS evaluated inside the transaction** | the referee's entire guarantee | structural (single serialized writer) | via opt-in tx only |
| Embedded, single-writer, **one file** | broadcast completeness; "just a project asset" | ✅ one file | directory + compaction churn |
| **Native secondary indexes** | project / author / assignee / state / labels | ✗ hand-built | ✗ hand-built |
| **Postings / inverted-index structure** | phase-2 keyword search | multimap tables (close) | composite keys (workable) |
| **Vector similarity** | duplicate detection — the highest-value agent search | ✗ | ✗ |
| **Queryable history** | acceptance-bar item 3; "what did this look like at step N" | ✗ (savepoints are internal) | ✗ (versioning is internal) |
| **TTL / expiry primitive** | message retention, dead registry entries | ✗ hand-written sweeps | ✅ compaction filters |
| Ordered range scans | scheduler due-time; per-recipient drain | ✅ | ✅ |
| Atomic writes across collections | record + event; message + receipt | ✅ same tx | ✅ cross-partition journal |
| Native durable change feed | notification completeness without relying on being sole writer | ✗ | ✗ |
| Small footprint, no C toolchain | item 2 requirement 4 | ✅ 1 crate | 41 crates |
| Format stability + migration path | a tracker outlives its engine version | ✅ shipped precedent | ✅ stated policy |
| Compression | text-heavy bodies and messages | ✗ | ✅ LZ4 default |

⚠⚠ **The three rows nobody fills are the interesting ones: vector similarity, queryable
history, and a native change feed.** Those — not raw storage — are where "predates
generative AI" becomes a concrete gap rather than an observation.

## 4. Where each falls short

**redb** — no secondary indexes, no TTL, no vector index, no queryable history, no change
feed, no compression; B+trees favour reads while the message bus is write-heavy; single
maintainer. ⚠ And its CAS is a **structural property you must use correctly**, not a named
primitive that enforces itself — safe, but it puts the burden at the call site, which is
precisely why the acceptance-bar probe needs a control.

**fjall** — transactions are **opt-in with a silent non-transactional path**; no secondary
indexes, no vector index, no queryable history, no change feed; 41 crates against 1; the
"100% safe Rust" claim covers the wrapper while `lsm-tree` carries 31 `unsafe` blocks;
background compaction and multiple files sit worse against "just a project asset"; API
revamped as recently as 3.0. ✅ Against that: compaction filters ≈ TTL, partitions give
cross-collection atomicity by construction, and LSM genuinely suits the write-heavy bus.

## 5. If the derivative is ever taken (owner: not unrealistic)

- ✅ **Licence-clean from either base**, and compatible with shipping Apache-2.0.
- ⚠⚠ **The line the resident would hold, offered as analysis not as a rule: inherit the
  crash-safety core, do not rewrite it.** That is what makes derivation different *in kind*
  from building from scratch — the layer where bugs are silent and surface months later
  under power loss is the layer you keep. Everything in §3's empty rows sits *above* that
  layer.
- ⚠ **A derivative forfeits upstream fixes** unless someone keeps merging — a standing cost,
  not a one-off.
- **The trigger already on the record** (item 9, latitude 3): a capability that cannot be
  built above the store's API, *named*. **The two live candidates are vector similarity and
  queryable history.** Both can currently be built above a key-value API, so neither has
  fired yet — the question is whether doing them below it buys enough.
- ⚠ **What would answer that is running the product.** Designing an engine now means
  designing for a workload that has never executed: no access patterns, no measured hot
  paths, and no evidence about whether the vector index belongs in the engine or above it.
- ✅ **Nothing is being given up by waiting.** Item 1's two-implementation floor already
  requires the store to sit behind an interface, so a later `FerroStore` slots in at the
  same seam. **That rule is what keeps this option free.**
