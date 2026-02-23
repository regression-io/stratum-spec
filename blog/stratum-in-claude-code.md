# Stratum as a Claude Code Execution Runtime

Claude Code is already doing what Stratum formalizes.

When you give Claude Code a task, it decomposes it into steps, calls tools, validates results, retries on failure, tracks a budget of context and attention, and produces typed artifacts. That's an `@infer`/`@compute`/`@flow` execution model. It's just not exposed as one.

This post is about what happens when it is.

---

## The Gap

Claude Code today has four enforcement primitives: Rules, Skills, Hooks, and MCP servers.

- **Rules** are guidance Claude is asked to follow. They're not enforced — Claude can drift.
- **Skills** are structured templates. They shape behavior but don't constrain outputs.
- **Hooks** run shell commands on events. They can block artifacts but not mid-execution state.
- **MCP servers** are tool registries. Claude decides whether to use them.

None of these give you typed inter-step contracts, structured retry with targeted feedback, hard budget enforcement, or a complete audit trail. They're conventions. Stratum is enforcement.

---

## What Stratum Adds to Claude Code

The honest version of this: Stratum as an MCP server is not a security boundary. Claude can ignore MCP tools entirely. What it is: a structured substrate that eliminates *accidental* drift — the kind that happens because Claude is pattern-matching on your CLAUDE.md, not executing a formal spec.

Four things change:

### 1. Plans are typed before anything executes

When Claude starts a non-trivial task, it calls `stratum_plan`. The result is a validated `.stratum.yaml` execution plan — a typed DAG of steps with declared inputs, outputs, and contracts. Claude presents this for review. Execution only begins after approval.

```
stratum_plan("Refactor the auth module and write tests for all edge cases")
→ ExecutionPlan {
    steps: [
      { id: "1", fn: "read_auth_module", output: SourceFile },
      { id: "2", fn: "identify_edge_cases", output: EdgeCaseList, depends_on: ["1"] },
      { id: "3", fn: "refactor", output: RefactoredCode, depends_on: ["1", "2"] },
      { id: "4", fn: "write_tests", output: TestSuite, depends_on: ["3", "2"] },
    ],
    estimated_cost: "12 tool calls",
    estimated_duration: "~4 minutes"
  }
```

For professional developers: this is the typed plan you review before Claude touches code. For vibe coders: this is the plain-language summary that tells you what Claude is about to do.

### 2. Retry is structured, not blind

When a step fails — tests don't pass, a file doesn't validate, an API returns an error — Claude calls `stratum_execute` on that step with the structured failure context. The retry gets only the specific failure, not the full task description again.

The difference in practice: instead of Claude re-reading the entire codebase on every retry, it gets "the test `test_auth_refresh` failed with `AssertionError: expected 401, got 200`" and retries with that context.

### 3. Token budget is visible

Claude Code has no native concept of token budget per task. Context windows fill up. Attention degrades. Steps that should be independent start colliding.

When Claude is operating through Stratum, each step has a declared `budget` that draws from a shared flow envelope. When the envelope is near exhaustion, `stratum_execute` surfaces that signal — Claude can compress context, conclude early, or surface the remaining work for the next session rather than degrading silently.

### 4. Every step is auditable

`stratum_audit` returns the structured trace of every step: what went in, what came out, how many retries, at what token cost. Not a transcript — a structured record you can query. When something goes wrong, you can identify the exact step, the exact input, and the exact failure reason.

---

## The Two Audiences

### Professional developers

You're building systems where the code Claude produces is reviewed, tested, and shipped. What you want from Claude Code + Stratum:

- See the execution plan before Claude writes a line
- Typed contracts between steps — no `KeyError` in step 4 because step 2 changed its output shape
- A test suite that validates the orchestration logic independently of the LLM
- An audit trail that can be committed to the repo alongside the code

The Stratum MCP server exposes `stratum_plan`, `stratum_validate`, `stratum_execute`, and `stratum_audit` as Claude Code tools. Register them in `.claude/settings.json`, add a `CLAUDE.md` note to use them for non-trivial tasks, and you're done.

### Vibe coders

You're using Claude Code to build things you couldn't build otherwise. You don't want to review typed execution plans — you want to describe what you need and get working code.

Stratum's value here is different: the code Claude produces is `@infer`-annotated. Every LLM call in the generated code has declared intent, typed outputs, and structured retry. When you come back to that code in three months and it's broken, you have trace records that tell you what changed. You're not debugging a transcript.

The flywheel: vibe coders get better output from Claude (because Stratum-annotated code is more reliable) → their codebases contain `@infer` and `@contract` annotations → professional developers they collaborate with encounter Stratum → adoption spreads without anyone explicitly adopting it.

---

## The Planning Skill

The entry point is a Claude Code skill at `.claude/skills/plan.md`:

```markdown
When starting a non-trivial task:
1. Call stratum_plan with the task description
2. Present the typed plan to the user
3. Wait for approval before executing
4. On approval, execute each step via stratum_execute
5. On step failure, retry with structured failure context
6. On completion, call stratum_audit and surface the trace
```

This is the structural embedding. The plan skill makes Stratum the default path, not an optional tool Claude might reach for.

---

## What It Looks Like in Practice

A developer asks Claude Code: "Add rate limiting to the API endpoints."

Without Stratum:
- Claude reads files, thinks, writes code, runs tests, iterates
- Context fills up partway through
- Retry on test failure replays the full context
- You get code that works (usually) with no audit trail of what changed

With Stratum + MCP:
- Claude calls `stratum_plan` → typed plan: read endpoints → identify patterns → implement middleware → write tests → validate
- Developer reviews the plan, approves
- Each step executes with typed inputs/outputs
- Test failure triggers targeted retry: "middleware did not apply to `/api/health` — add exclusion list"
- `stratum_audit` at the end: 5 steps, 2 retries on step 3, 4200 tokens, all passing

The code Claude writes is annotated:

```python
@infer(
    intent="Generate rate limiting middleware for these API endpoints",
    context="Exclude health check and metrics endpoints. Use token bucket algorithm.",
    ensure=lambda r: all(test_endpoint(e, r) for e in endpoints),
    budget=Budget(ms=2000),
    retries=3,
)
def generate_rate_limiter(endpoints: list[Endpoint]) -> RateLimiterConfig: ...
```

That annotation is the audit trail. It's the intent, the constraints, and the validation — committed to the repo, versioned, and readable by the next person who touches the code.

---

## The Recursive Insight

Stratum makes LLM coding agents better at building LLM systems.

Because when Claude Code writes code that uses `@infer` and `@contract`, it's producing systems that have the same formal properties as the Stratum runtime itself: typed contracts, structured retry, hard budget limits, full observability. The output code doesn't just work — it's maintainable.

The pattern propagates. Systems built with Stratum produce better LLM-generated components. Those components are easier for the next Claude Code session to extend. The runtime improves the output; the output improves the runtime's utility.

---

## The Enforcement Gap (Honest Version)

Nothing forces Claude Code to use the MCP server. This is worth saying directly.

The MCP strategy is viable for developers who want Stratum-aligned behavior in a controlled environment. It is not a security boundary against deliberate circumvention. What it is:

- **Environment restriction:** Configure Claude Code's settings to include Stratum MCP but exclude direct LLM API access. Claude has one path to LLM calls: through Stratum.
- **Hooks:** Detect when Claude writes manual retry loops or ad-hoc prompt construction in generated code. Flag it.
- **Skills:** Make the plan skill the structural entry point for non-trivial tasks.
- **Rules:** Fill the gaps — frame what Stratum can't structurally enforce.

These four layers together eliminate accidental drift. They don't prevent deliberate circumvention — but that's not the problem they're solving.

---

## Getting Started

1. Add the Stratum MCP server to `.claude/settings.json`:

```json
{
  "mcpServers": {
    "stratum": {
      "command": "python",
      "args": ["-m", "stratum.mcp"],
      "env": {}
    }
  }
}
```

2. Add a planning skill at `.claude/skills/plan.md` (see the MCP server docs)

3. Add to `CLAUDE.md`:

```markdown
For non-trivial tasks, use the stratum_plan tool before writing any code.
Present the typed plan for review before executing.
```

The MCP server ships with the Phase 2 library release. The specification describing exactly what it enforces is at [SPEC.md](../SPEC.md).

Questions and feedback: [GitHub Discussions](../../discussions).
