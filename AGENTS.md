# FerroTrack — rules of record

**A light AI-native issue tracking system for agentic workflows.** That is the owner's
repository description at creation (2026-08-28).

> **Provenance:** this file was drafted by the resident agent on 2026-08-28, the day the
> repo was created empty. It records standing workspace rules and the owner's direct
> instructions, each with its source — it makes no new rulings of its own. The owner has
> not yet ratified the file as a whole.

## What the owner has decided

1. **Sibling project: FerroStep.** FerroTrack is built in Rust as the agent-focused
   management system FerroStep loops ride on. **Intent (owner, 2026-08-28, stated to
   FerroStep's resident and relayed here the same day): FerroTrack is BOTH — FerroStep's
   out-of-the-box store AND a product in its own right.** Designing for either alone is
   wrong at the edges: error surfaces, upgrade guarantees, and who the API may break
   differ between the two roles. It supersedes **both existing adapters
   (`ferrostep-pocketbase` and `ferrostep-sqlite`) as the DEFAULT, and neither is
   retired** — "replacement" means the option you get without choosing, never the ones
   that stop being maintained. Consequence: FerroStep's two-implementation floor gets
   *stronger*, not retired — when the contract is awkward for redb, the move is never
   "bend the trait toward the native store."
2. **Backing store: UNDECIDED — that is the whole ruling** (owner, 2026-08-28). Names
   mentioned so far — redb (hypothetical), LibSQL, SurrealDB, a first-party store — are
   **examples, explicitly not exhaustive and not a shortlist**; do not treat the
   candidate set as narrowed. [`notes/redb-research.md`](notes/redb-research.md)
   characterizes the redb candidate and doubles as the evaluation template;
   [`notes/ferrostep-contract-fit.md`](notes/ferrostep-contract-fit.md) carries the
   store-neutral acceptance bar every candidate must meet.
3. **Same licence file as FerroStep** — Apache-2.0, copied verbatim.
4. **Research first.** No full architecture on the spot; the current phase produces
   research, not commitments.
5. **`assets/` at project root** carries the icon, themed to match FerroStep's.
6. **Review lanes are configuration-level, and the coupling stays loose** (owner,
   2026-08-29, declining to import the workspace's prior review-lane register into this
   repo's personas): a lane is a workflow definition plus a deployment's roster — data,
   not product — and **specific identities can never be 100% expected in every
   scenario**. Personas and product design must not assume a particular reviewer, a
   particular roster, or another deployment's conventions; a ruling from elsewhere
   enters this repo on its own merits, individually, or not at all.
7. **FerroStep is being installed with this repo as a deployment target** (owner,
   2026-08-28; install authorized the same day: the owner coordinates, **this repo's
   resident authors the config details, FerroStep's resident guides and reviews**). The
   deployment brief in
   [`notes/ferrostep-deployment-brief.md`](notes/ferrostep-deployment-brief.md) is the
   vocabulary. Settled the same day, superseding an earlier store-less reading: **the
   ledger is a PocketBase deployment in MAPPED shape** — FerroStep's referee over
   collections this repo itself introduces.
   - **What this repo authors and carries:** the workflow definition
     ([`workflow/research.json`](workflow/research.json)), the generation config
     ([`workflow/research.generate.json`](workflow/research.generate.json) — map plus
     explicit actor binding plus the `workflow` pointer that buys the create guard,
     and since FerroStep `ff95ff4` the ONE file both consumers
     read: the generator takes it whole and the CLI's `--map` accepts it directly), and
     the records collection's column spec
     ([`workflow/research.collection.json`](workflow/research.collection.json)).
     All contain only FerroTrack's own vocabulary — that is what makes them repo-safe, and
     it is a line, not an accident: ⚠⚠ **nothing that identifies the deployment enters
     this repo** (no host, port, route, credentials, or names of collections this repo
     did not introduce). Connection details are invocation parameters — environment or
     an untracked `.ferrostep/` file, never argv where avoidable.
   - **Install sequencing (corrected in sibling review, 2026-08-28):** the generated
     migration *finds* the records collection and *creates* only the events collection —
     mapped shape maps onto a collection you already have. So `ferrotrack_research` is
     created first, by whoever holds instance access, from the column spec above; access
     rules and the actor binding are deployment decisions made at install, deliberately
     not specified here. The `state` column is a select over exactly the definition's
     seven states, which is what gives `doctor` something real to check.
   - ✅ **The create-parity limit is CLOSED, opt-in taken** (FerroStep `39d6cc1`,
     2026-08-29; sibling resident measured it live in five cases). The generation
     config's `workflow` key points at the definition — resolved against the config
     file's own directory — and buys the CREATE guard: a record can only be born in
     the definition's `initial` state, read from the definition at emission and never
     copied into the map. ⚠ Opt-in is deliberate (an adopter refereeing pre-existing
     rows creates them mid-loop on purpose); **omitting the key emits the old
     update-only surface with no error**, the same silent-absence shape as the actors
     block. ⚠ **Install note: once installed, every research record must be CREATED
     with `state = "open"`** — the store now refuses any other birth state by name
     (`not_initial_state`), so "why was my record refused" has this as its first
     answer. Still true from the same review: in mapped shape the adapter refuses
     `ferrostep file` — filing is the collection's own procedure. ⚠ **The definition therefore carries NO `creation`
     block, deliberately** (an inert documented permission is worse than an honest
     absence); when filing becomes reachable it returns *with a population bound*, which
     is currently absent by the same honesty.
   - **Bootstrapping reading, flagged not asserted:** directing the install now, while
     FerroTrack is pre-alpha, reads as FerroStep's loop refereeing FerroTrack's own work
     before FerroTrack exists. Confirm with the owner before building anything on that
     reading.
   - ⚠ **`ferrostep doctor` run against a stale install reports store-side questions
     UNCHECKED and exits non-zero — that is correct behaviour, not a fault.**
     Regenerating an installed surface is the owner's action, coordinated by FerroStep's
     resident.
   - **Identity (owner, 2026-08-28; email domain corrected to the house `.io` the same
     day): the resident holds the `developer` entry and a second agent holds `reviewer`.**
     Both identities are SET in [`config.yaml`](config.yaml), the one copy — prose points
     there and deliberately restates no value. The `ferrostep` binary is installed from the sibling
     checkout at `c475ed5` (clean tree; reinstalled 2026-08-29 when the create guard
     landed — the emission lives in the generator, so an old emitter reads a config
     carrying the new `workflow` key and silently drops it: reinstall BEFORE
     regenerate, verify `not_initial_state` in the emitted file, then install.
     Previously `ff95ff4`, and `881e20aa` before that, each for the same reason),
     recorded here because an installed surface meets newer binaries later.
   - ⚠ **At install, expect the generated migration to create the events collection AND
     the actor collection, and to leave the records collection untouched** (measured in
     sibling review, 2026-08-29: three guarded creations — version field, events,
     actors; with `version` pre-declared, the records branch correctly skips). An actor
     collection appearing unasked is *expected* here, not a sign something else is
     wrong — and convenient, since the binding on this checklist needs one. The
     migration is measured idempotent: a second run with live rows present changed
     nothing and lost nothing.
   - ⚠⚠ **Never run the down-migration against this deployment** (measured, same date):
     `migrate down` deletes the events collection and every row in it, deletes the
     actor collection, and removes the version field — **while leaving the refereed
     records in place**, separating records from their history, which is the one
     property mapped shape exists to preserve. It prompts and cannot run unattended;
     nothing in an install or restart triggers it — that is the whole mitigation.
     ⚠ Every *copy* of the migration file carries the same down path: a second file
     installed beside the first (a forced re-run, a regenerate under a new name) is a
     loaded gun pointed at the history, and its filename will not say so.
   - ⚠⚠ **Install checklist, added when the roster gained a second agent (2026-08-28):
     the ACTOR BINDING is part of the install, not a retrofit.** On a mapped deployment
     without one, a role is a string any caller may claim — academic with one agent,
     not with two holding distinct roles. The binding names an auth collection and role
     field on the instance (deployment decisions, recorded there, never here) and
     creates no identities — the actor accounts must exist first, which is the owner's
     act. ⚠⚠ **Its `allow_unbound` flag defaults PERMISSIVE by design; flipping it to
     `false` once the actors exist is a SECOND deliberate act, named here so it cannot
     silently not happen.** Until that flip, the role check reads as enforcement and is
     not one — "we will add the binding later" and "the roles are enforced" are two
     beliefs that coexist comfortably, and only one is true.
     The binding is authored EXPLICITLY in the generation config — including
     `allow_unbound: true` for the install window, a *chosen* transition value, not an
     inherited default (omitting the block emits the permissive default silently; a
     value you chose and a value you inherited are different facts, and the emitted
     hooks document the chosen one in a comment). ⚠ **The flip is a
     regenerate-and-reinstall, not an instance toggle** — the permissive branch is baked
     into the emitted hooks, so flipping means: edit the generation config, re-emit,
     reinstall, restart. Budget it as such. **Actor accounts are needed for ALL THREE
     roles — including `owner`** — before the flip; creating them is the owner's act.
     ✅ The binding names `ferrotrack_actors` — **ruled by the owner, 2026-08-29:
     create, for now** — FerroTrack's own vocabulary, created by the migration when
     absent, which keeps the generation config committable in this public repo.
     **Bind-don't-mint stays on the record as a possible future transition for a public
     deployment** (owner, same ruling): binding an existing auth collection would move
     the generation config into the untracked `.ferrostep/` home, and that trade-off
     gets re-weighed if that transition is ever considered — not silently re-decided.
   - ✅ **The derived-map drift class is REMOVED, not mitigated** (owner-approved;
     landed as FerroStep `ff95ff4`, 2026-08-29): `--map` accepts the generation config
     directly, the derived `research.map.json` is deleted, and one file serves both the
     generator and the CLI — drift between them has nowhere to live. ⚠ A file carrying
     BOTH a `map` key and a top-level `records` key is refused with the remedy named,
     never guessed at — the mid-conversion case where picking a winner would silently
     use the half you are not editing.
   - ✅ **Creating `ferrotrack_research` on the instance is the sibling resident's act**
     (owner, 2026-08-29) — from the column spec, before the migration runs (the
     migration finds it, never creates it). Every step in the sequence now has an
     owner. ✅ **DONE the same day** (sibling; verified by reading the collection back:
     the spec's fields, `state` a select over exactly the seven states, `version`
     pre-declared). **Access rules set FAIL-CLOSED at creation — all five null — the
     sibling's deployment decision**: widening is a deliberate act by whoever holds the
     instance, never a default. Raised before install, as asked: the current loop needs
     no non-superuser read path (reads ride the standing store credential); the
     question returns at the `allow_unbound` flip, when there are actor accounts to
     widen *for*.
   - ⚠ **Creating a collection through the store's API auto-writes a `created_…`
     migration file on the instance** — expected and benign, authored by neither
     resident, and NOT a copy of the emitted one (different down path). A sweep for
     stray migration files will meet it; check provenance before treating it as one.
     ⚠⚠ **Consequence: the emitted migration must SORT AFTER it.** The emitted file
     *finds* the records collection the auto-file *creates*, and a fresh-database
     replay runs migrations in filename order — the wrong order fails at the find. The
     staged copy was renamed accordingly before install (2026-08-29).
   - ⚠ **On the target deployment, writing a hooks file IS the restart** (hooks
     watching measured on the deployment, 2026-08-29). There is no separate restart
     step to schedule: the migration file lands BEFORE the hooks file — the hooks
     write triggers the restart that applies it — and verification follows the write
     settling, not a command you run. **Installing the two staged files is the
     OWNER's act**, the last one in the sequence. ✅ **DONE 2026-08-29, sequence
     CLOSED, verified**: the anonymous ping answers from the new surface (the
     `columns` key only a current-generation emission carries), the events and actors
     collections exist, the records collection is untouched, and `ferrostep doctor`
     reports **0 faults, 0 unchecked, 6 agreed** — including the installed-write-path
     check, the first time this deployment could meet the detector and pass it whole.
     The referee is LIVE: research records are born `state = "open"` or refused, and
     refereed columns move only through the apply route. Next deliberate act on this
     surface (not part of the install): actor accounts for all three roles, then the
     `allow_unbound` flip — regenerate, reinstall, restart.
8. **FerroTrack ships with an installer — a product decision, recorded at research
   phase** (owner, 2026-08-29; ⚠ an over-read corrected the same day: this item is
   about **FerroTrack's future as a product**, NOT about the FerroStep-into-this-repo
   deployment in item 7 — the first draft conflated the two). When FerroTrack exists
   as a product, adopting it goes through an installation script or installer designed
   in from the outset, with the design bar set by what a hand-driven install was
   measured to cost the same week: **installation must not require broad filesystem
   grants**. Same spirit as "AI-native at the outset, not retrofitted" — the
   installation experience is part of the product, not an afterthought. ⚠ **FerroStep's
   own installation process — including its option to use FerroTrack as its built-in
   issue tracker — is the owner's conversation with FerroStep's resident**; recorded
   here only to mark the boundary, and this repo's resident does not open it.
   **Recorded intent, not a build item**: research-first (item 4) holds, and the
   installer's concrete shape waits on the backing-store decision (item 2) — flagged,
   not asserted: how much installing costs an adopter is itself a property stores
   differ on, so the evaluation bar may want a line for it.

## Standing rules that already bind this repo

1. ⚠ **This is a PUBLIC repo.** Nothing lab-internal enters it: no hostnames or ports,
   no credential locations, no internal service or session names, no lab tracker issue
   numbers. Establish disclosure context *before* material arrives here from a lab
   session, not after. (The same rule FerroStep, the sibling public repo, operates
   under.)
2. **Licence: Apache-2.0** — the house default for every new repo (owner, 2026-08-01).
   `LICENSE` carries the full text; declare it in any future `Cargo.toml` /
   `pyproject.toml` / `package.json`, and state it in the README.
3. **`north-star.md` is a gap to fill, not an exemption** (workspace convention, owner,
   2026-08-19/20: every product repo carries one, under exactly that filename). Its
   Vision section is the owner's to write or ratify. It is deliberately absent rather
   than agent-drafted from a one-line description.
4. **No process is adopted here yet.** No `workflow/` persona, no review lane — the
   review lane is piloted in Sonora only, and FerroStep adopted the identity portion
   only (workspace `AGENTS.md` §3). Until the owner directs otherwise, work here is
   owner-directed: ask rather than inventing a process.
