---
name: fable-advisor
description: |
  Consult Fable 5 as a one-shot planning advisor while keeping quota usage low.
  Use when the user says "ask fable", "ถาม fable", "consult fable", "fable-advisor",
  or asks for a second opinion / architecture plan / debugging strategy on a
  genuinely hard, high-stakes, or hard-to-reverse problem. Runs a gate first to
  decide whether the task actually needs Fable (an explicit user request to
  consult Fable always passes the gate), then hands Fable a curated brief (not
  the raw transcript), gets back a plan, checks it against the user's explicit
  instructions, and routes implementation to the current main model
  (Opus/Sonnet). Do NOT invoke for small, clearly-scoped, low-risk work — the
  gate exists precisely to keep those off Fable.
compatibility: claude-code
version: 1.1.0
allowed-tools:
  - Agent
  - SendMessage
  - AskUserQuestion
  - Read
  - Grep
---

# Fable Advisor: economical use of Fable 5

Fable 5 is the strongest, most expensive model available to this session. On a
Max 20x plan its quota is roughly half of the total budget, so it must be spent
like a scarce resource: called for genuine planning/debugging leverage, never
for routine work the main model already handles well.

This skill has four phases: **Gate → Brief → Spawn → Implement**, plus a
**Resume** path for follow-ups. Hard budget: **at most two Fable calls per
task** (one consult + one unblock) without telling the user first.

## Phase 1 — Gate (decide if Fable is even needed)

**Step 0 — intent before design.** If the ambiguity is about **what the user
wants** (not how to build it), resolve it with `AskUserQuestion` *before*
gating — Fable starts cold and cannot ask the user. Then gate the clarified
task.

**Explicit request overrides the gate.** If the user asked to consult Fable
("ask fable", "ถาม fable", etc.), consult — don't ask permission to skip. If
the task looks clearly routine, say so in one sentence *while proceeding*
("This looks routine, but consulting Fable as you asked"). Never silently
substitute your own answer for a requested consult.

**If Fable already produced a plan for this task earlier in the session**,
don't re-gate or re-brief — go straight to **Resume** below.

**Otherwise, the gate is a single question: does ANY criterion below apply?
If none do, SKIP.**

**CONSULT when ANY apply:**
1. Multiple viable approaches exist with real tradeoffs **and** the choice is
   expensive to reverse or will be lived with long-term (architecture, data
   model, library choice, migration strategy)
2. The change is cross-cutting — touches many files/systems, or the blast
   radius is large and hard to reverse
3. The change is **small but a mistake is costly, hard to detect, or
   irreversible** — auth/crypto/payments, destructive data migrations,
   concurrency, production infrastructure. Diff size is not the gate; blast
   radius is.
4. The problem is genuinely novel to this codebase — no existing pattern or
   prior art to copy from
5. **Two genuinely different attempts** at the problem have failed and the
   root cause is still unclear
6. Requirements are **technically** ambiguous enough that a plan should exist
   before any code is touched (intent-ambiguity went to the user in Step 0)

Routine work never triggers a consult on its own — a scoped bug fix with an
obvious root cause, cosmetic changes, replicating an existing pattern in the
codebase, anything answerable from docs/code/project memory. These are
illustrations, not a second checklist: a task that looks routine but matches
a numbered criterion above still consults.

If a **high-stakes** task is a genuine toss-up, ask the user once via
`AskUserQuestion` rather than guessing. Otherwise decide, and state the
decision in one sentence (e.g. "Scoped bug fix with an obvious cause —
skipping Fable and fixing it directly.").

**If SKIP:** proceed with the task yourself and stop here — do not run
Phases 2-4.

## Phase 2 — Brief (prepare curated context, not the raw transcript)

A fresh `Agent` call starts cold with zero memory of this conversation — that
is a feature here, not a limitation. It means Fable only ever sees what you
deliberately hand it, which is what keeps this cheap. Do not paste the whole
conversation. Write a tight brief:

```
## Brief for Fable
**Goal:** <one or two sentences — what decision or plan is needed>
**Repo / working dir:** <absolute path to the repo root — subagent cwds
  reset, so every path in this brief must be absolute>
**Constraints:** <stack, must-not-break, deadline, style/architecture rules;
  QUOTE the user's explicit instructions verbatim rather than paraphrasing>
**Context already known:** <relevant absolute file paths, prior decisions,
  relevant project memory — reference paths/names, don't paste full file
  contents unless essential; Fable can Read/Grep itself>
**What's been tried / ruled out:** <if anything, and why it failed. For
  debugging: paste the exact error/stack trace/failing command output
  VERBATIM — Fable cannot reproduce your interactive state>
**Acceptance:** <how success will be verified — the failing test to make
  pass, the command to run, the metric to hit>
**Specific question:** <the exact plan, diagnosis, or decision Fable should
  produce — be precise about the deliverable shape, e.g. "a step-by-step
  implementation plan with file-level detail" or "root-cause diagnosis plus
  a fix strategy". Add: "State your assumptions explicitly instead of asking
  questions back; list any decision that genuinely needs user input as an
  explicit open question in the plan.">
**Out of scope:** <what NOT to touch or reconsider>
```

Keep it under ~300-400 words, **excluding** verbatim error output or schema
snippets that are genuinely load-bearing — those are worth their length;
narrative padding is not. If the task needs actual file exploration, Fable can
do that itself once spawned — don't pre-digest more than necessary.

**Verify before spending:** confirm every path in the brief actually exists
before spawning. A brief pointing at a wrong path is the cheapest way to waste
an entire consult.

**Batch your questions:** if the task has several related open decisions, put
them all in this one brief. One consult covering three questions is far
cheaper than three consults.

## Phase 3 — Spawn (fable as a planning-only subagent)

Tell the user in 1-2 sentences that you're consulting Fable and why — do this
**before** spawning, since the foreground call blocks until Fable returns.

Spawn Fable as an advisor, not an implementer, by default:

```
Agent({
  description: "<3-5 word task description>",
  subagent_type: "Plan",       // no Edit/Write access — keeps Fable advisory-only
  model: "fable",
  run_in_background: false,     // default: you need the plan before continuing
  prompt: "<the brief from Phase 2>"
})
```

`subagent_type: "Plan"` is the default because it can Read/Grep/explore but
cannot Edit/Write — this matches the intended workflow (Fable plans, the main
model implements) and prevents accidental drift into Fable doing paid
implementation work. Only use `subagent_type: "general-purpose"` with
`model: "fable"` if the user explicitly wants Fable to write code itself, not
just plan.

Set `run_in_background: true` only when you have genuinely independent prep
to do while Fable thinks (e.g. reproducing the bug, running the test suite to
gather evidence for Phase 4) — otherwise stay foreground.

**If the spawn fails** (quota exhausted, model unavailable, tool error): tell
the user immediately and offer the choice — plan with the main model now, or
retry Fable later. Never silently downgrade a requested Fable consult to a
main-model answer.

Note the agent id/name the harness returns — you need it for Resume.

## Phase 4 — Implement

Before executing, spend one beat sanity-checking the plan:

1. **Conflicts with the user's explicit instructions win against Fable —
   always.** If the plan contradicts something the user explicitly asked for,
   do not silently follow Fable: surface the conflict and Fable's rationale to
   the user and let them decide.
2. **Open questions:** if the plan flags decisions needing user input, resolve
   them (with the user, or via one scoped `SendMessage` if it's a technical
   detail) before touching code. Don't guess on load-bearing ones.
3. **Relay the plan:** give the user a 3-5 line summary of the approach and
   key decisions — always, not just for big plans. If the plan is
   **high-stakes** (destructive, hard to reverse, or large-scale), wait for
   the user's go-ahead before making edits; otherwise proceed.

Then execute the plan yourself (or delegate to a `sonnet`/`opus` subagent for
large mechanical portions). Small deviations from the plan discovered during
implementation are normal — note them and continue. Do not re-consult Fable
to double-check routine implementation steps — that defeats the point of the
gate.

## Resume — when implementation gets stuck

If you hit a specific blocker while implementing Fable's plan:

1. **Prefer resuming the same agent** with `SendMessage` (using the agent id
   from Phase 3) rather than spawning a fresh one — a completed foreground
   agent auto-resumes on `SendMessage`, so this works even after Fable already
   returned its plan. It keeps Fable's prior context and avoids re-sending the
   whole brief. Scope the message to just the blocker: what you tried, the
   exact error/result verbatim, and the one question you need answered.
2. Spawn a **new** `Agent` call with `model: "fable"` only if the original
   agent is genuinely gone or unrelated to the blocker — and even then, quote
   the relevant excerpt of Fable's original plan in the fresh brief (Phase 2
   style, scoped to the blocker), since a new Fable has never seen it and may
   otherwise re-derive or contradict it.
3. **Hard cap: two Fable calls per task** (the consult + one unblock). A third
   call requires telling the user first and getting their go-ahead — at that
   point it's quota burning, not leverage. And a blocker is not the same as a
   broken plan: if the plan turns out fundamentally infeasible, don't
   ping-pong with Fable — summarize the state to the user and decide together.

Then go back to implementing with the main model.

## Quick reference

| Step | Tool | Model |
|---|---|---|
| Gate check | (reasoning only; `AskUserQuestion` for intent) | main model |
| Brief | (reasoning only; verify paths) | main model |
| Consult | `Agent` (`subagent_type: "Plan"`) | `fable` |
| Plan sanity-check + summary to user | (reasoning only; user wins conflicts) | main model |
| Implement | direct edits / `Agent` | `sonnet` or `opus` |
| Unblock (max 1 without asking) | `SendMessage` to existing agent, or new narrow `Agent` | `fable` |

## Changelog

- **1.1.0** — Fable design review (verified against actual harness behavior
  before applying): collapsed the Gate to a single CONSULT checklist and
  dropped the redundant SKIP list + tie-breaker; split intent-check into an
  explicit Step 0; explicit user requests now proceed immediately instead of
  offering to skip; added routing to Resume when Fable already has a plan for
  the task; added a spawn-failure fallback (never silently downgrade to
  main-model); added a hard cap of two Fable calls per task; Phase 4 now
  always relays a 3-5 line plan summary to the user, not just for high-stakes
  plans; added path verification before spending a Brief; Resume keeps
  `SendMessage`-first (confirmed completed foreground agents remain
  addressable) and now requires quoting the prior plan excerpt when a fresh
  agent is genuinely needed. Rejected two of Fable's proposed changes after
  verification: removing `allowed-tools` (it's soft pre-approval, not a
  restriction — nothing was broken) and demoting `SendMessage` to a fallback
  path (completed foreground agents do resume via `SendMessage`).
- **1.0.0** — Initial version: Gate → Brief → Spawn → Implement phases plus
  Resume path.
