# Training: From Chat to Agentic Developer

A 7-chapter Claude Code training curriculum for software development teams.

## Structure

| File | Content |
|------|---------|
| `chapter-01-basics.md` | Setup, CLAUDE.md, prompt engineering, core commands |
| `chapter-02-skills-commands-plugins.md` | Slash commands, skill files, plugins |
| `chapter-03-agents.md` | Agent files, memory, personas, plan mode |
| `chapter-04-agentic-workflows.md` | Sequential chains, parallel fan-out, orchestrators |
| `chapter-05-context-handling.md` | Context window mechanics, /compact, lean design |
| `chapter-06-cost-handling.md` | Model selection, lean prompts, tiered workflows |
| `chapter-07-ecosystem.md` | Production repos: claude-code, hooks, plugins, agents |

## Conventions

- **Language**: All hands-on exercises use TypeScript (not Ruby or Python)
- **Exercises**: Use real Claude Code files — `.claude/agents/*.md`, `.claude/skills/*.md`, `CLAUDE.md` — no framework setup required
- **Format**: Each exercise follows: create file → type prompt → observe → experiment (3 variations)
- **No build system**: Pure markdown content, no npm/build/test commands

## Editing Guidelines

- Keep exercises language-agnostic in concept; TypeScript in code examples
- Slash command syntax must be verified against actual Claude Code behavior
- Exercise file paths (e.g., `.claude/agents/`) must match real Claude Code conventions
