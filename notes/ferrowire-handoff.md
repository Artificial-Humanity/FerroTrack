# FerroWire — handoff document

**Purpose.** The owner moved the communication router and the carrier out of FerroTrack on
2026-09-07. They become a separate product with the name **FerroWire**, and the owner
handles that product separately. This document collects everything that FerroTrack decided
about those concerns, so that another agent can start from a record instead of from
nothing.

⚠ **This document is a transfer, not a specification.** Each item below shows its source
and its date. The FerroTrack repository is public, and this document contains no
deployment details.

⚠ **FerroTrack keeps none of these concerns.** The FerroTrack files now point here.

---

## 1. What FerroWire is

FerroWire carries messages between agents. Three of the owner's five original requirements
move to it (owner, 2026-09-03):

- **Inter-agent communications routing, on a registry.** Agents register themselves when
  they start. ⚠⚠ **The design is explicitly inter-brand.** Codex, Gemini, Grok, and local
  models must all work together. The owner named the gap this fills: Claude Code has
  messages between its own agents, but it has no registry.
- **A wake-up mechanism.** An agent must wake when a message arrives for it.
- **Task scheduling.** A person or the system gives a task to a persistent agent at a
  planned time. The task arrives as another wake-up message.

⚠ **The owner's vision statement puts these at the center.** Section 1 of FerroTrack's
`north-star.md` says: *"The registry and the communication router are the heart of it. Not
the issue tracker."* It also names the trend that motivates the product: **local agents and
frontier-vendor agents now work together in the same workflow, and nothing lets them find
each other.** That text is unratified, and it now describes FerroWire more than FerroTrack.

---

## 2. Decisions the owner already made

Each of these is an owner ruling with a date. FerroWire can keep them or change them, but a
change is a change and the record must show it.

| decision | ruling | date |
|---|---|---|
| **Liveness** | Connection-based. An agent is live if and only if it holds an open channel. | 2026-09-04 |
| **Delivery** | At-least-once, with an explicit acknowledgement and a TTL. | 2026-09-04 |
| **Addresses** | The product defines the format. Addresses are normalized and case-insensitive, and they name an **agent**, not a session. | 2026-09-04 |
| **Wake-up** | An **optional companion spawner**. The core never starts a process. | 2026-09-04 |
| **Protocol** | MCP. | 2026-09-05 |

⚠⚠ **Line-wide intent (owner, 2026-09-07): MCP is the standard throughout the whole Ferro
line, because these are agents-first systems.** The owner will settle the details later.
Treat it as the default that FerroWire must argue its way out of, not as a finished
specification. ⚠ **The push-direction problem in section 3 does not go away because of this
intent. A line-wide rule does not answer it.**

### Why each ruling went that way

**Liveness is connection-based** because it imposes nothing on clients except that they stay
connected. Heartbeats are a client obligation, and every other vendor must then implement
them. That cost is high for a protocol whose purpose is that other vendors adopt it.
"Live" then means exactly "deliverable now", which is the question the router asks.

**Delivery keeps a message until the agent acknowledges it**, because requirement 3 needs
messages to survive an agent that is down. The TTL bounds the growth. ⚠ **Agents must be
idempotent.** That is a contract obligation on every client, so it belongs in the protocol
documentation.

**The spawner is optional and separate** because a component that starts processes needs
broad permissions. FerroTrack's installer bar says an install must not need broad
filesystem grants. The split keeps the default install small and keeps the capability
reachable.

---

## 3. Open problems, with the analysis already done

⚠ These are the resident agent's analysis, not owner rulings.

**The registry has no identity story, and this is the largest hole.** Any program that can
reach the port can register with any address. That program then receives the messages for
that agent. The companion spawner can also start processes. "Can claim an address" and "can
start a process" are dangerous together. ⚠ Authentication is difficult to add later,
because other vendors write the clients. A shape exists in the sibling project FerroStep:
*bind an existing authenticated identity that carries a readable role. Bind, do not mint.*

**Connection-based liveness creates a false-death case.** An agent that runs but
disconnects for a moment looks dead. The spawner then starts a second copy. Single-flight
control stops ten messages from causing ten spawns. It does not stop one spawn from
duplicating a live agent. One answer makes registration idempotent per address.

**MCP works in one direction.** The agent is the client and it calls tools. Register, send,
and acknowledge fit that direction. **Wake-up and delivery go the other way.** MCP defines
notifications from the server to the client. ⚠ **Somebody must read the 2026-07-28
specification and learn if those notifications can carry messages.** The answer decides if
one protocol is enough, or if FerroWire needs two.

**Scheduling can be a property of a message, not a subsystem.** The owner described a
scheduled task as *"another wake-up message"*. Read literally, a scheduled task is a message
with a `deliver_after` time, and the scheduler is a scan of the messages that are due. ⚠ One
question decides it: does FerroWire need repeat schedules, such as every weekday at 09:00?
A single time value cannot express a repeat schedule.

**The protocol needs a version number from the first release.** Other people write the
clients. The release that adds a version number breaks every client that has no version
number.

---

## 4. ⚠⚠ The boundary question that FerroTrack cannot answer

**Where does the registry live?**

The owner's original brief (2026-09-03) said that **one registry serves both** issue
tracking and communications routing. The split makes that statement need a new decision.

The registry has two halves, and they divide cleanly:

- **Identity** — an address names an agent. Normalization and case rules. **FerroTrack needs
  this** for the author and the assignee of an issue.
- **Presence** — is this agent connected now. **Only FerroWire needs this.**

Three answers are available, and this document does not choose one:

1. FerroWire owns the whole registry. FerroTrack asks FerroWire for identity.
2. A shared library holds identity. Each product embeds it. FerroWire adds presence.
3. Each product keeps its own. The two accept that addresses can disagree.

⚠ **One technical constraint limits answer 1 and answer 3.** FerroTrack uses redb. redb
takes a file lock and permits one writer, so **two programs cannot open one store file**.
Two products mean two stores, or one product becomes a client of the other.

---

## 5. What FerroTrack keeps

For the next agent's orientation. FerroTrack keeps issue management, the FerroStep referee,
search, and its own store. It does **not** keep routing, delivery, wake-up, scheduling, or
presence.

FerroTrack's own decisions that FerroWire can reuse or reject:

- **Store**: a ranked ladder. redb first, fjall second, our own derivative work third.
  See `store-criteria.md` and `ideal-datastore.md`.
- **Distribution**: one static binary, plus a library crate for embedders.
- **Licence**: Apache-2.0.
- ⚠ **A lesson that transfers**: `persy` is MPL-2.0, and that licence removes it from a
  future derivative work. Examine the licence of a store before you evaluate it.
