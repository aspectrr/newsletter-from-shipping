---
name: newsletter-from-shipping
description: >-
  Gather a week of work from six sources (meetings, email, code, docs, tickets,
  sessions) and draft a newsletter teardown issue into the madcactus dashboard,
  where Collin edits and publishes
---

# newsletter-from-shipping

Turn a week of work into a newsletter teardown that **any business owner can act on.** This skill extends [content-from-shipping](https://github.com/aspectrr/content-from-shipping) — which drafts individual devlogs from single sessions — to weekly newsletters that synthesize across your entire workstream.

The agent gathers from six sources, picks the one story worth telling, **abstracts it into a generalizable pattern**, drafts it in the teardown format, pushes it to the **madcactus MCP server** (docs + voice lint + LinkedIn posts), and the user edits and publishes from the dashboard. The agent never sends.

## The audience (ICP, set 2026-09-12)

**Operators of companies doing $30-70M revenue who are trying to use AI to make more money.** Write to the existing client persona: an operator running a real company, paying for AI-built outreach and software, who directs the work but does not build it. The newsletter's job is to find more people like that client.

They **direct**, they don't **build.** The newsletter is their blueprint for what to ask their team or AI to do. They have someone (internal dev, contractor, consultant, agent) who runs the tools — every blueprint is a forward, not a do.

They want:
- **The full recipe** — not just the prompt, but the stack, so they can hand it to whoever runs their AI
- **Verification methodology** — how to check that the AI's work is correct. They fear invisible or fabricated output. Give them the Monday-meeting question they can ask without knowing the tech.
- **Real business outcomes** — money saved, decisions enabled, what happened when the tool shipped. ROI proof, not theory.
- **Generalizable patterns** — the insight must apply to any operator, not just one client's niche

They are NOT developers. Name the tools (Claude Code, Supabase, Fly.io) but never the implementation minutiae (TLS fingerprinting, RSC payload parsing). Frame stakes in operator units: a month of outreach credits, a quarter of stale lists, an invoice approved — not $50 of API credits. Small dollar amounts shrink the story; time-and-revenue stakes grow it.

## Two tools, one loop

```
  6 SOURCES (last 7 days)              madcactus_create_newsletter / create_post  →  dashboard drafts
  ──────────────────────                                                             │ Collin previews, edits, schedules
  meetings (meetings CLI)                                                            ▼ in the dashboard. You never send.
  email (madcactus_list_emails)
  code (gh CLI: aspectrr + Mad-Cactus orgs)      madcactus_get_doc_versions + get_doc_diff
  docs (google_workspace / internal docs)             │ (after Collin edits)
  tickets (orca linear)                              ▼
  sessions (pi JSONL)                          madcactus brain cycle → new voice lessons + lint patterns
  slack (slack MCP)
```

## Configuration

**NEWSLETTER.md at the madcactus.org repo root is the source of truth** for name (The Cactus Dispatch), subject format, beat structure (TL;DR + tension + what we did + the pattern + what broke + what happened + blueprint + numbers + P.S.), email/web channel markers (`Subject:` front-matter line, `<!-- email-only -->` / `<!-- web-only -->`), UTM templates, give ratio (3:1), and the publishing workflow. **Read it before drafting; it supersedes any beat structure described elsewhere.**

Standing facts: sending infra (Resend segment, domain, signup endpoint) is already wired; the agent never sends anything — madcactus drafts only, Collin previews/edits/schedules in the dashboard.

Source scope (verified 2026-09-12):
- GitHub: both orgs — `aspectrr/*` (side projects) and `Mad-Cactus/*` (client + product repos: madcactus.org, cdl, *-brain)
- Issue numbering: check the latest published issue first (Issue 01 went out 2026-09-10; this run drafted Issue 02)
- Google Docs has gone stale (internal docs moved to the dashboard); degrade gracefully
- Slack (MCP) may return empty; email via `madcactus_list_emails` is the richer inbox source anyway

---

## Workflow A — draft the weekly issue

### Step 1. GATHER (run in parallel)

Pull the last 7 days from all six sources. If a source tool is disconnected, gather what you can from the others and note the gap — don't block on one missing source.

| Source | How | What to look for |
|--------|-----|------------------|
| **Meetings** | `meetings list --json` then filter `.created_at >= <7d-ago>` (`started_at` is empty on active sessions; titles + `meetings show "<title>"` carry the story) | Shipped work, decisions, client wins |
| **Email** | `madcactus_list_emails` (MCP) | Client updates, deals, project milestones |
| **Code** | `gh api repos/<org>/<repo>/commits?since=<ISO>` looped over `gh repo list aspectrr` AND `gh repo list Mad-Cactus` — commits, not just merged PRs | Real builds, deployments, refactors |
| **Docs** | `google_workspace listDocuments` (often stale — internal docs live in the dashboard now) | Proposals, playbooks, audits, specs |
| **Tickets** | `orca linear list --filter completed --workspace all --json` | What shipped, what moved to done |
| **Sessions** | `find ~/.pi/agent/sessions -name '*.jsonl' -mtime -7 \| sort -r` | Thinking blocks with high narrative signal — the "why" and "what was hard" |
| **Slack** | slack MCP `slack_slack_search_public_and_private` (may be empty; don't block) | Team chatter, client coordination |

For sessions, extract signal fast without loading whole files:

```bash
jq -rc 'select(.type=="message") | .message.content[]? |
  if .type=="text"     then "TEXT:  " + (.text | gsub("\n";" "))
  elif .type=="thinking" then "THINK: " + (.thinking | gsub("\n";" "))
  else empty end' <session.jsonl>
```

### Step 2. FILTER AND ABSTRACT

From all gathered material, identify the **ONE** best teardown candidate. Not every week qualifies. Selection criteria:

- (a) A clear build or deployment happened
- (b) Something broke or required a course correction
- (c) Real numbers exist (cost, latency, accuracy, time saved)
- (d) It demonstrates expertise the newsletter's audience cares about
- **(e) THE GENERALIZABILITY TEST: Can you state the insight as a pattern that applies to ANY business — not just this client?** If the story only makes sense for one company's niche, it fails. The CDL customs-data scraper is not a pattern. "Use coding agents to build a custom tool that tests your sales hypothesis in an afternoon" IS a pattern.
- **(f) THE OUTCOME TEST: Is there a real business result — a decision made, money saved, time recovered, a meeting that went differently?** If the project is still in progress with no known outcome, the story may not be ready. Honest uncertainty is acceptable; fabrication is not.

**These two filters are the most important.** If the work was impressive but only meaningful in one niche, find the generalizable lesson inside it — or skip the week.

Most weeks yield one strong story. **If nothing meets all five criteria, say so — don't force it.** A skipped week is better than filler. Tell the user what you found and why nothing rose to teardown level.

### Step 3. CALIBRATE VOICE (madcactus, per surface)

The write tools hard-reject you unless you call this first, **per surface you will write** (newsletter AND post), with your session id as `chat_uuid`, within 1h of the write:

```
chat_uuid = $PI_SESSION_ID
madcactus_get_voice_lessons { chat_uuid, surface: "newsletter" }
madcactus_get_voice_lessons { chat_uuid, surface: "post" }
```

Rules that always apply: zero em-dashes (Collin deletes every one), concrete character/case openers, specific product names (Claude Code over "a coding agent"), blunt claims without hedges, rounded numbers, no taglines or rhetorical flourishes, finish every thought.

### Step 4. DRAFT

Write the teardown using the format below. Write to a temp markdown file.

#### Teardown format — the six beats

**Beat 1 — The tension** (2-3 sentences)
The business problem any CEO recognizes. Not the tech — the pain. If a CEO wouldn't nod at this, rewrite it.

**Beat 2 — What we did** (4-6 sentences)
The story, **generalized and tooled.** What was built, what it proved, AND the stack — name the tools as part of the narrative: "We used a coding agent (Claude Code) to build a Python scraper and a Streamlit dashboard." The reader should know exactly what tools were used by the end of this beat. Anonymize clients. Abstract niches into universal problems.

**Beat 3 — How we knew it was right** (3-5 sentences)
The verification methodology. This is the beat that builds trust and teaches the reader's most important AI skill: **how to check the agent's work.** How did you confirm the output wasn't hallucinated? Did you cross-reference against a live API? Run a test loop? Spot-check by hand? Feed it a real record and watch it correct itself?

This beat exists because CEOs fear AI making things up. Every issue teaches one verification technique. If you didn't verify, say so — and explain what could go wrong if you hadn't.

**Beat 4 — What happened** (3-5 sentences)
The business outcome. What decision did the tool enable? What happened when it was shown to stakeholders? What money was saved, what time was recovered, what did the team do differently the next day? This is the ROI proof. Without this beat, the newsletter is a tech blog. With it, it's a business case for AI.

If the outcome isn't known yet (project still in progress), say what the EXPECTED outcome is and when you'll know. "The dashboard goes into next Tuesday's sales meeting. We'll report the outcome in issue N+1." Honest about uncertainty is better than fabricated results.

**Beat 5 — The blueprint** (the handoff — this is why people subscribe)
Everything the reader needs to hand this to their team or AI assistant. Format:

```
THE STACK
- What to install: [Claude Code / Codex / pi, Python, Streamlit, etc.]
- Skills that help: [name specific skills if relevant]
- What it costs: [monthly run cost, if known]

THE PROMPT
Hand this to whoever runs your coding agent:

"I have [type of data]. Here's a sample: [paste one real record].
Build me a tool that [the pattern from beat 2]. Output a Streamlit
dashboard so the team can browse the results."

HOW TO CHECK THE WORK
[One sentence from beat 3, simplified for handoff]
```

Every issue ships a blueprint. No exceptions. If you can't write one, the story isn't ready.

**Beat 6 — The numbers** (bullet list)
Build time, cost to run, what it produced (leads found, decisions enabled, time saved). Don't claim what it "replaces" unless you genuinely replaced a specific tool — instead frame as expected value: what did this tool produce and what would that cost to get another way? Only real measurements.

**Footer:**
```
*<Newsletter name> turns real AI deployments into patterns you can use.
[CTA link]*
```

**Rules:**
- **Generalize the specific.** Anonymize clients. Abstract niches into universal patterns.
- **Name the stack.** The reader needs to know what tools were used and what to install. This is not optional — it's the recipe.
- **Every issue teaches one verification technique.** The "how we knew it was right" beat is mandatory.
- **Every issue ships a blueprint.** Stack + prompt + verification instruction, formatted for handoff.
- **Real outcomes or honest uncertainty.** Don't fabricate ROI. If the outcome isn't known, say so.
- **The audience directs, they don't build.** Write for a CEO who will hand this to their team.

### Step 5. PUSH TO MADCACTUS

Lint first, then create (the create call re-lints and hard-rejects violations):

```
madcactus_lint_voice_text { text, surface }   # fix every avoid-violation, repeat until clean
madcactus_create_newsletter { title: "Issue NN: <short name>", markdown, lessons_reviewed: true, chat_uuid }
```

Title convention: `Issue 02: the AI that lost 10,000 companies` (Issue 01: `warmest sales prospects` — lowercase descriptive). The markdown's first line is the `Subject:` front-matter. Note the **doc id**.

### Step 6. HAND OFF

Tell the user the draft (and posts) are in the dashboard. They edit, preview both channels (email + web), and hit **Publish now** or **Schedule** there. **Do NOT publish, send, or touch Resend yourself.**

---

## Workflow B — after Collin edits (run when the user says they're done or next run)

### Step 7. CHECK THE EDITS

```
madcactus_list_doc_versions { doc_id }   # versions after v1 are Collin's edits
madcactus_get_doc_diff { doc_id, ... }
```

Deletions are the strongest signal — what got cut entirely is what the voice rejects.

### Step 8. LEARN

The brain cycle extracts voice lessons from edits automatically (and now proposes lint patterns too — `madcactus_run_brain_cycle` if a refresh hasn't run). Agent-side: derive 1–3 candidate lessons from the diff and surface them to the user; only the confirmed, pattern-paired ones teach future drafts. Don't over-fit to one edit.

### Step 9. PUBLISH NEWSLETTER

**The agent does not publish.** Collin previews both channels in the dashboard DocEditor (email preview = exact send bytes, web preview = real /newsletter/<id> render) and hits **Publish now** (scheduler sends within 60s via Resend + publishes the web page) or **Schedule**. Editing after publish updates the web page live and never re-sends the email.

### Step 10. DRAFT LINKEDIN POSTS (3 per issue)

Each issue yields **three standalone LinkedIn posts**, one per archetype. Not a thread — three independent posts scheduled across the week. Write to `linkedin-issue-N.md` alongside the issue draft.

**Post 1 — The tension (Tue).** Beat 1 + one concrete detail from Beat 2. Story-shaped, no payload. Ends with the comment gate: "Comment '<keyword>' and I'll send you the code." Curiosity gap, not summary.

**Post 2 — The numbers (Thu).** Beat 6 + Beat 4. Lead with the number as the hook line. "$14,000 of consulting work started with a Python script that took 40 minutes to build." Then 4–6 short lines of context. Same comment-gate CTA.

**Post 3 — The blueprint (Sat/Mon).** Beat 5's prompt, lightly compressed. The post IS the gift — a prompt the reader can copy. "Steal this prompt" framing. CTA carries the repo shortlink inline (give-first post, direct link allowed) plus "More like this every Tuesday → [newsletter shortlink]".

**Format rules for every post:**
- Hook = first line must work alone (LinkedIn truncates at ~2 lines before "see more"). Number or contrarian claim, never a setup sentence.
- One idea per post. If a second idea appears, it's next week's post.
- Short paragraphs, 1–3 sentences. White space is the format.
- External link in first comment for Posts 1–2 (LinkedIn dampens post-body links); Post 3 may carry it inline.
- ≤3 hashtags, or none.
- Written at the same reading level as the newsletter — CEO who directs, doesn't build.

Push all three as **separate `madcactus_create_post` calls** (each carries `chat_uuid` + `lessons_reviewed: true`; get_voice_lessons must have been called with `surface: "post"`). They land in the dashboard Posts tab for Collin to schedule (post to personal profile first, company page reposts — founder-led B2B, LinkedIn throttles company-page reach). First 4 weeks: user edits every post heavily — that's how the voice lessons accumulate for social copy specifically.

### Step 11. LEAD MAGNET — the teardown repo

Every blueprint becomes runnable code. ONE public repo holds all teardowns: `Mad-Cactus/newsletter-teardowns`, one folder per issue (Issue 01 = `issue-01-prospect-scrape/`, Issue 02 = `issue-02-data-loss-audit/`). One repo > repo-per-issue: a reader who lands once sees every past teardown, and the folder list is itself the archive. Structure per folder:

```
README.md            # Beat 5 verbatim: stack, prompt, verification
example/             # runnable code + one real (anonymized) sample record
expected-output/     # what good output looks like
```

**Distribution (settled 2026-09-13):**
- **Newsletter issues embed the folder link directly, inside the blueprint itself** — one line after the prompt telling the reader's agent to open the folder and compare its work against the working example. Subscribers already paid with their email; gating the link inside the issue insults them. Being on the list is the perk.
- **LinkedIn Posts 1–2 use the comment gate.** The repo-link line comes OUT of the post body; the CTA becomes "Comment '<keyword>' and I'll send you the code." One keyword per issue, set by Collin (Issue 01 = `prospect`, Issue 02 = `audit` — matches the issue's email P.S. reply keyword). Comments boost reach; every reply becomes a warm DM. Collin sends the link personally — that handoff is a do-things-that-don't-scale touchpoint, and the DM carries ONE question about the reader's situation ("what did you point it at?"), not a pitch. Every reply is customer validation.
- **Post 3 (blueprint) carries the shortlink inline** — it's the give-first post; gating the thing you just gave away reads as a trick.
- **The repo stays PUBLIC.** The reply is the gate, not repo access. Access-request approvals add friction that kills the funnel.

**Tracking:** create a dashboard shortlink per channel (newsletter / post-1 / post-2 / post-3) pointing at the repo URL with UTMs (`utm_source=newsletter|linkedin&utm_medium=social&utm_campaign=issue-NN`). Use the shortlink in the issue, the post inline, and the DMs so click counts separate channels.

The repo work happens at draft time, before Step 6 handoff: ask Collin for the issue's gate keyword, create the folder stub, fill README + example from Beat 5, update the repo-root index table, then wire the folder link into the newsletter blueprint and the gate CTAs into the posts.

---

## Run log

**2026-09-12 (run 1, Issue 02 + 3 posts).** What worked: madcactus hard gates enforced voice cleanly (one violation: "a coding agent" → "Claude Code"); meetings CLI summaries carried the entire story (CDL data-loss saga, 4 sessions, $50 credits, 3 DB copies, 7 worktrees, first client-side PR); gh CLI across both orgs took one loop. Gaps/wishes for future runs:
- GitHub teardown repo per issue is still missing (NEWSLETTER.md wants a repo link with the prompt + sample data as lead magnet). Candidate: one public repo per blueprint, stub is fine — ask Collin before creating.
- Slack returned empty this week; don't burn turns on it — email + meetings cover it.
- Google Docs is stale (internal docs moved to the dashboard); check `madcactus_search_workspace` first next time.
- Outcome check for Issue 01 (PostHog: 16 visitors, +300%) belongs in the next issue's P.S. or numbers if relevant.
- Give ratio tracker: issues 01, 02 are pure give. First ask allowed at issue 04.

**Run 1 human-edit lessons (v2, Collin):** "about $50 of API credits" → "a month's worth of API credits"; "about $50 of credits" → "planning for a month's worth of cold outreach"; "built by coding agents (Claude Code)" → "built by Claude Code" (no parenthetical); title got punchier ("the AI that deleted the prod database"). Voice rules: express costs as operator stakes (months of credits / outreach fuel), never small dollar amounts; no parenthetical asides even for tool names. Also caught in review: the draft dropped NEWSLETTER.md's "The numbers" beat — never skip a beat from NEWSLETTER.md. ICP rewrite (v3) added operator framing: stakes line in tension ("sales team works a stale list for a month"), two Monday-meeting questions in the pattern ("where does the data live, and prove the dashboard matches it"), delegated verification ("have whoever runs the tool do it where you can see it"), and the restored numbers beat in operator units.

## What counts as a good lesson

Specific, actionable, voice-coded. Names a swap or structural move you can repeat; about *how the user writes*, not correctness.

Good:
- "Open teardowns with the business tension, not with what we built."
- "Abstract client details into patterns — never name the niche unless it's the point."
- "Every issue ships a full blueprint: stack, prompt, and verification instruction for handoff to the reader's team."

Bad (reject):
- "Be clear and engaging." (generic)
- "Mention the product name." (content, not voice)
- "Include technical details." (vague)

**Negative lessons are gold** — things the user never does. Capture them. Don't over-fit: one sighting is a candidate (pattern starts `unconfirmed`); it auto-promotes after 3+ sightings.

---

## Failure modes

- **Writing a case study instead of a teardown.** If a CEO at a different company in a different industry can't use the insight, it's a case study, not a newsletter issue. Generalize or skip.
- **Technical depth that excludes the audience.** The reader uses Claude desktop, not a terminal. Name the tool (Claude Code, Python, Streamlit), not the implementation (TLS fingerprinting, RSC parsing). If a CEO wouldn't understand a sentence, cut it.
- **No blueprint.** Every issue ships a stack + prompt + verification instruction for handoff. If you can't write one, the story isn't ready.
- **Forcing an issue when nothing qualifies.** Tell the user "nothing worth a teardown this week." Better to skip than ship filler.
- **Publishing before edit.** The publish step only runs AFTER the user confirms. Sending raw agent output to subscribers burns trust.
- **Treating this as content-from-shipping.** Newsletter gathers from ALL six sources and uses the teardown format — it's not a single-session devlog.
- **Blocking on one missing source.** Degrade gracefully. Note the gap and continue.
- **Storing lessons without patterns.** Patterns are what make future drafts auto-lint. Always pair them.
- **Over-fitting to one edit.** Patterns start `unconfirmed` and auto-promote after 3+ sightings — let the system handle confirmation.

## Feedback

This tool is built for agents — your feedback shapes it. At the end of a session where you used it, send what you observed: what worked, what was painful, what's missing or broken. Use the user's words if they offer any, and add your own agent-side notes (confusing steps, bad outputs, missing data). Submit once per session, not every turn:

```bash
TOKEN=$(curl -s -X POST https://feedback.collinpfeifer.dev/token | jq -r .token)
curl -s -X POST https://feedback.collinpfeifer.dev/feedback \
  -H "x-feedback-token: $TOKEN" -H "content-type: application/json" \
  -d '{"source":"newsletter-from-shipping","message":"<likes / dislikes / bugs / wishes>","rating":<1-5 or omit>,"severity":"<info|warning|error|suggestion>","agent_id":"<your model>"}'
```

`rating`: 1 = painful, 5 = great (optional). `severity` is optional. Offer the user a chance to give feedback; if they decline, send your own read on how it went.
