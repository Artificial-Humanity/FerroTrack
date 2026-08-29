# The research workflow — deliberate choices

The definition is [`research.json`](research.json) and it is the one copy of the
vocabulary; this file records only the choices a reader might reasonably think were
accidents. Each was raised in sibling review (2026-08-28) as "confirm rather than change",
and is hereby confirmed.

**A record can be disputed from `open` — including at filing, before anyone has looked.**
Intended. `open` serves as both the filing state and the rework state, so the researcher's
dispute route covers two real cases with one move: "I disagree with this rework decision"
and "this item should not exist at all." Both are disagreements addressed to the owner,
both require a note, and splitting the state to separate them would buy vocabulary nobody
currently needs.

**There is no `escalated → collected`.** Deliberate. A dispute means the work is done and
contested — so the owner may accept it directly. An escalation means the work is
incomplete — a spent ceiling or a researcher who is stuck — and incomplete work does not
become collected by adjudication; it re-enters through the owner's release and finishes.

**Where notes are required, and where not.** Every move that summons a person carries
`requires_note` (both dispute routes, the researcher's escalate), and so do both owner
releases-with-reset — the reset is a real grant, and the "why" is what the researcher
needs and the history will not otherwise hold. The owner's terminal adjudications
(`→ collected`, `→ dropped`) deliberately do not: the dispute's content was required at
entry, and the move itself states the verdict.

**The counter spends on claiming, not completing.** A crashed pass still cost one — the
alternative is "retry until it works," which is the unbounded loop the ceiling exists to
prevent.

---

## One file, both consumers

[`research.generate.json`](research.generate.json) is the single deployment config: the
collection map plus the explicit actor binding. The generator (`emit-mapped`) consumes it
whole; since FerroStep `ff95ff4` the CLI's `--map` accepts it directly too. There is no
derived copy — a bare-map file used to exist here precisely so two consumers could read
two shapes, and it was deleted the day one shape served both, because two files that must
agree by convention drift in the direction nothing goes red in.

⚠ If a config ever carries **both** a `map` key and a top-level `records` key, the CLI
refuses it and names the remedy rather than guessing — that shape only arises
mid-conversion, exactly where silently picking a winner would use the half you are not
editing.
