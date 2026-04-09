# Chapter 6: Cost Management — Running Agentic Systems Efficiently in Production

## Introduction

You've learned to build intelligent agents, compose them into workflows, and manage context carefully. Now comes the reality: **in production, every token costs money**. And unlike traditional software, where you pay for compute once per deployment, agentic systems incur costs per request, per token, every single day.

This chapter is about something many teams gloss over: running agentic systems profitably. Not cheaply — profitably. There's a crucial difference. A $0.10 API call that saves an engineer 20 minutes of work is a win. A $0.001 optimization that broke your code and cost you 2 hours of debugging time is a loss.

By the end of this chapter, you'll understand:
- How Claude's pricing works and why output tokens cost more than input tokens
- A decision framework for choosing between Haiku, Sonnet, and Opus at each step
- Techniques like prompt caching that can reduce your costs by 80-90%
- How to measure and monitor costs in real-world workflows
- When to optimize and when to leave well enough alone

Let's start with the fundamentals.

---

## 1. Understanding Claude's Pricing Model

### The Token Economy

Claude is priced per token — both input and output tokens. This is different from per-request pricing. A 1-token response and a 10,000-token response cost very differently.

**Quick math:**
- Input tokens: the tokens you send to Claude (your prompt + context)
- Output tokens: the tokens Claude generates in response
- **Output tokens cost 3-5x more than input tokens** on most models (more on why later)

### Current Pricing (March 2026)

Always check [anthropic.com/pricing](https://anthropic.com/pricing) for the latest rates, but here's the current baseline:

| Model | Input | Output |
|-------|-------|--------|
| Claude Haiku 3.5 | ~$0.80/MTok | ~$4.00/MTok |
| Claude Sonnet 4.5/4.6 | ~$3.00/MTok | ~$15.00/MTok |
| Claude Opus 4 | ~$15.00/MTok | ~$75.00/MTok |

*MTok = million tokens. So $0.80/MTok means $0.000000800 per token.*

### Understanding the Numbers

A "million tokens" (MTok) is approximately:
- 750,000 words
- 4 large PDF documents
- 10,000 lines of code

### Cost Calculation Examples

Let's ground this in reality. Here's a typical 10-turn agent session using Sonnet:

```
Turn 1: Agent receives 2,000 tokens of context + prompt
        Agent outputs 300 tokens
        Cost: (2,000 × $3/MTok + 300 × $15/MTok) / 1M = $0.0081

Turn 2-10: Similar pattern
           Average input: 1,500 tokens (context shrinks as goals narrow)
           Average output: 250 tokens
           Cost per turn: (1,500 × $3 + 250 × $15) / 1M = $0.0059

Total cost for 10-turn workflow:
  Turn 1: $0.0081
  Turns 2-10 (9 × $0.0059): $0.0531
  **Total: ~$0.061 per workflow execution**
```

For context: that's the cost of running a sophisticated analysis on a customer's code in less than 3 seconds. The engineer's time cost to do this manually: $50+. **The ROI is obvious.**

### Prompt Caching and Discount Pricing

Here's where it gets interesting. Claude supports **prompt caching** — you can cache the beginning of your prompt and reuse it across multiple requests at **~10% of the original cost** for cached tokens.

```
Standard request with 10,000-token context:
  Cost = 10,000 × $3/MTok ÷ 1M = $0.03

Cached request (same 10,000-token context reused):
  Cost = 10,000 × $0.30/MTok ÷ 1M = $0.003
  Savings: 90%
```

This is a game-changer for workflows that process the same large context repeatedly.

### Why Output Tokens Cost More

Output tokens cost 3-5x more than input tokens because:

1. **Inference cost**: generating tokens is computationally more expensive than reading them
2. **Uncertainty**: you control input length; output length depends on the task difficulty
3. **Quality**: you want Claude to think, not rush; longer outputs often signal deeper reasoning

Understanding this explains our optimization strategy: **reducing output token count is more impactful than reducing input tokens.**

---

## 2. Model Selection Strategy — The Intelligence Ladder

The single biggest lever for cost is **choosing the right model for each task**. This is where most teams waste money.

The wrong approach: use Sonnet for everything because it's your "production model."

The right approach: pick the least capable model that can reliably do the job, and spend the savings on better prompts and caching for tasks that truly need it.

### Claude Haiku 3.5 — The Workhorse

**Best for:** routing decisions, classification, simple summarization, lightweight orchestration, format conversion, straightforward Q&A

**Cost:** ~$0.80 input / ~$4.00 output (10x cheaper than Sonnet on input tokens)

**Speed:** fastest (lowest latency) — often responds in < 300ms

**Capabilities:**
- Reliable classification and routing
- Basic code analysis (finding where something happens)
- Structured data transformation
- Simple arithmetic and logic
- Following precise instructions with clear format specs

**Limitations:**
- Struggles with nuanced reasoning about code architecture
- Weak at complex multi-step problem solving
- Less reliable at novel tasks (things it wasn't trained thoroughly on)
- Can miss subtle patterns or edge cases

**Rule of thumb:** "If a competent junior engineer could do it in 30 seconds without thinking hard, use Haiku."

**Example use cases:**
```
✅ Use Haiku for these:
- Extract function signatures from a file
- Classify a GitHub issue into 5 categories
- Convert YAML to JSON
- Summarize a 500-word email into 1-sentence
- Route a customer request to the right team

❌ Don't use Haiku for these:
- Design a system architecture
- Write a complex financial calculation
- Debug a race condition
- Generate production code from requirements
```

### Claude Sonnet 4.5/4.6 — The Sweet Spot

**Best for:** code generation, code review, complex multi-step reasoning, most development tasks, nuanced analysis

**Cost:** ~$3.00 input / ~$15.00 output

**Speed:** balanced (typically 500ms-2s)

**Capabilities:**
- Generates production-quality code
- Sophisticated code review with context
- Complex reasoning chains
- Handles ambiguous requirements well
- Excellent instruction following
- Understands system design trade-offs

**Limitations:**
- More expensive than Haiku (but still affordable)
- Not required for routine classification or summarization
- Not optimal for extremely difficult reasoning (that's Opus's job)

**Rule of thumb:** "If you're building production code, doing code review, or need reliable multi-step reasoning, use Sonnet. It's the default."

**Example use cases:**
```
✅ This is Sonnet's bread and butter:
- Write a new feature in an existing codebase
- Review a pull request with security and performance review
- Generate test cases from a function spec
- Refactor legacy code
- Analyze a bug from a stack trace and suggest fixes

⚠️ Could be Sonnet, might need optimization:
- Summarize a 50-page architecture document (maybe use Haiku after Sonnet)
- Classify 1,000 support tickets (use Haiku in a loop instead)
```

### Claude Opus 4 — The Expert

**Best for:** architectural decisions, novel/ambiguous problems, research synthesis, principal-engineer-level reasoning, problems where cost is secondary to correctness

**Cost:** ~$15.00 input / ~$75.00 output (5x more expensive than Sonnet)

**Speed:** slowest (2-4s typical)

**Capabilities:**
- Exceptional reasoning on novel/ambiguous problems
- Architectural thinking and trade-off analysis
- Research synthesis and gap identification
- Highest accuracy on complex logic
- Best at learning new domains quickly

**Limitations:**
- Expensive — use sparingly
- Often slower (less suitable for real-time interactions)
- Overkill for routine tasks

**Rule of thumb:** "If you'd call in a principal engineer for advice, use Opus. If you wouldn't, you don't need it."

**Example use cases:**
```
✅ Worth the Opus cost:
- "Design a caching strategy for our distributed system"
- "Should we migrate from PostgreSQL to DuckDB? Show trade-offs."
- "Review the architectural implications of this change"
- "Identify systemic issues in this codebase"

❌ Wasting money:
- Classifying an issue
- Summarizing a document
- Simple code generation
- Format conversion
```

### The Tiered Model Pattern

**This is the core optimization strategy.** Use different models for different tasks in your workflow.

#### Pattern: Orchestration + Execution + Summarization

```
User request
    ↓
[Haiku] Route/Plan (what needs doing?)
    ↓
[Sonnet] Execute (do the work)
    ↓
[Haiku] Format/Summarize (prepare the response)
    ↓
Response
```

**Cost breakdown for a typical feature request:**
- Route with Haiku: 500 input, 100 output = $0.00041
- Execute with Sonnet: 3,000 input, 800 output = $0.01410
- Summarize with Haiku: 1,500 input, 200 output = $0.00122
- **Total: ~$0.0157 per workflow**

If you'd used Sonnet for all three: $0.0234 (49% more expensive for same capability)

> 💡 **Tip:** This tiered approach is more than just cost optimization. It's also **more reliable**. Haiku excels at classification, so you get better routing. Sonnet focuses on the hard work. The quality is often *better* than using one model for everything.

---

## 3. Prompt Caching — The Hidden Gem

Prompt caching is one of the most underused cost optimizations. Many teams don't even know about it.

### How It Works

Claude can cache the beginning portion of your prompt. Subsequent requests can reuse this cached portion at **~10% of the original cost**.

```
Request 1 (cache write):
├─ Cached prompt (10,000 tokens) → $0.03 cost
└─ New content (500 tokens) → $0.0015 cost
Total: $0.0315

Request 2 (cache read, same cached portion):
├─ Cached prompt (10,000 tokens) → $0.003 cost (90% discount!)
└─ New content (600 tokens) → $0.0018 cost
Total: $0.0048

Per-request savings: 85%
```

### What Gets Cached?

You mark a point in your prompt with a cache control directive. Everything before this point gets cached across subsequent requests with the same prefix.

### Cache TTL and Lifecycle

- **Cache duration:** ~5 minutes (currently in beta; may change)
- **Automatic invalidation:** if the cached prompt changes, a new cache is written
- **Cost:** you pay for cache writes (full price) and cache reads (10% price) separately

### Real-World Impact: Code Review System

Imagine a code review bot that reviews PRs:

```
System prompt (constant): 5,000 tokens
Codebase context (constant): 15,000 tokens
PR diff (varies): 2,000 tokens

Standard approach (100 PRs/day):
  100 × (20,000 × $3 + 500 × $15) / 1M = $6.07/day

With caching (assuming 20 requests within 5-min window):
  First batch (5 mins):
    - Request 1 (cache write): (20,000 × $3 + 500 × $15) / 1M = $0.0615
    - Requests 2-20 (cache read): each costs (2,000 × $3 + 500 × $15) / 1M = $0.0090
    - Batch cost: $0.0615 + (19 × $0.0090) = $0.2325

  Multiple batches (assuming 5 batches/day):
    - Total: ~$1.16/day
    - Savings: 81%
```

> ⚠️ **Warning:** Cache only works for **identical prefix content**. If your system prompt changes per request, caching won't help. Design your prompts with caching in mind.

### When Caching Matters Most

**High impact:**
- Code review systems (stable system prompt, stable codebase context)
- Documentation generators (fixed template, varying input)
- Multi-turn conversations with stable background context
- Batch processing with the same instruction set

**Low/no impact:**
- One-off analysis requests
- Conversations with changing system prompts
- Real-time chat where each message is unique

---

## 4. Prompt Efficiency — Writing Lean Prompts

**Every word in your prompt costs money.** A verbose 5,000-token prompt that could be 2,000 tokens is costing you 150% more than necessary.

### The Cost of Verbosity

```
Verbose prompt: 5,000 tokens × $3/MTok = $0.015 per request
Lean prompt: 2,000 tokens × $3/MTok = $0.006 per request

Difference: $0.009 per request
Over 1,000 requests/month: $9 extra cost
Over a year: $108 wasted on verbosity
```

That's just input cost. If verbosity also makes Claude's output longer (it often does), you waste even more.

### Anti-Patterns to Avoid

#### 1. Redundant Preambles

❌ **Bad:**
```
You are an expert software engineer. You have 20 years of experience.
You understand code deeply. You are thoughtful and careful.
Please analyze this code with care and attention to detail.
```

✅ **Good:**
```
Analyze this code for bugs and security issues.
```

Claude already knows how to reason carefully. You don't need to ask it to be an expert.

#### 2. Repeating Instructions Claude Already Knows

❌ **Bad:**
```
You are a code reviewer. Your job is to review code.
Review code carefully. Look for bugs, security issues, and performance problems.
Be thorough. Check everything.
```

✅ **Good:**
```
Review for bugs, security issues, and performance problems.
```

#### 3. Excessive Examples

❌ **Bad:** 5 examples, each 200 tokens
✅ **Good:** 1 example, 300 tokens (quality over quantity)

**Quality > quantity.** One thoughtful example beats three rushed ones.

#### 4. Hedging and Uncertainty

❌ **Bad:**
```
I'm wondering if perhaps you might consider reviewing this code?
If you think it's appropriate, could you maybe check it for issues?
```

✅ **Good:**
```
Review this code for issues.
```

Claude doesn't need social niceties. Be direct.

### Pro Patterns to Adopt

#### 1. Use XML Tags for Clarity

Instead of explaining what you're giving, use tags:

❌ **Bad (needs explanation):**
```
I have some code here and I want you to review it for performance:

[code block]
```

✅ **Good (tags make it clear):**
```
<code_to_review>
[code block]
</code_to_review>

Focus on performance.
```

This actually saves tokens because Claude doesn't need to re-read confusing prose to understand what you want.

#### 2. Structured Output Format

Instead of "return the results as JSON" (ambiguous, might produce prose), be explicit about the schema.

This prevents Claude from guessing, prevents you from needing follow-ups, and typically produces shorter output.

#### 3. Constraints Instead of Descriptions

❌ **Bad:**
```
Please write a concise summary of the key points from this document.
Make sure to capture the important information.
Keep it brief but comprehensive.
```

✅ **Good:**
```
Summarize in 200 words maximum.
```

### Prompt Comparison: Before and After

**Original Prompt (523 tokens):**
```
You are an expert software engineer with deep knowledge of Python.
Your task is to review the following Python code for potential issues.

Please look for the following kinds of problems:
1. Performance issues — look for inefficient loops, unnecessary allocations, etc.
2. Security issues — look for input validation problems, injection risks, etc.
3. Correctness issues — look for logic errors, edge cases not handled, etc.
4. Style issues — look for code that doesn't follow Python conventions
5. Testing — look for code that isn't testable or lacks test coverage

Consider the context of the code and what it's trying to do.
Be thorough but concise in your analysis.
For each issue, explain what the problem is, why it's a problem, and how to fix it.

Here is the code to review:
[code block - 200 tokens]

Please provide your analysis.
```

**Optimized Prompt (187 tokens):**
```
Review for performance, security, correctness, and style issues.
For each issue, explain the problem, impact, and fix.

<code>
[code block - 200 tokens]
</code>
```

**Savings: 64% of prompt tokens, and the output is likely better (more concise, no preamble).**

### Token-Saving Checklist

- [ ] Removed hedging language ("perhaps", "maybe", "if you don't mind")
- [ ] Removed preambles about what Claude is
- [ ] Condensed examples to one high-quality example
- [ ] Used XML tags instead of prose explanation
- [ ] Specified output format explicitly
- [ ] Removed phrases like "be careful", "be thorough"
- [ ] Cut redundant instructions
- [ ] Used constraints (word limit) instead of requests (be concise)

> 💡 **Tip:** For a year of daily agent runs, shaving 500 tokens off a prompt saves ~$120/year on input costs and often saves more on output (shorter, more focused responses).

---

## 5. Output Token Control

Output tokens cost 3-5x more than input tokens. **Controlling output length is the highest-leverage cost optimization.**

### Strategies to Reduce Output

#### 1. Request Structured Output

Structured output is shorter and more useful.

```
❌ Verbose response (35 tokens):
"The function has a bug: the loop on line 5 doesn't break when
it should. The condition should be `i > 10` instead of `i >= 10`."

✅ Structured response (15 tokens, 57% shorter):
{
  "line": 5,
  "issue": "off-by-one error",
  "current": "i >= 10",
  "fix": "i > 10"
}
```

#### 2. Set Output Length Limits

Use `max_tokens` to set a hard cap on output. Claude stops generating when it hits this limit.

#### 3. Specify Response Format

Instead of "respond naturally", be prescriptive:

❌ **Bad:**
```
Summarize this codebase.
```
Might get 2,000 tokens of flowing prose.

✅ **Good:**
```
Summarize in under 200 words.
Return as: DIRECTORY_STRUCTURE\nFILE_PURPOSES\nKEY_PATTERNS
```

Gets a focused, structured response.

#### 4. Request Diffs, Not Full Files

❌ **Bad:**
```
Refactor this code to use the new API.
[500-line file]
```
Claude might return the entire 500-line file (thousands of tokens).

✅ **Good:**
```
Refactor this code to use the new API.
Return only the lines that changed, as a diff.
[500-line file]
```

Claude returns a 100-token diff instead.

#### 5. Return Only Code/Results, No Explanation

```
❌ Verbose (~300 tokens):
The function should be:
[code]
This works because...

✅ Concise (~40 tokens):
[code]
```

### When NOT to Minimize Output

**Some tasks benefit from longer reasoning:**
- Complex architectural decisions (reasoning > brevity)
- Novel problem solving (let Claude think through it)
- Security-critical code review (thorough > concise)

When the task is genuinely hard, longer output often means better output. **Optimize for correctness first, cost second.**

### Real-World Impact: Code Generation

```
Task: "Generate a Python web scraper"

Verbose approach (no constraints):
  Output: ~1,500 tokens (full explanation + code + gotchas)
  Cost: 1,500 × $15/MTok ÷ 1M = $0.0225

Concise approach:
  Output: ~400 tokens (just code, no explanation)
  Cost: 400 × $15/MTok ÷ 1M = $0.006

Savings per request: $0.0165
Per 100 requests: $1.65
Per 10,000 requests/year: $165
```

That's real money.

---

## 6. Agentic Workflow Cost Optimization

Agentic systems are powerful but can become cost black holes if designed poorly. The issue: **every tool call, every decision, every retry is more tokens**.

### The Compounding Cost Problem

```
Single-turn request: 3,000 input tokens
    ↓
10-turn agentic workflow: 20,000+ input tokens
    ↓
Workflow with retries (avg 2 retries/agent): 40,000+ input tokens
    ↓
Workflow with multiple agents: 60,000+ input tokens
```

A "simple" workflow can balloon to massive cost quickly.

### Strategy 1: Pre-Filter Before Delegation

Don't send 50 files to an agent when 5 are relevant.

Create `.claude/agents/file-selector.md`:
```markdown
---
name: file-selector
description: Identifies relevant files for a task. Uses Haiku for cheap filtering.
tools: Read
model: haiku
---

Given a list of files and a task description, identify which files are relevant.
Return only the file names that matter.
```

**Setup:**
```
Create a file list (e.g., list.txt):
src/user_service.rb
src/auth_service.rb
src/database.rb
src/utils.rb
config/database.yml
config/env.rb
tests/user_service_test.rb
```

Then type:
```
Use the file-selector with this task: "We need to fix a bug in user authentication" and this file list: [read from list.txt]
```

**Observe:** Haiku filters 8 files down to 2 relevant ones. The cheaper filtering call ($0.005) saves massive context for the main agent call.

**What to experiment with:**
- Test how accurately Haiku filters different file types
- Find a task where aggressive pre-filtering saves the most tokens
- Measure: how much does filtered context reduce total workflow cost?

---

## 7. Exercises: Hands-On Cost Management

**Exercise 6-A: Feel the Model Difference**

**Goal:** Calibrate your intuition for cost vs quality by running the same code review task with Haiku, Sonnet, and Opus.

**Setup:** Create a file with a mix of obvious and subtle issues. Create `src/paymentProcessor.ts`:
```typescript
// Payment processing module
const API_KEY = "sk_live_abc123xyz";  // hardcoded production key

async function chargeCustomer(customerId: string, amount: number): Promise<boolean> {
  // No input validation
  // No idempotency key — double-charging possible if retried
  const response = await fetch("https://api.stripe.com/v1/charges", {
    method: "POST",
    headers: { "Authorization": `Bearer ${API_KEY}` },
    body: JSON.stringify({ amount, customer: customerId, currency: "usd" })
  });

  if (response.status === 200) {
    logCharge(customerId, amount);
    return true;
  }
  return false;  // silently swallows all errors
}

function logCharge(id: string, amount: number): void {
  console.log(`Charged ${id}: $${amount}`);  // logs to stdout, not a real logger
  fs.appendFileSync("charges.log", `${id},${amount}\n`);
}
```

Create three reviewer agents:

Create `.claude/agents/haiku-reviewer.md`:
```markdown
---
name: haiku-reviewer
description: Quick code review using Haiku. Fast and cheap. Use for routine reviews.
tools: Read
model: haiku
---
Review the given file. List the top 5 issues with: [SEVERITY] Issue description. Fix: one sentence. Nothing else.
```

Create `.claude/agents/sonnet-reviewer.md`:
```markdown
---
name: sonnet-reviewer
description: Thorough code review using Sonnet. Balanced quality and cost. Default for most reviews.
tools: Read
model: sonnet
---
Review the given file. For each of the top 5 issues: describe the problem, explain why it matters, and give a specific fix. Be thorough.
```

Create `.claude/agents/opus-reviewer.md`:
```markdown
---
name: opus-reviewer
description: Deep code review using Opus. Maximum reasoning depth. Use for critical security or architecture decisions only.
tools: Read
model: opus
---
Review the given file. Think carefully about non-obvious issues: subtle race conditions, security implications, failure modes under load, long-term maintainability risks. Focus on things a less careful reviewer would miss.
```

Run each one in sequence, using `/cost` to measure:
```
/cost
```
```
Use the haiku-reviewer on src/paymentProcessor.ts
```
```
/cost
```
```
Use the sonnet-reviewer on src/paymentProcessor.ts
```
```
/cost
```
```
Use the opus-reviewer on src/paymentProcessor.ts
```
```
/cost
```

**Observe:** Compare what each model finds. Haiku catches the obvious ones (hardcoded key, no validation). Sonnet finds more and explains them better. Opus may surface the idempotency risk and the silent error swallowing — things that only matter at scale.

**What to experiment with:**
- For this payment code, which model do you think is worth paying for? Write your answer in CLAUDE.md as a team guideline
- Find a simple config file or README in the project — run haiku-reviewer on it. Is Haiku perfectly adequate for non-critical files?
- Add this to your CLAUDE.md: "Model guidelines: haiku for formatting/docs/commits, sonnet for code, opus only for payment/auth/security"

---

**Exercise 6-B: The Tiered Workflow**

**Goal:** Build a workflow that routes tasks to the cheapest model that can handle them.

**Setup:** No new code files needed.

Create `.claude/agents/task-router.md`:
```markdown
---
name: task-router
description: Classifies a task and recommends which model tier to use. Use before any significant task to avoid over-spending.
tools: Write
model: haiku
---

Classify the task into a tier and write your decision to `.claude/routing-decision.md`:

**TIER 1 — Use Haiku:**
Commit messages, changelogs, renaming files, formatting code, simple summaries, moving content between files, writing docstrings for obvious functions

**TIER 2 — Use Sonnet (default):**
Implementing features, writing tests, fixing bugs, refactoring, API design, reviewing code, writing documentation for complex features

**TIER 3 — Use Opus (sparingly):**
Security-critical code (auth, payments, encryption), architectural decisions for core systems, diagnosing complex non-obvious bugs, anything where a mistake is very costly to fix

Output format:
## Routing Decision
**Task:** [description]
**Tier:** [1 / 2 / 3]
**Recommended model:** [haiku / sonnet / opus]
**Reason:** [one sentence]
```

Create `.claude/agents/commit-writer.md`:
```markdown
---
name: commit-writer
description: Writes a conventional commit message for staged changes. Tier 1 task — uses Haiku. Fast and cheap.
tools: Bash
model: haiku
---
Run `git diff --staged --stat` to see what changed. Run `git diff --staged` for the actual diff if needed.
Write a commit message following conventional commits: `type(scope): short description`
Types: feat, fix, docs, refactor, test, chore
Output only the commit message. Nothing else.
```

First, stage some changes:
```bash
git add src/paymentProcessor.ts
```

Then in Claude Code:
```
Use the task-router to classify this task: "Write a commit message for my staged changes"
```

Read `.claude/routing-decision.md`. Then:
```
Use the commit-writer to write the commit message.
```
```
/cost
```

Now route a harder task:
```
Use the task-router to classify this task: "Review the payment_processor.rb file for security vulnerabilities"
```

**Observe:** The router (Haiku) costs almost nothing. The commit writer (Haiku) also costs almost nothing. The security review routes to Sonnet or Opus — as it should.

**What to experiment with:**
- Run 5 different tasks through the router — do you agree with all its tier assignments?
- Ask the router to classify "Add error handling to the charge_customer function" — is Tier 2 correct?
- Add the tier guidelines to your CLAUDE.md permanently so the whole team uses them

---

**Exercise 6-C: Write Lean System Prompts**

**Goal:** Measure the token cost of verbose vs lean agent system prompts and develop an eye for waste.

**Setup:** Create two agents with identical purpose but very different system prompt lengths.

Create `.claude/agents/bloated-summarizer.md`:
```markdown
---
name: bloated-summarizer
description: Summarizes a file
tools: Read
model: haiku
---

You are an expert software engineer and technical writer with many years of experience reading and summarizing code across many different programming languages and paradigms. Your deep expertise allows you to quickly understand complex codebases and distill them into clear, concise summaries that are useful for developers of all experience levels.

When you are asked to summarize a file, please make sure to cover all of the following aspects: what the file does at a high level, what the main functions or classes are and what they do, any important patterns or conventions used, any dependencies or imports, and any notable features or issues you observe.

Please write your summary in a clear, well-organized format that would be helpful to a developer who is seeing this file for the first time. Make sure to use plain language and avoid jargon where possible. Be thorough and comprehensive while also being concise and easy to read.
```

Create `.claude/agents/lean-summarizer.md`:
```markdown
---
name: lean-summarizer
description: Summarizes a file
tools: Read
model: haiku
---

Summarize the given file in 3-5 sentences: what it does, its main functions, and anything notable.
```

Run both on the same file:
```
/cost
```
```
Use the bloated-summarizer on src/paymentProcessor.ts
```
```
/cost
```
```
Use the lean-summarizer on src/paymentProcessor.ts
```
```
/cost
```

**Observe:** The lean summarizer costs 40-60% less per call, and the output quality is nearly identical — Haiku doesn't need long instructions to summarize well.

Compare the summaries: is the bloated one actually better? Or does the extra instruction length just add cost without improving output?

**What to experiment with:**
- Rewrite your three reviewer agents (haiku/sonnet/opus) with lean system prompts — measure the savings
- Apply the lean prompt principle to the security-reviewer from Chapter 3 — can you cut it by 30% without losing quality?
- Count the words in each of your `.claude/agents/` files — try to get all system prompts under 100 words

---

**Exercise 6-D: The /cost Habit**

**Goal:** Build a cost-tracking practice so you can catch expensive agents early.

**Setup:** No new files needed — use the agents and files you've built across all chapters.

Create a cost log file: `.claude/cost-log.md`:
```markdown
# Agent Cost Log

Track token usage here to build intuition for what things cost.

| Agent | Task | ΔInput Tokens | ΔOutput Tokens | Notes |
|---|---|---|---|---|
| (example) hello-agent | summarize project | ~800 | ~200 | cheap, good for routing |
```

Now audit your 5 most-used agents. For each one:

```
/cost
```
(note the before number)

Run the agent on a representative task, then:
```
/cost
```
(note the after number, calculate the delta)

Update `.claude/cost-log.md` with the result.

Do this for: hello-agent, security-reviewer, tdd-implementer, feature-planner, and architecture-analyst.

**Observe:** Some agents are much more expensive than expected. Usually it's because:
- Their system prompt is long
- They read many files
- They produce long output

**What to experiment with:**
- Find your most expensive agent and reduce its cost by 25% — either lean the system prompt, add `model: haiku`, or limit the files it reads
- Set a personal rule: "No single agent run should cost more than 5,000 tokens" — which agents violate this?
- Add cost estimates to your agent descriptions: "Typical cost: ~1,500 tokens" so future-you remembers

---

**Exercise 6-E: Design a Cost-Efficient Agent from Scratch**

**Goal:** Build an agent with efficiency as the primary design constraint, not an afterthought.

**Setup:** Make a commit so git history exists to query:
```bash
git add -A
git commit -m "feat: add payment processor and multi-agent workflow examples"
```

Now design a `daily-digest` agent. The naive version would read all changed files — expensive. The efficient version uses git metadata only.

Create `.claude/agents/daily-digest.md`:
```markdown
---
name: daily-digest
description: Generates a brief daily summary of what changed in the last 24 hours. Designed to be cheap — reads only git output, not file contents.
tools: Bash, Write
model: haiku
---

Generate today's digest. Efficiency rules:
- Use ONLY git commands for data — do NOT read any source files
- Keep the entire digest under 150 words

Steps:
1. `git log --since="24 hours ago" --oneline` — get recent commits
2. `git diff --stat HEAD~1..HEAD` (or adjust range based on commit count) — get changed file names
3. Based only on commit messages and file names (not file contents), write `.claude/daily-digest.md`:

---
## Daily Digest — [date]
**Commits today:** [count]
**Files touched:** [count]
**What changed:** [3 bullets, inferred from commit messages]
**Worth a closer look:** [any file names that suggest risky or important changes]
---

If there are no commits in the last 24 hours, write: "No changes today."
```

In Claude Code:
```
/cost
```
```
Use the daily-digest to summarize today's changes.
```
```
/cost
```

**Observe:** This should be very cheap — Haiku model, no file reads, only a few bash commands. You get a useful summary for almost nothing.

Now ask Claude directly without the agent:
```
/cost
```
```
Summarize all the changes made to this project today, reading the relevant files for context.
```
```
/cost
```

**Observe:** The direct approach costs significantly more because it reads file contents.

**What to experiment with:**
- Schedule this agent to run every morning automatically (explore the `schedule` plugin: `/plugin install schedule@claude-plugins-official`)
- Add a `risk-level: LOW/MEDIUM/HIGH` field to the digest based on what kinds of files changed (src/ vs test/ vs config/)
- Modify it to cover the last 7 days — does the cost scale linearly with the number of commits?

---

## 8. Monitoring & Alerting

You can't optimize what you don't measure. Set up proper cost tracking and alerting.

### The `/cost` Command in Claude Code

In Claude Code, you can check API usage with:

```bash
/cost
```

This shows you:
- Tokens used this session
- Estimated cost for this session
- Models used and breakdown by model
- Cost trend (are you spending more or less over time?)

**Use this during development** to catch expensive patterns early.

### Anthropic Console Dashboard

Log into [console.anthropic.com](https://console.anthropic.com) to see:
- Total API usage across all projects
- Breakdown by model
- Trend graphs
- Cost forecasting ("at this rate, you'll spend $X this month")
- Budget alerts and rate limits

### Setting Budget Limits

In Anthropic Console, you can set spending limits:

1. Go to [console.anthropic.com/account/billing/overview](https://console.anthropic.com/account/billing/overview)
2. Click "Set spending limit"
3. Choose your monthly budget
4. Claude will reject requests once you hit the limit

### When You Notice Cost Spikes

If you see an unexpected cost spike:

1. Check your cost log to identify which agent/task is expensive
2. Use Exercise 6-C to lean down the system prompt
3. Consider switching to a cheaper model (Haiku instead of Sonnet)
4. Use pre-filtering to reduce context size
5. Enable prompt caching for repeated operations

---

## 9. Real-World Cost Scenarios

Let's work through realistic examples with actual numbers.

### Scenario A: Code Review Bot

**Workflow:**
1. User opens a PR with code changes
2. Bot extracts the diff
3. Bot reviews with Sonnet for issues
4. Bot formats output with Haiku
5. Bot comments on the PR

**Assumptions:**
- Average PR: 50 lines changed (500 tokens)
- System prompt + guidelines: 2,000 tokens
- Codebase context: 8,000 tokens (can be cached)
- Reviews per day: 20

**Cost calculation without caching:**
```
Per review:
  Input: (2,000 + 8,000 + 500) × $3/MTok = $0.0315
  Output (Sonnet): 400 tokens × $15/MTok = $0.006
  Summary (Haiku): 100 tokens × $4/MTok = $0.0004
  Total: $0.0379

Daily (20 reviews): $0.758
Monthly (500 reviews): $18.95
```

**Cost calculation WITH caching (80% cache hit rate):**
```
Per review (cache hit):
  Input (cached context): (8,000 × $0.30 + 2,500) × $3/MTok / 1M = $0.0090
  Output: same as before ($0.006)
  Total: $0.0150

Per review (cache miss / write):
  Input (full context): (10,000 × $3 + 500 × $3) / 1M = $0.0315
  Output: same as before ($0.006)
  Total: $0.0375

Average (80% hit, 20% miss):
  Cost = (0.80 × $0.0150) + (0.20 × $0.0375) = $0.0195

Daily (20 reviews): $0.390
Monthly (500 reviews): $9.75
Savings: 49% reduction
```

**Conclusion:** Add caching to your code review bot and cut monthly costs nearly in half.

### Scenario B: Batch Classification (100 support tickets)

**Task:** Classify 100 support tickets into categories

**Wrong approach (use Sonnet for each):**
```
100 tickets × 800 tokens per ticket × $15/MTok output = $1.20
```

**Right approach (Haiku + one Sonnet summary):**
```
Step 1: Route-and-classify with Haiku for each ticket
  100 × 400 tokens × $4/MTok = $0.16

Step 2: Summarize patterns with Sonnet
  8,000 tokens (compiled results) × $15/MTok = $0.12

Total: $0.28
Savings: 77% ($0.92 saved)
```

**Conclusion:** For repetitive classification, use Haiku in a loop and Sonnet for summary. Massive savings.

### Scenario C: Caching Impact Over Time

**Setup:** Code review bot, 100 PRs/day, 30-day month

**Month 1 (no optimization):**
```
100 reviews/day × 30 days = 3,000 reviews
3,000 × $0.0379 = $113.70/month
```

**Month 2 (add caching, improve prompt):**
```
Lean prompt: saves 300 tokens of input (~$0.001/review)
Caching at 70% hit rate: saves $0.018/review
New cost per review: $0.0379 - $0.001 - $0.018 = $0.0189

3,000 × $0.0189 = $56.70/month
Savings: $57/month (50% reduction)
Annual: $684
```

This is real cost management that scales.

---

## 🔗 Ecosystem Connection

The cost optimization techniques in this chapter are embedded into these repos at the design level:

| Repo | What it adds to this chapter |
|---|---|
| [SuperClaude-Org/SuperClaude_Framework](https://github.com/SuperClaude-Org/SuperClaude_Framework) | Token-Efficiency behavioral mode reduces token usage by 30-50% on exploratory/iterative work; the framework's smaller footprint leaves more context for actual code |
| [affaan-m/everything-claude-code](https://github.com/affaan-m/everything-claude-code) | The Longform Guide has a dedicated token optimization chapter with background process management, strategic model selection, and context window strategies from months of real usage |
| [obra/superpowers](https://github.com/obra/superpowers) | Git worktree isolation means each subagent gets a fresh context window — preventing context bloat from accumulating across a long workflow |

**Quick start:** After installing SuperClaude, run any exploratory task with `--mode token-efficient` and use `/cost` before and after to measure the difference.

> See **[Chapter 7 → Section 4](chapter-07-ecosystem.md)** for SuperClaude token-efficiency exercises and **[Chapter 7 → Section 3](chapter-07-ecosystem.md)** for everything-claude-code's Longform Guide on tokens.

---

## 10. Summary: Cost Management as a Discipline

Cost management isn't about being cheap — it's about being **deliberate**. Here's the summary:

1. **Model selection is the biggest lever.** Use Haiku for classification, Sonnet for code, Opus for hard decisions. Don't use expensive models for cheap tasks.

2. **Prompt caching is the second-biggest lever.** If you run the same agent multiple times with the same context, cache it. 90% cost reduction for cached tokens is massive.

3. **Lean prompts save continuously.** Every 500 tokens cut from a system prompt saves ~$120/year. This compounds across all your agents.

4. **Output control beats input reduction.** Outputting 200 tokens instead of 2,000 saves more money than reducing input. Be prescriptive about format and length.

5. **Measure before optimizing.** Use `/cost` during development. Build a cost log. You can't optimize what you don't measure.

6. **Design workflows with cost in mind.** Pre-filter files, route with cheap models, cache stable context. Don't let workflows grow unbounded.

7. **When to optimize vs. when to leave it.** If your code review task costs $0.05/PR and saves the engineer 10 minutes, the ROI is obvious. Don't over-optimize tasks where the cost is trivial relative to the value.

The goal isn't to make AI as cheap as possible. The goal is to **make AI more cost-effective than the alternative** — which is usually engineer time. A $0.10 API call that saves 30 minutes of engineer work is a win, every time.
