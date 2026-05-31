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
- **Knows the product.** A RAG pipeline scrapes the customer's own website, chunks and embeds it,
  so the agent answers real product, pricing, and feature questions from the actual site — and
  can pull in a new page on demand or web-search for facts that live off-site.
- **Drives the page with you.** While it speaks it scrolls to the right section, highlights the
  CTA, expands content, fills forms, and navigates — a guided tour, not a chat box in the corner.
- **Remembers every visitor.** Cross-session memory (name, company, interests, history, visit
  count) so a returning visitor is picked up in context, not greeted from scratch.
- **Captures and qualifies leads** naturally in the conversation (voice-first, no pushy popups),
  and **books meetings** (Cal.com).
- **Speaks 30+ languages**, switching mid-conversation, and adapts its tone using buyer psychology.
- **One embed, zero setup** — works on Webflow, Wix, Shopify, or a custom site.

In Foyer's own words: *"Your website is a brochure. Foyer makes it a conversation."* It's a real,
deployed product — the thing we improve, not the thing we built this weekend.

### The problem we attacked at the hackathon

Voice agents **fail silently.** A visitor asks something, the agent gives a vague non-answer or
doesn't know a fact, the lead cools — and nobody ever knows or fixes it. So we built a
**self-improvement layer** that turns every real conversation into training signal:

1. After an ElevenLabs call, the transcript is **scored** against the objection it raised
   (differentiation, price, trust, …) — by **Cekura**, or a **Nemotron** local-judge.
2. If the agent whiffed it → **RED**.
3. **Nemotron mines the failure into a concrete fix** and decides where it belongs:
   - **`skill`** — *how to handle* (appended to the agent's skill doc), or
   - **`brand`** — *a missing/incorrect fact* (appended to the agent's brand-facts).
4. The fix is written straight back into the agent's **real prompt surfaces**, and the
   strategy version is bumped. **No human edits it.**
5. The same scenario re-scores **GREEN**. A scoreboard shows the strategy version climbing,
   per agent.

So Foyer learns **two ways at once**: better *behavior* (skill) and filled *knowledge gaps*
(brand). The line that matters: **no human touched the agent between the red and the green —
it fixed itself.**

---

## 2. Demo (under 60 seconds)

▶️ **‹video link — to be recorded, ≤60s›**

Storyboard (what the clip shows, no narration needed):
1. The Self-Improve scoreboard for an agent — a scenario sitting at **RED**, strategy **v0**.
2. Hit **Run** → Cekura/Nemotron scores the live agent → it fails the objection (vague answer).
3. The loop **auto-mines the fix** and writes it into the agent's prompt (skill/brand).
4. Re-run → same scenario flips **GREEN**, strategy **v1**, the mined fix shown in plain English.
   No human edited the agent in between.

---

## 3. How we used Cekura, Nemotron, and Pipecat

### Cekura — the grader / validation signal
- Scenarios are `{ persona, expectedOutcome }` and **auto-sync to Cekura** — the integration
  captures the Cekura scenario id for you (no hunting for it in the dashboard).
- With `CEKURA_RUN_MODE=elevenlabs`, **Cekura calls the live ElevenLabs agent**, plays the
  visitor over a real voice call, and scores the conversation against our `expectedOutcome`.
  (It also scores already-stored real transcripts via `scenarios_run_text`.)
- Each Foyer agent is bound to its **own** Cekura agent — multi-tenant isolation, so scores
  never bleed across agents.
- **What we were testing:** objection handling — `differentiation`, `price`, `trust` (plus
  `skeptic`, `comparison`, `escalate`, `combined`). Every scenario's pass bar is the same
  shape: **a concrete, grounded answer — a specific differentiator, a real value lever, a
  named trust signal — not vague superiority.** Generic "we're the best / it's worth it" = FAIL.
- **How much it improved performance:** scenarios that score **RED** are re-scored after the
  autonomous fix and flip **GREEN** at the next strategy version — same agent, same scenario,
  no human edit. The scoreboard tracks pass/fail count, pass rate, and strategy version per
  agent so the lift is measured, not asserted. ‹drop in your final run's headline number, e.g.
  "pass rate v0 → v1: 1/3 → 3/3" — pull it from the scoreboard after your last run›.

### NVIDIA Nemotron — the judge and the miner
- Nemotron (`nvidia/nemotron-3-super`, OpenAI-compatible endpoint) does both LLM jobs in the
  loop: the **local-judge** (scores a transcript pass/fail when Cekura isn't in the loop) and
  the **miner** (reads the failed transcript and writes the corrected handling, *and* decides
  whether it's a `skill` or a `brand` fix).
- It runs behind a provider-agnostic shim; `llmLabel()` reports which model produced each
  verdict so results are never mislabeled, with an OpenAI/OpenRouter fallback so the loop
  always runs.

### Pipecat / voice
- The live agent runs on **ElevenLabs WebRTC** (Foyer's voice stack) — that's the agent Cekura
  calls and grades, and the prompt surface the mined fixes are written back into.
- The improvement loop itself was **prototyped on a Pipecat + Twilio reference bot**
  (our `lasso-voice` repo) and then ported into the ElevenLabs integration. We did not run
  Pipecat in the final demo — the hackathon notes it's fine not to use all three.

---

## 4. What we built during the hackathon

**New — the entire self-improvement layer (built here):**
- `packages/server/src/improve/` — `loop.ts` (score → mine+apply → re-score), `evaluator.ts`
  (Cekura + Nemotron judge), `miner.ts` (writes the fix, picks skill vs brand), `store.ts`,
  `cekura-mcp.ts`, `cekura-sync.ts`, `learned-context.ts`, `scenario-store.ts`.
- `packages/server/src/routes/improve.ts` — `/api/improve` (scoreboard, run, run-all, seed,
  scenarios, cekura-binding, diag).
- `packages/dashboard/src/pages/SelfImprove.tsx` — the per-agent scoreboard + scenario console.
- A judge-facing presentation scoreboard (RED→GREEN hero + per-scenario "learned fix in plain
  English" arc).
- **Writing mined fixes back into the live agent's skill docs / brand-facts** to close the
  loop on every real conversation.

**Pre-existing — Foyer the product:** the voice agent itself (ElevenLabs WebRTC, RAG over the
customer's website, the dashboard, widget, Shopify, Redis). Foyer existed before — it's the
*patient* the engine improves, not what we built today.

**Borrowed:** the loop design was ported from our Pipecat reference bot (`lasso-voice`);
Cekura, Nemotron, and ElevenLabs are the tools.

---

## 5. Feedback on the tools

### Cekura — building a self-improvement loop (real friction we hit, and what worked)
- **Runs are async with no obvious "done" signal.** We had to poll `runs_bulk_retrieve` until a
  terminal state; our first version assumed a synchronous result and got empty/partial data. A
  documented terminal-status field (or a webhook) would help a lot.
- **SSE responses were multi-line and got truncated.** The streamed result arrived as several
  `data:` frames that had to be concatenated before parsing — otherwise the JSON was cut off
  mid-object. This cost real time to diagnose.
- **Double-encoded JSON.** Some results came back as a JSON *string* nested inside JSON and
  needed a second parse.
- **Errors were swallowed.** When a run fell back or failed, the cause wasn't surfaced — we had
  to build a `/diag` endpoint just to see the real error and the raw `runs_bulk_retrieve` shape.
  Surfacing the underlying error by default would save hours.
- **What worked well:** the persona + expected-outcome model maps cleanly onto a pass/fail eval,
  and "Cekura calls the live agent and grades it" is exactly the right primitive for evaluating
  a real voice agent — it's what made our red→green honest instead of self-graded.

### NVIDIA Nemotron
- **Good at:** following structured judge/miner instructions and returning usable JSON verdicts;
  fast enough to keep the score→mine→re-score loop tight; OpenAI-compatible so dropping it in was
  a base-url + model swap; open weights meant no rate-limit anxiety while iterating.
- **Could be better:** strictness of JSON-only output on long transcripts (we defended with a
  tolerant parser), and pass/fail calibration could be sharper vs. a frontier judge — a borderline
  "concrete enough?" answer sometimes scored inconsistently between runs.

---

## 6. Live link

**https://app.tryfoyer.ai** — Foyer is deployed (Railway). The Self-Improve scoreboard lives in
the dashboard per agent (behind login). ‹add a shareable/demo agent link if you want judges to
click through›.

---

*Team: Sohazur, Shanzila, Maryam · Built at the YC Voice Agents Hackathon (Cekura × Pipecat/Daily).*
