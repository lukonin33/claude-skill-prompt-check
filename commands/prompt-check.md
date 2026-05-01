---
description: Audit and harden the last generated prompt before delivering it to the user. Optional argument `deep` (parallel subagent review) or `explain` (issues only, no rewrite).
argument-hint: "[deep|explain]"
---

# /prompt-check

Activate the `prompt-check` skill (`~/.claude/skills/prompt-check/SKILL.md`) on **the last prompt you generated in this session for the user**.

## How to identify "the last generated prompt"

It is **the text** that you, in one of your recent messages, offered to the user to **copy and paste into another chat** (recovery handoff / research prompt / handoff to another orchestrator / prompt for a new technical chat). Usually formatted as a code block with `START-...` / `END-...` markers, or as a long markdown block in a fenced code block.

If no such artifact exists in the current session — explicitly tell the user "no generated prompt found in this session for audit" and ask which text to check.

## Arguments

- **No argument** → Light mode (single-pass self-audit, ~1-2 min). Suitable for short/medium prompts.
- **`deep`** → Deep mode (~5-10 min, parallel subagents Critic + Strengthener + optional Domain-researcher). Use for long/strategic prompts or when the user explicitly asks for "deep audit".
- **`explain`** → Explain mode. All 5 layers run but the prompt is **not rewritten**. Each issue gets a "How to fix" hint and the user does the rewrite themselves. Useful for learning prompt engineering.

User argument: `$ARGUMENTS`

## What to do

1. Find the last generated prompt in conversation history
2. Read `~/.claude/skills/prompt-check/SKILL.md` and apply its 5 layers (consistency / completeness / dependencies / patterns scan / strengthening)
3. In Deep mode — launch subagents in parallel (Critic + Strengthener)
4. Return the structured deliverable per skill format:
   - `## prompt-check verdict` with categorized issues (Critical / Important / Nice-to-have)
   - `### What changed in the prompt` — diff or list of edits
   - `## Final hardened prompt` — finalized text for copy-paste (skipped in Explain mode)

For **0 issues** — verdict "delivery-ready as written" + unchanged prompt.

**Never silent rewrite** — the user must see the diff.

## Related

- Skill: `~/.claude/skills/prompt-check/SKILL.md` (full methodology)
- Project patterns reference: `<your-project>/patterns.md` (Layer 4 input, if your project has one)
