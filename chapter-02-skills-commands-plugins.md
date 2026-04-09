# Chapter 2: Skills, Commands & Plugins

Welcome to Chapter 2. In Chapter 1 you learned how to set up Claude Code, write a strong `CLAUDE.md`, and craft structured prompts. Now we go one level deeper: extending and customizing Claude Code itself.

Claude Code ships with a core set of slash commands, but that's just the foundation. You can install plugins to add new commands and capabilities, write skill files to give Claude reusable, consistent behaviors, and even create your own plugins to share with your team. This is the layer between "using Claude Code" and "programming Claude Code."

By the end of this chapter you will have a full command vocabulary, your first custom skill files, and a working plugin — all of which carry forward into Chapters 3 through 7.

---

## 1. Slash Commands In Depth

Slash commands are Claude Code's primary control interface. You met `/help`, `/clear`, `/doctor`, and `/exit` in Chapter 1. Here's the full reference.

### Core Session Commands

**`/help`** — Lists all available commands, including those added by plugins:
```
> /help
```
```
> /help plan
```
Shows detailed usage for a specific command. Always check this when a command doesn't behave as expected.

**`/clear`** — Resets conversation history while keeping `CLAUDE.md` loaded. Use this when a conversation has grown too long or you're switching topics.
```
> /clear
```

**`/memory`** — Shows what Claude currently has in context:
- The current `CLAUDE.md` content
- Loaded plugins and their commands
- Available MCPs (tool integrations)
- Session-level state

```
> /memory
```
Useful for debugging: "Why isn't Claude using the plugin I installed?" Check `/memory` to confirm it's loaded.

**`/exit`** — Exits the session gracefully:
```
> /exit
```

### Planning and Review Commands

**`/plan`** — Enters plan mode for complex tasks. Claude analyzes the task, breaks it into phases, identifies risks, and proposes an approach — **before touching any files.**

```
> /plan
> I need to migrate our authentication from session-based to JWT tokens. We have
> three services that consume the auth API, and we need to maintain backward
> compatibility during the migration.
```

Claude outlines the phases:

```
Plan Phase 1: Add JWT support alongside existing sessions
Plan Phase 2: Update all three consuming services to accept JWT
Plan Phase 3: Deprecate session-based auth
Plan Phase 4: Remove session code and clean up
```

You review and approve before any code is written. Without planning, Claude might start implementing before fully understanding the constraints — leading to wasted work, missed edge cases, or steps done in the wrong order.

**Always use `/plan` for:**
- Major refactoring or architectural changes
- Multi-file changes that could break things if done in the wrong order
- Database schema changes
- Security-sensitive changes

**You can skip it for:**
- Simple, well-defined single-file tasks
- Exploratory or read-only work (understanding code, researching a library)
- Quick bug fixes where the change is obvious

Build the habit now — when you start working with agents in Chapter 3, plan mode becomes even more critical because a wrong assumption at step 1 can cascade silently across many files.

**`/review`** — Runs a code review on recent changes or a specific file:
```
> /review
```
```
> /review src/models/user.ts
```
Claude analyzes code against your conventions from `CLAUDE.md` and provides structured feedback.

### Cost and Context Commands

**`/cost`** — Shows token usage and estimated cost for the current session:
```
> /cost
```
Outputs cumulative input and output tokens plus the estimated cost based on current Claude pricing. Develop the habit of running `/cost` at the end of each session — you'll quickly learn which tasks are expensive and why. Chapter 6 goes deep on cost management.

**`/compact`** — Compresses conversation history to save tokens:
```
> /compact
```
Claude summarizes the conversation, extracting essential information and condensing the rest. Use this when a long session is approaching token limits but you don't want to lose continuity with `/clear`. Chapter 5 covers context management strategies in detail.

### Plugin Management Commands

**`/plugin list`** — Shows all installed plugins and their commands:
```
> /plugin list
```

**`/plugin install`** — Installs a plugin from a registry:
```
> /plugin install commit-commands@claude-plugins-official
```

**`/plugin update`** — Updates an installed plugin to the latest version:
```
> /plugin update commit-commands
```

**`/plugin remove`** — Uninstalls a plugin:
```
> /plugin remove commit-commands
```

**`/plugin search`** — Searches the plugin registry:
```
> /plugin search typescript
```

### Creating Custom Slash Commands

You can define your own slash commands by creating markdown files in `.claude/commands/`. Each file becomes a `/command-name` shortcut.

Create `.claude/commands/standup.md`:
```markdown
---
name: standup
description: Summarize what changed in this repo since yesterday for a standup report
---

Look at the git log for commits in the last 24 hours. For each commit, summarize
in one sentence what changed and why. Group related commits together. Output a
bulleted standup-ready summary: what was done, what is in progress, any blockers
found in the commit messages.
```

Now type `/standup` in any session in that project. Claude runs the command exactly as defined.

This is the simplest form of extensibility — no packaging, no distribution, just a markdown file in your project. Skills (next section) take this further with structured invocation, frontmatter metadata, and reuse across projects.

---

**Exercise 2-A: Command Mastery**

**Goal:** Build the habit of using `/plan`, `/cost`, and `/compact` together on a real task.

**Setup:** Create `src/orderProcessor.ts` with some code to work with:
```typescript
function processOrder(items: Array<{price: number; quantity: number}>, discountCode: string): number {
  let total = 0;
  items.forEach(item => { total += item.price * item.quantity; });
  if (discountCode === "SAVE10") total *= 0.9;
  if (discountCode === "SAVE15") total *= 0.85;
  return total;
}

function applyTax(amount: number, region: string): number {
  const rate = region === "CA" ? 0.0875 : 0.05;
  return amount + amount * rate;
}
```

In Claude Code, type:
```
/plan Refactor processOrder to support a configurable discount table instead of hardcoded discount codes, and add input validation
```

**Observe:** Claude produces a plan with phases — it shows what it will change and why, before touching any file. Read the plan carefully: does it make the architecture decisions you would make?

Approve the plan:
```
Looks good, go ahead.
```

After Claude implements, run:
```
/cost
```

**Observe:** You now see exactly how many tokens the planning + implementation consumed. Note the split between input and output tokens.

Now run:
```
/compact
```

**Observe:** Claude summarizes the conversation. Run `/cost` again — the token count is lower. But what did `/compact` preserve vs. discard?

**What to experiment with:**
- Run the same refactor without `/plan` first — compare how Claude approaches it differently
- Use `/compact` before a task (not after) and observe how a leaner context changes Claude's output
- Add an entry to your `CLAUDE.md` saying "Always run `/plan` before any refactor" and see if Claude prompts you for it unprompted

---

## 2. Skills

A skill is a reusable, invocable instruction set stored in `.claude/skills/*.md`. Where a custom slash command in `.claude/commands/` is scoped to your project, a skill is a first-class Claude Code concept with a defined format and ecosystem support.

### What a Skill Is

Skills answer the question: "How should Claude approach this type of task?" You write the skill once, and it shapes Claude's behavior every time that type of task comes up — with consistent structure, steps, and constraints.

Skills are:
- **Reusable** — invoke the same skill across different tasks and projects
- **Composable** — combine multiple skills in one session
- **Shareable** — commit them to your repo or package them into a plugin
- **Stateless** — a skill shapes how Claude works, but doesn't maintain memory between invocations (that's what agents do, covered in Chapter 3)

### Skill File Format

Skills live in `.claude/skills/`. Each file has YAML frontmatter followed by the instruction body:

```markdown
---
name: skill-name
description: One sentence explaining when Claude should apply this skill.
model: sonnet          # optional — override the model for this skill
---

The instruction body. Write this as a structured process Claude should follow.
Be specific about steps, output format, and constraints.
```

The `description` field matters. Claude uses it to decide when to apply the skill automatically. Write it as: "Use when [situation]."

### A Complete Skill Example

Create `.claude/skills/write-tests.md`:
```markdown
---
name: write-tests
description: Write tests for any function or class. Use when asked to add test coverage to existing code.
---

When writing tests:
1. Read the function under test first — understand its inputs, outputs, edge cases, and failure modes
2. Write tests covering: the happy path, empty or null inputs, boundary values, and expected errors
3. Look at existing test files to match the project's testing framework and naming conventions
4. Add a one-line comment above each test explaining what behaviour it verifies
5. After writing the tests, check if they would actually catch the bugs they claim to test
```

With this skill present, when you ask Claude to "write tests for `running_total`", it follows this structured process rather than improvising. The output is consistent, thorough, and matches your project's style.

### Skills vs. Agents

The distinction between a skill and an agent is worth understanding clearly before Chapter 3:

| | Skill | Agent |
|---|---|---|
| **Scope** | One invocation, one task | Multi-step, autonomous loop |
| **Memory** | Stateless | Can maintain memory across steps |
| **File** | `.claude/skills/*.md` | `.claude/agents/*.md` |
| **Use when** | Consistent behavior for a task type | Autonomous multi-step work |

Think of a skill as a recipe and an agent as a chef who can follow multiple recipes, check the results, and adapt.

### Available Skills in the Ecosystem

Chapter 7 covers the full ecosystem, but know that community repos like `obra/superpowers` and `affaan-m/everything-claude-code` ship dozens of production-tested skills — test-driven-development, database-migrations, api-design, code-review checklists, and more. You can install them directly or use them as templates for your own.

---

**Exercise 2-B: Your First Skill File**

**Goal:** Create a reusable skill that shapes how Claude writes tests, then compare output with and without it.

**Setup:** Create `src/calculator.ts` if it doesn't exist:
```typescript
function runningTotal(transactions: number[]): number {
  let total = 0;
  let i = 0;
  while (i < transactions.length) {
    total += transactions[i];
    i++;
  }
  return total;
}
```

Create `.claude/skills/write-tests.md`:
```markdown
---
name: write-tests
description: Write tests for any function or class. Use when asked to add test coverage to existing code.
---

When writing tests:
1. Read the function under test first — understand its inputs, outputs, edge cases, and failure modes
2. Write tests covering: the happy path, empty or null inputs, boundary values, and expected errors
3. Look at existing test files to match the project's testing framework and naming conventions
4. Add a one-line comment above each test explaining what behaviour it verifies
5. After writing the tests, check if they would actually catch the bugs they claim to test
```

Now type:
```
Write tests for the runningTotal function in src/calculator.ts
```

**Observe:** Claude follows the skill's structured process — reads the function first, writes tests for edge cases, adds descriptive comments above each test.

Now temporarily rename the skill file (e.g., `write-tests.md.bak`) and run the exact same prompt again.

**Observe:** Without the skill, Claude's output is less structured. It may skip edge cases, omit comments, or invent a testing framework rather than matching your project's style.

**What to experiment with:**
- Add `model: haiku` to the frontmatter and re-run — does speed increase? Does quality change?
- Modify the skill to add: "Always include a test with the maximum realistic input size"
- Create a second skill `.claude/skills/explain-code.md` that explains any function in plain English, then invoke both on `process_order`

---

**Exercise 2-D: Build a Custom Skill**

**Goal:** Design a skill from scratch for a task you repeat often, then validate it produces consistent output.

**Setup:** No new files needed. Use your existing `src/orderProcessor.ts` and `src/calculator.ts`.

Create `.claude/skills/code-explainer.md`:
```markdown
---
name: code-explainer
description: Explain any function or class in plain English. Use when asked to explain, document, or describe what code does.
---

When explaining code:
1. Read the function or class fully before writing anything
2. Describe in one sentence what the code does (the "what")
3. Describe why it exists — what problem it solves (the "why")
4. List the inputs and what each one represents
5. Describe the output and what it means
6. Rate the complexity: Simple / Moderate / Complex, with a one-sentence justification
7. If there are non-obvious parts (clever logic, edge case handling, performance tricks), call them out explicitly
8. Keep the explanation accessible to a developer who hasn't seen this code before
```

Now type:
```
Explain the processOrder function in src/orderProcessor.ts
```

**Observe:** Claude follows the structured process — produces a consistent format with a "what", "why", input/output descriptions, a complexity rating, and callouts for non-obvious parts.

Run it again on `runningTotal`:
```
Explain the runningTotal function in src/calculator.ts
```

**Observe:** The output structure is identical even though the functions are very different. That consistency is the value of skills.

**What to experiment with:**
- Add an `audience` parameter to the skill: "If the requester specifies an audience (junior / senior / non-technical), adjust vocabulary and depth accordingly"
- Chain this skill with `write-tests`: explain a function first, then ask Claude to write tests based on that explanation
- Ask Claude to invoke the skill on a function it finds confusing — does the structured format help it catch its own misunderstandings?

---

## 3. Plugins

Plugins are the packaging and distribution layer for skills, commands, and integrations. While a skill file in `.claude/skills/` is local to your project, a plugin can be installed globally, shared with a team, and published to a registry.

### What Plugins Are and How They Work

A plugin is a package (distributed via npm) that Claude Code loads when you start a session. Each plugin can:

- Add one or more slash commands
- Ship skill files that become available immediately on install
- Integrate with external tools (language servers, build tools, CI systems)
- Define hooks that run automatically on events (e.g., before every commit)

When you run `/plugin install`, Claude Code downloads the plugin, registers it, and makes its commands and skills available. Installed plugins persist across sessions and are visible in `/plugin list`.

### The Official Plugin Ecosystem

The `claude-plugins-official` registry is the primary source. Key plugins:

**`claude-code-setup`** — Scans your codebase and recommends plugins, MCPs, hooks, and workflows for your stack.
```
> /plugin install claude-code-setup@claude-plugins-official
```
After installing, just ask Claude conversationally: "recommend automations for this project" or "what hooks should I use?" It analyzes your project structure and gives you a tailored setup checklist — no slash command needed.

**`commit-commands`** — Automates git workflows.
```
> /plugin install commit-commands@claude-plugins-official
```
Adds:
- `/commit` — stages and commits with a generated message
- `/commit-push-pr` — commits, pushes, and opens a PR
- `/clean-gone` — removes local branches tracking deleted remotes

**Language server plugins** — Give Claude the same real-time diagnostics and type-checking your editor has:
```
> /plugin install typescript-lsp@claude-plugins-official  # requires: npm install -g typescript-language-server
> /plugin install csharp-lsp@claude-plugins-official      # requires: dotnet tool install --global csharp-ls
> /plugin install pyright-lsp@claude-plugins-official     # requires: pip install pyright
```

**`feature-dev`** — Guides you through a structured 7-phase feature development workflow (Discovery → Exploration → Questions → Architecture → Implementation → Review → Summary):
```
> /plugin install feature-dev@claude-plugins-official
> /feature-dev add-user-invitations
```

**`code-review`** — Runs an automated PR review using 4 parallel agents (functionality, performance, security, style):
```
> /plugin install code-review@claude-plugins-official
> /code-review
```

**`claude-md-management`** — Keeps your `CLAUDE.md` accurate as your project evolves:
```
> /plugin install claude-md-management@claude-plugins-official
> /revise-claude-md
```
To audit your CLAUDE.md quality, ask Claude conversationally: "audit my CLAUDE.md files" — this triggers the `claude-md-improver` skill built into the plugin.

### Suggested Install Order

Start lean and add as you need:
1. `claude-code-setup` — let it recommend the rest
2. `commit-commands` — you'll use this daily
3. Your language's LSP plugin
4. `feature-dev` when you tackle larger features
5. `code-review` once you trust Claude's output quality

### Creating a Custom Plugin

A plugin is a directory with a `package.json` and a `plugin.md` (or similar entry point). At minimum:

```
my-plugin/
  package.json
  skills/
    my-skill.md
  commands/
    my-command.md
  README.md
```

`package.json`:
```json
{
  "name": "my-plugin",
  "version": "1.0.0",
  "description": "My custom Claude Code plugin",
  "claude-plugin": {
    "skills": ["skills/my-skill.md"],
    "commands": ["commands/my-command.md"]
  }
}
```

Install it locally:
```
> /plugin install ./my-plugin
```

It now appears in `/plugin list` and its skills and commands are available in every session.

---

**Exercise 2-C: Install Your First Plugin Set**

**Goal:** Bootstrap your project with the right plugins and observe how they extend Claude Code's capabilities.

**Setup:** Add a few more files to give the setup analyzer context.

Create `package.json`:
```json
{
  "name": "claude-training",
  "version": "1.0.0",
  "scripts": {
    "test": "jest",
    "dev": "ts-node src/app.ts"
  },
  "dependencies": {
    "express": "^4.18.0",
    "better-sqlite3": "^9.0.0"
  },
  "devDependencies": {
    "@types/express": "^4.17.0",
    "@types/jest": "^29.0.0",
    "jest": "^29.0.0",
    "ts-jest": "^29.0.0",
    "typescript": "^5.0.0"
  }
}
```

Create `src/calculator.test.ts`:
```typescript
import { runningTotal } from './calculator';

test('returns 0 for empty array', () => {
  expect(runningTotal([])).toBe(0);
});

test('returns value for single item', () => {
  expect(runningTotal([10])).toBe(10);
});
```

Install the foundational plugins:
```
/plugin install claude-code-setup@claude-plugins-official
```
```
/plugin install commit-commands@claude-plugins-official
```
```
/plugin install typescript-lsp@claude-plugins-official
```
```
/plugin install feature-dev@claude-plugins-official
```
```
/plugin install code-review@claude-plugins-official
```

Now run:
```
/plugin list
```

**Observe:** Each plugin appears with its version and the commands it added. Note how many new commands are now available that weren't there before.

Then let the setup analyzer scan the project:
```
Analyse this project and recommend automations.
```

**Observe:** Claude detects the TypeScript/Express/Jest stack and recommends specific hooks, MCPs, and workflow patterns tailored to it.

**What to experiment with:**
- Run `/feature-dev add-health-endpoint` and observe how the 7-phase workflow structures the conversation
- Try `/plugin install explanatory-output-style@claude-plugins-official` — notice how Claude's responses become more educational

---

## 4. Chapter Summary and What's Next

You now have a complete picture of Claude Code's extensibility layer.

### Key Takeaways

- **Slash commands are your full control interface.** `/plan`, `/compact`, `/cost`, `/review`, and `/memory` are as important as the basics. Build the habit of using `/plan` before complex tasks and `/cost` after them.

- **Skills make Claude consistent.** A skill file in `.claude/skills/` gives Claude a structured process for any repeatable task type. The output is predictable, thorough, and matches your project's conventions.

- **Plugins are distributable extensions.** They package skills and commands into shareable, installable units. The official plugin ecosystem gives you production-grade tools for git workflows, code review, feature development, and more.

- **You can build all of this yourself.** Custom slash commands, skill files, and full plugins are all within reach. The format is markdown — no framework required.

### Moving Forward

You're ready for Chapter 3: **Agents**.

Agents take everything you've learned — structured prompts, skills, plan mode — and add autonomy. An agent is a system that observes its environment, reasons about a goal, takes actions, and iterates based on the results. The skills you wrote in this chapter are the building blocks agents use. The `/plan` habit you built here is how you stay in control as agents take on more complex work.

Chapter 3 covers:
- What agents are and how the agent loop works
- Building `.claude/agents/` files
- Giving agents memory that persists across sessions
- Creating specialized personas (security reviewer, QA engineer, etc.)
- Plan → Execute patterns for safe, reviewable automation

### Before Chapter 3

1. Make sure you have at least one skill file committed to your project. You'll build on it in Chapter 3.

2. Run `/plugin install claude-code-setup@claude-plugins-official` on a real project and act on its recommendations. The setup it generates will be useful context as you start building agents.

3. Get comfortable with `/plan`. Before any non-trivial task in Chapter 3, you'll be expected to plan first. Make it a reflex now.
