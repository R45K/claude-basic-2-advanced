# Chapter 1: Basics — Understanding Claude Code and Agentic Development

Welcome to Claude Code. If you're reading this, you've likely spent time chatting with Claude about code, asking questions, getting explanations, maybe even having it write functions for you. That works fine for isolated snippets, but when you start working on real projects—codebases with dozens of files, complex interdependencies, evolving requirements, and the need for consistency across changes—the chat interface becomes a bottleneck.

Claude Code is different. It's not a chat interface pretending to be a developer tool. It's a CLI environment designed from the ground up for agentic development: Claude operating directly on your codebase, with persistent memory, real-time file access, the ability to execute tools, integrate with your workflows, and maintain context across sessions.

This chapter will take you from zero to productive with Claude Code. We'll start with the conceptual shift you need to make, walk through installation and setup, introduce you to project configuration, teach you how to prompt effectively, show you the tools at your disposal, and finally, give you hands-on exercises to cement your understanding. Expect to spend about 4 hours working through this chapter deeply.

---

## 1. Why Claude Code? Moving from Chat to CLI

### The Limitations of Chat for Developers

When you use Claude through the web chat interface, several constraints immediately become apparent once you start working with real projects:

**Context window exhaustion.** The chat interface has a context limit, typically around 200,000 tokens. For a small script or tutorial, that's fine. But add a 50KB codebase, multiple conversations about different parts, error logs, attempts at fixes, and you run out of space fast. When you hit that limit, the conversation ends. You lose continuity. You start fresh with a new chat, and Claude has no memory of what you were doing, why you made certain architectural choices, or what failed in previous attempts.

**No persistent file access.** Chat requires you to copy-paste code snippets in and out of the browser. This is error-prone: formatting gets mangled, indentation breaks, context is lost. For a 500-line module, you're either pasting the entire thing repeatedly (wasting tokens), or giving Claude only the relevant excerpt and hoping it understands the broader structure. Neither scales.

**No real-time tool execution.** In chat, Claude can't actually run your code, install dependencies, check compiler output, or see test results. It can tell you what it *thinks* will happen, but it's operating with incomplete information. When something goes wrong, you manually run the test, copy the error message back into chat, and Claude debugs blind.

**Context loss between sessions.** Each new chat is a fresh start. You have to re-establish context: "Here's the project structure. Here's our tech stack. Here's what we tried last time and why it didn't work." This repetition burns tokens and mental energy.

**No integration with your workflow.** You can't automatically commit code Claude writes, you can't run it through linters or type checkers, you can't trigger CI/CD pipelines. Every output requires manual integration.

**Limited agency.** Claude in chat is reactive: you ask, it answers. For complex tasks—refactoring a large module, adding a feature across multiple files, fixing a subtle bug—you need back-and-forth dialogue. This is slow and cognitively expensive for both you and Claude.

### What Changes with Claude Code

Claude Code fundamentally restructures how Claude interacts with your codebase. Here's what you get:

**Persistent context across sessions.** Claude Code maintains memory between conversations. Your project configuration (the `CLAUDE.md` file, which we'll cover in detail later) stays loaded. Previous decisions, conventions, and context don't evaporate when you close the terminal and come back tomorrow. This is how Claude transitions from "chatbot answering questions" to "team member who remembers the project."

**Direct file system access.** Claude can read and write files on your machine. No more copy-pasting. It can examine your entire codebase structure, understand how modules depend on each other, and make changes across multiple files with full visibility. When it needs to understand a function, it reads it directly from your source.

**Real-time tool execution.** Claude Code integrates with your development environment. It can run commands (npm, cargo, python, etc.), execute tests, check type errors, install dependencies. It sees the actual output, not a guess. If a test fails, it gets the error message immediately and can adjust its approach. This closes the feedback loop.

**Hooks and plugins.** You can extend Claude Code with custom commands and integrations. The ecosystem includes pre-built plugins for git workflows, language servers that provide real-time diagnostics, structured development processes (like feature development workflows), and more. Your workflow becomes a first-class citizen in Claude Code.

**Token efficiency.** Because Claude Code maintains context and doesn't lose information between sessions, you use tokens more efficiently. You're not re-explaining the project; you're building on what's already established.

**Agentic workflows.** Claude Code supports structured, multi-step workflows where Claude can plan a task, execute it in stages, get feedback, and refine. It's not just answering questions; it's operating with agency to accomplish goals.

### The Mental Model Shift: Claude as a Collaborator

Here's the conceptual leap that matters most: Stop thinking of Claude as a search engine or a smart autocomplete. Start thinking of it as a team member.

In chat, the relationship is transactional. You ask a question, Claude answers, the interaction ends. You're the driver; Claude is the passenger providing information.

In Claude Code, the relationship is collaborative. You set the project up, establish conventions and goals in `CLAUDE.md`, and then Claude operates with some autonomy within that framework. You give it a task: "Add pagination to the users API endpoint." Claude doesn't just write a response; it:

1. Reads your codebase to understand the current architecture
2. Checks your project conventions to match style and patterns
3. Makes changes across multiple files
4. Runs tests to verify the changes work
5. Commits the result with an appropriate message
6. Reports what it did and why

You don't copy-paste; you collaborate. You don't re-explain context; you let it reference the persistent memory.

This shift unlocks productivity. Instead of managing Claude's work (feeding it information, copying outputs, integrating manually), you're directing it at a higher level. Instead of "write a function that does X," you're saying "implement feature Y across the system, following our conventions, making sure tests pass, and clean up afterward."

---

## 2. Installing and Configuring Claude Code

### Installation

For installation instructions, prerequisites, and API key setup, follow the official documentation:

**https://docs.anthropic.com/en/docs/claude-code/getting-started**

The official docs are kept up to date with the latest requirements and platform-specific notes. Complete the installation and verify it works before continuing.

### First Run: Starting Claude Code

Navigate to a project directory (or create a test directory) and start Claude Code:

```bash
claude
```

You'll see a welcome message and a `>` prompt. This is the Claude Code REPL. You can now type prompts and interact with Claude in the context of your project.

**Initializing a new project.** If this is a new project with no `CLAUDE.md` yet, run:

```bash
claude init
```

`claude init` bootstraps the project by generating a starter `CLAUDE.md` based on your current directory structure, detected tech stack, and git history. It asks a few questions interactively and produces a ready-to-edit configuration. You should review and refine the output before committing it—this file will guide every Claude session on the project.

> TIP: Run `claude init` once per project, then keep `CLAUDE.md` updated as the project evolves. Treat it like any other source file: review it in PRs, update it when conventions change.

Let's verify everything is working:

```
> /doctor
```

### The `/doctor` Command

Running `/doctor` is your first diagnostic step. It checks:

- Node.js version
- npm configuration
- API key presence and validity
- Current directory structure
- Git configuration (if in a git repo)
- Installed plugins and hooks
- Available MCPs (tool integrations)

The output tells you what's working and what might need attention. For a fresh installation, you'll see that some things (plugins, MCPs) aren't set up yet. That's fine; we'll cover that in the next sections.

### First Run Tips

**Understand the session model.** Each time you run `claude`, you start a new session. The session maintains memory within that invocation, but when you exit (Ctrl+C), the session ends. Your persistent memory lives in `CLAUDE.md`—the project configuration file we'll cover extensively in the next section.

**Resuming a previous session.** If you closed Claude Code mid-task and want to pick up exactly where you left off—including the full conversation history and tool state—run:

```bash
claude resume
```

Claude Code stores recent sessions locally. `claude resume` lists them by project and timestamp so you can select the one to continue. This is different from just re-running `claude`: that starts fresh, while `resume` restores the conversation context, any pending tasks, and the thread of work you were doing. Use it whenever you need to continue a multi-step task that was interrupted.

**Always start in your project root.** When you `claude`, start from the root of your project. Claude Code reads the `CLAUDE.md` file in the current directory (and parent directories) to load context. If you start in a subdirectory and `CLAUDE.md` is at the root, it might not find it.

**Use `/help` liberally.** At any point, type `/help` to see available commands. Type `/help <command>` to see detailed help for a specific command.

**Session context.** Within a session, Claude maintains a conversation history. This history counts against your token budget. If you're working on a long task and notice performance degrading, you can use `/compact` to compress the conversation history, or `/clear` to clear it (which resets your session but keeps `CLAUDE.md` loaded).

---

## 3. CLAUDE.md — Your Project's Brain

### What CLAUDE.md Is and Why It Matters

`CLAUDE.md` is the single most important file in a Claude Code workflow. It's not code; it's configuration and context written in Markdown. When Claude Code starts, it reads `CLAUDE.md` from your project root and loads it into memory. This file becomes Claude's understanding of your project: its purpose, structure, conventions, tech stack, and guardrails.

Think of it as a standing brief. Instead of explaining your project every time you invoke Claude, you write it once in `CLAUDE.md`. Claude reads it automatically. You stay consistent because Claude follows the same conventions every session.

### What Goes in CLAUDE.md

An effective `CLAUDE.md` covers several categories:

**Project Overview.** A one-paragraph summary of what the project does, who uses it, and why it matters. This is the north star. When Claude needs to make decisions, it references this.

**Tech Stack and Architecture.** What languages, frameworks, and major libraries does the project use? How is it structured? What's the deployment target? Is it a monorepo, a microservice, a library? Are there external dependencies Claude should be aware of?

**Development Environment and Commands.** How do you run tests? How do you start the dev server? Are there setup scripts? What commands are important? Claude needs to know these so it can actually verify its work.

**Coding Conventions and Standards.** What's your preferred style? Do you have linting rules, formatting standards, naming conventions for files or functions? Are there architectural patterns Claude should follow? Do you avoid certain libraries or patterns?

**Important Files and Their Purposes.** For larger projects, call out key files. "The database models live in `src/models`, API routes in `src/routes`, tests in `tests/`." This helps Claude navigate without having to infer structure.

**What Claude Should NOT Do.** This is critical. Are there files Claude shouldn't modify? Should it never add dependencies without asking? Should it avoid certain patterns? Spell it out. This prevents mistakes.

**Decision Logs or Context.** Optionally, include notes about important past decisions. "We use PostgreSQL, not MongoDB, because..." or "We stay on Node 18 LTS for now because..." This helps Claude understand *why* things are the way they are.

### Example CLAUDE.md for a Web API Project

Here's a realistic example:

```markdown
# Acme User API

## Project Overview

The Acme User API is a RESTful service that manages user accounts, authentication,
and profile data for the Acme SaaS platform. It handles 10,000+ requests per second
in production, so performance and reliability are critical. The API is the single
source of truth for user identity across our microservice ecosystem.

## Tech Stack

- **Language:** Node.js (TypeScript)
- **Framework:** Express.js with custom middleware
- **Database:** PostgreSQL 14+ (via pg library)
- **Authentication:** OAuth 2.0 with JWT tokens (issued by auth-service, validated locally)
- **Testing:** Jest + Supertest
- **Deployment:** Docker containers on Kubernetes (via internal platform)
- **Monitoring:** Datadog + custom metrics

## Project Structure

src/
├── routes/          # Express route handlers
├── models/          # Database models and queries
├── middleware/      # Custom Express middleware
├── utils/           # Utility functions
├── tests/           # Test files (mirror src/ structure)
├── index.ts         # Application entry point
└── config.ts        # Environment configuration

## Development Environment

### Prerequisites
- Node.js 18+
- PostgreSQL 14+ (local dev instance, connection string in `.env`)
- Docker (optional, for containerized testing)

### Important Commands

- `npm install` — Install dependencies
- `npm test` — Run full test suite
- `npm run dev` — Start dev server on http://localhost:3000
- `npm run lint` — Run ESLint
- `npm run format` — Format code with Prettier
- `npm run build` — Compile TypeScript to JavaScript

### Database Setup

Local development uses a PostgreSQL database. On first setup:
1. Create a database: `createdb acme_user_api_dev`
2. Run migrations: `npm run migrate`
3. Seed test data: `npm run seed`

## Coding Conventions

### Style and Format

- **Formatter:** Prettier (configured in `.prettierrc`)
- **Linter:** ESLint (configured in `.eslintrc.json`)
- Before committing, run `npm run format && npm run lint`

### Naming

- **Files:** kebab-case for filenames (`user-controller.ts`, not `userController.ts`)
- **Functions/Classes:** camelCase for functions, PascalCase for classes
- **Constants:** SCREAMING_SNAKE_CASE
- **Database tables:** snake_case, plural (`users`, `user_sessions`)

### Architecture Patterns

- **Controllers:** Handle HTTP request/response. Keep thin; delegate to services.
- **Services:** Contain business logic. Database queries, validation, transformations live here.
- **Models:** Database queries and schema definitions. Don't mix with business logic.
- **Middleware:** Authentication, logging, error handling. Keep side-effect-free where possible.

### Error Handling

- Throw custom error classes (see `src/utils/errors.ts`) with appropriate HTTP status codes
- Always include an error message and, where helpful, an error code for clients
- Log errors with context (request ID, user ID, etc.) for debugging

## What Claude Should Do

- Write code that passes linting without comment
- Include tests for new functions or API endpoints
- Update database migrations if schema changes are needed
- Follow the service/controller/model separation strictly
- Use TypeScript types comprehensively; no `any` without strong justification

## What Claude Should NOT Do

- Never modify `package-lock.json` directly; only edit `package.json` and run `npm install`
- Never add new dependencies without asking first (performance and supply chain matter)
- Don't create new files outside the established structure without explicit approval
- Don't modify the Docker configuration or deployment setup
- Don't change existing API contract (routes, request/response schemas) without explicit approval
- Never commit directly; create a commit message and ask for approval before pushing

## Important Context

This API is consumed by three internal services: the web app, mobile app, and analytics service.
Changes to responses, authentication, or error handling can break these consumers. Always consider
backward compatibility, and if breaking changes are necessary, coordinate with service owners.
```

This example is detailed but not overwhelming. It answers the critical questions: What is this? How do I run it? What are the rules? What shouldn't I touch?

### How CLAUDE.md Persists Context Across Sessions

When you exit a Claude Code session and come back later, `CLAUDE.md` is still there. Claude loads it automatically. You don't have to re-explain the project. You just pick up where you left off.

More importantly, `CLAUDE.md` is *version controlled*. It lives in your git repo. It evolves as your project does. If you update your tech stack, move files, or change conventions, you update `CLAUDE.md`, and all future Claude Code sessions reflect those changes.

### The `/revise-claude-md` Command

As your project evolves, `CLAUDE.md` can drift out of sync with reality. The `claude-md-management` plugin (covered in section 6) provides a command to audit and update it:

```
> /revise-claude-md
```

This command asks Claude to review the current state of your project, check what's in `CLAUDE.md`, and suggest updates. It's useful after major refactors, dependency upgrades, or architectural changes.

TIP: Run `/revise-claude-md` periodically (e.g., quarterly) to keep your project documentation fresh. An out-of-date `CLAUDE.md` leads Claude astray.

---

## 4. Prompt Engineering Best Practices

The quality of Claude's output depends directly on the quality of your prompts. This is true in chat, but it's even more critical in Claude Code because you're asking Claude to operate with agency on your codebase. A vague prompt leads to wasted time and rework. A precise prompt gets you exactly what you need.

### Principle 1: Be Specific and Detailed

**Vague prompt:**
```
Add pagination to the users endpoint.
```

This is too open-ended. Claude doesn't know:
- What pagination scheme? Offset/limit? Cursor-based?
- How many results per page?
- Should it be required or optional?
- What response format should the paginated data have?
- Should there be sorting?
- Does pagination affect existing clients?

**Precise prompt:**
```
Add offset/limit pagination to the GET /users endpoint. Implement it so that:
- Clients can pass ?offset=0&limit=50 query parameters
- If not provided, default to offset=0, limit=25
- Include pagination metadata in the response: total_count, offset, limit, has_more
- Update the response schema in the API documentation (docs/api.md)
- Add tests covering edge cases: offset beyond total_count, negative limit, missing params
- Don't break existing clients; pagination should be optional for backward compatibility
```

The second prompt leaves nothing to interpretation. Claude knows exactly what to build, what to test, and what to document.

**The principle:** If you could hand this prompt to a junior developer, would they build it correctly without asking clarifying questions? If not, it's not specific enough.

### Principle 2: Use Positive and Negative Examples

Telling Claude what to do is good. Telling Claude what to do *and* what not to do is better.

**Example without negatives:**
```
Implement a caching strategy for database queries.
```

**Example with negatives:**
```
Implement a caching strategy for database queries. Use Redis for the cache.
Don't use in-memory caching; we need to share the cache across server instances.
Don't cache user-specific data that needs to reflect real-time changes.
Do cache results from expensive aggregate queries and read-heavy endpoints.
```

The negative examples set boundaries and prevent common mistakes. Claude now understands not just what to do, but what pitfalls to avoid.

### Principle 3: Encourage Step-by-Step Thinking

For complex tasks, explicitly ask Claude to think through the problem before jumping to code.

**Direct prompt:**
```
Refactor the authentication middleware to support multiple token types.
```

**Step-by-step prompt:**
```
We need to refactor the authentication middleware to support multiple token types
(currently only JWT). Before writing any code, think through:

1. What are the different token types we need to support?
2. How does each type validate differently?
3. What does the request flow look like for each type?
4. Where does token type detection happen?
5. What existing code will break if we change this?

After thinking it through, outline the refactoring plan, then implement it.
```

The step-by-step prompt forces deliberation. Claude is more likely to catch edge cases and design a robust solution rather than rushing into code.

### Principle 4: Use XML Tags for Structure

For complex requirements, use XML-style tags to structure the prompt. This is especially powerful for multi-faceted tasks.

**Example:**
```
<task>
Add a feature to export user data in multiple formats.
</task>

<requirements>
- Support CSV and JSON formats
- Include user profile, account settings, and activity logs
- Filter data by date range (optional)
- Anonymize sensitive fields (email, IP address) if requested
</requirements>

<constraints>
- Must complete within 5 seconds for users with <100k records
- Must not expose sensitive data unless explicitly requested
- Must use existing database queries where possible
- Must follow the error handling patterns in src/utils/errors.ts
</constraints>

<testing>
- Test with users who have various data volumes (1k, 100k, 1M records)
- Verify sensitive field anonymization works correctly
- Test the date range filter with boundary dates
</testing>

<output_format>
Create a new route GET /users/export?format=csv|json&dateFrom=&dateTo=&anonymize=true|false
Include tests in tests/export.test.ts
```

Structured prompts scale better than prose. They reduce ambiguity and make it easier for Claude to ensure it's covering all bases.

### Principle 5: Specify Output Format Explicitly

Tell Claude not just what to do, but how to present the result.

**Vague:**
```
Explain how to set up the dev environment.
```

**Specific:**
```
Write a setup guide for new developers. Format it as a step-by-step numbered list,
with commands in code blocks. Assume the reader has Node.js and PostgreSQL installed
but hasn't touched this codebase before. Include troubleshooting tips for common issues.
```

### Principle 6: Iterative Prompting

Complex tasks rarely succeed in a single prompt. Break them into stages.

**Wrong approach:** One big prompt asking Claude to refactor a complex module entirely.

**Right approach:**
1. First, have Claude analyze the current structure and suggest an approach.
2. Review the approach. Agree or refine it.
3. Have Claude implement phase 1 (e.g., new interfaces).
4. Have Claude implement phase 2 (e.g., migration logic).
5. Have Claude implement phase 3 (e.g., cleanup of old code).

Iterative prompting lets you steer the work. You catch problems early rather than discovering them when the entire refactor is done.

### Principle 7: Persona Prompting

Sometimes it helps to invoke a specific perspective. This is light, not heavy-handed.

**Example:**
```
Act as a senior engineer who prioritizes code clarity and test coverage.
How would you refactor the payment processing module? Think about:
- What would a careful peer review reveal?
- Where are the potential edge cases?
- What's the simplest, most maintainable approach?
```

This nudges Claude toward a certain mindset. Use sparingly; the goal is better thinking, not role-play.

### Principle 8: Use `/plan` Before Complex Tasks

Claude Code has a `/plan` mode that's specifically designed for complex tasks. You'll learn the details in the next section, but the principle is: before asking Claude to implement a complex feature, ask it to plan first.

```
> /plan

[Describe the task. Claude enters plan mode and thinks through it step-by-step.]

> Looks good, implement phase 1.

[Claude executes the plan.]
```

Plan mode separates thinking from doing. It lets you validate the approach before time and tokens are spent on implementation.

---

## 5. Essential Claude Code Slash Commands

Slash commands are shortcuts that trigger specific behaviors in Claude Code. They're prefixed with `/`. Here are the essential ones you need from day one.

### `/help`

Lists all available commands or shows help for a specific command.

```
> /help
```

Shows a full list of commands with one-line descriptions.

```
> /help plan
```

Shows detailed help for the `/plan` command.

### `/clear`

Clears the current conversation history within the session. This is useful when:

- You've been working on the same task for a long time and the conversation has gotten long and messy
- You want to start fresh on a new topic but keep the session running
- You're approaching your token budget and want to reset without exiting

```
> /clear
```

**Important:** `/clear` resets the conversation history but keeps `CLAUDE.md` loaded. Claude still has your project context; it just forgets the conversation leading up to this point.

### `/doctor`

Diagnostic command that checks your setup:

```
> /doctor
```

Outputs a report on:
- Node.js version
- npm configuration
- API key validity
- Current directory and project structure
- Git configuration (if in a repo)
- Installed plugins
- Available MCPs

Run `/doctor` when something feels off. It often catches misconfigurations.

### `/exit`

Exits the session gracefully:

```
> /exit
```

**There's much more.** The full command set — `/plan`, `/compact`, `/cost`, `/plugin`, `/review`, `/memory`, and more — is covered in depth in **Chapter 2: Skills, Commands & Plugins**.

---

## 6. Hands-On Exercises

The best way to internalize Claude Code is to use it. Here are 5 exercises, each designed to teach a specific skill. They build on each other, so work through them in order. Every exercise starts from a completely empty git repo.

**Setup that applies to ALL exercises in this chapter:**

```bash
mkdir claude-training && cd claude-training && git init
```

All exercises happen inside this folder.

---

**Exercise 1-A: Your First CLAUDE.md**

**Goal:** Create a CLAUDE.md that gives Claude meaningful project context, then see how it affects Claude's behavior.

**Setup:** In your empty `claude-training` folder, create a minimal stub project first.

Create `src/app.ts`:
```typescript
// A simple task management API

function createTask(title: string, dueDate: string): void {
  // TODO: implement
}

function listTasks(): void {
  // TODO: implement
}
```

Now create `CLAUDE.md` in the root:
```markdown
# Task Manager API

## What This Is
A simple REST API for managing tasks. This is a training project.

## Key Commands
- Run tests: `echo "no tests yet"`
- Start server: `echo "not implemented yet"`

## Conventions
- All functions must have comments explaining what they do
- Never delete tasks — mark them as archived instead
- Use camelCase for variable and function names, PascalCase for types and classes

## What Claude Should Never Do
- Add dependencies without asking first
- Modify this CLAUDE.md file
- Use global variables
```

Start Claude Code in this folder:
```bash
claude
```

Then type:
```
What does this project do and what rules should I follow when working on it?
```

**Observe:** Claude reads CLAUDE.md and answers from it — project purpose, conventions, and guardrails all come from the file you wrote.

Now ask:
```
Implement the createTask function in src/app.ts
```

**Observe:** Claude follows the conventions in CLAUDE.md without being told again — it adds comments, uses camelCase.

**What to experiment with:**
- Add a rule like "Always add a timestamp to every new function as a comment" — does Claude follow it?
- Add a contradictory instruction mid-conversation and see which wins: CLAUDE.md or your message
- Remove the "Never Do" section entirely and notice if Claude's behavior changes on the next task

---

**Exercise 1-B: Vague vs. Precise Prompts**

**Goal:** Feel the difference between a vague prompt and a well-structured one on real code.

**Setup:** Create a file with an intentional bug. Create `src/calculator.ts`:
```typescript
function runningTotal(transactions: number[]): number {
  let total = 0;
  let i = 0;
  while (i <= transactions.length) {  // bug: should be <, not <=
    total += transactions[i];
    i++;
  }
  return total;
}

console.log(runningTotal([10, 20, 30]));  // should print 60
console.log(runningTotal([]));            // crashes with TypeError
```

In Claude Code, first type this vague prompt:
```
Fix my code
```

Note the response — Claude will likely ask which file, what bug, or make assumptions.

Now type this structured prompt:
```
<context>
The file src/calculator.ts has a function called runningTotal.
It crashes with a TypeError when called with an empty array.
It also has an off-by-one error that causes incorrect results on non-empty arrays.
</context>

<task>
Find both bugs and fix them. Return only the corrected function.
</task>

<constraints>
- Do not change the function signature
- Do not add any packages or dependencies
- Preserve the existing comments
</constraints>

<output_format>
Return the fixed function only, in a code block. No explanation needed.
</output_format>
```

**Observe:** The structured prompt produces a targeted, correct fix. The vague one wastes a turn on clarification.

**What to experiment with:**
- Remove `<output_format>` and see how verbose the response becomes
- Add `Think step by step before writing any code` at the top — does the fix quality improve?
- Try rewriting the vague prompt just by adding one sentence of context — how much does that alone help?

---

**Exercise 1-C: Plan Mode for a Real Task**

**Goal:** Use `/plan` before executing a non-trivial change so you can review, adjust, and approve.

**Setup:** Your `claude-training` project now has `src/app.ts` and `src/calculator.ts`. Let's add one more stub to make the task meaningful.

Create `src/task_store.ts`:
```typescript
// In-memory task storage (not persistent)
const tasks: unknown[] = [];

function findTask(id: string): void {
  // TODO
}

function deleteTask(id: string): void {
  // TODO — but remember: we archive, never delete
}
```

Now type:
```
/plan Add a health-check endpoint to the app that returns the current time and the number of tasks in the store
```

**Observe:** Claude produces a step-by-step plan listing every file it will create or modify, the logic it will add, and the order of operations — without touching anything yet.

Read through the plan. Then try one of these responses:

If the plan looks good:
```
Looks good, go ahead.
```

If you want to change something:
```
Change step 2 — don't modify app.ts directly. Create a new file src/health.ts and import it in app.ts instead.
```

**Observe:** Claude updates the plan before executing. You maintained control over the architecture.

**What to experiment with:**
- Run the same task *without* `/plan` and compare how it approaches it
- After approving a plan, interrupt mid-execution with "Stop — actually skip the tests step"
- Ask Claude: "What's the riskiest part of this plan and why?"

---

## 7. Chapter Summary and What's Next

Congratulations. You've covered the foundation of Claude Code.

### Key Takeaways

- **Chat is a bottleneck.** Claude Code solves context loss, file access, tool execution, and integration by shifting from chat to a persistent CLI environment.

- **CLAUDE.md is central.** Your project configuration lives in `CLAUDE.md`. Claude loads it automatically, maintaining context across sessions. Invest in writing a good one.

- **Prompting matters.** The better your prompts, the better Claude's output. Specificity, structure, and positive/negative examples are critical.

- **Slash commands are your first interface.** `/help`, `/clear`, `/doctor`, and `/exit` are the commands you need right away. The full command set is in Chapter 2.

- **Iterative workflows beat one-shots.** Complex tasks are better tackled in phases — plan, implement, review, refine. This is more efficient than one giant prompt.

### Moving Forward

You're ready for Chapter 2: **Skills, Commands & Plugins**. Chapter 2 covers:

- The full slash command reference (`/plan`, `/compact`, `/cost`, `/plugin`, `/review`, `/memory`, and more)
- Skill files — reusable, invocable instruction sets that shape Claude's behavior
- Installing and using official plugins
- Creating your own skills and plugins

Chapter 3 then builds on this foundation with Agents — autonomous systems that observe, reason, and act across multiple steps.

Each chapter builds on the previous, so make sure you've internalized the concepts here before moving on.

### Before Chapter 2

1. Practice with Claude Code on a real project (or create a sample one). Get comfortable with CLAUDE.md and structured prompting.

2. Keep your `CLAUDE.md` updated. As you learn more about Claude Code, refine your project documentation.

3. Set up your development environment so Claude Code can run tests, linting, and other checks. This is what closes the feedback loop.

Welcome to agentic development. The work you do now, establishing good practices and understanding the tools, will pay dividends as you tackle more complex tasks.
