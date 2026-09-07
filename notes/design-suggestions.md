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

⚠ **The push-direction problem moved to FerroWire on 2026-09-07** — it was always about
delivery and wake-up, not about FerroTrack's own surface, which is request/response and
fits MCP cleanly. Kept below because the analysis transfers.
⚠ **What it does not obviously cover: the push direction.** MCP is a protocol in which the
agent is the *client* calling tools. Register, file, search, move, send, ack are all
natural tool calls. **Wake-up and delivery run the other way.** MCP does define
server→client notifications, so this may fit — ⚠ **VERIFY against the current spec
(2026-07-28) rather than assuming**, because the answer decides whether one protocol serves
or two are needed.

✅ **A neat interaction if it does fit** — ⚠ now FerroWire's to use, not FerroTrack's:
connection-based liveness means "holds an open channel", and "holds an open MCP session" is
a precise, already-specified form of that.

⚠ **Cost to weigh:** MCP serves *agents*. A human web UI and FerroStep's Rust adapter are
not MCP clients, so this likely means MCP for agents, REST for everything else, and the
library crate for embedders — three surfaces over one set of invariants. That multiplies
the bypass hazard already recorded against the library crate.

## 2 and 3 — MOVED TO FerroWire, 2026-09-07

⚠⚠ Suggestion 2 (**the registry has no identity story**) and suggestion 3 (**scheduling
probably does not need a subsystem**) both concern the communication product. They moved to
[`ferrowire-handoff.md`](ferrowire-handoff.md) with their full reasoning, and are not
restated here.

## 4. Build a walking skeleton, not an isolated CAS probe

The north-star names verification as the bottleneck, and the specific owed measurement is
the conditional update inside the transaction. ⚠ **Suggest widening that slightly**: rather
than a standalone CAS test, build the thinnest possible end-to-end slice — one agent
registers, a second sends it a message, it is delivered and acked, and one issue moves
state through the referee, all against a real redb file.

✅ **It verifies more per unit of work**: CAS at the call site *with a control*, the
single-writer property, and the MCP surface — in one artifact that is also the beginning of
the product rather than a test to be thrown away. It is the cheapest way to find out whether any of the last week's decisions do not
survive contact.

## 5. Version the protocol from the first release

The query surface and the read-your-own-writes promise are contracts (item 2), held with
clients this repo does not control. ⚠ **A version in the path or handshake costs nothing on
day one and cannot be added gracefully later** — the release that introduces versioning is
the one that breaks every unversioned client.

⚠ Note MCP carries its own spec versioning, which covers the transport but **not**
FerroTrack's own tool schemas and their semantics. Those still need our version.
