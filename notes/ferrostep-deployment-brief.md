# How FerroStep deploys today — the PocketBase deployment brief

**Source: FerroStep's resident, 2026-08-28, owner-directed.** Everything here is
FerroStep's published design, readable in the public `ferrostep-pocketbase` source, and was
cleared by its author for quoting in this repo. It deliberately contains nothing about any
particular deployment — no hosts, instance names, collection names, or accounts. This note
is the deployment-side companion to [`ferrostep-contract-fit.md`](ferrostep-contract-fit.md)
(the trait side) and the requirements bar for
[`redb-research.md`](redb-research.md) (the store side).

## What "using PocketBase" actually consists of

A **stock instance plus generated artifacts — install, not fork.** Five things:

1. A generated **migration** — adds the compare-and-swap column, creates the events
   collection.
2. A generated **hooks file** (JavaScript, dropped in `pb_hooks`) carrying the routes.
3. Optionally, a **refereed-columns guard** closing chosen columns to direct writes.
4. Optionally, a **role binding** naming which auth collection holds actors and which field
   carries their role.
5. Optionally, a **release hook**: a human writing a decision field performs a state
   release, with version and event.

Routes are collection-scoped (`/api/ferrostep/<records>/…`) so one instance carries several
refereed collections without colliding. Four of them: **ping** (anonymous; states what the
*installed* file can write), **schema** (authenticated; reads the collection live at request
time), **apply** (authenticated, transactional), **create** (generic shape only).

## Two deployment shapes, one wire contract

- **GENERIC** — collections the migration makes; counters and scope live as JSON blobs on
  the row. Clean; nobody's existing data is in it.
- **MAPPED** — an existing collection the deployment already lives in becomes the refereed
  record. A `CollectionMap` names which of its own columns hold state, version token,
  counters, scope labels, graded attributes. In mapped shape, filing stays with the
  collection's own procedure, so `create` is **refused by name** rather than half-supported.

⚠⚠ **The design property to carry into FerroTrack: the store's console stays the human view
of the same rows.** One record, one truth, no second chronology beside the first. A referee
that makes people keep a shadow copy has already lost.

⚠ A detail that saved a backfill: a version column's default of `0` on rows predating the
mapping **is a valid starting token** — adopting the referee over live data required no
migration of existing rows.

## The data model as it actually sits

One row per record: state (typically a select), version (integer), counters (integer
columns, one per name), scope labels (text columns), graded attributes (columns, typically
selects). Beside it an **events** collection, append-only, one row per decision: record,
actor, role, from_state, decision (JSON — the engine's whole Decision, verbatim), note.

⚠⚠ **That list is the whole prose surface.** `note` is the only free text FerroStep
persists. **No comments. No threads, replies, mentions, attachments, or relations between
records.** For a ledger that is a defensible boundary — the deployment's own collection
carries whatever else it wants. For a *tracker* it is most of the product, and it is the
largest single thing FerroStep will not tell FerroTrack how to design.

## What the referee needs from a store — the acceptance list for any backend

1. **Conditional update** — compare *inside* the transaction, beside the write.
2. **Multi-write atomicity** — row change and event append land together or not at all.
3. **History genuinely append-only** for non-administrators.
4. **Self-description** — which columns exist, which values each accepts, which the write
   path can reach; each answer three-valued.
5. **An identity the store authenticates, carrying a role the referee can read.** Bind,
   don't mint: it names an auth collection the deployment already has and never creates
   identities — the actors are not knowable when the loop is designed.
6. **Enumeration that does not silently truncate.**

## Roles, escalation, and the two things not modelled

**Roles:** a string on the actor's auth record; the check lives in a **hook, not an access
rule** (deliberate — see pain point 2). One flag governs whether a principal carrying no
role may act; it **defaults permissive** because a fresh install has no actors yet and a
strict default would refuse every write the moment the hooks landed. Flipping it once
actors exist is what makes an administrator's credentials insufficient to move a record.

**Escalation:** not an error type. A counter carries a max and an on-exhausted state, and
spending the last one **routes** the record there instead of failing. Separately, states
can be marked **halted** — a pause blocking the whole loop until a person acts; the release
hook is the human's way out.

**Not modelled — both FerroTrack product frontiers:**
- **Comments** (above).
- **Dissent.** A decision is allow, deny, or exhausted. There is no "I disagree and here is
  why" that is not a `note` on something else — the structural cause behind the
  zero-disputes measurement in `ferrostep-contract-fit.md`.

## Where PocketBase fights the referee — measured, not felt

1. **No conditional update in REST.** The obvious workaround — compare in a rule or
   request hook, then let the ordinary write proceed — measured *failing* under two
   concurrent writers **and intermittently passing, which is worse.** Hence a custom route
   whose compare runs inside the store's own transaction — the only write path shipped;
   REST-only was rejected rather than offered as a lesser mode.
2. **Rules bind ordinary callers; hooks bind administrators.** An access rule is not a
   control against a superuser. This is why the role check is a hook.
3. **Hook callbacks run in isolated runtimes.** File-scope helpers are invisible inside
   them; every handler must be self-contained. The duplication in the generated file is
   load-bearing, not tidiness waiting to happen.
4. **The generated file outlives the binary.** Installed once, met by newer adapters for
   years — the origin of the capability-signal obligations in `ferrostep-contract-fit.md`.
5. **One branch per declared column.** The route emits an `if` per name the map declares —
   the map is an allowlist by construction, and an *undeclared* column has no branch at
   all, not a rejecting one. Silent drop answering 200.
6. **Default page size 30, silently truncated.** Every enumeration asks for the 500 cap,
   never skips the total count, and verifies it read as many as the store said existed.
7. **Emitted JSON key order is not stable.** Nothing may assert on serialized text.
8. **The admin API 403s an ordinary actor.** Reading a collection's own schema needs
   administrator rights — so self-description became a generated, authenticated route of
   FerroStep's own (measured: ordinary token 403 from the admin path, 200 from theirs).

## What to collect next (research framing, per the brief)

- The **real-threads CAS battery with the always-true control** — shape available from
  FerroStep's resident on request.
- Whether an embedded store can honestly claim **append-only history, and against whom**.
  The existing flag's own definition scopes a store administrator out; FerroTrack may be
  able to do better — **measure rather than assert**, and if so it is a genuine advantage.
- What **self-description costs when you own the store** — the three-valued answer may
  collapse to two. If it does, **say why in the type rather than deleting the variant.**
- The **comment and dissent models** — product research, not storage research, and where
  the time should go.
