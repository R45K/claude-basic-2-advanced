# From Chat to Agentic Developer
### A Deep-Dive Training Program for Software Development Teams

---

## Welcome

This training program takes you from using Claude as a chat tool to building and orchestrating full agentic workflows. It is self-paced, hands-on, and **language-agnostic** — every exercise uses real Claude Code files (`.claude/agents/*.md`, `.claude/skills/*.md`, `CLAUDE.md`) that you create and run directly in Claude Code, regardless of your programming language.

Each chapter represents approximately **4+ hours** of study and practice.

---

## How to Use This Training

Every exercise follows the same format:

1. **Create a real file** — usually a `.claude/agents/` markdown file with a system prompt
2. **Type a prompt in Claude Code** — the exact text is provided
3. **Observe** — what happens, what to notice
4. **Experiment** — 3 variations to try on your own

No Python, Ruby, or C# required. Everything runs through Claude Code.

**Before you start:**
```bash
npm install -g @anthropic-ai/claude-code   # install Claude Code
claude                                      # start a session
/doctor                                     # verify setup
```

---

## Curriculum

| Chapter | Title | What You'll Learn |
|---|---|---|
| [Chapter 1](chapter-01-basics.md) | **Basics** | Claude Code setup, prompt engineering, CLAUDE.md, core slash commands |
| [Chapter 2](chapter-02-skills-commands-plugins.md) | **Skills, Commands & Plugins** | Slash commands in depth, building skill files, installing and creating plugins |
| [Chapter 3](chapter-03-agents.md) | **Agents** | What agents are, building `.claude/agents/` files, memory, personas, plan mode |
| [Chapter 4](chapter-04-agentic-workflows.md) | **Agentic Workflows** | Sequential chains, parallel fan-out, orchestrator-worker, git worktrees |
| [Chapter 5](chapter-05-context-handling.md) | **Context Handling** | Context window mechanics, `/compact`, session memory, lean agent design |
| [Chapter 6](chapter-06-cost-management.md) | **Cost Management** | Model selection, lean prompts, `/cost` habit, tiered workflows |
| [Chapter 7](chapter-07-ecosystem.md) | **The Ecosystem** | Hands-on exercises for the 4 key production repos that extend everything above |

---

## Recommended Learning Path

```
Week 1  ──▶  Chapter 1: Basics                   (install, configure, first prompts)
Week 2  ──▶  Chapter 2: Skills, Commands & Plugins (extend and customize Claude Code)
Week 3  ──▶  Chapter 3: Agents                   (build your first .claude/agents/ files)
Week 4  ──▶  Chapter 4: Workflows                (chain and orchestrate agents)
Week 5  ──▶  Chapters 5 + 6                      (optimize context and cost)
Week 6  ──▶  Chapter 7: The Ecosystem            (install and use production repos)
```

---

## Prerequisites

- A Claude Pro, Team, or API account
- Claude Code installed: `npm install -g @anthropic-ai/claude-code`
- Git installed and configured
- A real project to practice on (or any open-source repo)

---

*Built for software development teams transitioning from basic LLM usage to full agentic development.*
