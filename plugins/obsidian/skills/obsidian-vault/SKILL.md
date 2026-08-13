---
name: obsidian-vault
description: Vault conventions and formats for writing to the Obsidian vault — note frontmatter, session-log and journal snippets, evergreen project notes, .base database views, ADR format, and clipping URLs into the vault. Use when creating or editing vault notes, session logs, ADRs, or .base files. Defers `obsidian` CLI syntax and defuddle flags to the obsidian:obsidian-cli and obsidian:defuddle skills.
---

# Obsidian vault — templates and reference

The always-loaded rules (personal style rules, folder policy, prohibitions) live
in `~/CLAUDE.md`. This skill holds the bulky reference material those rules imply.

## Companion skills

Generic Obsidian reference lives in kepano's skills. Invoke them via the Skill
tool rather than restating them here:

| Skill | Purpose |
|---|---|
| `obsidian:obsidian-cli` | `obsidian` CLI syntax and command reference |
| `obsidian:obsidian-markdown` | Valid Obsidian Flavored Markdown |
| `obsidian:obsidian-bases` | `.base` database views of notes |
| `obsidian:json-canvas` | `.canvas` visual maps and diagrams |
| `obsidian:defuddle` | Clean markdown from any URL |

**REQUIRED:** `/plugin marketplace add kepano/obsidian-skills`, then install its
`obsidian` plugin — without it every pointer above is dead. Both that plugin and
this one are named `obsidian`, so they share the `obsidian:` prefix; the skill
names do not collide.

Sibling in this plugin: `/obs-close` writes the task-closing handoff into the
session note whose format this skill defines.

## Note frontmatter

Every note created by Claude must include frontmatter:

```yaml
---
title: <note title>
date: YYYY-MM-DD
categories:
  - projects          # pluralized
tags:
  - phoenix
  - firmware
status: active        # active | done | abandoned | reference
---
```

Property rules:

- `categories` — plural nouns, maps to a `.base` overview file
- `tags` — always plural; consistent across all categories
- `rating` — integer 1–7 only, never decimals or stars
- `due` / `start` / `end` — always `YYYY-MM-DD`, never freeform
- Reuse property names across categories so Bases can query across note types

## Vault CLI

Use the `obsidian` CLI for every read and write of an existing note. The one
exception is capturing a *new* file from outside the vault — `defuddle -o`
writes it directly (see Reading URLs), and the CLI then links it.

**REQUIRED SUB-SKILL:** Invoke `obsidian:obsidian-cli` for command syntax before
running any `obsidian` command — the `key="value"` parameter form, bare flags,
`file=` vs `path=` targeting, `vault=` selection, and the full command list.
The snippets below assume that syntax; do not reconstruct it from them.

Obsidian must be open — the CLI drives the running app. If it is not, fall back
to `cat` / `echo >>` / `tee` against the vault directly:

```bash
VAULT="$HOME/Documents/Obsidian Vault"          # the only vault; `vault=` is unnecessary
printf '\n### 14:32 — <title>\n<body>\n' >> "$VAULT/<note name>.md"
```

## Session-log snippets

Start of session — create the fragment and link it from the project note:

```bash
obsidian create \
  name="$(date +'%Y-%m-%d %H%M') <project> session" \
  content="---\ntitle: <project> session\ndate: $(date +'%Y-%m-%d')\ncategories:\n  - sessions\ntags:\n  - <project>\n---\n\n## Goal\n<inferred goal>\n\n## Log\n" \
  silent

obsidian append file="<Project Note>" \
  content="\n- [[$(date +'%Y-%m-%d %H%M') <project> session]]"
```

During the session:

```bash
obsidian append file="<session note name>" \
  content="\n### HH:MM — <title>\n<What happened and why. 1–4 sentences.>"
```

## Fractal journaling (quick thoughts)

```bash
obsidian create \
  name="$(date +'%Y-%m-%d %H%M') <short idea title>" \
  content="<thought>\n\nRelated: [[<project note>]]" \
  silent
```

These accumulate in the root. Periodically review, promote salient ones, link them
into evergreen notes. Do not pre-organize them into folders.

## Reading URLs (defuddle)

Prefer `defuddle` over raw `curl` or WebFetch for any web page — it strips nav,
ads and clutter, which saves tokens.

**REQUIRED SUB-SKILL:** Invoke `obsidian:defuddle` for flags and output formats
(`--md`, `-o`, `-p <property>`, `--json`), install steps, and the cases where it
is the wrong tool.

Vault convention — clippings land in `Clippings/`, never the root, and take the
same `YYYY-MM-DD Short title.md` prefix as every other dated note. Quote the
path; clipping names contain spaces (`$VAULT` as set under Vault CLI):

```bash
defuddle parse https://docs.nordicsemi.com/... --md \
  -o "$VAULT/Clippings/2026-04-01 Nordic BLE Spec.md"
```

Then give it frontmatter — `title`, `date`, `source` (the URL it came from),
`categories: [clippings]`, `status: reference` — and a wikilink from the
relevant project note. Check the file's first line first: if `defuddle` already
emitted a `---` block, merge into it rather than prepending a second one.
Obsidian reads only the first block.

## Evergreen project note

One flat note per project, lives in vault root, never expires:

```markdown
---
title: Project Phoenix
date: 2025-01-15
categories:
  - projects
tags:
  - phoenix
  - firmware
  - embedded
status: active
---

# Project Phoenix

Wrist-worn physiological monitoring device. PPG, AGC, optical contact detection.

## Architecture
[[Phoenix Firmware Stack]] | [[ADPD4100]] | [[BLE Protocol]]

## Key Decisions
- [[ADR-0001 Use dual-slot IR OCD]]
- [[ADR-0002 Perturbation method for contact detection]]

## Sessions
- [[2026-04-01 1400 phoenix session]]

## Open Issues
- [[AGC convergence slow on Fitzpatrick V-VI]]

## References
- [[Nordic NRF52840]]
```

## Database views (.base)

Create a `.base` file in root to surface notes by category:

```yaml
# sessions.base — all session notes
filters:
  and:
    - file.hasTag("sessions")

views:
  - type: table
    name: "Recent Sessions"
    order:
      - file.name
      - date
      - status
    limit: 30
```

```yaml
# decisions.base — all ADRs across projects
filters:
  and:
    - file.hasTag("decisions")

formulas:
  age: '(today() - date(date)).days'

views:
  - type: table
    name: "All ADRs"
    order:
      - file.name
      - date
      - status
      - formula.age
    groupBy:
      property: status
      direction: ASC
```

Embed any base inside a note:

```markdown
![[sessions.base]]
![[decisions.base#All ADRs]]
```

## ADR format

Filename: `ADR-XXXX Short title.md`, flat in the vault root.

```markdown
---
title: "ADR-0001: Use dual-slot IR OCD"
date: 2026-01-20
categories:
  - decisions
tags:
  - phoenix
  - firmware
status: accepted
---

# ADR-0001: Use dual-slot IR OCD

## Context
<What situation required a decision?>

## Decision
<What was decided?>

## Rationale
<Why this over alternatives?>

## Alternatives Considered
- **Option A** — rejected: <reason>
- **Option B** — rejected: <reason>

## Consequences
<Tradeoffs and effects>

Related: [[Project Phoenix]] | [[ADPD4100]]
```
