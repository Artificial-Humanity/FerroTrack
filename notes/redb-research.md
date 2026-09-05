# redb as FerroTrack's backing store — research

✅ **Status: SELECTED (owner, 2026-09-04) — redb is FerroTrack's backing store**, in the
stack `redb` + `axum` + `tokio`. ⚠ Since 2026-09-05 the store is a **ranked ladder of
three** — redb, then fjall, then our own derivative work; see
[`ideal-datastore.md`](ideal-datastore.md). [`AGENTS.md`](../AGENTS.md) item 2 is the one copy of that ruling, of what
the settlement rests on, and of what would reopen it; this note restates none of it and
loses to it on any disagreement.

⚠⚠ **Being selected does not make this note verified.** Nothing below has been checked
against a running redb, and **the acceptance bar is unmeasured** — the ruling was made on
characterization and says so. The "verify" markers are now work items rather than
suggestions, and two are named in AGENTS.md as owed: redb's maintainer concentration, and
the conditional update evaluated inside the transaction, tested at the call site with a
control.

This note has two jobs. It characterizes the redb *candidate*, and its structure — the
capability inventory, the obligations-that-move-to-FerroTrack list, and the acceptance-list
mapping — is **the template every other candidate is written against**; see
[`surrealdb-research.md`](surrealdb-research.md), the scratched candidate, kept as evidence
and readable side by side with this one.

⚠ **Two of its sections postdate the rest and are marked by date**: the fit assessment
against the owner's four requirements, and the cost section under it, were written
2026-09-03 after those requirements were stated and after the owner ruled that FerroTrack
provides the pub-sub model. Everything else is from 2026-08-28.

⚠ **A framing withdrawn 2026-08-28, kept visible because it was a real misread of an
instruction:** an earlier revision called redb "the owner's directed choice, not a
shortlist candidate." That read too much into the first instruction. The same over-read is
available today and would be easier to make — the ban on it is in AGENTS.md item 2.

Facts below were fetched 2026-08-28 from the project's README, docs.rs, and CHANGELOG —
sources at the bottom. Nothing here has been verified against a running redb yet; that is
the natural next step and each "verify" marker names what to probe.

## What redb is

A pure-Rust, embedded, ACID key-value store — LMDB-inspired, copy-on-write B+trees, no C
dependency. One file on disk. MVCC with **one writer and concurrent non-blocking readers**;
crash-safe by default; typed tables (`TableDefinition<K, V>`, `Key`/`Value` traits, a
`redb-derive` crate); multimap tables; savepoints with rollback; zero-copy reads via
`AccessGuard`. A stated file-format stability commitment: "a reasonable effort will be made
to provide an upgrade path" — and the v2→v3 migration actually shipped one
(`Database::upgrade()` in 2.6).

## Version and maintenance

| | |
|---|---|
| Current release | **4.2.0, 2026-08-17** — eleven days old at time of writing |
| Recent majors | 3.0.0 (2025-08-09, v3 file format, ~50KiB min db size), 4.0.0 (2026-04-02), 4.1.0 (2026-04-19) |
| In development | 4.3.0 (B-tree separator optimization, integrity-repair fixes) and a 5.0.0 behind an experimental flag (`no_std`, `KeyRange` API) |
| Health | actively maintained; the 4.1 release notes credit a large batch of bug fixes to AI coding agents |

Two recent fixes worth respecting because they name real hazards:

- **4.0.0**: `AccessGuardMut` had to gain `Drop` ordering because data in an accessor could
  be "written out after the transaction had already completed" — data loss. Consequence:
  stay current, and treat guard lifetimes as load-bearing.
- **4.1.0**: `restore_savepoint()` in a non-`Immediate`-durability transaction was unsound
  and now requires immediate durability. Consequence: if we use savepoints at all, the
  transactions that restore them run `Durability::Immediate`.

**Recommendation: pin the 4.2 line.** 5.0 churns the range/get API surface; nothing we need
is behind its flag.

## What we get, mapped to what an agent tracker needs

The capability questions come from FerroStep's store-adapter work — the ledger contract asks
a store what it can honestly promise. Against a hosted BaaS reached over REST (the
PocketBase deployment FerroStep currently rides), several answers were bad in measured,
public ways: the update verb takes no version predicate; rule-level predicates evaluate
outside the write's serialization window so racing writers both pass; administrators bypass
rules entirely (though not hooks); compare-and-swap is only honest when the compare runs
*inside the transaction*. An embedded store dissolves most of this class:

1. **CAS inside the transaction — native.** redb has a single serialized writer; a
   `WriteTransaction` that reads the version, compares, and writes IS the
   compare-inside-the-transaction shape. No hook machinery required to get it. ⚠ The
   capability flag still gets set from a real-threads measurement, not from this
   paragraph: what the measurement catches is not a broken store but a correct store used
   through a wrong call site — a path that reads the version *before* opening the write
   transaction reintroduces the stale-compare window on top of a perfectly serialized
   backend. The flag describes the call site, and the test needs a control (predicate
   replaced by always-true) so a conflict-free pass can be told apart from a vacuous one.
2. **One write path — structural.** Embedded means the FerroTrack process is the only door
   to the file. The rules-bind-users-but-not-superusers split has no analog: there is no
   side channel for a privileged credential to write around the referee. (Verify the
   corollary: a second process opening the same file must fail loudly — redb takes a file
   lock; probe the actual failure mode.)
3. **Two-record atomicity — same transaction.** Record + history/event row commit together.
   The disabled-batch-endpoint problem class disappears.
4. **Store-computed version token — native pattern.** The tracker computes the next version
   inside the write transaction and ignores caller-submitted values; a monotonic handler
   that cannot express a reset stays the shape (a reset is a distinct authorized operation
   at the API layer — that obligation is above the store and survives the store swap).
5. **Complete enumeration — snapshot cursors.** A read transaction iterates a consistent
   MVCC snapshot; there is no paginated REST view whose truncation flag defaults the count
   away. Truncation-by-design (limits we add) becomes our API's explicit choice to report.
6. **Typed schema at compile time.** `TableDefinition<K, V>` + derive moves the
   field-mapping risk (which field holds state/counters/version) from runtime config toward
   types the referee can actually see.

## What redb does NOT provide — obligations that move into FerroTrack

- **A server.** redb is in-process. The network surface (REST/whatever agents speak, auth,
  roles) is entirely ours. That is the point — the API layer is where the referee sits —
  but it means every guarantee above is only as good as the process that owns the file.
- **Secondary indexes.** Key-ordered B+tree tables only. Indexes (by state, by repo, by
  branch) are extra tables we maintain — transactionally, in the same write, which is
  strictly better than an external indexer, but it is code we own and must test.
- **Queries.** No filter language, no full-text search. Scans over snapshot cursors, or
  index tables we design.
- **Change notification.** No watch/subscribe. A realtime lane is an application-layer
  broadcast we build beside the write path. ✅ **As of the owner's 2026-09-03 ruling that
  FerroTrack provides the pub-sub model, this is the plan rather than a gap** — and see
  the note below on why redb's single-writer property makes that broadcast *sound* rather
  than best-effort.
- **Multi-process / HA.** One writer process. `ReadOnlyDatabase` (3.0+) allows other
  processes read-only access — potentially useful for a backup or inspection tool — but
  writes are ours alone. Backup strategy to design: savepoints, compaction
  (`Database::compact`), or copy-on-snapshot; verify what the crate actually blesses.
- **Migrations.** Table contents are our bytes; schema evolution discipline is ours. The
  file format itself has an upgrade story; our value encodings need one too (worth deciding
  before the first persisted byte — serde format choice is a compatibility commitment).

## Fit against the owner's four requirements (asked 2026-09-03)

The requirements are listed in [`AGENTS.md`](../AGENTS.md) item 2 and not restated here.
Scored honestly, redb is the best fit measured so far on three of them, and the fourth is
where its whole cost sits.

**1. The SQLite-to-PocketBase relation — structurally exact.** This is the relation redb
is built for: an embedded engine with FerroTrack as the only process that opens it. What
redb does *not* bring is the part PocketBase gets free from SQLite — a query language and
indexes. See the cost section below; that is the trade, and it is the whole trade.

**2. Light install — the best possible score, measured.** One crate, **zero dependencies**,
pure Rust, no C toolchain, one file on disk, ~50 KiB minimum database size. Nothing to
install, nothing to provision, no external process. Against item 8's bar — an installer
that must not require broad filesystem grants — a store that is one file the product
creates is the easiest possible thing to install.

**3. Pub-sub — and redb's structure makes FerroTrack's own broadcast SOUND, not merely
possible.** This is the non-obvious part and it is why the two rulings of 2026-09-03 fit
together rather than merely coexist.

A notification layer built above a store is only complete if **every** write passes through
the code that broadcasts. That is an assumption, and against a store that permits other
writers it is a false one: a server-mode database with its own auth, an administrative
credential, a REST path, a second process — each is a write that happens without your
broadcast running, and the subscriber never learns. The bug is silent and shows up as a
stale client.

redb removes the assumption instead of documenting it. The FerroTrack process is the only
door to the file, so "notify beside the write" and "notify on every write" are the same
statement. ⚠ **VERIFY the corollary** (already on this note's list): a second process
opening the same file must fail loudly — redb takes a file lock, but probe the actual
failure mode rather than trusting it.

⚠⚠ **The same property is what the referee's guarantee rests on** (see item 2 of the
capability mapping above: one write path, structural). So the store gives the referee and
the notification layer their soundness from one fact, not two. That is a real argument for
an embedded store over a server-mode one *given* the pub-sub ruling — and it would have
been an argument against the ruling's alternative, had it gone the other way.

**4. Rust-built with minimal dependencies — the measured floor.** 1 crate, zero
dependencies. There is no lighter answer available; this is the bottom of the table.

## ⚠⚠ The cost, stated plainly: FerroTrack becomes a small database

Every requirement above is met by redb *not doing things*. The bill arrives here, and it
should be read before redb is chosen, not after:

- **Secondary indexes are ours.** By state, by assignee, by label, by scope — each is a
  table FerroTrack maintains transactionally in the same write. Better than an external
  indexer, and it is code we own, test and keep consistent.
- **Query and filter behaviour is ours.** No query language. Scans over snapshot cursors,
  or index tables we design.
- ⚠ **Full-text search does not exist.** An issue tracker is expected to have search, and
  this is the largest product-surface gap on the list — larger than it looks next to the
  others, because it is a *user-visible feature* rather than an internal mechanism.
  Decide deliberately whether FerroTrack ships search, ships a weaker find, or takes a
  dependency for it — the last would spend some of the dependency budget redb just won.
- **Schema evolution is ours**, and ⚠ **the serialisation format is a compatibility
  commitment from the first persisted byte** — worth deciding before that byte exists.
- **One writer, one process.** `ReadOnlyDatabase` (3.0+) lets other processes read. This
  constrains the deployment shape: no second writer, no horizontal scale. For a tracker
  serving agent workflows that is likely fine — but it is a *product* property, not just
  an implementation detail, and adopters will meet it.

⚠ **AGENTS.md item 1's hazard inverts here, and the rule's wording was tuned for the other
case.** Against an opinionated store the danger is bending the ledger contract toward the
store's semantics. Against a minimal one the danger runs the other way: FerroTrack
accumulates database machinery, and that machinery leaks into the contract. Same rule,
opposite direction — worth naming because a reader checking item 1 against redb will find
its example pointing the wrong way.

✅ **The two-implementation floor gets easier, not harder.** A minimal store means
FerroTrack implements the ledger contract explicitly rather than mapping it onto another
database's opinions — which makes it more likely the trait stays honest about what a
second implementation must provide.

## The acceptance list this store must meet

The deployment brief ([`ferrostep-deployment-brief.md`](ferrostep-deployment-brief.md))
states the referee's six-item acceptance list for any backend. Mapping it here: items 1, 2
and 6 (conditional update inside the transaction, multi-write atomicity, non-truncating
enumeration) land on redb's transaction and snapshot properties above — subject to the
call-site measurement, not assumed. Item 3 (append-only history, *and against whom*) and
item 4 (what self-description costs when you own the store — the three-valued answer may
collapse to two; if so, say why in the type) are on the measurement list. Item 5 (bind an
existing authenticated identity carrying a readable role — bind, don't mint) is an
API-layer obligation redb neither helps nor hinders.

## The sibling brief

FerroStep's resident briefed FerroTrack on 2026-08-28 — the contract surface, the
retired-vs-surviving hazard split, and the decided-don't-re-propose list now live in
[`ferrostep-contract-fit.md`](ferrostep-contract-fit.md). Two corrections from that brief
matter here: the ledger contract already has **two working implementations** (SQLite, and
PocketBase against a live store), so FerroTrack would be a third with two references to
read; and **intentions**, initially open, were **answered by the owner on 2026-08-28:
both** — default ledger *and* standalone product, superseding both existing adapters as
the default while neither is retired (see `ferrostep-contract-fit.md`).

## Sources

- https://github.com/cberner/redb — README claims (ACID, MVCC, crash-safe, format stability)
- https://docs.rs/redb/latest/redb/ — API surface (transactions, savepoints, guards, tables)
- https://github.com/cberner/redb/blob/master/CHANGELOG.md — versions, dates, the 4.0/4.1 hazards
- https://www.phoronix.com/news/Redb-4.1-Released — 4.1 coverage
