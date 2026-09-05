# Design suggestions — the resident's, 2026-09-05

⚠⚠ **NONE OF THIS IS RULED. Every item here is the resident agent's proposal, offered when
the owner asked for suggestions, and none of it binds anything.** The file is separate from
`AGENTS.md` for exactly that reason: that file records what the owner decided, this one
records what was suggested. ⚠ If a line from here is ever cited as a constraint, it has
been promoted — which is the failure this workspace named on 2026-09-04 and is still
paying down.

---

## 1. Make MCP the inter-brand protocol

**The gap:** `AGENTS.md` item 9 establishes that the interface must be a network protocol
rather than a Rust API, because agents in other vendors' runtimes cannot link a crate. It
does not say *which* protocol, and a bespoke REST API means writing and maintaining a
client for every vendor FerroTrack wants to reach.

**The proposal:** expose FerroTrack's agent-facing surface as an **MCP server**.

Verified 2026-09-05: MCP was donated to the Linux Foundation's Agentic AI Foundation in
December 2025 (backed by AWS, Google, Microsoft, OpenAI, Cloudflare, Bloomberg) and has
native support across Claude, ChatGPT, Gemini, Microsoft Copilot, VS Code and Cursor, at
roughly 97M monthly SDK downloads as of March 2026.

✅ **This is not an additional surface — it is the answer to a question the brief asks and
has not answered.** One MCP server reaches every MCP-capable runtime with **no bespoke
client per vendor**, which is the inter-brand requirement almost exactly. It reduces work
rather than adding it.

⚠ **What it does not obviously cover: the push direction.** MCP is a protocol in which the
agent is the *client* calling tools. Register, file, search, move, send, ack are all
natural tool calls. **Wake-up and delivery run the other way.** MCP does define
server→client notifications, so this may fit — ⚠ **VERIFY against the current spec
(2026-07-28) rather than assuming**, because the answer decides whether one protocol serves
or two are needed.

✅ **A neat interaction if it does fit:** liveness is connection-based (item 2), and "holds
an open MCP session" is a precise, already-specified meaning for *connected*.

⚠ **Cost to weigh:** MCP serves *agents*. A human web UI and FerroStep's Rust adapter are
not MCP clients, so this likely means MCP for agents, REST for everything else, and the
library crate for embedders — three surfaces over one set of invariants. That multiplies
the bypass hazard already recorded against the library crate.

## 2. The registry has no identity story, and that is a hole

**Nothing in the brief or the record says who may register as whom.** As specified, anyone
who can reach the port can register under any address — and therefore receive that agent's
messages. With a companion spawner that starts processes, "may claim an address" and "may
cause a process to launch" sit uncomfortably close together.

⚠ **Authentication retrofits badly**, and it retrofits worst into a protocol whose clients
you do not control — which is the position item 9 puts FerroTrack in deliberately.

✅ **A shape already exists in the sibling contract**: FerroStep's acceptance list, item 5,
is *bind an existing authenticated identity carrying a readable role — bind, don't mint*.
FerroTrack currently has no answer to that item, and it is the one the registry most needs.
Worth deciding before the protocol is published rather than after.

## 3. Scheduling probably does not need a subsystem

Requirement 4 describes scheduled tasks arriving *"as another wake-up message"*. Taken
literally, **a scheduled task is a message with a `deliver_after` timestamp** — and the
scheduler is then a range scan over due-time on the message partition, which criterion 3
already requires the store to support.

✅ **If that holds, requirement 4 needs no cron parser, no separate timer service, and no
fourth workload — it collapses into requirement 2's machinery.** Four workloads become
three, and the sweep job the TTL already needs is the same loop that fires due messages.

⚠ Where it might not hold: recurring schedules ("every weekday at 9"). A single
`deliver_after` expresses one-shot delivery only. Whether recurrence is wanted is an owner
question, and it is the one thing that would justify a real scheduler.

## 4. Build a walking skeleton, not an isolated CAS probe

The north-star names verification as the bottleneck, and the specific owed measurement is
the conditional update inside the transaction. ⚠ **Suggest widening that slightly**: rather
than a standalone CAS test, build the thinnest possible end-to-end slice — one agent
registers, a second sends it a message, it is delivered and acked, and one issue moves
state through the referee, all against a real redb file.

✅ **It verifies more per unit of work**: CAS at the call site *with a control*, the
connection-based liveness model, at-least-once delivery, and the single-writer property —
in one artifact that is also the beginning of the product rather than a test to be thrown
away. It is the cheapest way to find out whether any of the last week's decisions do not
survive contact.

## 5. Version the protocol from the first release

The query surface and the read-your-own-writes promise are contracts (item 2), held with
clients this repo does not control. ⚠ **A version in the path or handshake costs nothing on
day one and cannot be added gracefully later** — the release that introduces versioning is
the one that breaks every unversioned client.

⚠ Note MCP carries its own spec versioning, which covers the transport but **not**
FerroTrack's own tool schemas and their semantics. Those still need our version.
