---
name: prompt-check
description: 'Audit and harden a generated prompt before delivering it to the user. Make sure to use this skill whenever you are about to output any text the user will copy-paste into another chat or send to an external orchestrator, OR whenever the user asks any variation of "re-read / check / analyze / improve / strengthen / refine / harden / audit / review the prompt", "finalize the prompt for another chat", "/prompt-check". Russian phrasings also trigger: «перечитай / проверь / проанализируй / улучши / усиль / доработай этот промпт / промт / ТЗ для чата / бриф для агента / промпт-блок / копи-паста блок». Trigger requires BOTH an action verb AND a prompt-object in the same request — does NOT trigger on "re-read this file / code / README / document" or "improve this function / SQL / README" where the object is not a prompt. Catches conflicts, ambiguities, missing data, broken dependencies, weak acceptance criteria, leading questions, hard-coded line numbers, missing rollback. Returns either a hardened prompt with diff, or the same prompt with explicit "no issues found" verdict. If a project-specific patterns file is provided (see Layer 4), Layer 4 will scan against it; otherwise a generic 12-point checklist is used. Specialized for prompt artifacts (≤2-3 screens, seconds-to-minutes) — for full multi-stage development workflow, use a dedicated bulletproof/code-review skill.'
license: MIT
---

# prompt-check

## Overview

A self-audit pass that the assistant runs on a prompt **it just generated** before sending it to the user. Catches typical defects (ambiguity, missing acceptance criteria, broken dependencies, leading questions, scope creep, hard-coded line numbers, missing rollback) **before** they cause a wasted chat session downstream.

Origin: pattern observed in long-running orchestrator workflows where a human reviewer repeatedly typed the same review request after each generated prompt («re-read, analyze, improve, give me the final version»). This skill formalizes that loop.

## When to Use

**Auto-trigger (preferred):** assistant just wrote a prompt for the user to copy-paste into another chat, or to send to an external orchestrator/agent. Before delivering — run prompt-check.

**Manual triggers** — phrases the user might type (any verb × object combination):

Verbs (en/ru):
- re-read / read again / перечитай / прочитай заново
- check / verify / проверь
- analyze / analyse / проанализируй
- improve / refine / улучши / улучшить
- strengthen / harden / усиль / усилить
- polish / доработай / refine

Objects (with common typos):
- prompt / промпт / промт (alt spelling without second «п» — common typo) / ТЗ / brief / бриф / задание
- "this block for copy-paste" / "the chat handoff" / handoff / handoff-prompt

Slash commands (via paired command file at `~/.claude/commands/prompt-check.md`):
- `/prompt-check` — Light mode
- `/prompt-check deep` — Deep mode with parallel subagents
- `/prompt-check explain` — Explain-only mode (issues without auto-rewrite, for learning)

> **Architectural note:** skills (`~/.claude/skills/<name>/SKILL.md`) activate via description triggers; they **do not auto-create** slash commands. To invoke this skill via `/`, install the paired file at `~/.claude/commands/prompt-check.md` (provided in this repo).

Examples that should trigger:
- "re-read the prompt" / "перечитай промт"
- "check the prompt" / "проверь промт"
- "analyze the prompt" / "проанализируй промт"
- "improve the prompt" / "улучши промт"
- "harden the prompt" / "audit the prompt" / "review the prompt"
- "ok now give me the final prompt" (right after the assistant produced a copy-paste block)

**Skip when:**
- Output is not a prompt for another chat (regular code / answer / explanation)
- Prompt is trivial (≤5 lines plain instruction without acceptance criteria)
- User explicitly said «no audit needed, just give me the prompt» / «without checking, deliver as is»

## Two Modes

### Mode A — Light (default, ~1-2 min)

Single-pass self-audit by the assistant. No subagents. Use for:
- Short to medium prompts (<300 lines)
- Tactical / operational prompts (one-off task, not stage-defining)
- Time-pressed contexts where deep analysis is overkill

### Mode B — Deep (`/prompt-check deep`, ~5-10 min, parallel subagents)

Spawns 2-3 parallel subagents:
1. **Critic** — finds defects, leading questions, ambiguities, missing fallbacks
2. **Strengthener** — proposes additions to make the prompt more useful
3. **Domain-researcher** (optional) — verifies factual claims if the prompt makes them (URLs alive, prices current, frameworks recommended are not deprecated)

Use for:
- Long prompts (>300 lines)
- Strategic prompts (research-prompts, build-prompts, multi-stage tasks)
- Prompts that will run for hours or be reused
- Anywhere the user explicitly asked for «deep audit» / «bulletproof prompt»

Selection rule of thumb: if the prompt is meant to start a **chat that will run >2 hours or cost >$1**, use Deep. Otherwise Light.

### Mode C — Explain (`/prompt-check explain`, learning mode)

Runs all 5 layers but **does not rewrite** the prompt. Returns issues categorized (Critical / Important / Nice-to-have) with **why** each one matters, plus suggested fix in plain prose. The user keeps full control over the rewrite. Useful when the user is learning prompt engineering or explicitly wants to make the changes themselves.

## The Checklist (all modes apply this; Deep adds subagent passes)

Run in this order. Each item is binary `[ ]` (passed or failed). Failed items → fix in the prompt or explicitly note as known limitation.

### Layer 1 — Internal consistency

- [ ] **No contradictions** between sections (e.g., section 1 says «do X» and section 5 says «don't do X»)
- [ ] **No double interpretations** of acceptance criteria («fast» — what is fast in numbers?)
- [ ] **Single voice / register** — not mixing formal and casual, not unintentionally mixing languages
- [ ] **No leading questions** that imply a single right answer («find competitors in country X» implies they exist; reword to «is there a competitor category, if not, why»)

### Layer 2 — Data completeness

- [ ] **Working directory specified** at the top (not guessed from context)
- [ ] **Traceable session identifier** (your project's convention — e.g., chat-id, session-id, ticket-id)
- [ ] **Files to read** listed explicitly (3-5 critical, not «read everything»)
- [ ] **Pre-flight block**: what must be ready before start (env-vars, accounts, prereq files)
- [ ] **Acceptance criteria measurable**: numbers, booleans, explicit thresholds — not «well», «optimal», «efficient»
- [ ] **Output artifact path specified**: where does the final result land
- [ ] **Language specified** if multi-language project

### Layer 3 — Dependencies and connections

- [ ] **No phantom references**: every file mentioned in «read this» actually exists (verify if possible)
- [ ] **No broken cross-refs**: sections reference each other only when they exist
- [ ] **Block order makes sense**: block 5 doesn't require output of block 7
- [ ] **External dependencies surfaced**: if prompt depends on user's API key / external service / human action — explicitly listed
- [ ] **Telemetry / logging**: how does the user know when the prompt's chat is done

### Layer 4 — Project-specific patterns scan (if available, otherwise generic)

If the user's project has a patterns/lessons file (e.g., `<PROJECT>/patterns.md`, `<PROJECT>/lessons-learned.md`, `docs/PATTERNS.md`), scan the prompt against the patterns recorded there. Common section types:

- Diagnostic methodology (verify-before-prescribe / smoking-gun query / clarifying questions before action)
- Migration patterns (auth/CORS/hardcoded URLs/webhook re-registration/schema changes)
- Scheduling/notification patterns (cron uniqueness, stale locks, cooldown, double-execution guards)
- System config gotchas (SSL/certbot / framework-specific quirks)
- Architecture (silent failures / recovery plan B / multi-layer state)
- Format checklist (cwd, traceable id, pre-flight, smoke-test, rollback)
- Stop-and-ask triggers (destructive / financial risk / architectural choice A vs B)

If **no patterns file is provided**, use the generic 12-point checklist below:

1. Verify before prescribe — diagnostic step before any change
2. Smoking-gun query first — narrow root-cause check
3. Pre-flight clarifying questions for ambiguity
4. Prerequisite chain explicit (X requires Y, which requires Z)
5. Disambiguate sources (which doc/branch/version)
6. No hardcoded URLs/PIDs/IPs
7. Webhook/integration re-registration after domain or env change
8. Schema migrations explicit, with rollback step
9. Cron uniqueness — only one runner if exclusive, otherwise lock
10. Cooldown for notifications (no double-fires within N seconds)
11. Recovery plan B / silent-failure detection / multi-layer state map
12. Stop-and-ask triggers for destructive or irreversible operations

### Layer 5 — Strengthening (additive, not corrective)

Even if Layers 1-4 pass, ask:

- [ ] **Decision tree instead of single recommendation** — for research/strategic prompts, return conditional branches (if X then A, if Y then B), not a single point
- [ ] **Top-3 surprises mandatory** — research prompts should have an explicit «surprises» field (or «no surprises, sources converge»)
- [ ] **Scoring matrix** for multi-option choices (criteria × options grid with numbers or explicit "unknown")
- [ ] **Rollback / fallback path** — what if this prompt's chat fails halfway
- [ ] **Quick-win MVP variant** — for build prompts, ask «is there a deliberately-dumb MVP that covers 80% in 20% of effort»

## Deliverable Format

Return to user **exactly this structure**:

```
## prompt-check verdict

**Mode:** Light | Deep | Explain
**Patterns reference used:** <path>/<filename> | generic-12-point-checklist
**Total issues found:** N

### Critical (must fix before delivery)
- [issue 1]: <what + why critical>
- [issue 2]: ...

### Important (recommended fix)
- [issue 3]: ...

### Nice-to-have (optional strengthening)
- [issue 4]: ...

### What changed in the prompt
[concise diff or list of edits — what sections were rewritten / added / cut]

---

## Final hardened prompt

[the full corrected prompt, ready for copy-paste]
```

In **Explain mode**: skip "Final hardened prompt" section. Instead, after each issue, give a one-line "How to fix" hint and let the user rewrite themselves.

If **no issues found** in any layer:

```
## prompt-check verdict

**Mode:** Light
**Total issues found:** 0
**Verdict:** prompt is delivery-ready as written.

[the same prompt, unchanged]
```

Never silently rewrite without showing what changed — the user must see the diff to learn for next time.

## Anti-patterns of the audit itself

- **Don't make cosmetic edits without justification** — if you renamed a variable / re-arranged paragraphs without arguing «this closes Layer X item Y», don't do it at all. Skill is about substantive defects, not «taste improvements».
- **Don't invent shortcomings** — if Layer 4 has no section for the topic at hand, don't stretch an unrelated pattern. Better to say «patterns file does not cover this class — consider adding a new pattern».
- **Don't go to Mode B without cause** — if the prompt is simple and Mode A found 0-1 issues, Mode B has diminishing returns. Spawn subagents only when the prompt size or stakes justify it.
- **Don't duplicate the patterns file** — if the project already has a patterns reference, use it. Don't re-write a parallel checklist inside the audit.

## Connection to other skills (general)

- **Bulletproof / full-stage development skills** — `prompt-check` is a **lite specialization** for one type of artifact (a prompt). It does not replace a multi-stage development audit, but it automates the prompt-specific layer.
  - **Edge case «is it a plan or a prompt?»**: if the artifact is a text block ≤2-3 screens for one task (handoff prompt, research prompt, recovery prompt) → **prompt-check**. If it's a multi-stage development plan / architecture doc / multi-week roadmap → use a dedicated bulletproof/plan-review skill. Boundary is fuzzy but works in 90% of cases.
- **Project patterns reference** — primary input for Layer 4. Skill reads it if present.

## History

- v1.0 (2026-05) — initial public release. Origin: repeated manual review request («re-read, check, improve, give me the final version») after generating a prompt. Skill formalizes that workflow with 5 layers + 3 modes (Light / Deep / Explain) + structured deliverable.

## Known limitations

- **Does not validate factual claims** in Light mode (e.g., «Service X costs $Y» won't be verified). Use Mode B + domain-researcher subagent with web search for that.
- **Does not know your product specifics** — generic skill, does not replace your product-area orchestrator. If the prompt has product-specific business logic, that part remains the orchestrator's job.
- **Layer 4 quality depends on patterns file**. Without one, the generic 12-point checklist gives baseline coverage but misses project-specific lessons. Recommend authoring a `patterns.md` for any long-running project.

## Contributing

PRs welcome at <https://github.com/lukonin33/claude-skill-prompt-check> for:
- Layer 4 generic checklist extensions
- New trigger phrases for other styles / registers / languages
- Translations of SKILL.md
- Mode C (Explain) refinements

License: MIT.
