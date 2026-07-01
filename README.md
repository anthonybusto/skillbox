# Skillbox

**Thinking tools for AI agents.**

Most agent skills tell the AI *what to do* — write a commit message, generate a README, lint some code. Skillbox skills tell the AI *how to think* — decompose problems before solving them, anticipate failures before building, surface hidden assumptions before they become bugs.

Small collection. High quality. Each skill is battle-tested and does one thing well.

## Skills

| Skill | What it does | Trigger phrases |
|-------|-------------|-----------------|
| [agent-convergence](skills/agent-convergence/) | Multi-agent debate loop with a cross-examining griller. Three agents argue, one tears them apart, repeat until consensus. | "debate", "convergence", "argue this out", "stress-test this idea" |

## Installation

Every skill follows the [Agent Skills](https://github.com/agentskills/agentskills) open standard. Works with **Cursor**, **Claude Code**, **Codex CLI**, **Gemini CLI**, and any tool that reads `SKILL.md`.

### Single skill

Copy the skill folder into your tool's skills directory:

```bash
# Cursor
cp -r skills/rubber-duck ~/.cursor/skills/

# Claude Code
cp -r skills/rubber-duck ~/.claude/skills/

# Project-scoped (any tool)
cp -r skills/rubber-duck .cursor/skills/   # or .claude/skills/
```

### All skills

```bash
cp -r skills/* ~/.cursor/skills/
```

## Philosophy

1. **Think before you code.** The most expensive bug is the one where you solved the wrong problem.
2. **Small and opinionated.** Each skill earns its place.
3. **Human in the loop.** These skills ask questions — they don't just generate output.
4. **Works everywhere.** Standard `SKILL.md` format. No vendor lock-in.

## License

MIT
