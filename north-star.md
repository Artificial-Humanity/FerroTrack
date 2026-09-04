# North Star — FerroTrack

> The map, not the implementation: the *why*, and the standing decisions the build serves.
> When a branch, experiment or detour raises "wait, what are we doing again?", this is the
> document that answers it.
>
> ⚠ **THIS FILE OWNS NOTHING BUT §1.** Every constraint and decision below is held in
> [`AGENTS.md`](AGENTS.md) or a note under `notes/`, and the owning file is named at each
> one. **If this file and the owning file disagree, the owning file wins and this one is
> the defect.** A second copy nobody compares is a failure this workspace has paid for
> repeatedly.
>
> ⚠⚠ **§1 IS THE OWNER'S AND IS CURRENTLY *UNRATIFIED*.** It was drafted 2026-09-04 by the
> resident agent from the owner's own answers in conversation that day — organised, not
> invented — and it is **awaiting the owner's rewrite or ratification.** Keep that order
> for any future edit to §1. The failure this procedure exists to prevent is on the record:
> an agent once wrote "a deliberate owner ruling" about a decision nobody could later
> verify, and it read as settled fact for two days.

---

## 1. The Vision — *drafted from the owner's words, UNRATIFIED*

**The registry and the communication router are the heart of it.** Not the issue tracker —
the issue tracker is what the registry makes possible first.

The gap FerroTrack exists to close: **combining locally hosted agents with frontier-vendor
agents is becoming common, and there is nothing that lets them find each other.** Claude
Code has messaging between its own agents but no registry. Nothing addresses the mixed
case — a local model and a frontier-vendor agent in the same workflow, needing to be
addressable, routable and wakeable by the same mechanism.

**It is built for us and shared with the world.** Both, genuinely. The owner's position:
*"I intend to both use the product and share it… valuable things should, ideally, be shared
with others if it's realistic."* There is **no charge, no commercial product and no service
entity** around it. We build it for ourselves and accommodate potential others.

⚠ **Being built for us is not a licence for it to stay rough.** In the owner's words:
*"just because it's built for me, it does not mean it remains in an unpolished state."*
That is a quality standard, not an aspiration, and it is the reason the API contracts in
`AGENTS.md` item 2 are treated as contracts rather than as internal conveniences.

**Intended for small teams and solo developers working with multi-agent workflows.**
⚠⚠ **That is a TARGET, not a wall — it is explicitly not a rule that FerroTrack never
becomes anything else** (owner, 2026-09-04, declining to state a "must never become"
constraint on the grounds that inferred rules of that shape have been hardening into walls
across this workspace). Read it as aim, and never cite it to rule something out.

⚠ **A correction the owner made while stating this, worth carrying:** an earlier framing —
that the owner was the customer — *"became both a rule and a wall. It was never the
intent."* If this document is ever used to narrow the audience, it is being misread.

---

## 2. What is ours, and what is rented

**Ours** — the referee semantics over the ledger, the agent registry, the communication
router, wake-up, task scheduling, the address format, the query surface, and the inverted
index behind it. Nearly everything the product *is*, because of §3.

**Rented** — `redb` (store), `axum` + `tokio` (server layer), and at phase 2 a stemmer.
Settled in `AGENTS.md` item 2.

⚠ **Both rented pieces of consequence are single-maintainer projects, measured 2026-09-04**
— redb is ~92% one author, and every candidate weighed was the same. This is a property of
the category, not of the choice. The standing mitigation is the permissive licence plus the
owner's latitude to build on a permissive base (item 9), which was authorised before the
risk was measured.

---

## 3. The One Organizing Principle

> **FerroTrack surfaces the feature. A dependency may merely facilitate it.**

The owner's own formulation (2026-09-04). It is the sentence that has actually decided
things: it settled which layer owns pub/sub, it set the store's evaluation axis to *how
much plumbing does FerroTrack handle* rather than *which features does the store have*, and
it is why requirement 5 — usable standalone — holds by construction rather than by effort.

When an argument stalls on "could the store do this for us", the answer is that it does not
matter who *can*; FerroTrack is what presents it either way, so the only question is how
much work the dependency saves.

---

## 4. Load-Bearing Constraints

Each is held elsewhere; this section points.

- **Public repo — nothing lab-internal, ever.** `AGENTS.md` standing rule 1.
- **Apache-2.0**, the house default. Aligns with §1's not-commercial, share-freely posture.
- **The interface is a network protocol, not a Rust API.** Forced by inter-brand: agents in
  other vendors' runtimes cannot link a crate. `AGENTS.md` item 9.
- **Installation must not require broad filesystem grants.** `AGENTS.md` item 8 — and the
  reason spawning lives in an optional companion rather than in core.
- **Single-writer store ⇒ the binary and the library crate are either/or per deployment**,
  never both against one file. `AGENTS.md` item 2.
- **The query surface and the read-your-own-writes promise are CONTRACTS.** They can be
  widened; narrowing them breaks clients this repo does not control.
- **FerroStep's two-implementation floor gets stronger, not weaker.** `AGENTS.md` item 1 —
  when the contract is awkward, the move is never to bend the trait toward the native store.

---

## 5. The Real Bottleneck

**Verification, not design.** As of 2026-09-04 the product is thoroughly *decided* and
entirely *unbuilt*, and the decisions rest on characterization rather than measurement. The
specific blocker: **the referee's whole guarantee is a conditional update evaluated inside
the transaction, and that has never been measured on redb** — at the call site, with a
control, so a conflict-free pass cannot be mistaken for a vacuous one.

⚠ That measurement outranks the reasoning that produced the store ruling. Until it is run,
every downstream decision inherits an unverified premise. It is the next piece of work.

---

## One-Breath Summary

A single Rust binary — or an embedded crate — that keeps a registry of agents, local and
frontier-vendor alike, routes messages between them, wakes them when work arrives, and
tracks the issues they work on. Built for us, polished as though for the world, and given
away.
