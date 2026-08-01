---
name: newsletter-from-shipping
description: >-
  Gather a week of work from six sources (meetings, email, code, docs, tickets,
  sessions) and draft a newsletter teardown issue into redline, then publish
  after edit. Turns recent shipping into a weekly newsletter in the user's
  voice. Activate when the user wants to draft a newsletter from recent
  activity, write the weekly issue, or publish after editing.
---

# newsletter-from-shipping

Turn a week of work into a newsletter teardown. This skill extends [content-from-shipping](https://github.com/aspectrr/content-from-shipping) — which drafts individual devlogs from single sessions — to weekly newsletters that **synthesize across your entire workstream.**

The agent gathers from six sources, picks the one story worth telling, drafts it in the teardown format, pushes to redline for editing, learns from the diff, and publishes after you confirm.

## Two tools, one loop

```
  6 SOURCES (last 7 days)          redline draft    →  redline app inbox
  ──────────────────────                                    │ you edit in the app
  meetings (hyprnote/Anarlog)                               ▼
  email (Gmail)             redline finalize  ◀────────  done
  code (GitHub PRs)               │
  docs (Google Docs)              ▼
  tickets (Linear)          redline add-lesson + add-pattern  →  voice corpus
  sessions (pi JSONL)              │
                                   ▼
                          Resend broadcast + social thread draft
```

The gather + filter + draft half is this skill's job. The store-drafts, compute-diffs, lint-against-voice, learn-from-edits half is **redline's job.** If the `redline` MCP server is connected, prefer its tools over shelling out.

## Configuration

Before the first run, establish these values (ask the user if unknown):

- **Newsletter name** — e.g. "The Cactus Dispatch"
- **From address** — e.g. "dispatch@example.com" (must be a Resend-verified domain)
- **Resend audience ID** — the audience to broadcast to
- **Email account** — which inbox to gather from (for the gmail step)
- **GitHub org/scope** — which org or repos to check for merged PRs
- **Content format** — default is the teardown format below; the user may prefer a different beat structure

Store these mentally for the session. The user may have a `NEWSLETTER.md` or similar doc with these details — check the repo root.

---

## Workflow A — draft the weekly issue

### Step 1. GATHER (run in parallel)

Pull the last 7 days from all six sources. If a source tool is disconnected, gather what you can from the others and note the gap — don't block on one missing source.

| Source | How | What to look for |
|--------|-----|------------------|
| **Meetings** | `meetings_list_meetings` (MCP) or meetings CLI | Shipped work, decisions, client wins |
| **Email** | `google_workspace listMessages q:"newer_than:7d"` | Client updates, deals, project milestones |
| **Code** | `gh pr list --state merged --search "merged:>{7d-ago-ISO}" --json title,body,url` across configured org/repos | Real builds, deployments, refactors |
| **Docs** | `google_workspace listDocuments modifiedAfter:"{7d-ago-ISO}"` | Proposals, playbooks, audits, specs |
| **Tickets** | `orca linear list --filter completed --workspace all --json` | What shipped, what moved to done |
| **Sessions** | `find ~/.pi/agent/sessions -name '*.jsonl' -mtime -7 \| sort -r` | Thinking blocks with high narrative signal — the "why" and "what was hard" |

For sessions, extract signal fast without loading whole files:

```bash
jq -rc 'select(.type=="message") | .message.content[]? |
  if .type=="text"     then "TEXT:  " + (.text | gsub("\n";" "))
  elif .type=="thinking" then "THINK: " + (.thinking | gsub("\n";" "))
  else empty end' <session.jsonl>
```

### Step 2. FILTER

From all gathered material, identify the **ONE** best teardown candidate. Not every week qualifies. Selection criteria:

- (a) A clear build or deployment happened
- (b) Something broke or required a course correction
- (c) Real numbers exist (cost, latency, accuracy, time saved)
- (d) It demonstrates expertise the newsletter's audience cares about

Most weeks yield one strong story. **If nothing meets all four criteria, say so — don't force it.** A skipped week is better than filler. Tell the user what you found and why nothing rose to teardown level.

### Step 3. CALIBRATE VOICE

Before writing a word:

```bash
redline lessons                    # voice rules derived from past edits
redline list-patterns              # matchable patterns the lint engine enforces
```

These are your constraints. Apply every applicable lesson, avoid every pattern. On cold start (empty), write to the user's general voice — direct, no hedging, receipts over rhetoric.

### Step 4. DRAFT

Write the teardown using the format below. Write to a temp markdown file.

#### Default teardown format

```
## What we built

[2-3 sentences. The thing, the context, the goal. No setup
padding — straight into it.]

## The architecture

[How it works. The stack, the flow, where the agent sits. Name
real tools and models.]

## What broke

[The failure modes. The wrong assumptions. This is the section
nobody else publishes — it's the whole reason people subscribe.
Be specific: "We assumed X. It cost us Y hours."]

## The numbers

- Cost per [unit]: $X
- Latency: Xs
- Accuracy: X%
- Time saved: X hrs/week

[Only real measurements. If you don't have a number, don't invent one.]

## The takeaway

[One paragraph. The one thing the reader should try this week.
A prompt pattern, a cost lever, a workflow fix. Actionable or
it didn't happen.]
```

**Rules:**
- Receipts over rhetoric. Every claim has a number or it gets cut.
- "What broke" is the moat — that's the section nobody else publishes.
- One takeaway, not three.
- The user's configured content format may differ — check for a `NEWSLETTER.md` or ask.

### Step 5. PUSH TO REDLINE

```bash
redline draft issue-N.md --context "newsletter: <newsletter name> issue N" --tags newsletter,content
```

Auto-lints against voice patterns. If violations: rewrite the file to fix them, then `redline delete-draft <id>` + re-push. Repeat until clean. Note the **draft id**.

### Step 6. HAND OFF

Tell the user the draft is in redline. They edit there. **Do NOT publish until they confirm the edit is done.**

---

## Workflow B — publish after edit (run when the user says they're done)

### Step 7. FINALIZE

```bash
redline finalize <draft_id>
```

Returns the **pair id** plus **diff analysis** (deletions, additions, word swaps, categorized changes, existing-pattern hits).

### Step 8. LEARN

Derive 1–3 voice lessons from the diff analysis. **Deletions are the strongest signal** — what got cut entirely is what the user's voice rejects.

Store each lesson **with a matching pattern:**

```bash
redline add-lesson <pair_id> "<specific, actionable lesson>" --tags newsletter,content
redline add-pattern --rule "<what the pattern enforces>" --pattern "<literal or regex>" --category style
```

Always pair lesson + pattern. Lessons without patterns don't lint — future drafts won't catch the issue. See *What counts as a good lesson* below.

### Step 9. PUBLISH NEWSLETTER

Send the **finalized** (edited) version — never the draft — via Resend. Two options:

**(a) Resend dashboard** — Broadcasts → compose with the edited markdown → send to the configured Audience. Simplest.

**(b) Resend API** — `resend.emails.send` (or the Node SDK) with:
- `from`: the configured from address (verified domain)
- `to`: the configured Resend audience
- `subject`: `Teardown #N: <headline>`
- `html`: the rendered final text

Use `RESEND_API_KEY` from env. **Never publish the draft — only the finalized version the user edited.**

### Step 10. DRAFT SOCIAL THREADS

Extract "What we built" + "The takeaway" into a 4–6 post thread for X/LinkedIn. Write to a temp file. The user pastes manually — do **not** attempt automated social posting (X API costs $100/mo, LinkedIn requires app review). When volume justifies it, add API integration here.

---

## What counts as a good lesson

Specific, actionable, voice-coded. Names a swap or structural move you can repeat; about *how the user writes*, not correctness.

Good:
- "Open teardowns with the tension, not a week-in-review."
- "Cut hedging: 'I think we should' → 'We should.'"
- "Never end a section with a significance summary."

Bad (reject):
- "Be clear and engaging." (generic)
- "Mention the product name." (content, not voice)

**Negative lessons are gold** — things the user never does. Capture them. Don't over-fit: one sighting is a candidate (pattern starts `unconfirmed`); it auto-promotes after 3+ sightings.

---

## Failure modes

- **Forcing an issue when nothing qualifies.** Tell the user "nothing worth a teardown this week." Better to skip than ship filler.
- **Publishing before edit.** The publish step only runs AFTER the user confirms. Sending raw agent output to subscribers burns trust.
- **Treating this as content-from-shipping.** Newsletter gathers from ALL six sources and uses the teardown format — it's not a single-session devlog.
- **Blocking on one missing source.** Degrade gracefully. Note the gap and continue.
- **Storing lessons without patterns.** Patterns are what make future drafts auto-lint. Always pair them.
- **Over-fitting to one edit.** Patterns start `unconfirmed` and auto-promote after 3+ sightings — let the system handle confirmation.
