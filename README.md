# Foyer — a voice agent that fixes its own failures

**YC Voice Agents Hackathon** · Cekura × NVIDIA Nemotron · voice on ElevenLabs

> Foyer is an existing product; the **self-improvement layer** is what we built at the
> hackathon. Code lives in our private product repo; this README summarizes what we built.

---

## 1. What is this?

### Foyer — the product

**Foyer turns a website into a conversation.** It's a voice AI sales agent that embeds on any
site with **one script tag** and talks to visitors out loud, in real time (ElevenLabs WebRTC,
sub-second latency). Instead of digging through a brochure, a visitor just asks — and the site
answers in a natural voice and *acts* on the page while it talks.

What it does on a live site:
- **Knows the product.** A site-crawling RAG knowledge base means the agent answers real product,
  pricing, and feature questions from the customer's own content — and can pull in a new page on
  demand or web-search for facts that live off-site.
- **Drives the page with you.** While it speaks it scrolls to the right section, highlights the
  CTA, expands content, fills forms, and navigates — a guided tour, not a chat box in the corner.
- **Remembers every visitor.** Cross-session memory (name, company, interests, history) so a
  returning visitor is picked up in context.
- **Captures and qualifies leads** naturally in conversation (voice-first, no pushy popups), and
  **books meetings** (Cal.com).
- **Speaks 30+ languages**, switching mid-conversation, and adapts its tone using buyer psychology.
- **One embed, zero setup** — works on Webflow, Wix, Shopify, or a custom site.

In Foyer's own words: *"Your website is a brochure. Foyer makes it a conversation."* It's a real,
deployed product — the thing we improve, not the thing we built this weekend.

### What we built at the hackathon

A **self-improvement layer** on top of the live agent: the agent is tested by Cekura against real
visitor scenarios, and when it fails to handle one — say, a skeptical visitor it can't convince —
the system **automatically mines the failure into a fix and pushes it into the agent's live
prompt**, so it handles that situation better next time, with **no human editing it. The agent
teaches itself from its own failures.**

How the loop runs:
1. After an ElevenLabs call (or a Cekura scenario run), the transcript is **scored** against the
   objection it raised — by **Cekura**, or a **Nemotron** local-judge.
2. If the agent whiffed it → **RED**.
3. **Nemotron mines the failure into a concrete corrective instruction** and decides where it
   belongs: **`skill`** (how to handle) or **`brand`** (a missing/incorrect fact).
4. The fix is pushed straight into the agent's **live system prompt**, and the strategy version is
   bumped. **No human edits it.**
5. The same scenario re-scores **GREEN**. A scoreboard shows the strategy version climbing.

The line that matters: **no human touched the agent between the red and the green — it fixed itself.**

---

## 2. Demo (under 60 seconds)

▶️ **‹video link — to be recorded, ≤60s›**

Storyboard (what the clip shows, no narration needed):
1. The Self-Improve scoreboard for an agent — a scenario sitting at **RED**, strategy **v0**.
2. Hit **Run** → Cekura/Nemotron scores the live agent → it fails the objection (vague answer).
3. The loop **auto-mines the fix** and writes it into the agent's live prompt (skill/brand).
4. Re-run → same scenario flips **GREEN**, strategy **v1**, the mined fix shown in plain English.
   No human edited the agent in between.

---

## 3. How we used Cekura, Nemotron, and Pipecat

The three hackathon themes map cleanly onto our three pieces: **voice** (the ElevenLabs agent),
**open-weights models** (Nemotron runs the learning), and **evaluating + improving agent
performance** (the Cekura loop).

### Cekura — evaluating and improving the agent (the core theme)
Cekura is the evaluation and learning signal for the whole loop.
- **Define once, test forever.** Scenarios — adversarial website visitors: a **GEO skeptic**, a
  **comparison shopper**, someone with **an ask the agent must escalate rather than bluff** — are
  authored per-agent in Foyer's Self-Improve dashboard and **synced to Cekura over its MCP
  server**, which auto-captures the Cekura scenario IDs and binds them to that agent. No hunting
  for IDs in a dashboard.
- **Grades the real agent.** With `CEKURA_RUN_MODE=elevenlabs`, Cekura calls the live ElevenLabs
  agent, plays the visitor over a real voice call, and scores the conversation against our pass
  criterion. Each Foyer agent is bound to its own Cekura agent (multi-tenant isolation), so scores
  never bleed across agents.
- **What we were testing:** whether the agent handles adversarial visitors with a concrete,
  grounded response instead of vague claims — and, when it can't, whether the loop teaches it to.
- **How much we improved performance:** ‹pending the demo run — the scoreboard reports pass/fail
  and pass-rate per strategy version (v0 → vN); drop in the demo's headline before→after here›.

### NVIDIA Nemotron — the open-weights model that runs the learning
Nemotron (`nemotron-3-super`, served over an OpenAI-compatible vLLM endpoint) does three jobs:
- **Judge** — scores a transcript pass/fail with a one-line rationale (strict: fails vague/fluffy,
  passes concrete).
- **Miner** — reads the failed transcript and writes a tight, on-voice corrective fix that names
  the actual lever/fact, and decides whether it's a `skill` or `brand` fix.
- **Simulator** — drives the simulated visitor for the local-judge path.

It dropped straight in as all three (no SDK changes), with an OpenAI/OpenRouter fallback so the
loop always runs.

### Pipecat / voice
The live voice runtime is **ElevenLabs WebRTC** — the agent Cekura calls and grades, and the
prompt surface the fixes are written back into. The improvement loop's design traces to our
**Pipecat** reference agent (lasso-voice), where we prototyped the score→mine→re-score pattern
before wiring it into Foyer.

---

## 4. What we built during the hackathon

**Old (already existed) — Foyer itself:** the agent, the **ElevenLabs voice runtime**, the
dashboard, and the **site-crawling knowledge base** (RAG). Foyer is the product.

**New (built today) — the self-improvement loop:**
- **Cekura evaluation wired into Foyer** — Cekura runs scenarios against the live agent and scores
  how it handles them (Nemotron + Cekura stack).
- **A failure→fix loop** — when the agent fails a scenario, the system mines the failed transcript
  into a concrete corrective instruction and **pushes it into the agent's live system prompt**, so
  the next conversation is better — autonomously.
- **A per-agent scenario UI** — define a visitor scenario (persona + pass bar) per agent in the
  Self-Improve dashboard; it syncs to Cekura and runs on demand.
- Under the hood: `packages/server/src/improve/` (loop, evaluator, miner, store, cekura-mcp,
  cekura-sync, scenario-store), the `/api/improve` routes, the `SelfImprove.tsx` dashboard page,
  and a judge-facing presentation scoreboard.

**Borrowed:** the loop design was ported from our Pipecat reference bot (lasso-voice); Cekura,
Nemotron, and ElevenLabs are the tools.

---

## 5. Feedback on the tools

### NVIDIA Nemotron (`nemotron-3-super`)

**Did well**
- **Excellent strict judge** — reliably failed vague/fluffy answers, passed concrete ones, with a
  useful one-line rationale. Consistent enough to drive an automated pass/fail loop.
- **Strong miner** — wrote tight, concrete, on-voice fixes that named the actual lever/fact, not
  generic "be more helpful" filler.
- **Zero-friction integration** — the OpenAI-compatible vLLM endpoint dropped straight in as
  judge + miner + simulator with no SDK changes.

**Could be better**
- **Over-performs as a role-play simulator** — it gave the simulated agent discounts/levers its
  prompt never granted, producing false passes until we explicitly constrained it to "only use
  what's in the prompt." Worth a note for anyone using it to simulate conversations.
- **JSON mode is a sharp edge** — `response_format: json_object` plus a prompt asking for prose
  silently returns empty/garbage. The format and the prompt have to agree or you get nothing, with
  no error.
- Needs tight `max_tokens` + "output only X" guardrails to stay on format.

### Cekura — self-improvement loops + bugs

**Bugs / friction we hit**
- **SSE response framing (biggest one)** — MCP responses came back as multi-line / concatenated
  `data:` SSE frames, plus a trailing call-id line after the JSON. Naive parsing truncated or
  failed; we had to parse each event separately and strip the trailing line. Easy to get wrong —
  worth documenting or fixing server-side.
- **Async runs aren't obvious** — `scenarios_run_*` returns run IDs and you must poll
  `runs_bulk_retrieve` until terminal. A documented "wait-until-terminal" pattern (or a sync
  option) would save a lot of time.
- **`text_only` override rejected on voice agents** — running against a live ElevenLabs agent
  fails with "Override for field text_only is not allowed by config." You have to whitelist it on
  the ElevenLabs side, and the error doesn't say where.
- **Empty 0-turn conversations reach a terminal status** — they look like a result but no
  conversation happened; we had to treat them as "pending," not a failure. Surfacing *why* a run
  produced 0 turns would help.
- **Live fan-out floods easily** — one run with N scenarios × M personalities = N×M real
  ElevenLabs calls. We had to cap it; a built-in cap/warning would help.
- **Result-shape discovery** — we built a diag endpoint to dump `runs_bulk_retrieve` and
  reverse-engineer the real result shape. Clearer schema docs would help.
- **Closing the loop** — after a mined fix, re-scoring races the live prompt propagation to
  ElevenLabs (debounced). A "the agent under test just changed — re-evaluate now" hook would make
  self-improvement loops much cleaner.

**Positives**
- The MCP server was clean to integrate (initialize → session id → tools/call).
- Per-agent scenario sync + auto-binding the scenario IDs worked great once wired.
- Once you poll correctly, real grading + the real conversation transcript come back — exactly
  what a learning loop needs.

---

## 6. Live link

**https://app.tryfoyer.ai** — Foyer is deployed (Railway). The Self-Improve scoreboard lives in
the dashboard per agent (behind login).

---

*Team: Sohazur, Shanzila, Maryam · Built at the YC Voice Agents Hackathon (Cekura × Pipecat / Daily).*
