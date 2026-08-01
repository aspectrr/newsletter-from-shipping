# newsletter-from-shipping

Turn a week of work into a newsletter teardown — automatically gathered from meetings, email, code, docs, tickets, and coding sessions — drafted in your voice, and published after you edit.

**It's just a skill.** No server, no CLI bridge of its own. The agent gathers activity from six sources, picks the one story worth telling, and drafts it into [redline](https://github.com/aspectrr/redline) using a teardown format. You edit. Voice compounds. Then it publishes.

Built on [content-from-shipping](https://github.com/aspectrr/content-from-shipping), which handles the session→draft→edit→learn loop for individual devlogs. This skill extends that loop to weekly newsletters that synthesize across your entire workstream.

## The loop

```
  6 SOURCES (last 7 days)          redline draft    →  redline app inbox
  ──────────────────────                                    │ you edit
  meetings (hyprnote/Anarlog)                               ▼
  email (Gmail)             redline finalize  ◀────────  done
  code (GitHub PRs)               │
  docs (Google Docs)              ▼
  tickets (Linear)          redline add-lesson + add-pattern  →  voice corpus
  sessions (pi JSONL)              │
                                   ▼
                          Resend broadcast + social thread draft
```

1. **Gather** — the agent pulls the last 7 days from all six sources in parallel.
2. **Filter** — picks the ONE story worth a teardown. Most weeks produce one. If nothing qualifies, it tells you and skips.
3. **Calibrate** — reads accumulated voice lessons and patterns from redline before writing a word.
4. **Draft** — writes the teardown (what we built / architecture / what broke / numbers / takeaway), auto-lints against your voice.
5. **You edit** — in the redline app.
6. **Learn** — the agent reads the draft→final diff, derives voice lessons, stores them. Future drafts get sharper.
7. **Publish** — sends the finalized version via Resend. Drafts social threads for manual posting.

## What's here

Just one thing: `skills/newsletter-from-shipping/SKILL.md` — the skill.

## Install

```sh
git clone git@github.com:aspectrr/newsletter-from-shipping.git
ln -sfn "$PWD/newsletter-from-shipping/skills/newsletter-from-shipping" ~/.pi/agent/skills/newsletter-from-shipping
```

### Prerequisites

- [redline](https://github.com/aspectrr/redline) installed (`redline` on PATH). For the MCP server (preferred — no subprocess per call), add to pi's config:
  ```json
  { "mcpServers": { "redline": { "command": "redline", "args": ["mcp"] } } }
  ```
- [Resend](https://resend.com) account with a verified domain + API key for publishing.
- Source connections (the agent degrades gracefully if some are missing):
  - **Meetings:** [hyprnote](https://hyprnote.com)/Anarlog local DB (via the `meetings` MCP or CLI)
  - **Email:** Gmail (via `google_workspace` MCP)
  - **Code:** GitHub (`gh` CLI)
  - **Docs:** Google Docs (via `google_workspace` MCP)
  - **Tickets:** Linear (via `orca linear` CLI)
  - **Sessions:** pi agent session files at `~/.pi/agent/sessions/`

Then in any pi session: "draft this week's newsletter." The skill activates.

## Why a skill, not an app?

Same reason as content-from-shipping: the agent can gather and synthesize directly, and redline already owns the draft→edit→diff→learn loop. This skill just adds the gather-from-everywhere step and the teardown format on top. No server to maintain, no API to break.
