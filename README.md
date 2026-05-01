# prompt-check — a Claude Code skill

Audit and harden a generated prompt before delivering it to the user. Catches conflicts, ambiguities, missing data, weak acceptance criteria, leading questions, hard-coded references, missing rollback. Returns a hardened prompt with a diff (or "no issues found" verdict).

**Origin:** pattern observed in long-running orchestrator workflows — a human reviewer repeatedly typing «re-read, check, improve, give me the final version» after each generated prompt. This skill formalizes that loop with a structured 5-layer audit and 3 modes.

## What it does

5-layer self-audit before delivering a prompt to the user:

1. **Internal consistency** — contradictions, ambiguity, leading questions, voice/register
2. **Data completeness** — cwd, traceable id, files-to-read, measurable acceptance criteria, output path
3. **Dependencies** — phantom references, broken cross-refs, block order, external dependencies
4. **Project-specific patterns scan** — uses your project's `patterns.md` if available, otherwise a generic 12-point checklist
5. **Strengthening** — decision tree, top-3 surprises, scoring matrix, rollback, MVP variant

3 modes:

- **Light** (default, 1-2 min) — single-pass self-audit
- **Deep** (`/prompt-check deep`, 5-10 min) — parallel subagents (Critic + Strengthener + optional Domain-researcher)
- **Explain** (`/prompt-check explain`) — issues only, no auto-rewrite (learning mode)

## Installation

### Option 1 — manual copy (recommended)

```bash
git clone https://github.com/lukonin33/claude-skill-prompt-check.git
mkdir -p ~/.claude/skills/prompt-check
cp claude-skill-prompt-check/prompt-check/SKILL.md ~/.claude/skills/prompt-check/

# optional — slash command
mkdir -p ~/.claude/commands
cp claude-skill-prompt-check/commands/prompt-check.md ~/.claude/commands/
```

Restart Claude Code (Reload Window in VS Code).

### Option 2 — git submodule

```bash
cd ~/.claude/skills
git clone https://github.com/lukonin33/claude-skill-prompt-check.git prompt-check-repo
ln -s prompt-check-repo/prompt-check prompt-check
```

## How to invoke

- **Auto-trigger** — skill activates automatically when the assistant is about to deliver a prompt to the user (any text formatted for copy-paste into another chat)
- **Manual phrases** (en/ru) — "check the prompt", "improve the prompt", "harden the prompt", "audit the prompt", "review the prompt"; «перечитай промпт», «улучши промт», «проверь промпт», «проанализируй промт»
- **Slash commands** — `/prompt-check`, `/prompt-check deep`, `/prompt-check explain`

Trigger requires both an action verb AND a prompt-object — does not trigger on "re-read this file/code/README" where the object is not a prompt.

## Connecting your project's patterns file

Layer 4 looks for a project-specific patterns/lessons file. If you have one (e.g., `<your-project>/patterns.md` or `docs/PATTERNS.md`), Layer 4 will scan against the patterns recorded there. If you don't, a generic 12-point checklist is used.

Recommended sections for your `patterns.md`:

- Diagnostic methodology
- Migration patterns
- Scheduling/notification patterns
- System config gotchas
- Architecture patterns (silent failures, recovery)
- Format checklist
- Stop-and-ask triggers

## License

MIT © 2026 Maxim Lukonin

## Contributing

PRs welcome for:
- Layer 4 generic checklist extensions
- New trigger phrases for other styles / registers / languages
- Translations of SKILL.md
- Mode C (Explain) refinements
