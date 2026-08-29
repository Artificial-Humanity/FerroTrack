# The FerroStep ledger contract, from FerroTrack's side — research

**Status: research.** Source: a briefing from FerroStep's resident agent (2026-08-28),
corroborated against FerroStep's public source where marked. Adopters of FerroStep are
described by shape, never named — that discipline applies to everything in this repo.

## The contract FerroTrack would implement

FerroStep keeps its core pure (no IO, no async, no clock, no store, no network) and puts
everything store-shaped behind a `Ledger` trait — verbs: `capabilities`, `load`, `create`,
`apply`, `select`, `history`, `store_shape` (corroborated at
`ferrostep-ledger/src/lib.rs`, `pub trait Ledger`).

**Two implementations exist and are exercised today — SQLite, and PocketBase against a
live store.** FerroTrack would be the *third*, not the first. That changes the research
posture: read what the existing two had to do, and more usefully, read where they
**refuse**.

The parts that matter more than the verbs:

- **`apply(record, event) → Version`** lands the state change, the counter changes, and
  the event append **together or not at all**. Compare-and-swap on a version token is the
  mechanism; everything else follows from that being real.
- **`capabilities`** is where a store states what it actually guarantees. The standing
  warning from the existing adapters: **set the CAS flag from a measurement under
  concurrent writers, never from reading your own API.** A store can evaluate a version
  predicate with perfectly correct semantics and no serialization behind it — a writer
  arriving between check and commit passes a predicate that is already stale. Correct
  mechanism, absent atomicity: different properties, and only the second licenses the
  flag. Obligation: a real-threads test that *fails* when the compare is outside the
  transaction.
- **`store_shape`** asks the store to describe itself: which columns exist, which values
  each accepts, which the write path can reach. Every field is a **three-valued** answer —
  *stated*, *stated there is nothing to constrain*, or *nobody asked* — because the last
  two collapse together under an `Option`, and only the first is a verified all-clear.

## What an embedded native store retires — and what survives it

Retired by owning the store in-process (see [`redb-research.md`](redb-research.md)):
the CAS-outside-the-transaction hazard; the split where rules bind ordinary callers and
hooks bind administrators; the acute form of the installed-generated-file problem.

**Not retired — these are FerroTrack design obligations, whatever the backend:**

1. **A write path derived from a declaration has NO branch where the declaration is
   silent** — not a rejecting branch, none — so the request is accepted, the field
   dropped, and success returned, with an event recording an update that never happened.
   Rule: **refuse the rest, never ignore it.**
2. **A capability signal is worthless unless something reads it.** Design test: does
   adding a case to the emitter *force* a case in the consumer? If not, the two sides
   will ship separately and the gap will read as compliance.
3. **Two checks answering two different questions do not cover for each other** — a
   per-name allowlist saying "fine" does not mean the per-kind question was asked.
4. **Whether a check RUNS is a runtime property.** No amount of asserting on an
   artifact's text reaches it; if FerroTrack emits anything executable, budget
   known-answer fixtures through the real code path.
5. **A ceiling clearable by the actor it constrains is not a ceiling.** The reason the
   referee exists at all.
6. **A no-op and a no-run are the same diff.** Any check whose pass condition is
   "nothing happened" — a dry run, an idempotent upgrade, a guard that let something
   through unchanged — needs a separate proof, *in the same run*, that the thing
   executed at all. Learned during install verification (2026-08-29): an idempotency
   re-run that read as a clean no-op had possibly never run — migrations apply in
   filename order and the copy sat below the store's highwater mark — and it became
   evidence only once a marker in the same restart proved new files execute.

## Decided in FerroStep — do not re-propose against these

Pure core behind the `Ledger` trait. Workflows are **data** (JSON definitions, not code).
Counters spend on **entry**; exhaustion is **routing**, not an error. Grow a decision
kind's **fields**, never the set of kinds. A refusal names its subject and states the
remedy — never approximate, never silently degrade. Apache-2.0.

## Handed to FerroTrack as open design questions

- **The refereeable categories are the load-bearing choice.** The referee today guards
  state, version, counters, scope, and graded attributes — and has *no category for
  anything else*. That limit has already cost an adopter whose merge gate keyed on a
  field the referee could not reach. An issue tracker's field set is much wider than the
  guarded set; deciding what FerroTrack makes refereeable is the central design act.
- **Model disagreement explicitly.** A loop with no vocabulary for dissent reports zero
  dissent — measured in a real agent loop, with a structural cause, not a compliance one.
  If FerroTrack models agent-to-agent work, dissent needs first-class representation or
  the tracker will report a consensus that never existed.
- **Intentions — ANSWERED (owner, 2026-08-28, via FerroStep's resident): both.**
  FerroTrack is the default ledger the referee ships against *and* a product with its own
  users. It supersedes both existing adapters (SQLite and PocketBase) as the default;
  **neither adapter is retired** — they stay real and exercised, because two independent
  implementations are the floor that keeps the `Ledger` trait honest. Standing
  constraint that follows: when something in the contract is awkward for the native
  store, the answer is never to bend the trait toward it — the native store is the
  implementation with the most leverage to deform the interface, which is exactly what
  the floor exists to catch. And because FerroTrack has users who are not FerroStep, the
  refereeable-categories question above hardens: a tracker's fields are its product
  surface, and the ones the referee cannot reach are precisely where an adopter's gate
  ends up keyed.

## The loose-coupling bar

**"A lane is a workflow definition plus a deployment's roster — data, not product"**
(owner, 2026-08-29, ruling on this repo's personas; measured true of FerroStep's engine
the same day — zero hardcoded role names, lane shapes, or assumed identities in non-test
code, with the only lane-shaped material living in `examples/` under an explicit
illustrations-not-standards disclaimer). This is the design bar for everything FerroTrack
builds above the store: rosters and lanes the product has never seen must fit, and a
feature that only works for one arrangement of roles is lane-shaped and does not travel.
The register question produced the test in passing: **a rule that is definition-driven
travels; a rule that is lane-shaped does not** — which is why exactly two of the
workspace's review-lane rulings entered this repo's personas and the rest did not.

## Process

Cross-repo discipline: a FerroTrack finding that would land as a change in FerroStep
routes through FerroStep's resident rather than being opened there directly, and the
reverse holds. Both repos are public; in commits, issues and docs, describe an adopter's
shape rather than quoting it.
