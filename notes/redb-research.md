# redb as FerroTrack's backing store — research

**Status: candidate research — the store is UNDECIDED, and that is the whole of the
ruling** (owner, 2026-08-28: "I don't expect you to fully architect the solution on the
spot. I just want research for now"; and, same day, that redb is *hypothetical* — LibSQL,
SurrealDB, or a first-party store were named as possibilities, **explicitly not
exhaustively**: the candidate set is open, not narrowed to the names mentioned). An earlier revision of this paragraph called redb "the owner's
directed choice, not a shortlist candidate"; that read too much into the first instruction
and is withdrawn. This note now characterizes the redb *candidate*, and its structure — the
capability inventory, the obligations-that-move-to-FerroTrack list, and the acceptance-list
mapping — is the template the other candidates should be evaluated against.

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
- **Change notification.** No watch/subscribe. A realtime lane (SSE or similar, if agents
  need it) is an application-layer broadcast we build beside the write path.
- **Multi-process / HA.** One writer process. `ReadOnlyDatabase` (3.0+) allows other
  processes read-only access — potentially useful for a backup or inspection tool — but
  writes are ours alone. Backup strategy to design: savepoints, compaction
  (`Database::compact`), or copy-on-snapshot; verify what the crate actually blesses.
- **Migrations.** Table contents are our bytes; schema evolution discipline is ours. The
  file format itself has an upgrade story; our value encodings need one too (worth deciding
  before the first persisted byte — serde format choice is a compatibility commitment).

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
