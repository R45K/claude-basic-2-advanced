# Chapter 4: Agentic Workflows

## Introduction

In Chapter 3, you learned to build a single agent—a focused AI system that could handle a specific task with intelligence and autonomy. But real-world software development rarely works in isolation. Features are complex, teams are specialized, and time is precious. This chapter teaches you how to orchestrate *multiple* agents into workflows that accomplish more together than any single agent could alone.

By the end of this chapter, you'll understand how to design workflows where agents collaborate, hand off work to each other, run in parallel, and recover from failures. You'll learn the patterns that power production systems, the tooling that makes it practical, and how to avoid the common pitfalls that make multi-agent systems brittle.

---

## 1. What is an Agentic Workflow?

### Definition

An **agentic workflow** is a coordinated system of multiple agents working toward a shared goal, with explicit handoff points, communication protocols, and error handling. Unlike a monolithic agent, a workflow decomposes the work into specialized tasks, each handled by an agent optimized for that domain.

Think of it like a software development team: you have a planner who breaks down requirements, developers who implement, QA who tests, and a tech lead who reviews. Each person is an agent; the team structure is the workflow.

### The Spectrum: Single Agent → Workflow → Pipeline

Understanding the distinctions matters for choosing the right approach.

**Single Agent**
- One AI system handles the entire task
- Fast to build, easy to reason about
- Works well for bounded, well-defined problems
- Limits: token context, no specialization, no parallelism
- Examples: a code completion tool, a simple chatbot, a documentation writer

**Workflow**
- Multiple agents, explicit roles, structured communication
- Agents are aware of each other (orchestrator knows about workers)
- Handoffs are programmatic: agent A writes output, agent B reads it
- Failures in one agent can be caught and recovered
- Examples: CI/CD pipelines, code review systems, multi-step planning

**Pipeline**
- Specialized case of workflow where work flows linearly (A → B → C → D)
- Each stage is independent and stateless (or nearly so)
- Highly parallelizable if stages are replicated
- Examples: data processing ETL, image generation pipelines, batch code analysis

A rule of thumb: *Start with a single agent. Build a workflow when one agent isn't enough. Build a pipeline when the workflow becomes purely linear and you need parallelism at scale.*

### Why Workflows? The Compelling Reasons

#### Context Limits

Modern LLMs have large context windows (100K+ tokens), but they're not infinite. A large codebase, comprehensive requirements, and iterative refinement can exhaust even generous contexts. Workflows break the problem into pieces, each fitting within a context window.

#### Specialization

An agent fine-tuned to write Python is better at writing Python than a generalist. An agent trained on test patterns writes better tests. Workflows let you deploy the right tool for each job.

#### Parallelism

A single agent processes sequentially. A workflow can split work across multiple agents running concurrently. Analyzing 10 modules in parallel takes 1/10th the wall-clock time (roughly).

#### Reliability

If one step fails, a workflow can catch it, retry, or route to a different agent. Fallback strategies are built in. A monolithic agent either fails entirely or is too cautious to be useful.

#### Token Efficiency

Breaking a problem into pieces lets each agent focus narrowly, using fewer tokens overall. A 100K token problem becomes five 20K token problems.

### When Workflows Are Overkill

Not every problem needs a workflow. Overhead includes:

- **Communication latency**: each handoff takes time (network round trips, file I/O)
- **Complexity**: coordinating agents requires state management, error handling, retry logic
- **Debugging difficulty**: understanding why a workflow failed across three agents is harder than tracing a single agent
- **Cost**: running five agents instead of one costs more in API fees

**When to stick with a single agent:**
- The task is small, well-defined, and fits in one context window
- Latency matters and you can't afford inter-agent communication delays
- The problem is inherently non-parallelizable
- You're prototyping and need fast iteration

**When a workflow becomes essential:**
- The task doesn't fit in a single context window
- Different steps benefit from different model configurations or specializations
- Parallelism would deliver meaningful speedup
- You need human oversight at checkpoints
- Reliability and error recovery matter

### Real-World Analogy: CI/CD Pipelines

A continuous integration/deployment pipeline is an agentic workflow you already know:

```
Code Push
    ↓
[Agent: Build] → Compile code, link, check syntax
    ↓ (on success)
[Agent: Test] → Run unit tests, integration tests
    ↓ (on success)
[Agent: Security Scan] → Check for vulnerabilities
    ↓ (on success)
[Agent: Deploy] → Push to staging/production
    ↓ (on success)
[Agent: Monitor] → Check health, alert on errors
```

Each stage is a specialized agent. They communicate via artifacts (compiled binaries, test results). If any stage fails, the pipeline stops. Humans review outputs at checkpoints. This is the pattern we'll teach you to apply to software development.

---

## 2. Core Workflow Patterns

### Pattern A: Sequential Chain

The simplest workflow: work flows through agents in a strict sequence, each consuming the previous agent's output.

```
Agent A (Input)
    ↓
[Process]
    ↓
Output A
    ↓
Agent B (Input: Output A)
    ↓
[Process]
    ↓
Output B
    ↓
Agent C (Input: Output B)
    ↓
[Process]
    ↓
Final Output
```

**When to use:** When steps have strict dependencies and must occur in order.

**Classic examples:**
- Requirements analysis → Architecture design → Implementation → Testing
- Planner agent (break down task) → Coder agent (implement) → Reviewer agent (verify)
- Content writer (draft article) → Editor (refine) → Publisher (format and release)

**How it works:**

1. Agent A receives input, processes it, outputs structured result (file or structured output)
2. Orchestrator (could be you, or another agent) reads Agent A's output
3. Orchestrator passes Agent A's output as input to Agent B
4. Agent B processes and outputs
5. Repeat

**Key advantages:**
- Clear dependencies prevent wasted parallel effort
- Each agent focuses narrowly on its stage
- Easy to debug: failures point to specific stage
- Natural fit for requirements → design → build → test → deploy

**Pitfalls:**
- If Agent A produces malformed output, Agent B fails. Add validation between steps.
- The chain is only as fast as its slowest step; no parallelism benefit.
- Failures require understanding which step broke and why.

Tip: Add a validation step after each agent that checks output structure before handing off.

---

### Pattern B: Orchestrator-Worker

A central orchestrator agent breaks down the task, dispatches work to specialized worker agents, and aggregates results. This pattern is more flexible than sequential chains.

```
                    ┌─→ [Worker: Code Writer] → code.py
                    │
[Orchestrator] ─→ [Decompose Task]
                    │
                    ├─→ [Worker: Test Writer] → test.py
                    │
                    └─→ [Worker: Doc Writer] → README.md

                         (Orchestrator gathers all outputs)
                         ↓
                    [Final Artifact]
```

**When to use:** When a task naturally decomposes into independent subtasks that different specialists should handle. When you want flexibility in *how* the work is split.

**Example scenario:** Building a new feature
- Orchestrator reads requirements, breaks into: "Write core logic", "Write tests", "Update docs"
- Sends each to a specialized worker
- Collects all outputs, ensures consistency, produces final artifact

**How the orchestrator communicates:**

Workers are typically subagents spawned by the orchestrator. Communication is explicit via shared files:

```
Orchestrator creates:
  .claude/subtasks/write-code.md
  .claude/subtasks/write-tests.md
  .claude/subtasks/write-docs.md

Worker 1 reads: .claude/subtasks/write-code.md
         writes: src/implementation.py

Worker 2 reads: .claude/subtasks/write-tests.md
         writes: tests/test_implementation.py

Worker 3 reads: .claude/subtasks/write-docs.md
         writes: docs/api.md

Orchestrator reads all outputs, merges and validates
```

**Key advantages:**
- Flexible decomposition: orchestrator decides how to split work
- Specialization: each worker is optimized for one task
- Better context usage: each worker focuses narrowly
- Easier to add/remove workers without rewriting the chain

**Pitfalls:**
- Orchestrator becomes a bottleneck; it must be smart and efficient
- Dependencies between subtasks can emerge (e.g., tests need code first)
- Aggregating results requires careful handling of inconsistencies

Warning: If workers have dependencies (Test Writer needs Code Writer's output), you're not purely parallel. Consider a hybrid: orchestrator identifies dependencies and either sequences them or passes outputs explicitly.

---

### Pattern C: Parallel Fan-Out / Fan-In

A specialized case of orchestrator-worker: the orchestrator splits a task into *independent* parallel subtasks, launches all concurrently, then merges results.

```
                    ┌─→ [Agent 1] → result_1
                    │
[Orchestrator] ─→ [Fan Out]
                    │
                    ├─→ [Agent 2] → result_2
                    │
                    └─→ [Agent 3] → result_3

                    [Fan In / Merge Results]
                         ↓
                    [Final Output]
```

**When to use:** When you have independent work that can be parallelized. The wall-clock time goes from `3x agent_time` to roughly `agent_time` (ignoring overhead).

**Real-world example:** Code review at scale.
- Orchestrator: "Review this codebase for: performance issues, security vulnerabilities, code style"
- Agent 1 (Performance): Analyzes for bottlenecks
- Agent 2 (Security): Looks for security issues
- Agent 3 (Style): Checks code conventions
- All three run in parallel, results merged into one comprehensive review

**Another example:** Testing across modules.
- Orchestrator: "Test these 4 modules"
- Agents 1-4 each test one module in parallel
- Results merged: "2 modules pass, module 3 has 5 failures, module 4 has 1 warning"

**Key advantages:**
- Massive speedup: running 5 agents in parallel is nearly 5x faster
- Agents remain independent; easy to add more parallel workers
- Results merge cleanly if structured consistently

**Pitfalls:**
- Merge logic must handle inconsistencies (same file reviewed by 2 agents → conflicts)
- If one agent is slow, all others must wait (bottleneck on slowest task)
- Hard to debug cross-agent issues (which agent reported what?)

---

### Pattern D: Feedback Loop (Generate → Test → Fix)

A single agent or small team loops until output meets a success criterion. Common in TDD, iterative refinement, and validation pipelines.

```
[Agent: Generator]
    ↓
[Produce Output]
    ↓
[Agent: Validator]
    ↓
[Check: Does output pass?]
    ↓ (No) ← Feedback Loop
    ↓ (Yes)
[Success: Output approved]
```

**When to use:** When you have a clear success criterion but the path to success is uncertain. When iterative refinement is cheaper than getting it right the first time.

**Real-world examples:**
- TDD: write test → implement → run test → fix implementation → repeat until green
- Prompt engineering: generate response → evaluate quality → refine prompt → repeat
- Code generation: generate code → run linter → fix style issues → repeat

**Key advantages:**
- Natural for TDD and iterative development
- Success is objective (tests pass, criteria met)
- Agents learn from feedback; loops often converge quickly
- Reduces need for perfect instructions upfront

**Pitfalls:**
- Can loop forever if criterion is unreachable or too strict
- Each iteration costs API tokens and time
- Need explicit retry limits and circuit breakers

---

### Pattern E: Git Worktrees for True Parallel Isolation

When agents must modify the same codebase simultaneously without conflicts, use git worktrees to give each agent its own isolated branch.

```
Main Repository (main branch)
    ↓
git worktree add ../feature-auth -b feature/auth
    ↓
[Session 1: Agent in feature-auth worktree]
    ↓
[Modifies code in isolated branch]
    ↓
[When complete: git merge feature/auth back to main]
```

**When to use:** When you need true parallel agent work on the same codebase without git conflicts. Each agent gets its own branch and can commit independently.

**Real-world scenario:**
- Two agents building different features simultaneously
- Each works in its own worktree branch
- No merge conflicts because they never touch the same files
- When both finish, merge both branches sequentially into main

**Key advantages:**
- Complete isolation: agents don't see each other's uncommitted changes
- Git handles merging automatically if files don't conflict
- Safe for long-running parallel tasks
- Natural fallback to conflict resolution if overlaps occur

---

## 3. Hands-On Exercises

The following exercises teach workflow patterns through Claude Code agents. Each exercise builds real agents that coordinate via shared files or git. **Critical rule:** Every exercise must work from a completely empty git repo. Any code the exercise works on is created as part of the exercise setup steps.

### Exercise 4-A: Sequential Chain — Planner → Coder → Reviewer

**Goal:** Build a three-agent chain where each agent picks up where the previous one left off, communicating via files.

**Setup:** Create a new feature stub that the chain will implement. Create `src/search.rb`:

```ruby
# Search module — finds tasks matching a query string
# TODO: implement search_tasks(query)
# Should search task titles and descriptions (case-insensitive)
# Returns array of matching tasks, empty array if none found
```

Now create the three agents:

Create `.claude/agents/feature-planner.md`:

```markdown
---
name: feature-planner
description: Plans a feature implementation and writes the plan to .claude/current-plan.md. Use as the first step before any coding work.
tools: Read, Write, Glob
model: sonnet
---

You create implementation plans. When asked to plan a feature:
1. Read the relevant stub or description
2. Write a plan to `.claude/current-plan.md` using exactly this structure:

## Feature: [name]
## Goal
[one paragraph — what this does and why]
## Files to Create
- [path]: [what goes in it]
## Files to Modify
- [path]: [what changes and why]
## Implementation Steps
1. [specific step]
2. [specific step]
3. [specific step]
## Tests to Write
- [test description]
- [test description]
## Risks
- [risk]: [mitigation]

Do not write any code. Plan only.
```

Create `.claude/agents/feature-coder.md`:

```markdown
---
name: feature-coder
description: Implements a feature by following the plan in .claude/current-plan.md. Always reads the plan first. Use after feature-planner has run.
tools: Read, Write, Bash, Glob
model: sonnet
---

You implement planned features.
1. Read `.claude/current-plan.md` — this is your specification
2. Follow the plan exactly, step by step
3. After each file is written, confirm: "Step [n] complete: [what you did]"
4. When done, append to `.claude/current-plan.md`:

## Implementation Notes
- [what you built]
- [any deviations from plan and why]
- [anything the reviewer should pay attention to]
```

Create `.claude/agents/feature-reviewer.md`:

```markdown
---
name: feature-reviewer
description: Reviews a completed implementation against its original plan. Use after feature-coder has run.
tools: Read, Glob, Grep
model: sonnet
---

You review implementations against plans.
1. Read `.claude/current-plan.md` (including the Implementation Notes at the bottom)
2. Read all files that were created or modified
3. Check:
   - Does the code match the plan?
   - Are there any obvious bugs?
   - Were the planned tests written?
   - Is there anything missing?
4. Write a verdict to `.claude/review-result.md`:

## Review Result: [APPROVED / NEEDS CHANGES / REJECTED]
### What's good
- [finding]
### Issues found
- [issue]: [specific fix needed]
### Verdict
[one sentence summary]
```

In Claude Code, run the chain step by step:

```
Use the feature-planner to plan the implementation of search_tasks in src/search.rb
```

Read `.claude/current-plan.md`. Then:

```
Use the feature-coder to implement the plan in .claude/current-plan.md
```

Then:

```
Use the feature-reviewer to review the implementation
```

**Observe:** Each agent picks up state from the files left by the previous one. Read all three output files to see the complete audit trail.

**What to experiment with:**
- Read `.claude/current-plan.md` after the planner runs — is it specific enough you could implement it yourself?
- After the reviewer flags an issue, ask the coder to fix only that specific issue
- Add a 4th agent `.claude/agents/feature-documenter.md` that writes a docstring summary to a `docs/` folder

---

### Exercise 4-B: Parallel Fan-Out — Three Agents, One Report

**Goal:** Run three specialist review agents simultaneously and merge their findings into a single document.

**Setup:** Create a file with different types of issues for each reviewer to find. Create `src/invoice_generator.rb`:

```ruby
# Invoice generator — has performance, style, and security issues intentionally

def generate_invoices(customer_ids)
  invoices = []
  customer_ids.each do |id|
    # N+1 query pattern — performance issue
    customer = DB.query("SELECT * FROM customers WHERE id = #{id}")
    orders = DB.query("SELECT * FROM orders WHERE customer_id = #{id}")

    # Method is 40+ lines — style issue
    total = 0
    tax = 0
    discount = 0
    orders.each do |o|
      items = DB.query("SELECT * FROM order_items WHERE order_id = #{o['id']}")  # another N+1
      items.each do |item|
        total += item['price'] * item['quantity']
        if item['taxable'] == 1
          tax += item['price'] * item['quantity'] * 0.1
        end
        if customer['loyalty_years'].to_i > 3
          discount += item['price'] * item['quantity'] * 0.05
        end
      end
    end

    # No authorization — any caller can generate any customer's invoice
    # SQL injection via string interpolation — security issue
    invoices << { customer: customer, total: total + tax - discount }
  end
  invoices
end
```

Create `.claude/agents/performance-reviewer.md`:

```markdown
---
name: performance-reviewer
description: Reviews code for performance problems: N+1 queries, missing indexes, inefficient loops. Use for performance audits.
tools: Read
model: haiku
---

Find the top performance issues in the given file. For each:
- **Issue:** what the problem is
- **Location:** line number
- **Impact:** why this is slow at scale
- **Fix:** the specific change needed

Focus on database query patterns first. Limit to 5 findings maximum.
```

Create `.claude/agents/style-reviewer.md`:

```markdown
---
name: style-reviewer
description: Reviews code for style, naming, and readability issues. Use for style audits.
tools: Read
model: haiku
---

Find the top style and readability issues in the given file. For each:
- **Issue:** what the problem is
- **Location:** line number
- **Fix:** specific improvement

Check for: methods over 20 lines, poor naming, deep nesting, magic numbers, missing comments. Limit to 5 findings.
```

In Claude Code:

```
Run the performance-reviewer and style-reviewer in parallel on src/invoice_generator.rb, then compile all findings into a single file at .claude/audit-report.md, grouped by category and sorted by severity.
```

**Observe:** Claude delegates to both agents. Each runs independently with its own context. The orchestrating Claude collects their outputs and writes the merged report.

**What to experiment with:**
- Open `.claude/audit-report.md` and count how many issues were found across both agents
- Run just one reviewer without the other — is the merged report better than any single one?
- Add a 3rd reviewer: create `.claude/agents/security-reviewer.md` that checks for SQL injection, hardcoded secrets, and missing validation

---

### Exercise 4-C: The Feedback Loop — Generate, Test, Fix, Repeat

**Goal:** Build an agent that loops until its code passes tests — entirely autonomously.

**Setup:** Create a stub function with failing tests. Create `src/roman_numerals.rb`:

```ruby
# Converts an integer (1-3999) to a Roman numeral string
# Examples: 1 => "I", 4 => "IV", 9 => "IX", 14 => "XIV", 1994 => "MCMXCIV"
# Returns nil for out-of-range input
def to_roman(number)
  # TODO: implement
  nil
end
```

Create `test/roman_numerals_test.rb`:

```ruby
require 'minitest/autorun'
require_relative '../src/roman_numerals'

class RomanNumeralsTest < Minitest::Test
  def test_simple_values
    assert_equal "I", to_roman(1)
    assert_equal "V", to_roman(5)
    assert_equal "X", to_roman(10)
  end

  def test_subtractive_notation
    assert_equal "IV", to_roman(4)
    assert_equal "IX", to_roman(9)
    assert_equal "XL", to_roman(40)
    assert_equal "XC", to_roman(90)
  end

  def test_complex_numbers
    assert_equal "XIV", to_roman(14)
    assert_equal "XLII", to_roman(42)
    assert_equal "MCMXCIV", to_roman(1994)
  end

  def test_edge_cases
    assert_nil to_roman(0)
    assert_nil to_roman(4000)
    assert_nil to_roman(-1)
  end
end
```

Create `.claude/agents/tdd-implementer.md`:

```markdown
---
name: tdd-implementer
description: Implements a stubbed function until all existing tests pass. Runs tests and self-corrects. Use when you have a stub and a test file.
tools: Read, Write, Bash
model: sonnet
---

Your job: make all tests pass. Your loop:

1. Read the stub file and the test file
2. Write an implementation
3. Run: `ruby test/roman_numerals_test.rb`
4. If tests fail:
   - Read every failure message carefully
   - Identify the specific case your implementation gets wrong
   - Fix only that issue, don't rewrite everything
   - Run again
5. Repeat up to 5 times
6. Report: tests passing, iterations needed, final implementation approach

Rules:
- Never modify the test file
- Use only Ruby standard library
- Stop and explain if you cannot pass all tests after 5 attempts
```

In Claude Code:

```
Use the tdd-implementer to implement the to_roman function.
```

**Observe:** The agent writes an implementation, runs tests, reads failure output, fixes specific cases (likely IV, IX subtractive notation fails first), and iterates until all 12 tests pass.

**What to experiment with:**
- After it succeeds, add a new failing test: `assert_equal "MMXXIV", to_roman(2024)` — ask the agent to make it pass
- Change `model: haiku` — does it solve Roman numerals correctly, or does it need more iterations?
- Break the implementation deliberately (add a bug back) and ask the agent to fix it from the test failures

---

### Exercise 4-D: Orchestrator + Workers

**Goal:** Build an orchestrator that decomposes a task into briefs and delegates each to a specialist.

**Setup:** No new code files needed — the orchestrator will plan new functionality for the project.

Create `.claude/agents/project-orchestrator.md`:

```markdown
---
name: project-orchestrator
description: Breaks a complex feature request into subtask briefs for specialist agents. Use to plan large features before implementation starts.
tools: Read, Write, Glob
model: sonnet
---

You plan complex work. You NEVER write code — you write briefs for other agents.

When given a feature request:
1. Read the current project structure (use Glob to understand what exists)
2. Break the feature into 3-5 focused subtasks
3. For each subtask, write a brief to `.claude/subtasks/[subtask-name].md`:

---
## Subtask: [name]
## What to Build
[specific description — clear enough for a junior developer]
## Files to Create or Modify
- [path]: [what changes]
## Inputs
[what this subtask needs from other subtasks, if anything]
## Outputs
[what this subtask produces for other subtasks]
## Success Criteria
- [checkable condition]
- [checkable condition]
## Recommended Agent
[suggest: feature-coder, test-writer, security-reviewer, or other]
---

4. After writing all briefs, list the subtasks in execution order with dependencies noted.
```

In Claude Code:

```
Use the project-orchestrator to break down this feature: "Add user registration and login to the task manager — users should be able to sign up with email and password, log in, and only see their own tasks"
```

**Observe:** The orchestrator reads the current project, identifies what exists, and produces separate brief files in `.claude/subtasks/`. Read each one — they should be specific enough to hand to a developer or agent.

**What to experiment with:**
- Pick one brief from `.claude/subtasks/` and ask the feature-coder to implement it
- Ask the orchestrator: "Which subtask is most risky and why?"
- Ask it to produce briefs for a feature in your real project — evaluate whether the briefs are actually useful

---

### Exercise 4-E: Git Worktrees — True Parallel Isolation

**Goal:** Run two agents simultaneously on separate branches with zero risk of conflict.

**Setup:** Your project needs at least one commit first. In your terminal:

```bash
cd claude-training
git add -A
git commit -m "initial project setup"
```

Now create two isolated worktrees:

```bash
git worktree add ../claude-training-search -b feature/search
git worktree add ../claude-training-auth -b feature/auth
```

Open **two terminal windows**. In terminal 1:

```bash
cd ../claude-training-search
claude
```

In terminal 2:

```bash
cd ../claude-training-auth
claude
```

In terminal 1, type:

```
Use the tdd-implementer to fully implement the search_tasks function in src/search.rb, writing tests in test/search_test.rb
```

In terminal 2, type (simultaneously):

```
Use the tdd-implementer to implement a validate_email function in src/string_utils.rb that checks if a string is a valid email address, with tests in test/email_test.rb
```

Both agents run simultaneously on isolated codebases. Let them finish.

Back in the main terminal:

```bash
cd claude-training
git merge feature/search
git merge feature/auth
git worktree remove ../claude-training-search
git worktree remove ../claude-training-auth
```

**Observe:** Both agents worked in parallel without any file conflicts. The merge is clean because they touched different files on different branches.

**What to experiment with:**
- Intentionally assign both agents to modify `src/app.rb` — observe the merge conflict and resolve it manually
- Add a third worktree and run a third agent in parallel
- Add `.claude/agents/merge-helper.md` that reads a conflict and suggests how to resolve it

---

## 4. Patterns in Practice: Common Scenarios

### Scenario 1: Adding a Complex Feature

**Workflow:** Orchestrator-Worker + Sequential Chain

1. Use project-orchestrator to break down the feature (e.g., "add payments")
2. Orchestrator writes subtask briefs
3. For each subtask, use feature-planner → feature-coder → feature-reviewer chain
4. Orchestrator collects all reviewed work and writes final summary

**Benefits:**
- Orchestrator ensures all subtasks are covered
- Each subtask gets planning, implementation, and review
- Clear handoffs and traceability

---

### Scenario 2: Code Quality Audit

**Workflow:** Parallel Fan-Out

1. Use performance-reviewer, style-reviewer, security-reviewer in parallel
2. Each scans the codebase independently
3. Compile results into one audit report
4. Triage findings by severity and assign to sprints

**Benefits:**
- Audit runs fast (parallel, not sequential)
- Specialized agents catch different issues
- Results are comprehensive and prioritized

---

### Scenario 3: Prototyping with TDD

**Workflow:** Feedback Loop

1. Use tdd-agent to build a new function
2. Agent writes tests, implements, refactors until tests pass
3. Code is production-ready when agent declares COMPLETE

**Benefits:**
- Tests drive design
- Implementation is validated by tests
- Refactoring is safe

---

### Scenario 4: Large Parallel Feature Sprint

**Workflow:** Git Worktrees + Sequential Chains

1. Create worktree for each feature
2. In each worktree, run feature-planner → feature-coder → feature-reviewer
3. When all features are done, merge worktrees sequentially into main
4. Resolve any conflicts that arise

**Benefits:**
- Teams work in parallel without blocking each other
- Each feature gets full planning and review
- Git handles merge coordination

---

## 5. State Management and Communication

### Shared Files as Communication

Agents communicate by reading and writing shared files (in `.claude/` or your working directory):

```
agent-1 writes: .claude/plan.md
     ↓
agent-2 reads: .claude/plan.md
     writes: .claude/implementation.md
     ↓
agent-3 reads: .claude/implementation.md
     writes: .claude/review.md
```

**Advantages:**
- Explicit handoffs, easy to trace
- Humans can read intermediate artifacts
- No special coordination framework needed

**Pitfalls:**
- Files can become stale; agents must read current version
- File locking issues if two agents write simultaneously
- Large files are slow to parse

### State Tracking

Use `.claude/state.md` or similar to track workflow progress:

```markdown
## Workflow State

### Plan
- Status: READY
- File: .claude/current-plan.md
- Last updated: 2026-03-31T10:00:00Z

### Implementation
- Status: IN_PROGRESS
- Agent: feature-coder
- Last updated: 2026-03-31T10:05:00Z

### Review
- Status: PENDING
- Agent: feature-reviewer
- Last updated: [not started]
```

Agents can check this state before proceeding, ensuring they don't run before dependencies are ready.

---

## 6. Error Handling and Retries

### Validation Between Handoffs

After each agent produces output, validate before handing off:

1. **Check file existence**: Does the expected output file exist?
2. **Check structure**: Is JSON valid? Does it match schema?
3. **Check content**: Does it have required fields?

If validation fails, retry or escalate:

```
Agent A produces malformed output
    ↓
Validation fails
    ↓
Log error to .claude/errors.md
    ↓
Option 1: Retry Agent A with corrected prompt
Option 2: Escalate to human review
Option 3: Use fallback agent
```

### Timeout and Circuit Breakers

For feedback loops, set limits:

```
Iteration 1: Try implementation, run tests
Iteration 2: Tests fail, fix and retry
Iteration 3: Tests fail again, fix and retry
Iteration 4: Tests fail again, fix and retry
Iteration 5: Tests fail again, GIVE UP (circuit breaker)
    ↓
Report to human: "TDD failed after 5 iterations, needs manual review"
```

---

## 7. Debugging Multi-Agent Workflows

When a workflow fails, trace the issue systematically:

1. **Identify the failing agent** by reading status files (`.claude/state.md`, agent-specific notes)
2. **Check the input** the agent received (previous agent's output)
3. **Review the agent's logs** or output
4. **Reproduce in isolation** by running the agent with the same input
5. **Fix the agent** or the input, then retry the workflow

**Common issues:**
- Previous agent produced unexpected output format → add validation step
- Agent ran out of context → split the task smaller
- Agent hit rate limits → add delays between agents
- Agent produced code with syntax errors → add linting step before next agent

---

## 8. Cost Optimization

Running multiple agents costs more than a single agent. Optimize:

1. **Use haiku for simple tasks** (style checking, basic validation)
   Use sonnet for complex tasks (planning, implementation)

2. **Cache common contexts** — if two agents both need to understand the same codebase, cache it

3. **Parallelize when possible** — fan-out saves time and can reduce total API cost

4. **Limit feedback loops** — too many iterations waste tokens. Cap at 3-5 attempts.

5. **Measure and monitor** — log token usage per agent, identify wasteful patterns

---

## 9. When NOT to Use Multi-Agent Workflows

Agentic workflows add complexity. Stick with a single agent if:

- **The task is small and well-defined** (fits in one context window, 5K tokens or less)
- **Latency matters** (each handoff adds 1-2 seconds)
- **You're prototyping** (workflows slow down iteration)
- **The task is inherently sequential** with no parallelism benefit (e.g., reading docs → writing summary)

---

## 🔗 Ecosystem Connection

The workflow patterns in this chapter are at the heart of what these repos provide as production-ready systems:

| Repo | What it adds to this chapter |
|---|---|
| [obra/superpowers](https://github.com/obra/superpowers) | The complete 7-phase workflow (Brainstorm → Worktrees → Plan → Execute → TDD → Review → Complete) is exactly the orchestrator-worker + feedback-loop pattern from this chapter, pre-built and enforced as a plugin |
| [affaan-m/everything-claude-code](https://github.com/affaan-m/everything-claude-code) | `/multi-plan`, `/multi-execute`, and `/orchestrate` commands implement the parallel fan-out pattern; 30 agents implement the specialist-worker pattern |
| [SuperClaude-Org/SuperClaude_Framework](https://github.com/SuperClaude-Org/SuperClaude_Framework) | Orchestration behavioral mode and the PM Agent coordinate complex multi-agent workflows; `/sc:implement` uses sub-agent delegation internally |

**Quick start:** `/plugin install superpowers@claude-plugins-official` — then type `Use the superpowers workflow to implement [any small feature]` and watch it run the complete 7-phase pattern you built manually in this chapter.

> See **[Chapter 7 → Section 2](chapter-07-ecosystem.md)** for full Superpowers exercises and **[Chapter 7 → Section 3](chapter-07-ecosystem.md)** for everything-claude-code multi-agent commands.

---

## Chapter Summary & What's Next

You now understand the core patterns for multi-agent workflows:

- **Sequential chains** for strict dependencies
- **Orchestrator-worker** for flexible decomposition
- **Parallel fan-out** for independent parallel tasks
- **Feedback loops** for iterative refinement
- **Git worktrees** for safe parallel development

These patterns combine. A large project might use:
- Orchestrator to decompose into features
- For each feature: planner → coder → reviewer chain
- Across features: git worktrees for parallel isolation
- Performance/security/style agents run in parallel

### Applied to Real Development

Imagine building a SaaS app with authentication, payments, and search:

1. **Day 1:** Use orchestrator to break into 3 features (auth, payments, search)
2. **Day 1-2:** Create 3 git worktrees, one per feature
3. **In each worktree:** Use planner → coder → reviewer chain
4. **Each review:** Use parallel performance/security/style reviewers
5. **End of Day 2:** Merge all worktrees, resolve any conflicts, deploy
6. **Day 3+:** If bugs found, use TDD-agent to fix with tests first

This workflow scales from 1 person to a team of 50, all coordinated via agents and files.

---

## Appendix: Quick Reference

### Agent File Locations

```
.claude/agents/feature-planner.md
.claude/agents/feature-coder.md
.claude/agents/feature-reviewer.md
.claude/agents/performance-reviewer.md
.claude/agents/style-reviewer.md
.claude/agents/security-reviewer.md
.claude/agents/tdd-agent.md
.claude/agents/project-orchestrator.md
```

### Shared State Files

```
.claude/current-plan.md          (feature-planner output)
.claude/implementation-notes.md  (feature-coder output)
.claude/review-verdict.md        (feature-reviewer output)
.claude/performance-review.md    (performance-reviewer output)
.claude/style-review.md          (style-reviewer output)
.claude/security-review.md       (security-reviewer output)
.claude/tdd-state.md             (tdd-agent state)
.claude/audit-report.md          (aggregated review results)
.claude/subtasks/*.md            (orchestrator output)
```

### Git Worktree Commands

```bash
# Create worktree
git worktree add ../feature-X -b feature/X

# List worktrees
git worktree list

# Clean up
git merge feature/X
git worktree remove ../feature-X
```

### Tips for Success

1. **Always validate between handoffs** — malformed output breaks the next agent
2. **Use shared files as your communication protocol** — explicit and debuggable
3. **Add state tracking** — it helps you understand workflow progress
4. **Test agents in isolation first** — debug a single agent before coordinating many
5. **Keep agents focused** — one job per agent, well-defined outputs
6. **Parallelize when independent** — fan-out saves time and money
7. **Use git worktrees for true parallel isolation** — agents can modify same files safely
8. **Monitor and log** — track which agent failed and why
9. **Set retry limits** — feedback loops can loop forever if not capped
10. **Start small, scale up** — master sequential chains before adding parallelism
