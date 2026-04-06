# Chapter 5: Context Handling

## Introduction

Context window management is one of the most critical but often overlooked skills in agentic development. While Chapters 1-3 taught you how to build agents and orchestrate workflows, this chapter focuses on one of production agentic systems' hardest problems: keeping your agent's working memory under control.

Most developers don't hit context limits in simple scripts. But when you're building agents that read files, call APIs, orchestrate multiple sub-agents, and maintain conversation state—you hit these limits fast. The difference between a production-grade agent and a proof-of-concept is often not fancy architecture; it's disciplined context management.

By the end of this chapter, you'll understand not just how context works, but how to architect systems that stay lean, responsive, and cost-effective even when running for days or weeks.

---

## 1. What is the Context Window?

### Defining Context

The context window is the total amount of "working memory" available to an LLM in a single interaction. Think of it as the model's conversation buffer. Everything you feed into Claude in a single request—the system prompt, the conversation history, the tools available, the files you've pasted in, the results from tool calls—all of it counts against your context window.

The context window has two separate limits:

1. **Input context limit**: How much you can send to Claude (measured in tokens)
2. **Output context limit**: How many tokens Claude can generate in response (typically ~8,000 tokens maximum)

This chapter focuses primarily on managing input context, though output limits matter for streaming and long-form generation tasks.

### Understanding Tokens vs. Words

Tokens are the currency of context. A token is not a word—it's a smaller unit. This distinction matters because it directly affects how many tokens you actually use.

**Rough heuristics for token counting:**

- **English prose**: ~1 token ≈ 4 characters (or ~0.75 tokens per word)
- **Code**: Much denser. ~1 token ≈ 2.5 characters. Code uses more tokens per line than English because of special characters, indentation, and syntax
- **JSON**: Dense like code. Bracket heavy, so higher token density
- **Whitespace and newlines**: Each typically costs 1 token per line, even if the line is just indentation

For example:
- A 10,000-word technical document might be ~13,000 tokens
- A 1,000-line source file might be ~2,500–3,500 tokens (because of code density)
- A JSON file with 100 objects might be ~2,000 tokens depending on structure

The best way to get exact counts is to use Claude Code's `/cost` command, which shows you actual token usage after each interaction.

### Current Claude Model Limits

As of March 2026, here are the context limits for Claude models:

| Model | Input Context | Output Context | Best For |
|-------|---------------|----------------|----------|
| Claude Haiku 3.5 | 200K tokens | ~8K tokens | Speed-critical agents, high-volume processing |
| Claude Sonnet 4.5 | 200K tokens | ~8K tokens | General-purpose agentic work (recommended) |
| Claude Sonnet 4.6 | 200K tokens | ~8K tokens | Improved reasoning for complex tasks |
| Claude Opus 4 | 200K tokens | ~8K tokens | Deep reasoning tasks, less latency-sensitive work |

200K tokens is substantial—roughly equivalent to:
- A 150,000-word book
- 60,000 lines of code
- 15 hours of conversation transcript

But in production agentic systems, that fills up faster than you'd expect.

### What Counts Toward Your Context

Everything you send to Claude counts:

1. **System prompt** (CLAUDE.md, instructions, rules)
2. **Conversation history** (every message you've sent, every response Claude has given)
3. **Tool definitions** (all the functions available to the agent; typically 1K–5K tokens for a large tool set)
4. **File contents** (every file you read, every pasted code block)
5. **Tool results** (every API response, command output, file read result)
6. **Current request** (the question you're asking right now)

None of this is "free." A 100-line file you casually paste in costs ~250 tokens. A tool result from an API call that returns 50 objects costs ~2K tokens. These add up quickly in agent loops.

### What Happens When You Hit the Limit

If you send Claude a request that exceeds the input context limit, here's what happens:

1. **Truncation**: Claude Code typically truncates the conversation history, keeping the most recent messages and dropping older ones. You lose the conversational context from earlier in the session.
2. **API Error**: The Anthropic API rejects the request with a `context_length_exceeded` error.
3. **Degraded Quality**: Before hitting hard limits, as context grows beyond ~150K tokens, the model's attention dilutes. Response quality can degrade; the model "loses track" of things.
4. **Increased Cost**: You're paying for every token. 200K tokens at Claude Sonnet rates is ~$1.20 per request.

The output limit is even more restrictive. If you ask Claude to generate 20,000 tokens of output, it will stop at ~8,000 tokens and return a message saying "I've reached my response limit."

---

## 2. Why Context Management Matters for Agents

Agents are context-hungry machines. Unlike a simple Q&A system where you send a question and get an answer, agents accumulate context across multiple turns as they read files, call tools, and make decisions.

### How Context Explodes in Agent Workflows

Consider a simple file-reading agent that processes a codebase:

1. **Initial system prompt**: 500 tokens
2. **User request**: "Analyze the codebase" (50 tokens)
3. **Agent reads file 1** (`server.js`, 5KB): +1,500 tokens
4. **Agent reads file 2** (`database.js`, 4KB): +1,200 tokens
5. **Agent reads file 3** (`auth.js`, 3KB): +900 tokens
6. **Agent generates analysis**: Claude responds with 1,500 tokens of analysis
7. **User asks a follow-up question**: +200 tokens

After just 3 files and one follow-up, you've used ~5,850 tokens of a 200K budget. That seems fine until you realize:

- If each file is larger (20KB instead of 5KB), you've quadrupled the cost
- If the agent reads 20 files (a realistic codebase), you're at ~30K tokens just from file contents
- If you have 5 follow-up questions, you've added another 1K tokens of history
- In a multi-agent workflow, each sub-agent call produces results that get added to the context

A file-reading agent processing a medium-sized codebase can exhaust context in minutes if not managed carefully.

### The Three-Fold Cost: Speed, Money, Quality

Poor context management affects three critical dimensions of agentic systems:

1. **Speed**: Each token you send to Claude adds latency. A 150K-token request is 10x slower than a 15K-token request. Agents need to be fast; excess context kills performance.

2. **Cost**: At Anthropic's rates, 200K tokens costs ~$1.20 for input with Claude Sonnet. If you burn through a full context window on every agent call, costs skyrocket. A workflow that should cost $0.50 costs $12.

3. **Quality**: The "lost in the middle" problem is well-documented in LLM research. When a model receives extremely long contexts (>100K tokens), its attention becomes diluted. Critical information in the middle of the context gets passed over. The model performs worse on tasks requiring detailed recall of information embedded deep in the conversation.

### The "Lost in the Middle" Problem

Claude (like all LLMs) doesn't have perfect recall of everything in its context. Information placement matters:

- **Start of context**: Strong attention. System prompt and initial instructions are well-attended.
- **Middle of context**: Weak attention. A tool result buried 50K tokens into the context might be overlooked.
- **End of context**: Strong attention. Recent messages and the current request are priorities.

This means that in very long contexts, information in the middle is effectively "lost." If you have:

```
[System Prompt: 500 tokens]
[Instructions about error handling: 1K tokens]
[50K tokens of file contents and tool results]
[Current question: 200 tokens]
```

The error-handling instructions (positioned ~1K tokens in) might be ignored because the model's attention is focused on the recent 50K tokens of file contents and your current question.

The fix: structure your context so that the most important information is either at the very beginning or very end, never buried in the middle.

---

## 3. Measuring and Monitoring Context

You can't manage what you don't measure. Monitoring context usage is as important as monitoring memory usage in traditional applications.

### Using `/cost` in Claude Code

The Claude Code CLI provides a `/cost` command that shows you exactly how many tokens you've used in the current session:

```
> /cost

Session Summary:
  Input tokens used: 45,230
  Output tokens generated: 3,450
  Total session cost: $0.36
  Input context remaining: 154,770 tokens (~77% capacity)
```

Run this frequently. Get in the habit of checking `/cost` after large operations (reading files, running agent loops, calling tools). This gives you visibility into whether you're trending toward context limits.

### Watching the Context Bar

In Claude Code's UI, you'll see a context bar that visualizes your token usage. When it fills up, you're approaching limits. It's a visual cue to take action (compact, clear, or restructure) before you hit hard limits.

### Setting Up Alerts and Budgets

In production agentic systems, set explicit context budgets and alert thresholds. Monitor token usage and log warnings when approaching limits:

```
CONTEXT_BUDGET = 150000  # Leave 50K buffer
ALERT_THRESHOLD = 0.8 * CONTEXT_BUDGET  # Alert at 80%
```

When current usage exceeds the alert threshold, pause and consider compacting or clearing the session. This prevents surprises in production workflows.

---

## 4. Strategies for Managing Context

Managing context is an art and a science. Different strategies suit different situations. Here are the core patterns used in production agentic systems.

### a) CLAUDE.md — The Permanent Context Layer

Every Claude Code session automatically loads a file called `CLAUDE.md` from your project root. This file contains permanent instructions, conventions, and rules that apply to all interactions in that project.

**Why CLAUDE.md counts against context:**

`CLAUDE.md` is loaded at the start of every session and stays in context for the entire session. It counts toward your 200K token limit. Don't treat it as free memory.

**How to structure CLAUDE.md:**

Keep it focused. Not a dump of everything, but a curated set of the most important conventions. Order matters—put the most critical rules first:

```markdown
# Project Conventions

## Core Principles (Most Important)
- All API responses must be validated before use
- Critical errors must be logged to `errors.log`
- Never modify production database without explicit approval

## Code Style
- Use camelCase for JavaScript variables
- Maximum line length: 100 characters
- Always include JSDoc comments for public functions

## Architecture
- API handlers in `src/handlers/`
- Database models in `src/models/`
- Utilities in `src/utils/`

## Deployment
- Staging: automatic on merged PRs
- Production: manual deployment with approval

## Rarely-Used Details
- Historical context about why we chose Postgres over MongoDB
- Edge cases in our payment system
```

The critical rules come first (rules that affect behavior), architecture details come next, and historical context comes last.

**Trimming CLAUDE.md as projects evolve:**

Every few weeks, review CLAUDE.md and remove rules that are no longer relevant. Old rules accumulate token cost with zero value. If you've migrated away from a tool, remove the section about it. If a convention became standard practice and doesn't need to be said, remove it.

> Tip: Keep CLAUDE.md to 500–1000 tokens maximum. Anything larger is probably too detailed for a permanent context layer. Move very specific details to other documents that agents can read on demand.

### b) `/compact` — Context Compaction

As a session grows, conversation history accumulates. You might have 50 back-and-forths with Claude, all of which are still in context. `/compact` summarizes the conversation, replacing the full history with a compressed version.

**What `/compact` does:**

- Reads the entire conversation history
- Generates a summary capturing key decisions, completed tasks, and current state
- Replaces the verbose history with the summary
- Preserves the current question or task being worked on

Result: you reclaim 30K–50K tokens by replacing a 50-message conversation with a 2-paragraph summary.

**When to use `/compact`:**

- After completing a major phase of work
- When context usage is 60%+ but you need to continue the session
- Before switching to a different sub-task (compact, then work on the new task)
- Periodically in long sessions to keep context lean

**What gets preserved and lost:**

- **Preserved**: Task summary, key architectural decisions, current problem statement, important discoveries
- **Lost**: Exact wording of earlier messages, debugging steps that were ultimately abandoned, conversation tangents

If you need to refer to the exact wording of something discussed 30 messages ago, you'd need to scroll back or ask Claude to find it before compacting.

**How to trigger `/compact`:**

```
> /compact
```

Claude Code handles the compaction. A few seconds later, your conversation history is replaced with a summary.

**Automatic compaction in Claude Code:**

You can configure Claude Code to automatically compact when context reaches a certain threshold. This prevents accidental context limit hits:

```
compact_at_percentage: 70
```

This tells Claude Code to automatically compact the conversation when 70% of context is used, keeping you safely below the limit without manual intervention.

### c) `/clear` — Fresh Start

`/clear` wipes the conversation history and starts fresh. Use it when you've completed a task and are starting something new.

**When to use `/clear`:**

- Task is complete, next task is unrelated
- Context has degraded and starting fresh would be better
- You're switching from coding to writing or vice versa (different context needs)
- Starting a completely new project or workstream

**What it does:**

- Clears conversation history (you lose everything said before `/clear`)
- Keeps CLAUDE.md and project configuration
- Sets token usage back to baseline

**Key principle: don't carry context you don't need**

Many developers keep a session open for weeks, accumulating context they'll never need again. Every tool result from Tuesday is still taking up tokens on Friday. Every debugging conversation from a resolved bug is still there. Use `/clear` liberally to start fresh when old context is dead weight.

> Warning: `/clear` is irreversible. If there's any chance you'll need something from the current context, save it to a file before clearing.

### d) Selective Context Loading

Don't load files wholesale. Load only what you need, when you need it.

**Anti-pattern: dumping entire files**

```
> Read me the entire `auth.js` file so you understand the authentication system.
```

If `auth.js` is 500 lines, that's ~1,200 tokens. You might only need 50 lines—the login function.

**Better: read specific sections**

```
> Read lines 50–100 of `auth.js` (the login function). I want to understand how we handle JWT tokens.
```

Or:

```
> Show me just the `validateToken()` function from `auth.js`.
```

This loads ~200 tokens instead of 1,200.

**Querying for specific data**

If you're working with databases or APIs, query for specific records rather than dumping entire datasets. Instead of fetching and including all users, fetch only recent users. This pattern saves orders of magnitude in context tokens.

**Designing agents to request information**

Structure agents so they ask for specific information rather than being handed everything. Give agents tools to read files and query databases rather than pre-loading all data. This keeps context lean by default.

### e) Summarization Agents

A powerful pattern for multi-agent workflows: before passing results from one agent to another, run them through a "summarizer" agent.

**How it works:**

```
Agent A: Analyzes codebase, produces 5,000 tokens of analysis
    ↓
Summarizer Agent: Reads Agent A's output, produces 500-token summary
    ↓
Agent B: Receives 500-token summary instead of 5,000 tokens
```

Agent B gets the essential findings without the noise, saving 90% of context.

**When to use this pattern:**

- Multi-agent workflows where agents pass results to each other
- Tool results that are verbose but contain a simple answer (e.g., a 2,000-token API response that really says "no results found")
- Reducing context bleed between workflow stages

**When NOT to use this pattern:**

- If Agent B needs detailed information (e.g., specific code snippets, exact metrics)
- If summary loses critical nuance
- In single-agent workflows (summaries add latency with no context benefit)

### f) External Storage — Keep Context Lean

The strongest context management strategy: don't keep data in context at all. Store it externally and retrieve on demand.

**Patterns:**

1. **File-based state**: Store intermediate results in JSON files. Agents read these files only when needed.

2. **Database queries**: For large datasets, query the database for specific records instead of loading everything into context.

3. **Vector stores**: For large knowledge bases, use vector embeddings to retrieve only relevant sections.

4. **APIs and webhooks**: Instead of embedding API data in context, call the API when needed and use only the response.

Benefits:
- Context stays lean (5K tokens instead of 50K)
- Data is always fresh (not stale information from when you loaded it)
- Scales to datasets of any size
- Agents can run longer without context issues

> Tip: The "retrieve on demand" pattern is the foundation of truly scalable agentic systems. Build it early.

### g) System Prompt Hygiene

Your system prompt (CLAUDE.md, instructions, tool definitions) is loaded on every single turn. Even if you've cleared the conversation, the system prompt is still there.

**Costs of a bloated system prompt:**

- Every request includes it (1K-token system prompt means 1K token minimum per request)
- Duplicates waste space: if your system prompt contains the same rule twice, you're paying for both
- Unused tools: if your system prompt defines 50 tools but the agent only needs 3, the other 47 are dead weight

**Keep system prompts concise:**

- Remove tools that aren't being used
- Use references instead of duplication: "Follow the conventions in CLAUDE.md" rather than listing them all
- Delete rules that are no longer relevant
- Group related rules so you can delete entire sections when contexts change

Example of bloated system prompt:

```markdown
You are an AI assistant for our engineering team.

Always validate inputs. Always validate inputs. Always validate inputs.
  (redundancy: same rule listed 3 times)

Use camelCase for variable names. Use camelCase for variable names.
  (redundancy: same rule listed twice)

Here are all 47 tools available to you...
  (bloat: agent only needs 3)
```

Clean system prompt:

```markdown
You are an AI assistant for our engineering team.

Follow all conventions in CLAUDE.md.

Available tools: AuthManager, FileHandler, DatabaseQuery.

For architecture questions, refer to docs/architecture.md.
```

The clean version is 1/10 the size but conveys the same information, because it uses references instead of duplication.

---

## 5. Context Management in Multi-Agent Workflows

In multi-agent workflows, each agent gets its own context window. This is actually a feature, not a bug.

### Isolated Context Windows

When Agent A calls Agent B, Agent B starts with a fresh context window. Agent B doesn't see Agent A's internal reasoning, tool results, or debugging steps. Agent B only sees what Agent A explicitly passes to it.

**Implication:** Context usage is local to each agent. Agent A can use 50K tokens and still leave Agent B with a fresh 200K tokens.

**Trade-off:** You have to be explicit about what information to pass between agents. You can't assume Agent B knows about decisions made in Agent A.

### The Orchestrator Pattern

In a multi-agent system, the orchestrator agent coordinates other agents and summarizes results before passing them along:

```
User Request
    ↓
Orchestrator Agent
    ├─ calls Agent A (research)
    ├─ calls Agent B (analysis)
    ├─ calls Agent C (report generation)
    ↓
Orchestrator summarizes all 3 results
    ↓
Returns final report to user
```

The orchestrator:
1. Calls each agent with a specific, focused task
2. Receives results from each
3. Synthesizes results into a cohesive response

**Context benefit:** Each agent's context is isolated. Even if Agent A produces 10K tokens of research, Agent B doesn't carry that weight. The orchestrator receives results from all three agents but summarizes them before returning to the user, keeping the final context lean.

### Context Budgets per Agent

In complex workflows, explicitly allocate context budgets to each agent. This helps prevent context exhaustion and forces agents to be focused and efficient.

Each agent is designed to know its budget and rejects tasks that would exceed it, or splits large tasks into smaller subtasks that fit within the budget.

This prevents context exhaustion and forces agents to be focused and efficient.

### The "Need to Know" Principle

Agents should receive only the information they need for their specific task.

**Anti-pattern:**

```
Orchestrator passes everything to Agent A:
  - Entire codebase
  - All recent issues
  - All deployment logs
  - All team decisions

Agent A then searches through irrelevant information to find what's needed.
```

Agent A gets 100K tokens of irrelevant information. The codebase and recent issues might be relevant, but deployment logs and team decisions probably aren't.

**Better pattern:**

```
Orchestrator passes only what Agent A needs:
  - Specific module from codebase
  - Issues related to the task
  - Only relevant metadata

Agent A receives exactly what's needed for efficient work.
```

Agent A gets 5K tokens of directly relevant information. Cleaner, faster, cheaper.

---

## 6. Hook-Based Memory Persistence

For long-running projects, you need memory that persists across sessions. Hooks provide this capability.

### What Are Hooks?

Hooks are scripts that run automatically at session start and end. They let you:
- Load context from files at the start of a session
- Save state to files at the end of a session
- Maintain "memory" across multiple Claude Code sessions

### The SessionStart Hook

The `SessionStart` hook runs when you start a new session. It automatically loads relevant context files and reminds the agent about the project's state.

When you start a new session, this hook runs and loads:
1. A project memory file (decisions made, completed work, open questions)
2. A decisions file (architectural choices, why certain approaches were taken)
3. Current context (what project/sprint you're working on)

**Result:** The agent starts the new session with full context of the project, even though it's a fresh session.

### The SessionEnd Hook

The `SessionEnd` hook runs when you end a session. It saves important state to files for the next session to load.

**Workflow:**

1. You end the Claude Code session
2. SessionEnd hook triggers
3. Agent writes a summary of work completed
4. Agent writes any decisions made
5. Files are saved for the next session

### Using Hooks for Memory Persistence

The pattern uses hooks to maintain project memory:

1. **SessionStart**: Load project memory and recent decisions
2. **Work**: Agent completes tasks, makes decisions
3. **SessionEnd**: Agent saves work summary and decisions
4. **Next session**: SessionStart loads the saved files

Result: each session starts with full context of the project, even across day boundaries.

---

## 7. Long-Running Projects — Context Strategy

For projects running days, weeks, or months, in-context memory is useless. You need external memory systems.

### Why In-Context Memory Fails

If you keep the same Claude Code session open for 3 weeks:
- Context grows to 180K tokens
- Work from day 1 is still in context but irrelevant
- You're paying for all those tokens every request
- Response time slows down

Better: close sessions regularly and use persistent memory files.

### Build a "Project Memory" File

Maintain a running log of decisions, completed tasks, and open questions in a `project-memory.md` file:

```markdown
# Project Memory

## Completed Tasks
- [x] Set up PostgreSQL database
- [x] Implement user authentication
- [x] Deploy staging environment

## Current Work
- [ ] Implement payment processing
- [ ] Write API documentation
- [ ] Optimize database queries

## Key Decisions
- Using PostgreSQL instead of MongoDB for relational data integrity
- Authentication via JWT tokens, refresh tokens stored in secure HTTP-only cookies
- Deployment via GitHub Actions + AWS CodeDeploy

## Open Questions
- Should we implement rate limiting at the API level or use a reverse proxy?
- Do we need a message queue for async processing, or is basic job scheduling sufficient?

## Architecture Notes
- Monolithic backend (src/), could split into microservices later if needed
- Frontend is SPA in Next.js, separate from backend
- Database in separate RDS instance for scalability
```

At the start of each session, the agent reads this file and understands the project context without needing to review the entire conversation history.

### Agent "Briefing" Pattern

For long-running projects, structure sessions around a briefing document. At the start of each session:

1. Agent reads the project memory file
2. Agent reads the current briefing document (what you're working on today)
3. Agent starts work
4. At end of session, agent updates the memory file

**Briefing template:**

```markdown
# Session Briefing: 2026-03-31

## What We're Doing Today
Implementing payment processing with Stripe integration.

## What's Already Done
- User authentication working
- Database schema designed
- API routes stubbed out

## What We Need to Do
1. Create Stripe account and get API keys
2. Implement payment endpoint
3. Add payment UI to frontend
4. Test with Stripe's test card numbers

## Context from Previous Sessions
- Rate limiting was decided to be at API level (see decisions.md)
- We're using JWT + secure cookies (see architecture.md)
- Stripe was chosen over PayPal on 2026-03-28 (see decisions.md)

## Potential Blockers
- Need Stripe API keys from finance team (email sent, waiting)
- Payment endpoints need to handle currency conversion (research in progress)
```

Agent reads this, understands exactly what to do, and gets to work.

### Three-File Memory System

Maintain three files for comprehensive project memory:

| File | Purpose | Size | Updated |
|------|---------|------|---------|
| `CLAUDE.md` | Permanent conventions and rules | 500–1000 tokens | Rarely (every few weeks) |
| `decisions.md` | Architectural and design decisions | 1K–3K tokens | Every session |
| `progress.md` | Task list, completed work, current status | 1K–2K tokens | Every session |

**Workflow:**

- Session starts: load all three files (2.5K–5K tokens)
- Agent works based on these files plus current conversation
- Session ends: update `decisions.md` and `progress.md`
- Next session: load updated files

This keeps context usage low (you're reading files on demand, not carrying weeks of conversation history) while maintaining full project memory.

---

## 8. Debugging Context Issues

When things go wrong—Claude forgets earlier instructions, contradicts itself, seems confused—context issues are often the culprit.

### Symptoms of Context Problems

1. **Forgotten instructions**: You told Claude a rule in message 1, and by message 50, it's ignoring that rule
2. **Contradictions**: Claude says "We decided to use PostgreSQL" then later says "Let's use MongoDB"
3. **Lost context**: Claude says "I need the codebase to answer that" even though you provided it earlier
4. **Degraded quality**: Response quality seems to decline as conversation progresses
5. **"I don't have access to..." errors**: Claude forgets tools or file contents it had earlier

### How to Diagnose

1. **Check token usage**: Run `/cost` and see how many tokens you're using
   - If over 150K: context is crowded
   - If over 180K: you're in degraded territory
   - If over 190K: imminent context exhaustion

2. **Review what's in context**: Use `/compact` to summarize. Does the summary mention the rule Claude forgot? If not, the rule was lost in context.

3. **Look at conversation structure**: Are important instructions early in the conversation? Or buried 30 messages in?

4. **Check CLAUDE.md**: Is the rule in CLAUDE.md? If yes, and Claude is ignoring it, CLAUDE.md might not be loaded properly, or it's buried too deep.

### Fix Patterns

**If context is over 150K:**

```
> /compact
```

Compacting summarizes the conversation. You reclaim 30K–50K tokens. Compression often makes Claude more accurate (simpler context, less lost-in-the-middle effect).

**If you're contradicting yourself:**

```
> /clear
```

If the session has grown messy and contradictions are proliferating, starting fresh might be better than trying to untangle it.

**If Claude forgets a critical rule:**

Move the rule to the top of CLAUDE.md:

```markdown
# CLAUDE.md

## CRITICAL: This Rule Must Never Be Forgotten
- Always validate API responses before using them
- Never proceed without explicit user approval for production changes
- All errors must be logged

## Other rules (less critical)...
```

Rules at the top get more attention due to the "lost in the middle" problem.

**If a tool result is being ignored:**

Don't rely on inline tool results buried in the conversation. Instead, save the result to a file and refer to it:

```
I've saved the API response to results.json. Use that to answer the question.
```

This concentrates information at the end of the context (where attention is stronger) rather than buried in the middle.

### The "Instruction Drift" Problem

In long agent loops, instructions drift. An agent follows Rule A for the first 10 iterations, then gradually starts ignoring it by iteration 50.

**Cause:** Rule A is early in the context. As the conversation grows, Rule A gets deeper and deeper in the context, until it's buried and ignored.

**Prevention:**

1. **Use hooks to re-inject instructions**: At the start of each agent iteration, re-state critical rules.
2. **Use external files for critical rules**: Instead of embedding rules in context, store them in a file and have the agent reference them each iteration.
3. **Short sessions**: End sessions before context grows too large. Start fresh with hooks reloading the rules.

Even if the previous iterations' context grows large, each new iteration starts with critical rules at the top, preventing drift.

---

## 9. Hands-On Exercises

These exercises teach context management through practice. Every exercise is self-contained: all files needed are created as part of the setup, and the learner starts with the baseline project from Chapters 1-3 (with `src/`, `test/`, `CLAUDE.md`, and `.claude/agents/`).

---

**Exercise 5-A: Feel the Context Fill Up**

**Goal:** Directly observe context growing and learn to manage it with `/cost` and `/compact`.

**Setup:** Add several files to the project so there's something to read. Run these commands in your terminal to create stub files:

```bash
# Create src/billing.rb
cat > src/billing.rb << 'EOF'
# Billing module
def calculate_invoice(order_id); end
def apply_discount(invoice, code); end
def send_invoice_email(invoice, customer); end
EOF

# Create src/notifications.rb
cat > src/notifications.rb << 'EOF'
# Notifications module
def send_welcome_email(user); end
def send_password_reset(user, token); end
def send_order_confirmation(order); end
EOF

# Create src/reports.rb
cat > src/reports.rb << 'EOF'
# Reporting module
def daily_sales_report(date); end
def customer_activity_report(customer_id, start_date, end_date); end
def inventory_report; end
EOF
```

Now start a fresh Claude Code session:
```bash
claude
```

First, check your baseline:
```
/cost
```

Note the number. Now ask Claude to read everything:
```
Read every file in the src/ directory and give me a one-paragraph summary of what each module does.
```

After it finishes:
```
/cost
```

**Observe:** Each file read adds to the context. The delta shows exactly how many tokens those reads cost.

Now run:
```
/compact
```

Then `/cost` again.

**Observe:** Tokens drop significantly. Claude has replaced the full conversation history with a compressed summary, but it still knows what it was doing.

**What to experiment with:**
- Ask a follow-up question after `/compact` — does Claude still have the context it needs?
- Try `/clear` instead of `/compact` and compare: compact preserves the task summary, clear wipes everything
- Create a very large file (copy-paste the contents of any 3 files into a new `src/combined.rb`) and check how much one large file read costs

---

**Exercise 5-B: CLAUDE.md as Context Architecture**

**Goal:** Understand how CLAUDE.md size affects every single session, and how to keep it lean.

**Setup:** You need two versions of CLAUDE.md to compare. First, make your current CLAUDE.md intentionally bloated.

Replace your `CLAUDE.md` with this verbose version:

```markdown
# Task Manager API — Comprehensive Developer Guide

## Project Overview
This is a comprehensive task management API built with Ruby. The project aims to provide a robust, scalable, and maintainable solution for managing tasks, users, billing, and notifications. The codebase follows object-oriented principles and is designed with extensibility in mind.

## Architecture
The project uses a modular architecture where each concern is separated into its own module. The src/ directory contains the core business logic, while test/ contains unit tests written with Minitest. We follow the single responsibility principle throughout the codebase.

## Complete Coding Conventions
- Use snake_case for all variable names, method names, and file names
- Use CamelCase for class names and module names
- All public methods must have a comment explaining what they do
- All methods should be under 20 lines; if longer, extract helper methods
- Never use global variables; pass dependencies as parameters
- Use frozen string literals at the top of every file
- Never use `puts` for logging; use the logger module
- All database queries must go through the DB module, never use raw SQL
- Never hardcode credentials; use environment variables
- Always handle errors with begin/rescue, never let exceptions propagate uncaught
- Write at least one test per public method
- Tests should be deterministic; never use sleep() in tests
- Use descriptive variable names; single-letter variables are only acceptable in short loops

## Development Workflow
- Always run tests before committing
- Use conventional commit messages: feat:, fix:, docs:, refactor:, test:
- Create a feature branch for each new feature
- Request code review before merging
- Update the CHANGELOG.md with every significant change

## What Claude Should Never Do
- Modify production config files
- Delete any data without explicit confirmation
- Add gems without checking with the team
- Push directly to main branch
```

Start a fresh session and check the baseline cost:
```
/cost
```

Now replace CLAUDE.md with a lean version:

```markdown
# Task Manager API

## Critical Rules
- Never delete data — archive only
- No raw SQL — use DB module
- No hardcoded credentials — use ENV
- Tests required for all public methods

## Commands
- Tests: `ruby -Itest test/**/*_test.rb`

## Conventions
- See `docs/CONVENTIONS.md` for full style guide (read on demand)
```

Create `docs/CONVENTIONS.md` with all the verbose content from before (so nothing is lost, just moved):

```markdown
# Detailed Coding Conventions

## Variable and Method Naming
- Use snake_case for all variable names, method names, and file names
- Use CamelCase for class names and module names

## Method Design
- All public methods must have a comment explaining what they do
- All methods should be under 20 lines; if longer, extract helper methods
- Use descriptive variable names; single-letter variables are only acceptable in short loops

## Code Patterns
- Never use global variables; pass dependencies as parameters
- Use frozen string literals at the top of every file
- Never use `puts` for logging; use the logger module
- All database queries must go through the DB module, never use raw SQL
- Never hardcode credentials; use environment variables
- Always handle errors with begin/rescue, never let exceptions propagate uncaught

## Testing
- Write at least one test per public method
- Tests should be deterministic; never use sleep() in tests

## Development Workflow
- Always run tests before committing
- Use conventional commit messages: feat:, fix:, docs:, refactor:, test:
- Create a feature branch for each new feature
- Request code review before merging
- Update the CHANGELOG.md with every significant change

## Guardrails
- Never modify production config files without explicit approval
- Never delete data — archive only
- Never add gems without checking with the team
- Never push directly to main branch
```

Start a fresh session and check baseline cost again:
```
/cost
```

**Observe:** The lean CLAUDE.md has a noticeably lower baseline token count — and this difference applies to every single API call made in every session.

**What to experiment with:**
- Ask Claude "What are the coding conventions for this project?" — it reads `docs/CONVENTIONS.md` on demand instead of always loading it
- Measure how much the baseline drops when you halve your CLAUDE.md
- Add the instruction "If asked about architecture, read docs/ARCHITECTURE.md" — Claude will only read that file when needed

---

**Exercise 5-C: Build a Session Memory System**

**Goal:** Create an agent that gives you a daily briefing and captures what was accomplished — persistent across `/clear` and new sessions.

**Setup:** No extra files needed. The agent creates everything itself.

Create `.claude/agents/session-keeper.md`:

```markdown
---
name: session-keeper
description: Maintains a running log of project work across sessions. Invoke with "start" at the beginning of each day and "end" at the end.
tools: Read, Write
model: haiku
---

You maintain `.claude/session-memory.md` — the project's persistent memory.

**When invoked to START a session:**
1. Read `.claude/session-memory.md` if it exists
2. Summarize the last session in 2-3 sentences
3. List any open questions or blockers from last time
4. Say: "Good morning. Last session: [summary]. Open items: [list]. What are we working on today?"

**When invoked to END a session:**
1. Ask: "What did we accomplish today? Any decisions or blockers?"
2. Wait for the response, then append to `.claude/session-memory.md`:

---
## Session: [today's date]
**Completed:** [bullet list]
**Decisions made:** [bullet list, or "none"]
**Still open:** [bullet list, or "none"]
---

**Memory hygiene:** If `.claude/session-memory.md` exceeds 150 lines, summarize entries older than 2 weeks into a single "### Older history" block.
```

Start a session:
```
Use the session-keeper to start today's session. This is our first day — let's work on adding pagination to the task list.
```

Do some work (ask Claude to implement a `paginate` function in `src/app.rb`):
```
Add a paginate function to src/app.rb that splits a list into pages. Function signature: paginate(items, page_size = 10)
```

End the session:
```
Use the session-keeper to end today's session.
```

Type `/clear` to simulate the next day. Start fresh:
```
Use the session-keeper to start today's session.
```

**Observe:** The agent reads yesterday's notes from the file and briefs you on what was done and what's open — even though the conversation context was cleared.

**What to experiment with:**
- Run this across 5 real working sessions and open `.claude/session-memory.md` — is it useful?
- Share the memory file with a teammate and ask them to invoke the session-keeper — they get instant project context
- Ask: "Use the session-keeper — what decisions have we made about the database?"

---

**Exercise 5-D: The Summarizer Agent**

**Goal:** Add a context-efficient summarization layer between data-gathering and analysis to dramatically cut token usage.

**Setup:** Create a set of source files large enough to be meaningful. Run in your terminal:

```bash
# Create src/middleware.rb
cat > src/middleware.rb << 'EOF'
# Request/response middleware pipeline
# Handles: authentication, rate limiting, request logging, response compression
# Used by all API endpoints

module Middleware
  def self.authenticate(request)
    token = request.headers['Authorization']&.split(' ')&.last
    return false unless token
    JWT.decode(token, ENV['JWT_SECRET'])
    true
  rescue JWT::DecodeError
    false
  end

  def self.rate_limit(ip, limit: 100, window: 3600)
    key = "rate_limit:#{ip}"
    count = Redis.current.incr(key)
    Redis.current.expire(key, window) if count == 1
    count <= limit
  end
end
EOF
```

Now create two competing agent designs:

Create `.claude/agents/codebase-scanner.md`:

```markdown
---
name: codebase-scanner
description: Scans the full codebase and writes a structured summary to .claude/codebase-summary.md. Run this BEFORE any analysis agent to save context tokens.
tools: Read, Write, Glob
model: haiku
---

Scan the entire project and write a concise summary to `.claude/codebase-summary.md`:

# Codebase Summary
## What This Project Does
[2 sentences max]

## Files and Their Purpose
[One line per file: `path` — what it does]

## Key Patterns
[3-5 bullet points: repeated patterns, conventions, important design decisions]

## Entry Points
[Where to start reading for: API endpoints, database, auth, tests]

Be ruthlessly concise — this file is read by other agents. Every word costs tokens.
```

Create `.claude/agents/architecture-analyst.md`:

```markdown
---
name: architecture-analyst
description: Analyses project architecture using the codebase summary. Always reads .claude/codebase-summary.md instead of scanning files directly.
tools: Read
model: sonnet
---

Read `.claude/codebase-summary.md` first. Use ONLY that summary unless you need one specific file to answer a question.

Based on the summary:
1. Identify the top 3 architectural strengths
2. Identify the top 3 architectural risks or weaknesses
3. Recommend 2 concrete improvements with rationale

If you need to read a specific file, state which one and why, then read only that file.
```

Run the two-stage approach:
```
/cost
```
```
Use the codebase-scanner to scan and summarize the project.
```
```
Use the architecture-analyst to review the architecture.
```
```
/cost
```

Now try the naive approach in a fresh session:
```
/cost
```
```
Read all the source files and analyze the architecture of this project.
```
```
/cost
```

**Observe:** The scanner+analyst pipeline uses significantly fewer tokens for the analysis step because the analyst reads a 30-line summary instead of raw source files.

**What to experiment with:**
- Open `.claude/codebase-summary.md` — is it accurate enough to be useful to a human too?
- Run the analyst again without re-running the scanner — it still uses the cached summary from the file
- Ask the analyst a specific question that requires reading one file: "Is the middleware thread-safe?" — watch it minimally expand context

---

**Exercise 5-E: Greedy vs. Lean Agent Design**

**Goal:** Design an agent with context efficiency as a first-class constraint, and measure the difference.

**Setup:** Introduce a real bug to fix. Add this buggy function to `src/calculator.rb`:

```bash
cat > src/calculator.rb << 'EOF'
def percentage_change(old_value, new_value)
  # BUG: crashes when old_value is 0
  ((new_value - old_value) / old_value) * 100
end
EOF
```

Create the greedy version first:

Create `.claude/agents/greedy-debugger.md`:

```markdown
---
name: greedy-debugger
description: Debugs issues by reading the full codebase context first.
tools: Read, Glob
model: sonnet
---

When debugging, build complete context first:
1. Read ALL files in src/ to understand the full system
2. Read ALL test files to understand expected behaviour
3. Read CLAUDE.md and any documentation
4. Only then diagnose the issue
```

Create `.claude/agents/lean-debugger.md`:

```markdown
---
name: lean-debugger
description: Debugs issues by reading minimally — only what's needed to diagnose the specific problem.
tools: Read, Glob, Grep
model: sonnet
---

Debug with the minimum reads necessary:
1. Read only the file containing the error
2. If the error references another file, read only that file
3. Stop reading as soon as you have a hypothesis
4. State your hypothesis, then verify it — don't keep reading "just in case"
5. Target: solve this by reading at most 3 files

Before each file read, state: "Reading [file] because [specific reason]."
```

Check token baseline:
```
/cost
```

Run the greedy debugger:
```
Use the greedy-debugger to find and fix the bug in the percentage_change function
```
```
/cost
```

Fresh session, then run the lean debugger on the same (unfixed) code:
```
/cost
```
```
Use the lean-debugger to find and fix the bug in the percentage_change function
```
```
/cost
```

**Observe:** The lean debugger solves the same problem (divide by zero when old_value is 0) with dramatically fewer tokens because it reads only `src/calculator.rb` and stops.

**What to experiment with:**
- Introduce a bug that genuinely requires reading 2 files to diagnose (e.g., a function in `src/billing.rb` that calls a function from `src/calculator.rb` incorrectly) — does the lean debugger still solve it?
- Add to the lean agent: "If you can't solve it in 3 files, list exactly what you'd need to read next" — a graceful degradation pattern
- Apply the lean principle to one of your other agents — rewrite its system prompt to be more context-frugal

---

## 🔗 Ecosystem Connection

The context management patterns in this chapter are solved at scale by the memory and hook systems in these repos:

| Repo | What it adds to this chapter |
|---|---|
| [affaan-m/everything-claude-code](https://github.com/affaan-m/everything-claude-code) | Full hook-based memory persistence system (SessionStart/SessionEnd hooks) that automatically saves and loads context — the production version of the session-keeper agent you built here |
| [SuperClaude-Org/SuperClaude_Framework](https://github.com/SuperClaude-Org/SuperClaude_Framework) | Automatically reads `PLANNING.md`, `TASK.md`, and `KNOWLEDGE.md` at session start — a structured three-file memory system built into the framework |
| [obra/superpowers](https://github.com/obra/superpowers) | Uses git worktrees (which have isolated context per branch) as a structural solution to context pollution between parallel workstreams |

**Quick start:** In the SuperClaude Framework, create `PLANNING.md` (architecture decisions), `TASK.md` (current sprint), and `KNOWLEDGE.md` (accumulated learnings) in your project root — SuperClaude reads these automatically on every session start.

> See **[Chapter 7 → Section 3](chapter-07-ecosystem.md)** for everything-claude-code hook exercises and **[Chapter 7 → Section 4](chapter-07-ecosystem.md)** for SuperClaude's structured memory system.

---

## 10. Chapter Summary & What's Next

### Key Takeaways

1. **Context is a finite resource.** Your 200K token budget must be managed deliberately. Every file you read, every tool result, every message costs tokens.

2. **Measure what you manage.** Use `/cost` obsessively. Log token usage in agent loops. Set alerts at 60%, 80%, and 95% of budget. You can't manage what you don't measure.

3. **CLAUDE.md is your permanent context layer.** Structure it carefully—critical rules first, rarely-used details last. Keep it under 1000 tokens.

4. **Use `/compact` to reclaim context.** When context grows large, compacting summarizes conversation history and reclaims 30K–50K tokens. Use it liberally between major task phases.

5. **Selective loading beats bulk loading.** Don't read entire files; read specific sections. Don't load all data; query for what you need. Design agents to request information rather than be handed everything.

6. **External storage is your best friend.** Files, databases, vector stores—keep data outside context and retrieve on demand. This scales to unlimited data sizes.

7. **Hooks enable memory across sessions.** Use SessionStart and SessionEnd hooks to load/save project memory. Maintain a three-file system: CLAUDE.md (rules), decisions.md (architecture), progress.md (tasks).

8. **Multi-agent workflows naturally isolate context.** Each agent gets fresh context. Use the orchestrator pattern to summarize results before passing to the next agent. This prevents context bleed.

9. **The "lost in the middle" problem is real.** Information in the middle of very long contexts gets overlooked. Put critical rules at the start or end; never bury important info in the middle.

10. **Long-running projects need external memory.** For projects lasting days or weeks, don't rely on in-context history. Use persistent memory files (project memory, decisions, progress) and load them at session start via hooks.

### What's Next: Chapter 6

With context management mastered, you're ready to tackle error handling and resilience—the next frontier in building production agentic systems. Chapter 6 covers:

- **Graceful degradation**: What happens when tools fail or APIs are down?
- **Retry logic**: When and how to retry failed operations
- **Circuit breakers**: Preventing cascading failures in multi-agent systems
- **Fallback strategies**: Continuing work when the ideal path is blocked
- **Observability**: Instrumenting agents so you can debug production issues

These patterns transform agents from fragile prototypes into systems reliable enough to run unattended for days.

---

## Further Reading & Resources

- **Token Counting**: Use the Anthropic token counter (in the API docs) for exact counts
- **Model Documentation**: Current context limits for all Claude models: https://docs.anthropic.com/claude/reference/getting-started-with-the-api
- **Vector Storage**: Pinecone, Weaviate, or Milvus for large-scale knowledge bases
- **State Management**: Redis or DynamoDB for persistent agent state across sessions
- **Monitoring**: DataDog, New Relic, or custom logging for production agent monitoring
