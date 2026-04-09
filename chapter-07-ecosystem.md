# Chapter 7: The Ecosystem
### Production-Grade Repos That Level Up Your Claude Code Practice

---

## Introduction

You've completed Chapters 1–5 and mastered the fundamentals: agents, skills, commands, and token management. Now it's time to layer in four production-grade repositories that extend Claude Code into an enterprise-ready SDLC platform.

**How they layer:**
1. **claude-plugins-official** (foundation) — LSPs, git tools, code review, markdown management
2. **Superpowers OR Everything-Claude-Code** (workflow discipline) — choose one or both
   - Superpowers: strict TDD, enforced code review, 7-phase workflow
   - Everything-Claude-Code: 30 language-specific agents, 136+ battle-tested skills, flexibility
3. **SuperClaude** (orchestration layer) — 30 commands, 20 agents, research, PM, behavioral modes

**When to use which:**
- Building a team enforcing TDD? → Start with **Superpowers**
- Working in a specific language needing depth? → Use **Everything-Claude-Code**
- Managing large features across sprints? → Add **SuperClaude**
- Want LSP, git automation, code review? → You always need **claude-plugins-official**

**Your starting point:** You have `claude-training/` with src/, test/, CLAUDE.md, and `.claude/agents/` containing agents you've built.

---

## Section 1: anthropics/claude-plugins-official

**What it is:** The official Anthropic plugin marketplace — 32 production plugins you install with `/plugin install`. The foundation layer everyone needs.

**Key plugins:**
- `claude-code-setup` — scan your project, recommend automations
- `commit-commands` — `/commit`, `/commit-push-pr`, `/clean_gone` for git automation
- `feature-dev` — 7-phase structured feature workflow
- `code-review` — 4 parallel review agents
- `pr-review-toolkit` — 6 specialist review agents
- `claude-md-management` — `/revise-claude-md` to keep CLAUDE.md current
- `hookify` — create rules that intercept dangerous operations
- `ralph-loop` — self-iterating agent loop for automation
- Language servers — typescript-lsp, csharp-lsp, pyright-lsp, gopls-lsp, rust-analyzer-lsp

---

### Exercise R-1-A: Explore the Plugin Marketplace

**Goal:** Discover what plugins exist and understand what they do for your project.

**Setup:** You're in a Claude Code session in your `claude-training/` folder with git initialized.

Then type:
```
/plugin install claude-code-setup@claude-plugins-official
```

**Observe:** Claude Code installs the plugin and confirms it's ready.

Now type:
```
/plugin list
```

**Observe:** See all installed plugins and their descriptions.

**What to experiment with:**
- Install `explanatory-output-style` and ask Claude a question — notice the different response format
- Install `code-review` and type `/code-review src/app.ts` — see a full code review run
- Run `/plugin search authentication` to find plugins for a specific domain

---

### Exercise R-1-B: Automate Your Git Workflow with commit-commands

**Goal:** Use commit-commands to streamline the commit → push → PR workflow.

**Setup:** Make a change to a file in your project (edit `src/app.ts` or similar), then stage it:
```
git add src/app.ts
```

Install commit-commands:
```
/plugin install commit-commands@claude-plugins-official
```

Then type:
```
/commit
```

**Observe:** Claude analyzes your staged diff and generates a conventional commit message. It shows you the message before committing.

Now make another change, stage it, and type:
```
/commit-push-pr
```

**Observe:** Claude commits, pushes to a new branch, and creates a pull request.

After PR creation, type:
```
/clean_gone
```

**Observe:** Claude removes stale local branches.

**What to experiment with:**
- Make three small commits, then run `/commit` on each — notice the conventional message format
- Create a multi-file change and run `/commit-push-pr` — compare the PR title and description
- Run `/clean_gone` after multiple branch creations — see which branches it cleans

---

### Exercise R-1-C: Structure a Feature with feature-dev

**Goal:** Experience a guided 7-phase feature development workflow.

**Setup:** You're ready to build a new feature. Install feature-dev:
```
/plugin install feature-dev@claude-plugins-official
```

Then type:
```
Use the feature-dev workflow to add a pagination feature to the task list
```

**Observe:** Claude activates the 7 phases in sequence:
1. **Discovery** — what is pagination? what are the requirements?
2. **Exploration** — what libraries/patterns exist?
3. **Questions** — Claude asks clarifying questions (page size, sorting, cursor vs offset?)
4. **Architecture** — proposed structure (models, API response format)
5. **Implementation** — write code phase by phase
6. **Review** — self-review against requirements
7. **Summary** — recap what was built and what's next

**What to experiment with:**
- Interrupt at the Architecture phase and redirect the design — see how Claude adapts
- Complete the feature, then ask "what would a production-grade implementation look like?" — see the level of detail difference
- Run feature-dev on a bug fix instead of a feature — notice how it adapts the phases

---

### Exercise R-1-D: Enforce Rules with hookify

**Goal:** Create hooks that enforce rules and block dangerous operations.

**Setup:** Install hookify:
```
/plugin install hookify@claude-plugins-official
```

Then type:
```
Use hookify to create a rule: never run 'git push --force' without asking me first
```

**Observe:** Hookify creates a git hook that intercepts `git push --force` and requires your confirmation before proceeding.

Now create a second rule:
```
Use hookify to add a rule: block any deletion of files in src/ or test/ directories unless I explicitly confirm
```

**Observe:** Hookify creates a pre-delete hook.

Try to trigger one of the rules:
```
git push --force
```

**Observe:** The hook intercepts and asks for confirmation.

**What to experiment with:**
- Create 3 rules for different operations (e.g., `git reset --hard`, `rm -rf`, database migrations)
- Try to bypass a hook (you can't with hookify—it's strict)
- Review the hook files in `.git/hooks/` — see what hookify generates

---

### Exercise R-1-E: Build Your Own Plugin (Intro)

**Goal:** Understand plugin structure so you can extend Claude Code for your team's workflow.

**Setup:** Install the plugin-dev plugin:
```
/plugin install plugin-dev@claude-plugins-official
```

Then type:
```
/plugin-dev:create-plugin standup-helper
```

**Observe:** Claude walks you through an 8-phase plugin creation process:
1. **Define purpose** — what does the plugin do?
2. **Choose agents** — which agents does it need?
3. **Define skills** — what skills will agents use?
4. **Define commands** — what commands trigger it?
5. **Setup file structure** — scaffold the plugin directory
6. **Write agent code** — create the agents
7. **Write skill code** — create the skills
8. **Test and verify** — confirm it works

The plugin-dev plugin generates all the necessary files.

**Observe:** After completion, you have a working plugin in `.claude/plugins/standup-helper/` with agents and skills ready to use.

**What to experiment with:**
- Open the generated plugin files and read the structure — identify agents, skills, and command definitions
- Modify one skill and test it with the plugin live
- Create a second plugin for a different use case (e.g., "daily-review", "api-doc-generator")

---

> 💡 **Tip:** The plugins-official layer is your foundation. Everyone on the team should install commit-commands, claude-code-setup, and code-review first. Everything else is optional.

---

## Section 2: obra/superpowers

**What it is:** A plugin that enforces disciplined, TDD-first development with git isolation, parallel agent dispatch, and mandatory code review.

**Install:**
```
/plugin install superpowers@claude-plugins-official
```

**The 14 skills:**
- brainstorming, writing-plans, test-driven-development
- systematic-debugging, using-git-worktrees, dispatching-parallel-agents
- subagent-driven-development, requesting-code-review, receiving-code-review
- executing-plans, verification-before-completion, finishing-a-development-branch
- using-superpowers, writing-skills

**The 7-phase workflow:**
1. **Brainstorming** — clarify requirements and design
2. **Git Worktrees** — create isolated branch with verified baseline
3. **Planning** — break feature into 2–5 minute tasks
4. **Execution** — subagents handle each task in parallel
5. **TDD** — RED → GREEN → REFACTOR for each task
6. **Code Review** — severity-based review, blocks on critical issues
7. **Branch Completion** — verify tests pass, merge cleanly

---

### Exercise R-2-A: Install and Explore Superpowers

**Goal:** Get Superpowers installed and understand its 14 skills and 7-phase workflow.

**Setup:** You're in a Claude Code session in `claude-training/`.

Then type:
```
/plugin install superpowers@claude-plugins-official
```

**Observe:** Superpowers installs as a plugin.

Now type:
```
What skills does the superpowers plugin provide?
```

**Observe:** Claude lists all 14 skills with descriptions of when each is used.

Type:
```
Explain the superpowers 7-phase workflow and why each phase matters
```

**Observe:** Claude describes the workflow and the reasoning (isolation, testing, review, verification).

**What to experiment with:**
- Ask "which superpowers skill should I use for X?" where X is a task you're working on
- Request "show me an example of the RED-GREEN-REFACTOR cycle in the test-driven-development skill"
- Ask "what's the difference between subagent-driven-development and executing-plans?"

---

### Exercise R-2-B: Run Brainstorm → Plan → Execute

**Goal:** Experience the first three phases of Superpowers on a real feature.

**Setup:** You're ready to build a feature. You have a clear idea of what you want (e.g., "add rate limiting to the API").

Then type:
```
I want to add a rate-limiting feature to the task manager API. Use the superpowers brainstorming and planning skills to prepare for implementation.
```

**Observe:**
- Claude activates brainstorming — asks clarifying questions (rate limit per user? per IP? token bucket or sliding window?)
- You answer the questions
- Claude activates planning — breaks the feature into 2–5 minute tasks with clear dependencies

A good plan looks like:
```
Task 1 (5 min): Add rate_limit config to CLAUDE.md
Task 2 (3 min): Create src/middleware/rateLimiter.test.ts with failing tests
Task 3 (4 min): Implement RateLimiter class in src/middleware/rateLimiter.ts
Task 4 (2 min): Integrate RateLimiter into src/app.ts
Task 5 (3 min): Run full test suite, verify coverage
```

**What to experiment with:**
- Answer the brainstorm questions in minimal detail vs thorough detail — notice how plan quality changes
- Ask Claude to split Task 3 into smaller sub-tasks
- Request "reorder the plan to get a demo-able state faster" — see how it prioritizes

---

### Exercise R-2-C: Mandatory TDD with Superpowers

**Goal:** Experience the RED-GREEN-REFACTOR cycle enforced by Superpowers.

**Setup:** You have a plan. You're ready to implement one task. Example: implement a `rate_limit?` method.

Then type:
```
Use the test-driven-development skill from superpowers to implement an isRateLimited function in src/middleware/rateLimiter.ts. Here's the requirement: it should return true if the current user has exceeded their rate limit (100 requests per hour).
```

**Observe:** Superpowers forces this sequence:
1. **RED** — write a failing test first in `src/middleware/rateLimiter.test.ts`
   ```
   test('returns true if user exceeds 100 requests per hour', () => {
     // setup: create a user with 101 requests in the last hour
     // assertion: expect(isRateLimited(user)).toBe(true)
   });
   ```
2. Run the test — it fails (RED)
3. **GREEN** — implement the method in `src/middleware.rb`
4. Run the test — it passes (GREEN)
5. **REFACTOR** — improve the code without changing behavior
6. Re-run the test — still passes

If you try to skip "write failing test first," Superpowers will ask "Where's the test?" and won't let you proceed.

**What to experiment with:**
- Intentionally try to implement before writing a test — Superpowers blocks you
- Write a test that's too broad, then refine it — see how Superpowers guides you to smaller, focused tests
- After GREEN, ask "what refactors make sense?" — Superpowers suggests improvements (extract method, rename variable, etc.)

---

### Exercise R-2-D: Dispatch Parallel Agents

**Goal:** Use Superpowers to run three independent tasks simultaneously.

**Setup:** You have three independent functions to implement (no dependencies between them).

Then type:
```
Use the dispatching-parallel-agents skill to implement these 3 functions simultaneously:
- paginate_results in src/app.ts (takes query results, returns paginated array)
- format_currency in src/billing.rb (takes amount, returns formatted currency string)
- sanitize_input in src/string_utils.rb (takes string, returns sanitized for HTML)
```

**Observe:**
- Superpowers creates three git worktrees (one for each task)
- Dispatches three subagents, one per task
- Each subagent works independently in parallel
- Status updates show progress across all three tasks
- Subagents commit work to their branches
- Main superpowers agent merges all three branches back to main when complete

**What to experiment with:**
- Add a fourth task and watch it run in parallel with the other three
- Interrupt one task partway through and ask for help — superpowers handles the interruption
- After completion, run `git log --oneline` and count the commits — you'll see clean, atomic commits from each subagent

---

### Exercise R-2-E: Full 7-Phase Feature Development

**Goal:** Complete an entire feature using the Superpowers 7-phase workflow end-to-end.

**Setup:** You're ready for a full feature. You have a clear feature requirement (e.g., "add task priority levels: high, medium, low").

Then type:
```
Use the full superpowers workflow to add task priority levels (high/medium/low) to the task manager. Users should be able to set priority when creating a task, and filter tasks by priority.
```

**Observe:** Superpowers runs all 7 phases:
1. **Brainstorming** — Claude asks clarifying questions
2. **Git Worktrees** — creates a worktree with clean baseline
3. **Planning** — breaks into 2–5 minute tasks
4. **Execution** — subagents implement tasks (possibly in parallel)
5. **TDD** — each task is RED-GREEN-REFACTOR
6. **Code Review** — Superpowers reviews code, flags severity levels, blocks on critical issues
7. **Branch Completion** — merges to main after all checks pass

When complete, you have:
- A fully implemented feature with passing tests
- Clean git history with atomic commits
- A code review record of what was checked
- No manual merge conflicts (Superpowers handles them)

**What to experiment with:**
- After the workflow, run `git log --oneline` and count commits — notice the clarity
- Ask "what would a v2 enhancement look like?" at the end — see how Superpowers sets you up for iteration
- Run `git diff origin/main` — see exactly what changed (should be minimal, focused)

---

> ⚠️ **Warning:** Superpowers enforces TDD strictly. If you try to skip the "write test first" step, it won't let you. This is intentional—it's the discipline enforcement.

---

## Section 3: affaan-m/everything-claude-code

**What it is:** A comprehensive optimization system from 10+ months of intensive daily use. Not a plugin — it's a repository you clone. Provides 30 agents, 136+ skills, 60+ commands, and language-specific rules.

**5 pillars:**
- Guides (Shorthand, Longform, Security)
- 30 Specialized Agents (language-specific, domain-specific)
- 136+ Battle-Tested Skills
- 60+ Commands
- Language Rules (Python, Ruby, Go, TypeScript, etc.)

**Install:**
```bash
git clone https://github.com/affaan-m/everything-claude-code.git
cd everything-claude-code
# Follow README for setup — copy agents/ and skills/ into your .claude/ folder
```

---

### Exercise R-3-A: Read the Shorthand Guide First

**Goal:** Understand the philosophy before using any tools.

**Setup:** Clone the repo:
```bash
git clone https://github.com/affaan-m/everything-claude-code.git
```

Then type:
```
Read everything-claude-code/guides/shorthand-guide.md and summarize the 5 most important principles for using this system effectively
```

**Observe:** Claude reads the guide and extracts the core principles (likely: research before coding, token management, memory persistence, reusable patterns, iterative refinement).

Now type:
```
Which of these 5 principles have you already been using in chapters 1-5? Which are new?
```

**Observe:** Claude relates the guide to what you've learned, showing continuity and new insights.

**What to experiment with:**
- Ask "what's the token cost difference between a shorthand vs longform guide for the same task?"
- Request "give me 3 examples of research-first development from chapter 3"
- Ask "how does the security guide complement the agent design from chapter 2?"

---

### Exercise R-3-B: Use a Language-Specific Agent

**Goal:** Invoke a specialized agent that knows your language deeply.

**Setup:** Copy the TypeScript language agent from the cloned repo into your project:

```bash
cp everything-claude-code/agents/typescript-reviewer.md .claude/agents/
# (or python-reviewer, go-reviewer, etc.)
```

Now you have a specialized agent in your `.claude/agents/` folder. Type:
```
Use the typescript-reviewer to review src/paymentProcessor.ts and flag any security or performance issues
```

**Observe:** The typescript-reviewer agent has deep TypeScript-specific knowledge:
- Idiomatic patterns (type narrowing, generics, utility types)
- Security (injection patterns, unsafe type assertions, sensitive data logging)
- Performance (unnecessary re-renders, promise chains, memory leaks)
- Testing patterns (mocking, async tests, type-level tests)

**What to experiment with:**
- Compare typescript-reviewer output vs your generic code-review from R-1-B — notice the TypeScript-specific depth
- Ask the typescript-reviewer "what would production TypeScript look like?" on one of your files
- Use the typescript-reviewer on test files — it will flag weak test patterns

---

### Exercise R-3-C: Install a Skill Pack

**Goal:** Add battle-tested domain skills from everything-claude-code to your project.

**Setup:** Browse the `skills/` folder in the cloned repo. Pick 3 skills that match your work:
- `skills/tdd.md` (comprehensive test-driven development)
- `skills/database-migrations.md` (safe DB changes)
- `skills/api-design.md` (REST principles and patterns)
- `skills/error-handling.md` (comprehensive error strategies)

Copy them to your project:
```bash
cp everything-claude-code/skills/tdd.md .claude/skills/
cp everything-claude-code/skills/database-migrations.md .claude/skills/
```

Now type:
```
Use the tdd skill to add test coverage to src/search.ts. Current coverage is 45%. Target is 85%.
```

**Observe:** The skill file walks you through:
- Coverage analysis (which lines/branches are uncovered?)
- High-impact tests (what tests catch the most bugs?)
- Edge cases (what inputs break the code?)
- Refactoring for testability (if code is hard to test, how do you change it?)

The skill from everything-claude-code is more battle-tested than one you'd write from scratch.

**What to experiment with:**
- Read the skill file you copied — identify 3 things it does better than the write-tests skill from Chapter 2
- Apply the database-migrations skill to a schema change you're planning
- Ask "which parts of this skill are most relevant to my project?"

---

### Exercise R-3-D: Multi-Agent Planning with Orchestration Commands

**Goal:** Use everything-claude-code's orchestration commands to break down large features.

**Setup:** Everything-claude-code provides `/multi-plan` and `/orchestrate` commands. Type:
```
/multi-plan Implement a complete user authentication system for the task manager. We need registration, login, session management, password reset, and 2FA support.
```

**Observe:** The command breaks the feature into workstreams:
- **Workstream A (3 people):** Database schema, user model, password hashing
- **Workstream B (2 people):** Registration API, email verification
- **Workstream C (2 people):** Login, session token generation, logout
- **Workstream D (1 person):** Password reset flow
- **Workstream E (1 person):** 2FA integration

Each workstream has dependencies documented (Workstream A blocks B, B blocks C, etc.).

Compare with a single-agent plan from Chapter 6 — notice the parallelization.

**What to experiment with:**
- Ask `/multi-plan` to optimize for "fastest demo" instead of "minimal team headcount"
- Request "show me the critical path" — which workstreams block others?
- Add a new constraint: "We only have 2 developers" — see how the plan adapts

---

### Exercise R-3-E: Session Memory via Hooks

**Goal:** Enable session-to-session memory so Claude auto-loads your project state.

**Setup:** Everything-claude-code provides hooks for SessionStart and SessionEnd. Copy them:
```bash
cp everything-claude-code/hooks/* .claude/hooks/
```

These hooks automatically:
- Save your project context at session end
- Load it at session start
- Update CLAUDE.md with progress

Start a new Claude Code session and observe:
```
/briefing
```

**Observe:** Claude shows your project state, recent changes, and open issues — without you having to re-explain anything.

Make a change, then end the session and start a fresh one:
```
What did I change in the last session?
```

**Observe:** Claude recalls the change automatically.

**What to experiment with:**
- End a session mid-task, start a new one, resume the task — see continuity
- Modify the hook to save more/less context
- Run `cat .claude/hooks/session-end-hook.md` to see what's being saved

---

> 💡 **Tip:** Everything-claude-code is deepest for the language you use daily. If you code in Python, the python agents and skills will be most valuable.

---

## Section 4: SuperClaude-Org/SuperClaude_Framework

**What it is:** A meta-programming configuration system with 30 commands, 20 coordinated agents, 7 behavioral modes, and optional MCP integrations. This transforms Claude Code into a full SDLC platform.

**Install:**
```bash
pipx install superclaude
superclaude install
superclaude doctor
```

Or from source:
```bash
git clone https://github.com/SuperClaude-Org/SuperClaude_Framework.git
cd SuperClaude_Framework
./install.sh
```

**Key commands:** `/sc:research`, `/sc:brainstorm`, `/sc:implement`, `/sc:test`, `/sc:pm`, `/sc` (list all 30)

**7 behavioral modes:** Brainstorming, Business Panel, Deep Research, Orchestration, Token-Efficiency, Task Management, Introspection

---

### Exercise R-4-A: Install SuperClaude and Verify

**Goal:** Get SuperClaude installed and explore the 30 commands.

**Setup:** Install SuperClaude:
```bash
pipx install superclaude
superclaude install
```

Verify installation:
```bash
superclaude doctor
```

**Observe:** Output shows all components installed and working (agents, commands, MCP integrations).

In Claude Code, type:
```
/sc
```

**Observe:** Claude lists all 30 commands with descriptions:
- `/sc:research` — autonomous research (up to 5 hops)
- `/sc:brainstorm` — structured ideation
- `/sc:implement` — implementation with behavioral modes
- `/sc:test` — testing and coverage
- `/sc:pm` — project management and roadmaps
- etc.

**What to experiment with:**
- Type `/sc:help` to see detailed help for any command
- Ask "what's the difference between /sc:brainstorm and the superpowers brainstorming skill?"
- Request "show me the behavioral modes and when to use each"

---

### Exercise R-4-B: Research with /sc:research

**Goal:** Use SuperClaude's autonomous research capability.

**Setup:** You have a technical question to research. Example: "What are best practices for rate limiting in Ruby web APIs?"

Then type:
```
/sc:research What are the best practices for rate limiting in TypeScript/Node.js APIs?
```

**Observe:** SuperClaude:
1. Formulates search queries
2. Searches (up to 5 iterative searches)
3. Scores results by relevance
4. Synthesizes a comprehensive answer
5. Lists sources

Compare with a naive question without SuperClaude:
```
What are the best practices for rate limiting in TypeScript/Node.js APIs?
```

**Observe:** SuperClaude's answer is deeper, cites more sources, covers edge cases.

Experiment with exhaustive depth:
```
/sc:research --depth exhaustive What are production patterns for JWT authentication in Node.js?
```

**Observe:** Longer, more thorough research with more sources.

**What to experiment with:**
- Research a topic you know well and verify the quality of SuperClaude's answer
- Ask "which patterns would work best for our task manager?" after the research
- Try `/sc:research --depth quick` on a simpler topic and compare token cost

---

### Exercise R-4-C: Brainstorm with /sc:brainstorm

**Goal:** Use structured brainstorming mode for a technical decision.

**Setup:** You have a design question or architectural decision to make.

Then type:
```
/sc:brainstorm How should we structure the user authentication system for our task manager? We need to support both API keys and JWT tokens.
```

**Observe:** SuperClaude activates Brainstorming mode:
- Explores multiple approaches (session-based, token-based, hybrid)
- Documents trade-offs (stateless vs stateful, scalability, security)
- Proposes decision criteria
- Recommends an approach with reasoning

Now switch to Business Panel mode:
```
/sc:brainstorm --mode business How should we prioritize the auth system given a 2-week deadline and 2 developers?
```

**Observe:** Different perspective — less technical depth, more pragmatic constraints, resource planning.

**What to experiment with:**
- Use Brainstorming mode on a feature you've already decided on — see if it changes your mind
- Ask "what would the Enterprise mode recommend?" (a hint about other behavioral modes)
- Request "show me the trade-offs in a table format"

---

### Exercise R-4-D: Token-Efficiency Mode

**Goal:** Activate the mode that reduces token usage by 30–50%.

**Setup:** You're working exploratory or iteratively. You don't need verbose explanations — you want quick answers.

Get a baseline:
```
/cost
```

Then type:
```
/sc:implement --mode token-efficient Add a simple health check endpoint to src/app.ts
```

**Observe:** SuperClaude in token-efficient mode:
- Uses shorter explanations
- Asks fewer clarifying questions
- Generates minimal comments in code
- Skips nice-to-have details

After the task, check token cost:
```
/cost
```

Compare with the same task in normal mode (start a fresh session, same task, no `--mode` flag). The token-efficient version should be 30–50% cheaper.

**What to experiment with:**
- Time both versions (normal vs token-efficient) — which feels faster?
- Which types of tasks benefit most from token-efficiency? (exploratory > architectural)
- Run a complex task in token-efficient mode and measure the quality trade-off

---

### Exercise R-4-E: PM Agent for Project Coordination

**Goal:** Use SuperClaude's PM agent to organize a multi-sprint roadmap.

**Setup:** You have a large feature to break into sprints.

Then type:
```
/sc:pm Create a 3-sprint roadmap for adding complete user authentication to the task manager. We need: registration, login, session management, password reset, and 2FA. We have 1 person, 2 weeks per sprint.
```

**Observe:** SuperClaude activates the PM agent:
- Breaks into 3 sprints (Sprint 1: registration + login; Sprint 2: session + password reset; Sprint 3: 2FA + hardening)
- Lists deliverables per sprint
- Flags dependencies (Sprint 1 must complete before Sprint 2)
- Estimates effort
- Identifies risks (2FA complexity, timezone handling in password reset)
- Suggests cut-line if scope overruns

Now ask the PM agent to re-prioritize:
```
The client needs login working by end of sprint 1 — reorganize accordingly
```

**Observe:** PM agent adapts the roadmap, possibly deferring password reset or 2FA to later sprints.

**What to experiment with:**
- Add a new constraint: "We have 2 developers, each can do 2 features per sprint"
- Ask "what's the critical path?" — which features block others?
- Request "show me a risk/effort matrix for each feature"

---

> ⚠️ **Warning:** SuperClaude is powerful but adds overhead. Use it for research-heavy tasks, large features, and PM tasks — not every simple coding task.

---

## Section 5: How They Work Together — The Full Stack

**The layering:**

1. **Layer 1 (Everyone):** claude-plugins-official
   - Install: commit-commands, code-review, claude-md-management, claude-code-setup
   - Benefit: better git workflow, code quality, markdown management

2. **Layer 2 (Team discipline):** Choose superpowers OR everything-claude-code OR both
   - Superpowers: for teams that want strict TDD enforcement and 7-phase discipline
   - Everything-claude-code: for teams that want language depth and flexibility
   - Both: for teams that want discipline + specialized agents

3. **Layer 3 (Large-scale):** SuperClaude
   - For research-heavy projects
   - For multi-sprint roadmapping
   - For SDLC orchestration
   - For resource-constrained teams (token-efficient mode)

**When each activates:**
- You run a `/commit-commands` action → Layer 1 kicks in
- You invoke `/using-superpowers` → Layer 2 (Superpowers) kicks in
- You run `/sc:research` → Layer 3 kicks in
- You use a skill from everything-claude-code → Layer 2 (Everything-Claude-Code) kicks in

They don't conflict — they're designed to work together.

---

### Exercise R-5-A: The Full Stack in Action

**Goal:** Use all 4 repos on a single real feature to see how they orchestrate.

**Setup:** You have a feature to build. Install everything:
- Layer 1: `/plugin install commit-commands code-review claude-md-management claude-code-setup`
- Layer 2a: `/plugin install superpowers`
- Layer 2b: Clone everything-claude-code and copy agents/skills
- Layer 3: `superclaude install`

Now build a feature using all layers:

**Step 1: Research** (Layer 3 — SuperClaude)
```
/sc:research What are the current best practices for implementing user authentication in web APIs (2026)?
```

**Observe:** SuperClaude researches and synthesizes best practices.

**Step 2: Brainstorm** (Layer 3 — SuperClaude)
```
/sc:brainstorm Based on the research, how should we architect authentication for our task manager?
```

**Observe:** SuperClaude brainstorms the architecture.

**Step 3: Plan** (Layer 2a — Superpowers)
```
Use the superpowers planning skill to break down the authentication implementation into 2-5 minute tasks
```

**Observe:** Superpowers creates a granular plan.

**Step 4: Implement** (Layer 2b — Everything-Claude-Code language agent)
```
Use the typescript-reviewer and tdd skill to implement the authentication according to the plan
```

**Observe:** Language-specific agent implements with TDD.

**Step 5: Code Review** (Layer 1 — claude-plugins-official)
```
/code-review src/auth/
```

**Observe:** Automated code review with the code-review plugin.

**Step 6: Commit** (Layer 1 — claude-plugins-official)
```
git add .
/commit-push-pr
```

**Observe:** commit-commands creates a clean commit and PR.

**Final: Verify full integration**
```
/briefing
```

**Observe:** Claude shows the complete context (Layer 1 + Layer 2 + Layer 3 all operating together).

**What to experiment with:**
- Remove one layer (e.g., skip /sc:research) and feel the difference in depth
- Build the same feature using only Layer 1 + 2a (Superpowers) — compare quality and token cost
- Identify which tool you'd drop first if you could only keep 3 — that tells you which matters most

---

> 💡 **Tip:** The full stack is powerful but expensive (tokens). For small features, Layer 1 + 2 is usually enough. Add Layer 3 for research-heavy work, multi-sprint planning, or when you're stuck.

---

## Section 6: Learning Path for Each Repo

### claude-plugins-official

**Day 1:**
- Install commit-commands, code-review, claude-md-management
- Run `/commit` on a real change
- Run `/code-review` on one file
- Run `/revise-claude-md` to update your CLAUDE.md

**Week 1:**
- Install feature-dev and run a complete 7-phase feature
- Install hookify and create 2–3 rules (git push --force, file deletion)
- Use /plugin search to find plugins for your specific language/domain
- Install LSP for your language (ruby-lsp, pyright-lsp, etc.)

**Month 1 — Mastery:**
- You run `/commit-push-pr` without thinking — git workflow is automated
- Code reviews are systematic (every PR goes through /code-review)
- Your CLAUDE.md stays current (run /revise-claude-md weekly)
- You've created 1–2 custom plugins for your team's workflows
- **How you know you've reached mastery:** New teammates ask you how to set up git automation — you have a repeatableprocess

---

### Superpowers

**Day 1:**
- Install superpowers
- Run the 7-phase workflow on one small feature
- Experience RED-GREEN-REFACTOR in the test-driven-development skill

**Week 1:**
- Complete a full feature using all 7 phases
- Run dispatching-parallel-agents on a 3-task feature
- Try interrupting a phase and redirecting — get comfortable with the workflow
- Build your first custom skill (use writing-skills)

**Month 1 — Mastery:**
- You automatically reach for Superpowers for any significant feature
- TDD is instinctive (test first, every time)
- Parallel agent dispatch is your default (never sequential if tasks are independent)
- You've mentored 1–2 teammates through the workflow
- **How you know you've reached mastery:** You feel uncomfortable coding without Superpowers — the discipline has stuck

---

### everything-claude-code

**Day 1:**
- Clone the repo
- Read guides/shorthand-guide.md
- Copy the language-specific agent to your .claude/agents/
- Run that agent on one file

**Week 1:**
- Copy 3–5 skills to your project (pick ones for your language/domain)
- Use /multi-plan on a 3-task feature
- Set up the session hooks (SessionStart, SessionEnd)
- Read the language rules for your language

**Month 1 — Mastery:**
- You've used 15+ skills from the repo (you know what's available)
- Session continuity is automatic (you never lose context)
- The language-specific agent is your default for reviews (better than generic)
- You've contributed back 1–2 custom skills or agents to your team
- **How you know you've reached mastery:** A teammate asks "how should we structure this?" and you reply with a link to a skill from everything-claude-code — it's your reference system

---

### SuperClaude

**Day 1:**
- Install SuperClaude
- Run `superclaude doctor` to verify
- Run `/sc` to see all 30 commands
- Try `/sc:help` on 3 commands

**Week 1:**
- Run `/sc:research` on a technical question you're researching
- Run `/sc:brainstorm` on an architectural decision
- Compare token cost: normal vs `/sc:implement --mode token-efficient`
- Create a 3-sprint roadmap with `/sc:pm`

**Month 1 — Mastery:**
- You use `/sc:research` for any research task (vs hunting Stack Overflow)
- `/sc:pm` is your go-to for roadmapping (instead of manual spreadsheets)
- Token-efficient mode is your default for iterative work
- You've identified 3–4 behavioral modes that match your workflow
- **How you know you've reached mastery:** You explain the behavioral modes to a new teammate and they understand when to use each — you're the expert

---

## Key Takeaways

1. **Start with plugins-official** — it's the foundation. Every team needs commit-commands and code-review.

2. **Pick one for Layer 2** (workflow discipline):
   - Superpowers if your team values strict TDD and enforced review
   - Everything-Claude-Code if your team wants language depth and flexibility

3. **Add SuperClaude only if** you do research-heavy work, multi-sprint planning, or need token optimization.

4. **Each repo solves a different problem:**
   - plugins-official: automation + foundational tools
   - Superpowers: discipline + enforced workflow
   - everything-claude-code: depth + battle-tested patterns
   - SuperClaude: orchestration + research + SDLC

5. **They layer without conflict** — run them together, not separately.

6. **The full stack is powerful but expensive** — start with Layer 1, add Layer 2, add Layer 3 only when you need it.

---

## Next Steps

1. Pick one exercise from each section (R-1-B, R-2-B, R-3-B, R-4-B) and complete them today
2. Complete Exercise R-5-A this week (the full stack)
3. Follow the learning path for the repos your team chooses
4. Review with your team in 1 month: which layer is most valuable? which should you expand?

Good luck building with the full Claude Code ecosystem.
