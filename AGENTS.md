# Your Mother — AGENTS.md

A chores-charging Plow agent, and the third agent built on the Vigia
template — the first with **no external sources**: the data source is the
user's own word, recorded. Read this before changing anything; it says who
owns what and where a change goes.

## The one test

**Who else would have to change if this fact changed?** That owner is where
the change goes.

| Path | Owns | Never here |
| --- | --- | --- |
| `plow-pbc/plow-hermes-agent` (base) | boot, `plow-init`, gateway config, base persona, plugin pin, agent-index reporter | anything Mother-specific |
| `persona.md` | Mother's voice: dry charge, the one-word approval, honesty about what was *said* vs *done*, photo escalation etiquette | per-turn plumbing, tool how-tos |
| `skills/mother-checkup/` | the ledger: models, store, config, streaks, checkup, recap — and its SKILL.md | chat delivery (post_chat.py is its only exception, mechanically) |
| `skills/mother-recap/` | the Sunday verdict conversation | week math (engine owns that) |
| `skills/mother-onboarding/` | first contact, seed tasks, timezone, cron registration | the engine's defaults (config.py owns those) |
| `skills/mother-checkup/scripts/kit/` | generic infrastructure: clock, jsonio — domain-free | anything that knows what a chore is |
| `image/` | TZ cont-init | gateway config, plow-init, agent-index reporter — the base's |
| `Dockerfile` / `compose.yml` | how this content ships | base-image behavior |

Sibling repos [`vigia-hermes-agent`](https://github.com/emanuellcoelho/vigia-hermes-agent)
and [`fandom-hermes-agent`](https://github.com/emanuellcoelho/fandom-hermes-agent)
share this structure; `kit/` stays byte-identical across forks until a
third fork confirms the pattern graduates it into a package.

## Conventions

- **Scripts are mechanical.** One JSON object on stdout, exit 0/1/2, no
  human words. The SKILL.md files are the voice. No prompt text in Python.
- **stdlib only.** No new dependencies in scripts or the image.
- **The engine is pure** (`mother/engine/`): dates in, numbers out, no IO,
  no clock reads — time comes from `kit.clock`, injectable in tests. The
  escalation rule (photo after `proof_after` misses) lives in
  `checkup.py`, provable without a clock.
- **Honesty is architectural**: the ledger records what the user SAID
  (`done`/`skip`), and only `checkup --mark-missed` charges silence. The
  engine never invents an answer, and a later word overwrites the same
  day's entry on the record.
- **Writes are atomic** (`kit.jsonio.save_json_atomic`). The store file is
  a contract: `mother/models.py` owns its shape, explicitly serialized.
- **State lives in `/var/lib/hermes/mother/`** (override with
  `MOTHER_HOME` for tests). Nothing under this tree carries a credential,
  a chat id, or a person's data beyond the chore list they chose.
- **TZ is fixed at boot** from `mother/config.json`
  (`image/cont-init.d/10-mother-timezone`). `hermes cron create` takes no
  per-job zone, so onboarding refuses to register schedules while the
  container's zone and the config disagree — a restart applies the zone
  first.

## Commits

Conventional, scoped by the table above, imperative, one concern per commit:
`feat(engine):`, `feat(skills):`, `fix(...)`, `docs:`, `chore:`. The body
says **why**; the diff says what. Never in a commit: `plow-credentials`,
state files, anything under a `MOTHER_HOME`. A pin bump
(the base digest) is its own commit naming what moved
and why.

## Tests

    python3 -m pytest tests/ -q

Pure and fixture-fed: no network, no clock sleeps. A change to streak
math, escalation, or store shape starts in `tests/`.

## Schedules (the registered spec)

| name | schedule (container TZ) | deliver |
| --- | --- | --- |
| `mother-checkup` | `30 21 * * *` (onboarding writes the user's time) | post_chat.py per pending task; all done = one "ok." |
| `mother-recap` | `0 20 * * 0` (Sunday) | post_chat.py weekly verdict |

Changing these rows is an edit to `skills/mother-onboarding/SKILL.md` and
this table together.

## Forking this into a new agent

The structure is deliberately the template:

1. Copy the tree; change `persona.md` and the skills' domain layer
   (`mother/` → your domain; `kit/` stays).
2. Your data seam here is the user's word, not a source adapter. Agents
   with external data keep Vigia's/Fandom's `sources/` shape instead;
   `alerts.py`-style purity still applies.
3. New `AGENT_ID`, new registration (`agent_index_client.py --register`),
   new compose `AGENT_ID`.
4. LICENSE stays MIT; NOTICE keeps the Apache attributions.

Three siblings confirm the pattern — `kit/` graduates into its own package
when a fourth arrives.