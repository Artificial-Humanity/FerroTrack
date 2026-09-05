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
   *stronger*, not retired — when the contract is awkward for whichever store FerroTrack
   lands on, the move is never "bend the trait toward the native store."
2. ✅ **SETTLED (owner, 2026-09-04): the stack is `redb` + `axum` + `tokio`.** This closes
   the backing-store question opened 2026-08-28.
   - ✅ **The store is a RANKED LADDER of three (owner, 2026-09-05), not a choice with a
     fallback:** **1. redb — we proceed. 2. fjall — backup. 3. our own derivative work —
     third.** ⚠⚠ **What changed on 2026-09-05: a derivative is now ON THE LADDER rather
     than a hypothetical.** It was previously an authorised escape hatch (item 9, latitude
     3) holding no position; it is now a direction the project may deliberately take, on
     the owner's stated grounds that **deriving from a permissive base has become
     materially more reachable in recent years.**
     The reasons, the ideal-datastore requirements, and where redb and fjall each fall
     short are recorded in [`notes/ideal-datastore.md`](notes/ideal-datastore.md).
     ⚠ **This does not soften item 1.** Whichever rung is standing, the move is never to
     bend the ledger contract toward the native store — and that rule is also what keeps
     rung 3 cheap, since the two-implementation floor already requires the store to sit
     behind an interface a later `FerroStore` can slot into.
   - ⚠⚠ **What the settlement RESTS ON, recorded so it can be revisited deliberately rather
     than silently:** characterization, **not measurement**. Two things were owed at the
     moment of the ruling and are still owed:
     1. ✅ **RESOLVED 2026-09-04 — checked, and it does NOT move the ruling.** redb's
        concentration is real: cberner 1551 commits, next human 32 (~92%). **But the
        condition as written required the secondary to be better, and it is not.** fjall
        is marvin-j97 1933 / next human 32, and `lsm-tree` — the crate that actually
        persists — is marvin-j97 2568 / next human 20. ⚠⚠ **Both candidates are
        single-maintainer projects in near-identical proportion, so bus-factor-of-one is a
        property of this CATEGORY, not a discriminator between these two.** On the
        surrounding signals redb is the safer of the pair: 8 years old against 3, ~2× the
        stars and forks (4770/237 vs 2304/119), 11 open issues against 39, and pushed the
        same day this was checked.
        ✅ **The standing mitigation is already on the record: latitude 3.** The risk is
        abandonment, the licence is `MIT OR Apache-2.0`, and the owner has already
        authorised building on a permissive base — so the escape hatch was approved before
        the risk was measured. ⚠ It stays a real risk, correctly weighted: it is now a
        reason to keep the fork option live, not a reason to prefer fjall.
     2. ⚠ **The acceptance bar in [`notes/ferrostep-contract-fit.md`](notes/ferrostep-contract-fit.md)
        is unmeasured on redb** — above all the conditional update evaluated INSIDE the
        transaction, tested at the call site with a control so a conflict-free pass cannot
        be mistaken for a vacuous one. That measurement outranks the reasoning that
        produced this ruling.
   - **Why redb over fjall** (full comparison in [`notes/store-criteria.md`](notes/store-criteria.md)):
     ACID single-writer is redb's default and cannot be opted out of, where fjall's
     transactions are opt-in with a **silent** non-transactional path — a weak guarantee in
     a codebase agents contribute to. fjall's clearest advantage, compaction filters,
     deflated on inspection: it saves the sweep *job*, not the logic (liveness and
     deliverability are read-time questions either way), and item 9's scheduling machinery
     makes periodic sweeps nearly free. Plus 56 crates against 84, one file on disk, and
     `MultimapTable` fitting both the structured indexes and phase-2 postings lists.
   - ⚠ **`axum` and `tokio` entered this record as a MEASUREMENT ASSUMPTION and were never
     independently evaluated.** They were the stand-in for "a server layer" when the
     whole-product dependency stacks were measured, and the ruling adopted them along with
     the store. tokio is near-inevitable for async Rust; **axum is one of several reasonable
     choices that were never compared.** Flagged as a gap in the evaluation, not as an
     objection to the ruling.
   - ✅ **The settled stack excludes tantivy**, which places search at phases 0–2 of the
     roadmap in [`notes/store-criteria.md`](notes/store-criteria.md).
   - ✅ **SEARCH IS NOW FULLY DECIDED (owner, 2026-09-04)** — (b), (c) and (d) all answered:
     - **(b) The public query surface promises KEYWORD SEARCH, FIELD-WEIGHTED**: multi-word
       boolean matching over title / description / body, with a title match outranking a
       body match. ⚠ This is a **contract**, not an implementation note — inter-brand
       clients code against it, widening is easy and **narrowing is a breaking change for
       clients this repo does not control.**
     - **(c) The API PROMISES READ-YOUR-OWN-WRITES**: an issue is findable immediately
       after it is filed. Free today, because the index commits in the same transaction as
       the write. ⚠ It does **not** foreclose tantivy at phase 3, but it prices it: tantivy
       would need synchronous commits, which are slower and fine at this volume — **budget
       that when phase 3 is considered, rather than discovering it then.**
     - **(d)** Retained and indexed: title, description, body as text; project, author,
       assignee as structured id lookups. Recorded in the note.
     - ⚠⚠ **Consequence worth naming: phase 2 is now REQUIRED, not optional.** Phases 0–1
       (structured indexes, prefix/substring find) do not deliver keyword-field-weighted
       search, so **search cannot be exposed at all until the hand-built inverted index
       exists.** Phases 0–1 may still ship first — with no search endpoint published.
       Tantivy remains the phase-3 option and stays backfillable because (d) retains text.
   - ✅ **PROTOCOL RULED (owner, 2026-09-05): MCP is FerroTrack's protocol.** This answers
     the question item 9 left open — that item established the interface must be a network
     protocol rather than a Rust API, and never said which. MCP is a cross-vendor standard
     as of 2026 (donated to the Linux Foundation's Agentic AI Foundation, December 2025;
     native across Claude, ChatGPT, Gemini, Copilot, VS Code and Cursor), so **one server
     reaches every MCP-capable runtime with no bespoke client per vendor** — the inter-brand
     requirement almost exactly, and it reduces work rather than adding it.
     - ⚠ **OPEN, and it decides whether one protocol serves or two:** MCP is
       agent-as-client calling tools, so register / file / search / move / send / ack are
       natural — but **wake-up and delivery run the other way.** MCP defines server→client
       notifications; whether they suit a general delivery channel must be **verified
       against the current spec (2026-07-28), not assumed.**
     - ✅ **If they do fit, liveness gains a precise definition for free**: connection-based
       liveness means "holds an open channel", and "holds an open MCP session" is an
       already-specified form of exactly that.
     - ⚠ **MCP serves AGENTS.** A human UI and FerroStep's Rust adapter are not MCP
       clients, so the likely shape is MCP for agents, the library crate for embedders, and
       possibly REST for a UI — each another place the one set of invariants must hold. See
       the bypass hazard recorded against the library crate above.
   - ✅ **`axum` ACCEPTED AS RULED (owner, 2026-09-04)** — the gap above is closed by
     decision rather than by study, on the stated grounds that a server framework is
     replaceable behind our own handler layer in a way an on-disk format is not. ⚠ Recorded
     as a *decision not to evaluate*, which is different from an evaluation, so a later
     reader does not mistake it for one.
   - **History of the decision, kept because the reasoning is the useful part:** undecided
     from 2026-08-28; SurrealDB named front runner and scratched within 2026-09-03 on
     measured dependency weight; redb and fjall compared as full stacks 2026-09-03/04;
     settled 2026-09-04.
   - **What it was scratched ON, so the ruling can be revisited deliberately rather than
     silently:** 305 resolved crates on its most minimal configuration against redb's 1,
     and a C toolchain that is **hard-coded per target family rather than feature-gated**,
     so a consumer cannot switch it off. ✅ Upstream ships a pure-Rust path for wasm, so
     if that selection ever becomes a feature the dependency objection changes shape. That
     is the condition; it has not happened.
   - ✅ **Licence question, RULED ADEQUATE by the owner 2026-09-03** and recorded even
     though the candidate is out, because it was a real ruling and the candidate could
     return: SurrealDB's core is BSL-1.1, not an open-source licence, converting to
     Apache-2.0 four years per release. ⚠ Ruled against BSL-1.1 specifically; an upstream
     relicence would be a new fact. ⚠ **The scratch was NOT on licence grounds** — the two
     must not be conflated later.
   - **What the store is FOR — the owner's stated requirements (2026-09-03). These are the
     product-side half of the acceptance bar, and they outlived the candidate that
     prompted them:**
     1. **Issue tracking, in the SQLite-to-PocketBase relation.** FerroTrack occupies
        PocketBase's seat — the product — and the store is the engine underneath it.
     2. **A light install: embedded into the project, or carried as a project asset.** The
        same bar item 8 sets for the installer, arriving from the other direction.
     3. **Pub-sub, ideally — to carry inter-agent communication routing.** Stated as a
        preference; see the ruling below for where it now lives.
     4. **Rust-built, with minimal dependencies — ideal.** ⚠ **Two separate tests.** The
        scratched candidate passed the first easily and failed the second by two orders of
        magnitude, which is the whole reason to keep them apart.
   - ✅ **RULED (owner, 2026-09-03): FERROTRACK provides the pub-sub model, not the
     store.** This settles the layer question that the store decision was turning on, and
     it **widens the candidate set rather than narrowing it**: the engine no longer needs
     native pub-sub, only embedded operation, ACID, a conditional update evaluated inside
     the transaction, and a point at which FerroTrack can observe its own writes.
     ⚠ Consequence worth stating, because it is a *gain* and reads like a loss: a store's
     lack of watch/subscribe is no longer a defect against it.
     ⚠⚠ **Boundary unchanged: the inter-agent messaging system itself is the owner's design
     space and spans more than this repo.** This ruling says which layer owns pub-sub. It
     does not design the routing, name an addressing scheme, or settle which component
     hosts it — and this repo's resident does not open that.
   - ✅ **Dependency weight, MEASURED 2026-09-03** (resolved graphs, minimal feature sets,
     no build — method and fairness caveat in the SurrealDB note): **redb 1 crate (zero
     dependencies); rusqlite-bundled 9; sled 16; fjall 41; libsql 136; SurrealDB 305.**
     ⚠ Read across scopes carefully: redb is a key-value store and SurrealDB a whole
     database, so much of that tree is machinery FerroTrack would pull in anyway. What
     survives the caveat is the C requirement and *inherited versus chosen* — 305 arrive
     as a set.
   - **Candidate research:** [`notes/redb-research.md`](notes/redb-research.md) is the
     evaluation template every candidate is written against;
     [`notes/surrealdb-research.md`](notes/surrealdb-research.md) is the scratched
     candidate, kept as evidence. [`notes/ferrostep-contract-fit.md`](notes/ferrostep-contract-fit.md)
     carries the store-neutral acceptance bar. ⚠⚠ **No candidate has been evaluated
     against that bar — both notes are characterization.** The one hard number in the
     whole evaluation is the dependency measurement above.
   - ⚠ **The over-read this repo keeps meeting is now RESOLVED, not merely warned about.**
     Three earlier corrections turned on "use redb" being read as "redb decided"; as of
     2026-09-04 redb **is** decided, by the owner, on a recorded comparison. The warning is
     retired — but the shape it taught is not: **a question is not a ruling, and an
     agent's recommendation is not one either.** This one became a ruling because the owner
     made it, on a date, in words this file quotes.
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
     Both identities are SET in [`config.yaml`](FerroStep/config.yaml), the one copy — prose points
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

9. **What FerroTrack is FOR — the product brief** (owner, 2026-09-03, stated to enable a
   proper evaluation of the stack). Five things, in the owner's order:
   1. **Issue management** — PocketBase-like, but tailor-made for agents.
   2. **Inter-agent communications routing, on a REGISTRY.** Agents starting up under
      FerroStep management register themselves in the FerroTrack registry, and that one
      registry serves **both** issue-tracking and comms routing. ⚠⚠ **Explicitly
      INTER-BRAND** — Codex, Gemini, Grok, Muse and others "all play together". The
      owner's note on the gap this fills: Claude Code has built-in messaging between
      agents but **no registry**.
   3. **A wake-up mechanism** — agents must be able to be *awakened* when a communication
      arrives. An extension of item 2, not a separate feature.
   4. **Task scheduling** — tasks handed to persistent agents at a scheduled time,
      arriving as another wake-up message, but originated by the system or a human user
      rather than by another agent.
   5. ⚠ **Usable standalone.** FerroTrack is a dependency for FerroStep, but a user who
      wants only FerroTrack's features — without FerroStep's workflow management — must
      be able to run it on its own. **This is a product requirement, not an aspiration.**
   - **Evaluation latitude the owner granted the same day** — what the *store* is allowed
     to not do. ⚠ **Corrected by the owner within the day, and the correction is the
     version that binds**: the first statement of this latitude named FerroStep as the
     carrier of these features; **it is FERROTRACK that carries them.** Recorded because
     the earlier wording would otherwise support a contradiction with requirement 5 that
     does not exist — do not re-derive one from it.
     1. ✅ **FerroTrack surfaces the feature; the database may merely FACILITATE it.**
        Full-text search is FerroTrack's to support. Pub/sub likewise: a store with
        something like SurrealDB's open (live) queries facilitates it, and a store without
        one does not prevent it — **the feature is surfaced by FerroTrack either way.**
        ⚠⚠ **So the real evaluation axis is the owner's own question: HOW MUCH OF THE
        PLUMBING DOES FERROTRACK HANDLE?** Not "which features does the store have" —
        every feature in item 9 is FerroTrack's to present regardless. See
        [`notes/store-criteria.md`](notes/store-criteria.md), which is organised on that
        axis and carries the measurement.
     2. **Other databases may be considered** if they support these features more readily
        and spare FerroTrack the plumbing.
     3. ✅ **Building a new product on a permissively-licensed base is on the table**
        (owner's clarification, 2026-09-03: by "forking" he meant *creating a new product
        using an existing permissively-licensed product as a base* — not maintaining a
        divergent copy of someone else's tree), rebranded (FerroStore, FerroData).
        ⚠ **Conditional, and the condition is the owner's: "only if the arguments for it
        are compelling enough."** ⚠ Verified 2026-09-03: **redb, sled and fjall are all
        `MIT OR Apache-2.0`**, so the base is permitted and compatible with standing rule 2.
   - ✅ **Requirement 5 is not in tension with the latitude above** — once the features are
     FerroTrack's, a standalone FerroTrack has them by construction. Recorded positively
     because an earlier revision of this item claimed the opposite at length.
   - ⚠⚠ **Requirements 3 and 4 are not database features and no store choice will supply
     them.** A database can notify a *connected* subscriber; it cannot start a process
     that is not running, and it cannot run anything at a scheduled time by itself. Both
     need a **resident FerroTrack process**. Combined with requirement 2's inter-brand
     goal — agents in other vendors' runtimes cannot link a Rust crate, so the interface
     must be a network protocol rather than an API — the shape the brief implies is
     **FerroTrack as a server with an embedded store**, which is what requirement 1's
     PocketBase analogy said all along. ✅ **Consequence for the evaluation: the database
     question gets SMALLER, not larger.** See [`notes/store-criteria.md`](notes/store-criteria.md).
   - ✅ **RESOLVED (owner, 2026-09-04): an OPTIONAL COMPANION SPAWNER.** Core FerroTrack
     **never spawns a process.** A separate, opt-in component holds the per-vendor launch
     commands and whatever grants spawning needs, so the default install stays inside item
     8's bar and the capability is still reachable. ⚠ *(Resident's inference, 2026-09-04 —
     the SPLIT is the owner's ruling; this is only a prediction about how it erodes.)*
     Letting spawn logic drift back into the core on convenience grounds would undo it.
     - **The loop this closes:** a message arrives for an agent that is not connected →
       the companion notices and launches it → the agent registers itself on startup
       (brief requirement 2) → FerroTrack delivers over the new channel. The companion is
       therefore an ordinary FerroTrack *client* with spawn config, not a privileged
       insider — worth preserving, because it keeps core's API the only way in.
     - ⚠ **Two hazards to design against, named now rather than discovered:** the companion
       needs **single-flight/debounce per agent** or ten queued messages become ten spawn
       attempts; and it needs an identity and authorisation like any other client, since
       "may ask FerroTrack who is down" and "may start processes" is a potent pair.
   - ✅ **Runtime semantics DECIDED (owner, 2026-09-04), the three that the brief's
     requirements 2–4 turn on:**
     - **Liveness is CONNECTION-BASED.** An agent is live if and only if it holds an open
       channel — so "live" means exactly "deliverable right now", which is the question the
       router asks, and it imposes nothing on clients beyond staying connected. That last
       point matters more here than elsewhere: heartbeating would be a client obligation
       every *other vendor* has to implement, on a protocol whose whole purpose is that
       they adopt it.
       ⚠⚠ **The cost is a false-death case that the spawner turns into a real bug**: an
       agent running but briefly disconnected reads as dead and gets launched again.
       Single-flight is necessary but **not sufficient** — it stops ten messages causing
       ten spawns, not one spawn duplicating a live process. *(Resident's inference, not
       an owner ruling.)* Making registration idempotent per address would stop a duplicate
       that does start from becoming a second agent under the same address — one way to
       close this, not the only one, and not yet chosen.
     - **Delivery is AT-LEAST-ONCE, with explicit ack and a TTL.** Retained until the
       recipient acknowledges, expired after a configured age. **Agents must therefore be
       idempotent**, and that is a contract obligation on every client, so it belongs in
       the protocol documentation rather than in a design note. ⚠ redb has no expiry
       mechanism, so **the TTL sweep is one of the hand-written background jobs** — cheap,
       because item 9's scheduler is being built anyway, but it is now committed work.
     - **Distribution is a SINGLE STATIC BINARY** that creates its redb file on first run —
       ✅ the most literal answer to item 2's "embedded into the project or just a project
       asset", and it sits well with the store: one binary, one file — **AND a LIBRARY
       CRATE is published alongside it** (owner, same day), so Rust embedders, FerroStep
       included, can link FerroTrack in-process. The two answers complete each other rather
       than competing: standalone users run the binary, embedders link the crate.
       ⚠⚠ **Because redb takes a file lock and permits one writer, these are EITHER/OR per
       deployment, never both.** An embedder owns the file and **no FerroTrack binary can
       run beside it.** That is an adopter-facing constraint, so it belongs in the product's
       documentation and not only here — it is the kind of thing discovered at 2am
       otherwise.
       ⚠⚠ **The in-process path must enforce the SAME invariants as the network path.** If
       the library lets an embedder write while network clients go through validation, the
       referee has a bypass — and this workspace already has that exact lesson on record
       from a store where rules constrained users but not superusers. *(Resident's
       inference, not an owner ruling.)* Two surfaces over one set of invariants is the
       shape that avoids it; the library is the surface that will be tempted.
     - **Agent ADDRESSES are defined by FerroTrack: normalized, case-insensitive, and they
       name an AGENT rather than a session** — so an address survives a restart. FerroTrack
       owns validity, normalization and collision rules, so every vendor's client gets one
       rule instead of each inventing its own.
       ⚠ **Ruled for THIS repo on 2026-09-04**, which is what makes it binding here: a
       ruling of the same shape existed elsewhere from 2026-09-02, and item 6 requires that
       such a ruling enter on its own merits rather than by import. It now has.
       ⚠ *(Resident's inference, not an owner ruling.)* Normalizing in exactly one place
       — the normalized form as the key, the caller's original kept for display — avoids
       the way case-insensitivity usually decays into case-sometimes-sensitive.
       ✅ **Note this is deliberately separable from liveness**: the registry entry persists
       across restarts because it names an agent, while liveness is the channel's state.
       An address that exists but is not currently deliverable is the normal case, not an
       inconsistency — and it is precisely the case the companion spawner acts on.
   - ⚠ **A prior owner ruling on agent ADDRESSING exists outside this repo** (2026-09-02)
     and bears directly on requirement 2. It has **deliberately not been imported**: item 6
     says a ruling from elsewhere enters this repo on its own merits, individually, or not
     at all, and standing rule 1 bars lab-internal material. Ask the owner to restate it
     here if it is to bind this repo.
   - **This brief is the first material that could fill `north-star.md`** (standing rule
     3). Not drafted: §1 Vision is the owner's to write or ratify.

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
4. **No review lane is adopted here.** The `FerroStep/personas/` files are the IDENTITY
   portion only — who commits as whom, per `FerroStep/config.yaml` (decided item 7) — the same
   shape FerroStep adopted; the review lane itself is piloted in Sonora only (workspace
   `AGENTS.md` §3). ⚠ An earlier revision of this line said "no `workflow/` persona";
   that was true on 2026-08-28 and stale the same day the personas landed. Until the
   owner directs otherwise, work here is owner-directed: ask rather than inventing a
   process.
