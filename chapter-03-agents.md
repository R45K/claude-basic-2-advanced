# Chapter 3: Agents

Welcome to Chapter 3. You've now completed Chapters 1 and 2 and understand the basics of Claude Code and how to extend it with skills, commands, and plugins. In this chapter, we go deeper into one of the most powerful concepts in modern AI development: **agents**.

An agent isn't just a chatbot responding to individual questions. It's an autonomous system that perceives its environment, reasons about problems, takes actions to move toward a goal, and iterates based on the results. This chapter will teach you not only what agents are, but how to build them, how to make them reliable, and how to use specialized techniques to control their behavior.

By the end of this chapter, you will have built multiple working agents and understand how to architect complex multi-agent systems.

## 1. What is an Agent?

### Definition

An **agent** is an AI system that can:

1. **Perceive** its environment (read files, make API calls, get feedback from tools)
2. **Reason** about what to do next (plan steps, choose actions)
3. **Act** by invoking tools or taking steps toward a goal
4. **Iterate** by observing the results and repeating the cycle

This might sound abstract, so let's ground it in an example. Consider two scenarios:

**Scenario A (Not an agent):** You ask Claude, "Summarize this document." Claude reads the document once and gives you a summary. Done. Single request-response pair.

**Scenario B (An agent):** You ask an agent, "Find all bugs in this codebase, write tests for them, and fix them." The agent then:
- Reads files across the codebase
- Identifies patterns that might be bugs
- Writes test cases to verify the bugs
- Modifies the code to fix them
- Runs the tests to confirm the fix works
- Reports back what it did

In Scenario B, the agent is continuously cycling through perception, reasoning, and action.

### The Agent Loop: Observe → Think → Act → Observe

Every agent operates on this fundamental cycle:

```
┌─────────────────────────────┐
│  Observe Environment        │
│ (read files, check status)  │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│  Think                      │
│ (plan next action)          │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│  Act                        │
│ (call tools, make changes)  │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│  Observe Results            │
│ (get feedback from tools)   │
└──────────────┬──────────────┘
               │
               └───────────────┘
                  (repeat until done)
```

The agent starts with information (what's in the repo? what files exist?), reasons about what to do (which file should I check first? is this really a bug?), takes an action (read this file, run this test), and then observes the result. Based on that result, it decides what to do next.

This cycle can repeat dozens of times for a single task. Unlike a simple prompt-response interaction, the agent's behavior is shaped by everything it learns along the way.

### How Agents Differ from Assistants and Chatbots

Let's clarify three terms you'll hear often:

**Chatbot:** A system that responds to user messages based on training data. It has no tools and cannot take actions in the real world. It's purely conversational. Think of customer service bots: they can answer questions but can't actually modify your account.

**Assistant:** A system (usually Claude) that can use tools to take actions, but operates on a request-response basis. You ask for something, Claude does it, and returns a result. Each request is largely independent. Most use cases in Chapter 1 are assistants—you prompt, Claude responds with an action taken.

**Agent:** A system that autonomously works toward a goal over multiple steps, using its own judgment about what to do next. The human sets a high-level goal ("find and fix all bugs") and the agent breaks it down, makes decisions, and iterates without asking for permission at each step.

In practice, the line is fuzzy. But the key difference is *autonomy* and *iteration*. An agent decides what to do next; an assistant waits for the next instruction.

### When to Use an Agent vs. a Simple Prompt

Here are some decision criteria:

**Use a simple prompt (call Claude directly) when:**
- The task is well-defined and has a clear, single solution path
- You need a quick answer or analysis (summarize a file, review a snippet of code)
- The user interaction is conversational and back-and-forth
- You want to minimize latency and cost
- The task doesn't require iteration or learning from tool results

**Use an agent when:**
- The task is complex and has multiple solution paths
- The agent needs to make decisions based on what it discovers
- The goal is high-level, but the steps are unclear upfront
- The task might require retries, verification, or iteration
- The agent might spawn sub-tasks or work on multiple things in parallel
- You need the system to work autonomously without human guidance at each step

### Real-World Examples

**Not an agent:**
```
User: "Read this test file and tell me what it's testing."
Agent: Reads the file and responds with an explanation.
```

Why? Single read, single response. No iteration needed.

**Is an agent:**
```
User: "Find all failing tests in this repo, identify the root cause of each failure,
       propose a fix, implement it, and verify the fix works."
Agent: (1) Finds test files, (2) Runs them, (3) Reads code, (4) Analyzes failures,
       (5) Modifies code, (6) Re-runs tests, etc.
```

Why? Multiple steps, decisions made along the way, iteration based on results.

## 2. The Claude Agent Architecture

An agent's behavior is built on a few key components. Understanding these will help you design agents that are reliable and predictable.

### Tools: What They Are and How Claude Uses Them

A **tool** is any function that an agent can call to interact with the world. Tools give Claude concrete ways to act. Common tools include:

- **File I/O:** read, write, and delete files
- **Execution:** run shell commands
- **Search:** find files by name or search code for patterns
- **Web requests:** fetch URLs, make API calls
- **Interaction:** ask the user a question

When you build an agent in Claude Code, you define which tools are available. Claude then learns which tool is appropriate for each situation.

For example, if you define tools like `Read`, `Glob`, `Grep`, and `Bash`, Claude might reason: "The user wants to find all uses of a deprecated function. I should use Glob to find relevant files first, then Grep to search for the pattern." It then calls these tools, gets results, and decides what to do next.

### The Tool-Use Loop: Thinking, Choosing, and Acting

Here's what happens when you run an agent:

1. **Human sends a goal:** "Fix all TypeScript errors in this project."

2. **Claude thinks:** Based on the tools available and the goal, Claude plans a strategy. It might think, "I need to find TypeScript files, check them for errors, understand the errors, then fix them."

3. **Claude chooses a tool:** Claude decides which tool to call first. It might choose `bash` to run the TypeScript compiler.

4. **Tool executes, returns result:** The compiler runs and returns a list of errors.

5. **Claude observes and reasons:** Claude reads the error output and decides the next action. Maybe it needs to read a specific file to understand the context of an error.

6. **Repeat:** Steps 2-5 repeat until Claude decides the task is complete.

This is the **tool-use loop**, and it's the heart of agentic behavior. Each iteration, Claude has new information and can make a better decision about what to do next.

### System Prompts: Shaping Agent Behavior

A **system prompt** is the underlying instruction set that tells Claude how to behave. For agents, a good system prompt:

- Defines the agent's expertise and perspective (e.g., "You are a senior backend engineer")
- Specifies constraints and goals (e.g., "Always prioritize security")
- Guides decision-making (e.g., "When in doubt, ask the user")
- Explains which tools to use and when

System prompts are powerful. A generic system prompt might produce a generic agent; a carefully crafted system prompt produces an agent that behaves like a specialist.

### The Role of Context in Agent Decision-Making

Every tool result becomes **context** for the next decision. If a file read returns 500 lines of code, Claude can use all of that to decide what to do next. If a search returns 10 file paths, Claude can prioritize which ones to read based on the filenames and what it's trying to accomplish.

The quality of your agent depends partly on the quality of the context it receives. Tools should return clear, well-structured information so Claude can make good decisions. For example:

**Poor:** A bash tool that returns a wall of unstructured output
**Better:** A tool that parses the output and returns structured results (e.g., a JSON list of errors)

### Max Turns and Stopping Conditions

An agent loop must have a stopping condition. Otherwise, it could loop forever. In Claude Code, agents have built-in stopping mechanisms:

- The agent will stop after it decides the task is complete
- You can set a context length limit to prevent excessive iterations
- You can use the `/plan` command to require approval before major actions

Most agents should have reasonable safeguards to prevent runaway loops and excessive API costs.

> Tip: When building agents, start small and test thoroughly. Use `/plan` mode to review the agent's proposed steps before execution.

## 3. Plan Mode — Thinking Before Acting

Chapter 2 covered `/plan` in the command reference — the mechanics, when to use it, and when to skip it. The short version: always run `/plan` before any non-trivial task.

It's worth restating here because plan mode becomes *more* critical as agents take on more work autonomously. When you're guiding Claude step-by-step, a wrong assumption surfaces quickly. When an agent is operating across multiple files and tool calls, that same assumption can silently cascade into dozens of wrong moves before you notice.

The habit to build: before handing a complex task to an agent you've defined, run `/plan` first. Let Claude surface its assumptions. Challenge them. Then proceed.

```
/plan

Add a new `user_roles` table to support role-based access control.
Migrate existing user data to use roles. Update the authentication
middleware to check roles. Write tests.
```

This upfront conversation — "What about existing admins?" "Should we use an enum instead of a separate table?" — saves hours of rework that would otherwise happen inside an agent loop where you can't intervene as easily.

---

### Exercise 3-D: The Plan → Execute Pattern

**Goal:** Use plan mode with an agent to safely refactor code you can review before it changes anything.

**Setup:** Create a file with messy, working code that's a good refactoring target. Create `src/orderProcessor.ts`:
```typescript
function p(o: any[]): number {
  let t = 0;
  for (const i of o) {
    if (i["qty"] > 0) {
      if (i["price"] > 0) {
        t = t + (i["qty"] * i["price"]);
        if (i["discount"]) {
          if (i["discount"] > 0) {
            t = t - (t * i["discount"] / 100);
          }
        }
      }
    }
  }
  return t;
}
```

Create `.claude/agents/refactor-agent.md`:
```markdown
---
name: refactor-agent
description: Refactors code for clarity and maintainability without changing its behaviour. Always explains what it will change before touching anything.
tools: Read, Write
model: sonnet
---

You are a careful refactoring engineer. Your rule: never change behaviour, only structure.

Before making any changes:
1. Read the file
2. Explain what the current code does in plain English
3. List every specific change you plan to make (rename, extract, simplify)
4. Write: "Shall I proceed with these changes?" and stop

Only after receiving confirmation:
5. Apply all changes
6. Briefly confirm what was done
```

In Claude Code:
```
/plan Use the refactor-agent to improve src/orderProcessor.ts
```

Read the plan. Review it. Then:
```
The plan looks good — proceed.
```

**Observe:** The agent explains the current code, lists its planned changes (renaming `p` to `calculateOrderTotal`, extracting the discount logic, flattening the nested conditionals), and waits for your go-ahead before touching anything.

**What to experiment with:**
- After seeing the plan, say "Don't rename the function — only fix the nesting" and see if it respects the constraint
- Remove the "stop and wait" instruction from the agent and re-run — notice how it changes from safe to aggressive
- Give it a file you actually want refactored from your own codebase

---

## 4. Building Agents in Claude Code

Now it's time to build real agents. In Claude Code, agents are defined as markdown files in the `.claude/agents/` directory. Each agent is a reusable component with its own system prompt, available tools, and model selection.

### What Makes a Claude Code Agent

A Claude Code agent is a simple markdown file that specifies:

- **name:** A unique identifier for the agent
- **description:** What this agent does and when to invoke it
- **tools:** Which Claude Code tools it has access to (Read, Glob, Grep, Bash, Write, etc.)
- **model:** The Claude model to use (haiku for fast tasks, sonnet for complex reasoning)
- **System prompt:** The detailed instructions that shape the agent's behavior

The agent is invoked by typing a prompt like "Use the hello-agent to summarize this project." Claude Code then runs the agent in an isolated context with the specified tools and instructions.

### Agent File Format

An agent markdown file looks like this:

```markdown
---
name: agent-name
description: What this agent does and when to invoke it.
tools: Read, Glob, Grep, Bash
model: haiku
---

[Your detailed system prompt here]
```

The YAML front matter (between the `---` markers) defines the agent's metadata. The rest is the system prompt—the detailed instructions Claude uses when running this agent.

---

### Exercise 3-A: Hello, Subagent

**Goal:** Create your first `.claude/agents/` file, invoke it, and understand how subagents work.

**Setup:** You only need the `claude-training` folder from Chapter 1 (or any folder with a few files in it). No extra setup needed.

Create `.claude/agents/hello-agent.md`:
```markdown
---
name: hello-agent
description: A simple test agent that summarizes what it finds. Use when asked to explore a directory.
tools: Read, Glob
model: haiku
---

You are a file explorer. When invoked:
1. List all files in the current project (use Glob with pattern "**/*")
2. Write a short paragraph (3-5 sentences) describing what the project appears to be about
3. Mention the primary language and any frameworks or libraries you can identify

Be concise. Do not read file contents unless the filenames are ambiguous.
```

In Claude Code:
```
Use the hello-agent to summarize this project.
```

**Observe:** Claude delegates to the subagent which runs in its own context window. The agent sees the file tree, infers the project purpose, and returns a summary. Notice that it uses only Glob — it doesn't read file contents — because that's all it needs.

**What to experiment with:**
- Change `model: haiku` to `model: sonnet` and re-run — does the description improve?
- Add `Grep` to the tools list and update the prompt: "Also search for any TODO comments" — see what it finds
- Change the description to something unrelated (e.g., "Use when asked about databases") and try to invoke it again — notice how the description affects when Claude chooses to use it

---

## 5. Subagents — Delegating Work

Complex tasks often benefit from delegation. In Claude Code, you can spawn subagents (using the agents you've created) to handle specific subtasks, then observe how they work in parallel or sequence.

### What a Subagent Is

A **subagent** is an agent spawned to handle a specific, well-defined task. A parent workflow can invoke multiple subagents, wait for them to complete, and then integrate their results.

Think of it like project management: a project manager (parent workflow) might ask one agent to "refactor the authentication system" and another to "write tests for it", then review and combine their work.

### Why Delegate? The Benefits

**Parallel Work:** If you have multiple independent tasks, invoke different agents. They work in sequence (one after another) in your workflow, but you can reason about them separately.

**Specialization:** A security-focused agent can audit code while a performance-focused agent optimizes it. Each has the right perspective and tools.

**Context Isolation:** A subagent has focused context and instructions, which can reduce confusion and lead to more reliable output.

**Scale:** For very complex projects, break it into smaller agent-sized chunks that are easier to manage.

### Using Multiple Agents in Claude Code

In Claude Code, you use multiple agents by explicitly invoking them with commands like:

```
Use the security-reviewer to audit the auth module.

Use the refactor-agent to clean up the database layer.

Use the test-writer to add integration tests.
```

Each command invokes the named agent with your prompt. The agent runs independently and reports back.

### Practical Orchestration Example

Here's a realistic workflow that uses multiple agents:

```
First, use the hello-agent to summarize the current project structure.

Based on that summary, use the security-reviewer to audit the main
authentication and API routes.

Then use the refactor-agent to improve code quality in any files flagged
as problematic.

Finally, use the test-writer to add tests for the refactored code.

Once all agents are done, synthesize their findings into a prioritized
list of improvements.
```

Each agent runs in turn, and you can review the results. This orchestration approach is simpler and more transparent than trying to build one agent that does everything.

---

### Exercise 3-F: Orchestrate Three Agents

**Goal:** Chain hello-agent, security-reviewer, and refactor-agent together in sequence, passing results from one to the next via file-based state.

**Setup:** You need the `hello-agent.md`, `security-reviewer.md`, and `refactor-agent.md` agent files from the earlier exercises. Ensure `src/userAuth.ts` from Exercise 3-C also exists.

In Claude Code:
```
Use the hello-agent to find all TypeScript files in this project and write
the list to .claude/ts-files.md.
```

Then:
```
Read .claude/ts-files.md. Use the security-reviewer to audit any files
with "auth" in the name, and write a findings report to
.claude/security-findings.md.
```

Then:
```
Read .claude/security-findings.md. Use the refactor-agent to clean up
any file the security reviewer flagged as NEEDS WORK or FAIL.
```

**Observe:** Each agent reads the output file from the previous step — this is file-based state passing. The hello-agent discovers the project layout; the security-reviewer narrows its focus to auth files; the refactor-agent acts only on what was flagged. No single agent needs to understand the whole pipeline.

**What to experiment with:**
- Run the sequence again after the refactor-agent finishes — does the security-reviewer now rate the file better?
- Add a fourth step: "Use the memory-agent to document what all three agents did" — observe how it synthesizes across the output files
- Deliberately introduce a new vulnerability into `userAuth.ts` mid-sequence and see whether the pipeline catches it on the next run

---

## 6. Agent Teams — Peer Collaboration at Scale

Subagents are hierarchical: a parent delegates a task, a worker completes it and reports back. **Agent teams** flip this model. Instead of one agent orchestrating subordinates, you get a group of fully independent Claude Code sessions working as peers — each with its own context window, communicating directly with each other through a shared task list and mailbox.

> Agent teams are experimental. Enable them by adding `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` to your environment (via `~/.claude/settings.json` or shell config) and ensure you're on Claude Code v2.1.32 or later.

### The Architecture

| Role | What it does |
|---|---|
| **Team lead** | Your main Claude Code session. Creates the team, spawns teammates, coordinates. |
| **Teammates** | Independent Claude Code instances, each with their own context. |
| **Task list** | Shared, mutable queue of work items. Teammates claim tasks and mark them complete. |
| **Mailbox** | Async messaging between teammates and the lead. |

### Creating a Team

Describe the goal and ask Claude to form a team:

```
Create an agent team to investigate this performance regression from three angles:
one teammate on database query analysis, one on memory profiling, and one playing
devil's advocate challenging the findings of the other two.
```

Claude creates the team, distributes the work, and teammates begin in parallel. You can also specify the composition explicitly:

```
Create a team with 3 teammates using Sonnet. Assign one to the frontend,
one to the backend, and one to write integration tests.
```

### Interacting With the Team

In **in-process mode** (single terminal), cycle through teammates with `Shift+Down`. In **tmux/split-pane mode**, each teammate gets its own pane — click in to interact directly.

You can message teammates at any time without going through the lead:

```
Tell the database teammate to also check for missing indexes on the orders table.
```

Teammates communicate with each other and with the lead through the mailbox. The shared task list keeps everyone aligned — tasks are self-claimed as teammates finish their current work.

### Subagents vs. Agent Teams

| | Subagents | Agent Teams |
|---|---|---|
| **Communication** | Report to parent only | Peer-to-peer + shared task list |
| **Context** | Focused, subordinate | Independent, full Claude instances |
| **Coordination** | Parent manages all | Self-coordinating with task queue |
| **Cost** | Lower (one main session) | Higher (N full Claude instances) |
| **Best for** | Focused, well-defined delegation | Parallel exploration, debate, cross-layer work |

### When to Use Agent Teams

**Strong fit:**
- **Parallel investigation** — multiple teammates explore competing hypotheses simultaneously (faster convergence, avoids anchoring bias)
- **Cross-layer features** — frontend, backend, and tests owned by separate teammates without file conflicts
- **Review with multiple lenses** — security + performance + test coverage reviewers running at the same time

**Avoid teams when:**
- Tasks must happen in a specific sequence
- Multiple teammates would edit the same files
- You're on a tight token budget — each teammate is a full Claude Code instance

### Quality Gates via Hooks

You can use hooks to enforce standards on the team:

```json
{
  "hooks": {
    "TaskCompleted": "npm test -- --testPathPattern={{task.files}}",
    "TeammateIdle": "echo 'Check task list before going idle'"
  }
}
```

`TaskCompleted` runs when a teammate marks a task done — exit code 2 blocks completion until it passes. `TeammateIdle` fires when a teammate has no more work, letting you inject additional instructions.

---

### Exercise 3-G: Your First Agent Team

**Goal:** Enable agent teams and observe peer messaging, the shared task list, and parallel investigation in action.

**Setup:** Add `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` to your `~/.claude/settings.json` (or export it in your shell config). Confirm you are on Claude Code v2.1.32 or later with `claude --version`. You will use the TypeScript files from the earlier exercises as the codebase under investigation.

In Claude Code:
```
Create a team of 2 teammates to investigate a performance regression in
this project. Teammate 1 should profile the TypeScript code from the
earlier exercises — look for algorithmic complexity issues, blocking
operations, and hot paths. Teammate 2 plays devil's advocate: look for
performance causes that Teammate 1 might miss or dismiss, including
environmental factors, dependency overhead, and test-harness overhead.
Each teammate should write their findings to a separate file under
.claude/ before reporting back to you.
```

Watch the shared task list populate. Use `Shift+Down` (in-process mode) to switch between teammates as they work.

**Observe:** The two teammates operate independently and can send each other messages through the mailbox when one discovers something the other should know. The task list shows which items are claimed, in-progress, and complete. After both finish, compare the two findings files — they will often surface complementary issues.

**What to experiment with:**
- Message Teammate 2 directly: "Also check whether the test setup in stringUtils.test.ts could be inflating runtime" — see how it picks up the new angle mid-flight
- After both teammates finish, ask the lead: "Synthesize both findings files into a ranked list of the top 3 performance risks"
- Note the total token usage compared to running a single agent with the same prompt — this is the real cost of peer collaboration

---

## 7. Specialized Agent Personas

Agents don't have to be generic. You can create specialized agents with specific expertise and perspectives. This is done through the system prompt.

### What Personas Do

A **persona** is a detailed description of expertise, perspective, and values that shapes an agent's behavior. The same goal can produce very different results depending on the persona.

Example: "Find issues in this code"
- **Generic agent:** Might flag any unusual pattern
- **Security auditor:** Focuses on authentication, injection, data exposure
- **Performance specialist:** Focuses on loops, database queries, memory usage
- **Backend architect:** Focuses on system design, scalability, maintainability

### How to Write a Good System Prompt Persona

A good persona has:

1. **Role and title:** "You are a senior Ruby engineer..."
2. **Experience level:** "...with 10 years of experience..."
3. **Domain expertise:** "...specializing in Rails APIs..."
4. **Values and priorities:** "...who prioritizes clean, tested, maintainable code..."
5. **Communication style:** "...and explains your reasoning clearly"

Here are some examples:

**Senior Backend Engineer:**
```
You are a senior backend engineer with 10 years of experience building
production systems. You prioritize:
1. Code that is clean, tested, and maintainable
2. Systems that scale and are reliable
3. Security and data protection
4. Performance optimization

When reviewing code, look for:
- Proper error handling and validation
- Adequate test coverage
- Scalability concerns (database queries, caching, etc.)
- Security vulnerabilities (injection, auth issues, etc.)
- Code style and maintainability

Explain your reasoning clearly and provide specific, actionable feedback.
```

**Security Auditor:**
```
You are a security auditor with 8 years of experience in application security.
Your job is to find vulnerabilities and security weaknesses. You focus on:
1. Authentication and authorization flaws
2. Data exposure and privacy issues
3. Input validation and injection attacks
4. Cryptography and key management
5. Third-party dependency vulnerabilities

When reviewing code, ask:
- What would a bad actor try to do here?
- Is there insufficient validation?
- Could data be leaked?
- Are secrets properly protected?

Report findings with severity levels and remediation guidance.
```

**QA and Test Engineer:**
```
You are a QA engineer with 7 years of experience. You are creative in finding
edge cases and breaking software. When given code or a feature, you:
1. Think of unusual inputs and edge cases
2. Consider state transitions and race conditions
3. Test error paths and failure modes
4. Consider performance under load
5. Verify that requirements are actually met

Write thorough test cases that cover:
- Happy paths
- Error conditions
- Edge cases
- Boundary values
- Integration points

Make your tests clear and well-documented.
```

### Creating Specialized Agents

To create a specialized agent, define a new markdown file in `.claude/agents/` with a focused system prompt. For example:

```markdown
---
name: performance-specialist
description: Profiles code for performance issues and suggests optimizations. Use when you suspect code is slow or inefficient.
tools: Read, Bash, Grep
model: sonnet
---

You are a performance engineer with 8 years of experience optimizing
production systems. Your focus is finding where code is slow and making
it fast.

When analyzing code:
1. Look for N+1 queries, inefficient loops, unnecessary allocations
2. Consider algorithmic complexity (is this O(n²) when it could be O(n)?)
3. Check for blocking I/O operations that could be parallelized
4. Identify hot paths that are executed frequently
5. Suggest concrete optimizations with expected impact

Always provide benchmarks or metrics when possible.
```

### When to Use a Persona vs. Generic Agent

**Use a persona when:**
- You want specialized expertise for a specific task
- You need a particular perspective (security vs. performance)
- You're reviewing or auditing code
- You want consistent, expert-level thinking

**Generic is fine when:**
- The task is simple and straightforward
- You're just gathering information
- The perspective doesn't matter
- You want balanced, generalist thinking

---

### Exercise 3-C: The Security Reviewer Persona

**Goal:** Create a specialized expert agent and give it vulnerable code to find real issues.

**Setup:** You need code with real, findable security problems. Create `src/userAuth.ts`:
```typescript
// User authentication module
const DB_PASSWORD = "admin123";  // hardcoded secret

function login(username: string, password: string) {
  // Direct string interpolation — SQL injection vulnerability
  const query = `SELECT * FROM users WHERE username='${username}' AND password='${password}'`;
  const result = DB.execute(query);
  return result[0];
}

function getUserData(userId: string) {
  // No authorization check — any user can access any user's data
  const query = `SELECT * FROM users WHERE id=${userId}`;
  return DB.execute(query)[0];
}

function updateEmail(userId: string, newEmail: string) {
  // No input validation on email format
  // No check that the current user owns this account
  DB.execute(`UPDATE users SET email='${newEmail}' WHERE id=${userId}`);
}
```

Create `.claude/agents/security-reviewer.md`:
```markdown
---
name: security-reviewer
description: Reviews code for security vulnerabilities. Use when asked to audit a file or feature for security issues.
tools: Read, Glob, Grep
model: sonnet
---

You are a senior application security engineer. You find vulnerabilities before attackers do.

When reviewing code:
1. Look specifically for: SQL injection, hardcoded credentials, missing authentication/authorization, lack of input validation, sensitive data exposure
2. For each finding state:
   - **What:** name of the vulnerability
   - **Where:** exact file and line number
   - **Risk:** why this is dangerous in plain language
   - **Fix:** the specific change needed
3. Rate each finding: CRITICAL / HIGH / MEDIUM / LOW
4. End with a one-line verdict: PASS / NEEDS WORK / FAIL

Be direct. Do not soften findings. A developer's career depends on you being honest.
```

In Claude Code:
```
Use the security-reviewer to audit src/userAuth.ts
```

**Observe:** The agent should find all three issues — SQL injection (×2), hardcoded credential, and missing authorization — with correct severity ratings. Count how many it catches.

**What to experiment with:**
- Change `model: haiku` and re-run — does it miss any of the three issues?
- Add a fourth vulnerability to `user_auth.rb` (e.g., logging the password: `puts "Login attempt: #{password}"`) — does the agent catch it?
- Ask the agent to "suggest a fully secure rewrite of the login function"

---

## 8. Common Agent Patterns

Over time, patterns have emerged for building reliable agents. Understanding these patterns helps you design better agents for your specific tasks.

### ReAct: Reason + Act

**ReAct** stands for "Reasoning + Acting." The idea is to force the agent to explicitly state its reasoning before each action.

Without ReAct:
```
Agent: [decides to read file1.py]
Agent: [reads file1.py]
Agent: [decides to read file2.py]
```

With ReAct:
```
Agent: I need to understand the codebase structure. Let me start by reading the main configuration file.
Agent: [reads config.py]
Agent: I can see the project uses PostgreSQL and requires authentication. Now let me read the auth module.
Agent: [reads auth.py]
Agent: I found that auth.py uses basic token validation, but doesn't implement token expiration. This is a security gap.
Agent: Let me check if there are any tests for auth.
Agent: [reads test_auth.py]
```

The ReAct pattern makes the agent's thinking transparent and often leads to more careful decision-making.

**How to implement in a system prompt:**
```
Before taking any action, state your reasoning:
1. What do I know so far?
2. What's my goal?
3. What should I do next and why?
4. What tool should I call?

Then execute the tool and observe the result.
```

### Chain of Thought: Breaking Reasoning Into Explicit Steps

Similar to ReAct, **Chain of Thought (CoT)** breaks down reasoning into numbered steps.

Example system prompt enhancement:
```
When solving problems, use the following approach:

1. Understand the problem - what are we trying to achieve?
2. Break it down - what are the sub-tasks?
3. Consider constraints - what are the limitations?
4. Plan the approach - in what order should we do things?
5. Execute - do each sub-task carefully
6. Verify - did we achieve the goal?

Use this structure in every response.
```

This is especially useful for complex problems where the steps aren't obvious.

### Reflection: Agent Reviews Its Own Output

**Reflection** means the agent checks its own work before finishing.

Pattern:
```
1. Attempt the task
2. Review the output: "Is this correct? Complete? Well-tested?"
3. If not satisfied, refine and try again
4. Repeat until satisfied
```

Example system prompt addition:
```
After completing your work:
1. Review what you've done
2. Ask yourself: Is it correct? Is it complete? Does it follow best practices?
3. Are there edge cases I missed?
4. If you're not satisfied, refine your solution before reporting completion
```

This requires the agent to demonstrate its own verification, not just declare the task done.

### Verification Loop: Testing the Agent's Own Work

**Verification** means the agent checks its work by running tests or validating output.

Pattern:
```
1. Implement a solution
2. Write tests for the solution (or run existing tests)
3. If tests pass, done
4. If tests fail, debug and refine
5. Repeat until tests pass
```

This is powerful because the agent gets immediate feedback on whether its work actually works. You see this in Exercise 3-E (test-writer), where the agent runs tests after writing code.

---

### Exercise 3-E: The Self-Verifying Agent

**Goal:** Build an agent that writes tests, runs them, reads the failures, and fixes them — without you in the loop.

**Setup:** Create a file with a stubbed function that needs an implementation. Create `src/stringUtils.ts`:
```typescript
// Returns the most frequently occurring word in a string.
// If there's a tie, returns the word that appears first.
// Ignores punctuation and is case-insensitive.
// Returns null for empty or null input.
export function mostFrequentWord(text: string | null): string | null {
  // TODO: implement
  return null;
}
```

Create `src/stringUtils.test.ts`:
```typescript
import { mostFrequentWord } from './stringUtils';

test('returns most frequent word', () => {
  expect(mostFrequentWord("the cat sat on the mat")).toBe("the");
});

test('is case-insensitive', () => {
  expect(mostFrequentWord("Hello hello HELLO world")).toBe("hello");
});

test('returns null for empty string', () => {
  expect(mostFrequentWord("")).toBeNull();
});

test('returns null for null input', () => {
  expect(mostFrequentWord(null)).toBeNull();
});
```

Create `.claude/agents/test-writer.md`:
```markdown
---
name: test-writer
description: Implements a stubbed function so that all existing tests pass. Runs the tests and fixes failures before finishing. Use when you have a stub function and existing tests.
tools: Read, Write, Bash
model: sonnet
---

Your workflow is always:

1. Read the stub function and understand its contract from the comments and tests
2. Write an implementation in the stub file
3. Run the tests with Bash: `npx jest src/stringUtils.test.ts`
4. If any tests fail: read the failure message carefully, fix the implementation, run again
5. Repeat step 4 up to 4 times
6. Report: what you implemented, which tests pass, and any that still fail

Do not report success until all 4 tests actually pass. Do not modify the test file.
```

In Claude Code:
```
Use the test-writer to implement the mostFrequentWord function.
```

**Observe:** The agent reads both files, writes an implementation, runs the tests, reads the failure output if any, fixes the code, and loops until green — all autonomously.

**What to experiment with:**
- Deliberately make one test incorrect (e.g., `expect(mostFrequentWord("the cat sat on the mat")).toBe("cat")`) — watch the agent try and eventually give up after 4 retries
- Add `model: haiku` to the agent — does it still implement correctly on the first try?
- Add a 5th harder test case to the test file: `expect(mostFrequentWord("one, two, one! Two. One?")).toBe("one")` (requires punctuation stripping)

---

## 9. Memory — How Agents Remember Things

A single agent session might last minutes. But what about an agent that needs to work across multiple sessions? How does it remember what happened yesterday? Or what the project constraints are?

Agents have four distinct memory mechanisms, each with different tradeoffs.

### In-Context Memory (Ephemeral)

**What it is:** Information in the current conversation. Everything Claude knows about the task comes from what you've said and what it discovered during the current invocation.

**How it works:** Each message and tool result is part of the current context. Claude reads the entire history when deciding what to do next.

**Pros:**
- Immediate—no setup or delays
- Perfect recall within a single session
- No external dependencies

**Cons:**
- Lost when the session ends
- Limited by the context window (though Claude models have large windows)
- Can become slow if the conversation gets very long

**When to use:** Short tasks, single-session work, exploratory analysis.

### External Memory (Files)

**What it is:** Persistent information stored in files that an agent reads at the start of a session and updates throughout.

**How it works:** At the start of a session, read a state file (like `.claude/project-memory.md`) containing important context. As the agent works, it updates this file. Future sessions read the updated file.

**Pros:**
- Persistent across sessions
- Shareable with other agents or humans (e.g., via version control)
- Scalable—can store large amounts of structured data
- Can be version-controlled to track history

**Cons:**
- Requires file I/O (slower than in-memory)
- Need to manually structure and parse the data
- Requires careful synchronization if multiple agents access the same file

**When to use:** Multi-session work, structured state, team projects.

**Example usage in a system prompt:**
```
Maintain a memory file at .claude/project-memory.md:

1. At the start of your work, read this file to see what's been done
2. Complete your task
3. Update the memory file with a brief summary of what you found or did

Memory entry format:
## [Date]
- [What you did]
- [Key findings]

Keep entries concise and useful for future sessions.
```

### CLAUDE.md as Project Memory

**What it is:** A file in your project root that contains project-level instructions and context that all agents (and humans) should follow.

**How it works:** Claude Code automatically reads `CLAUDE.md` in your project. This file contains "never do X" rules, project conventions, architectural notes, etc.

**Pros:**
- Automatically discovered—no setup needed
- Applies to all agents in the project
- Easily version-controlled
- Can include links to documentation

**Cons:**
- Limited scope (project-level only, not session-level)
- Changes require manual updates

**When to use:** Team-wide conventions, architectural constraints, security policies, "never do X" rules.

**Example `CLAUDE.md`:**
```markdown
# Project Guidelines

## Architecture
- All API routes must use Express middleware for auth
- Database access only through the ORM (Sequelize)
- No raw SQL queries

## Code Quality
- All new functions must have JSDoc comments
- Test coverage must be >80%
- Never commit without running the linter

## Security
- Never log passwords or API keys
- All user input must be validated
- SQL injection prevention: always use parameterized queries

## Team
- Backend stack: Node.js + Express + Sequelize
- Database: PostgreSQL
- For questions about architecture, ask @alice-lead-engineer
```

When agents read this, they understand the project's constraints and conventions without needing to ask.

### External Databases and Vector Stores

**What they are:** Semantic databases (like pgvector, Pinecone, or Weaviate) that allow you to search large amounts of information by meaning, not just keywords.

**How they work:** You embed documents (code, documentation, etc.) into semantic vectors. An agent can then query the database with natural language and get the most relevant documents back.

**Pros:**
- Handles large knowledge bases (thousands of documents)
- Semantic search—find relevant info even without exact keywords
- Scalable and fast
- Can be shared across teams

**Cons:**
- Requires setup and infrastructure
- Additional cost for embeddings and storage
- More complex to implement

**When to use:** Large codebases, documentation search, knowledge retrieval at scale, multi-agent systems sharing common knowledge.

**Tools available:**
- **pgvector:** PostgreSQL extension for semantic search (free, self-hosted)
- **Pinecone:** Managed vector database service (easy to use, paid)
- **Weaviate:** Open-source vector search engine (flexible, self-hosted)
- **Milvus:** Another vector database option (self-hosted, highly scalable)

---

### Exercise 3-B: An Agent With External Memory

**Goal:** Build an agent that persists what it learns across sessions using a file.

**Setup:** No extra files needed — the agent creates its own memory file.

Create `.claude/agents/memory-agent.md`:
```markdown
---
name: memory-agent
description: Explores and documents the project, remembering findings across sessions. Use when you want to build up a knowledge base about this codebase.
tools: Read, Write, Glob
model: sonnet
---

You maintain a persistent memory file at `.claude/project-memory.md`.

Every time you are invoked:
1. Read `.claude/project-memory.md` if it exists — this is your memory from previous sessions
2. Complete the task you've been asked to do
3. Append what you learned to `.claude/project-memory.md` using this format:

---
## [Today's date] — [task summary in 5 words]
**Findings:** [bullet points of what you discovered]
**Still unknown:** [things you noticed but didn't investigate]
---

If the memory file doesn't exist yet, create it with a header:
# Project Memory
*Started: [date]*
```

In Claude Code:
```
Use the memory-agent to document what's in the src/ directory.
```

After it runs, open `.claude/project-memory.md` and read what it wrote.

Start a fresh session (`/clear`) and run:
```
Use the memory-agent to document what's in the test/ directory.
```

**Observe:** The second invocation reads the first session's notes and builds on them, even though the conversation was cleared.

**What to experiment with:**
- Ask: "Use the memory-agent — what do you already know about this project?"
- Add a task: after 5 sessions, ask the agent to "summarise and condense the memory file"
- Try `/clear` between every invocation — the agent should still accumulate knowledge via the file

---

## 10. Advanced Patterns for Production Agents

As you build more complex agents, certain patterns emerge that make them more reliable, transparent, and efficient.

### The Plan-Review-Execute Pattern

This is plan mode in practice. Before an agent makes changes:

1. **Plan:** The agent proposes what it will do
2. **Review:** You read the plan and provide feedback
3. **Execute:** The agent proceeds with the approved plan

This prevents wasted work and catches misunderstandings early.

### The Verify-and-Report Pattern

After completing work, the agent:

1. **Verifies:** Runs tests or validation checks
2. **Reports:** Documents what it did, what worked, and what didn't

This gives you confidence in the agent's output. You see both the work and the verification, not just the agent's claim that it's done.

### The Delegation Pattern

Break complex tasks into smaller pieces and assign them to specialized agents:

1. **Orchestrator agent** understands the overall task
2. **Delegator** breaks it into subtasks
3. **Specialist agents** handle each subtask
4. **Synthesizer** combines the results

This is what you practiced in Exercise 3-D.

---

## 🔗 Ecosystem Connection

The agent patterns you built in this chapter are already implemented at scale in these repos — battle-tested over months of real production use:

| Repo | What it adds to this chapter |
|---|---|
| [affaan-m/everything-claude-code](https://github.com/affaan-m/everything-claude-code) | 30 pre-built specialized agents for specific languages and domains (TypeScript, Python, Go, Ruby, security, database, E2E) — ready to drop into `.claude/agents/` |
| [SuperClaude-Org/SuperClaude_Framework](https://github.com/SuperClaude-Org/SuperClaude_Framework) | 20 coordinated specialized agents (PM Agent, Security Engineer, Backend Specialist, Frontend Specialist, Deep Research Agent) that auto-coordinate based on context |
| [anthropics/claude-plugins-official](https://github.com/anthropics/claude-plugins-official/tree/main/plugins) | The `agent-sdk-dev` plugin scaffolds full Claude Agent SDK projects; `skill-creator` helps you build and benchmark your own skills |

**Quick start:** Clone `affaan-m/everything-claude-code`, browse `agents/`, and copy the reviewer agent for your primary language into your project's `.claude/agents/`.

> See **[Chapter 7 → Section 3](chapter-07-ecosystem.md)** for everything-claude-code hands-on exercises and **[Chapter 7 → Section 4](chapter-07-ecosystem.md)** for SuperClaude agents.

## 11. Chapter Summary and What's Next

You've now learned what agents are, how they work, and how to build them. Let's review the key concepts:

### Key Takeaways

1. **Agents are autonomous systems** that perceive, reason, act, and iterate toward goals. They're more than chatbots or simple assistants.

2. **The agent loop** (Observe → Think → Act) is the fundamental pattern. Claude decides what to do next based on what it discovers.

3. **Tools are the bridge** between Claude's reasoning and real-world action. Define tools clearly with good descriptions.

4. **Claude Code agents** are defined as markdown files with metadata and system prompts. They're composable and reusable.

5. **Memory takes multiple forms:**
   - In-context (ephemeral, immediate)
   - File-based (persistent, shareable)
   - Project-level (`CLAUDE.md`)
   - Semantic databases (for scale)

6. **Plan mode** is essential for complex tasks. Always use it before major changes — especially when handing off to an agent that will operate autonomously.

7. **Agent teams** enable peer collaboration between independent Claude sessions with a shared task list and peer-to-peer mailbox. Use them for parallel investigation and cross-layer work where teammates can explore competing angles simultaneously; avoid them for sequential tasks or tight token budgets, since each teammate is a full Claude Code instance.

8. **Specialization through personas** produces better agents. A security-focused agent finds different issues than a generic one.

9. **Patterns like ReAct, Chain of Thought, Reflection, and Verification** make agents more reliable and transparent.

10. **Delegation and composition** let you break complex work into focused subtasks handled by specialist agents.

11. **Agents require clear stopping points.** Design them to know when the task is complete and report their findings.

### Common Pitfalls to Avoid

- **No clear stopping conditions:** Always design agents to recognize when they're done
- **Unclear tool descriptions:** If the agent doesn't understand what tools do, it won't use them well
- **Too much context:** Long, rambling system prompts confuse agents. Be clear and concise
- **Ignoring errors:** Always handle tool errors gracefully and provide clear error messages
- **No verification:** Don't assume the agent's work is correct. Have it test its own output
- **Single agent for everything:** Break work into specialized agents when possible
- **Agent teams for sequential work:** Teams shine for parallel exploration, not for tasks that must happen in order — the coordination overhead isn't worth it

### What's Next: Chapter 4

In Chapter 4, we go deeper into **Advanced Agent Techniques**. You'll learn:

- **Multi-agent orchestration:** Building coordinated networks of agents that collaborate
- **Advanced tool design:** Creating sophisticated tools that handle complex interactions
- **Prompt engineering for agents:** Fine-tuning system prompts for maximum reliability
- **Debugging agent behavior:** Understanding why agents fail and how to fix them
- **Cost optimization:** Making agents efficient and cost-effective
- **Production deployment:** Running agents reliably with monitoring and error recovery

By the end of Chapter 4, you'll be able to design and deploy production-grade multi-agent systems.

---

**Chapter 3 Complete.** You now understand agents deeply. Time to start building.
