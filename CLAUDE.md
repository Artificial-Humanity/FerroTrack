# FerroTrack

**[AGENTS.md](AGENTS.md) is this repo's rules of record. Read it before you do anything else.**
It is not loaded for you — only this file is — so nothing else will put it in front of you.

## Your role is assigned in config.yaml, not assumed

**If nothing in your system prompt says otherwise, you are the repo's default agent.**
`default_agent` in [`config.yaml`](FerroStep/config.yaml) names your entry, and the entry carries
your title, your name, your commit identity, and the path to your persona. That persona is
imported on the next line, so it is already in your context by the time you read this
sentence.

@FerroStep/personas/DEVELOPER.md

⚠ The `@import` above is a literal path — imports cannot read `config.yaml` — so it is the
one deliberate second copy of the default agent's `persona` value. Change the two together.

**Keep this file short.** It exists to route and to import; the rules live in `AGENTS.md`,
the procedures in the persona, and the configurable values in [`FerroStep/config.yaml`](FerroStep/config.yaml). Anything
restated here becomes a second copy to drift.
