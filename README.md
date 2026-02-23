# Stratum

**Stop babysitting your LLM calls.**

Stratum is a Python library where `@infer` (LLM calls) and `@compute` (normal functions) compose identically. Typed contracts flow between steps. The runtime handles retry, budget enforcement, and observability — so you don't have to wire them up yourself.

```python
@infer(
    intent="Classify the emotional tone of customer feedback",
    ensure=lambda r: r.confidence > 0.7,
    budget=Budget(ms=500, usd=0.001),
    retries=3,
)
def classify_sentiment(text: str) -> SentimentResult: ...
```

That's it. If the LLM returns low confidence, it gets told exactly what failed and tries again — not a blank retry, a targeted one. If it hits the budget, it stops. Every call produces a structured trace record you can query.

---

## Why

LLM calls in production have a few recurring problems:

- **Retry is brute force.** Most frameworks replay the full prompt on failure. Stratum injects only the specific postcondition that failed.
- **Budget is an afterthought.** Soft hints don't stop a runaway `refine` loop. Stratum enforces hard limits — `BudgetExceeded` is an exception, not a bill.
- **Flows are opaque.** When a multi-step pipeline fails, you want to know which step, with what input, after how many retries, at what cost. Stratum traces every call structurally.
- **LLM steps and regular functions don't compose.** Stratum makes `@infer` and `@compute` indistinguishable by type — swap one for the other and nothing downstream changes.
- **Agent outputs can hijack downstream agents.** `opaque[T]` fields are passed as structured data, never inlined into instruction text. The parameterized query pattern, applied to prompts.
- **Human-in-the-loop is a custom build every time.** `await_human` genuinely suspends execution and returns a typed `HumanDecision[T]`.

---

## Status

This repository contains the **v1 specification**. The reference implementation is in development.

- [`SPEC.md`](SPEC.md) — the normative specification
- [`blog/introducing-stratum.md`](blog/introducing-stratum.md) — detailed walkthrough of the design
- [`blog/stratum-in-claude-code.md`](blog/stratum-in-claude-code.md) — how Stratum works as a Claude Code execution runtime
- [`blog/stratum-in-codex.md`](blog/stratum-in-codex.md) — how Stratum works as a Codex execution runtime

**Feedback:** Use [GitHub Discussions](https://github.com/regression-io/stratum-spec/discussions) to ask questions, propose changes, or discuss the design. Use [Issues](https://github.com/regression-io/stratum-spec/issues) for specific bugs or gaps in the spec.

---

## Core Concepts

### `@infer` and `@compute` are the same type

```python
@infer(intent="Classify document topic", ensure=lambda r: r.confidence > 0.8)
def classify(doc: Document) -> Category: ...

@compute
def classify(doc: Document) -> Category:
    return rule_based_classifier(doc)
```

These have identical signatures. A `@flow` that calls `classify` doesn't know or care which one it gets. This means:

- **Testing:** Mock `@infer` calls with `@compute` stubs. Test orchestration logic without touching an LLM.
- **Migration:** Start with LLM, replace with rules as patterns emerge. No downstream changes.
- **Cost control:** Swap expensive inference for fast lookup when coverage allows.

### Contracts are typed boundaries

```python
@contract
class SentimentResult:
    label: Literal["positive", "negative", "neutral"]
    confidence: Annotated[float, Field(ge=0.0, le=1.0)]
    reasoning: str
```

A `@contract` class compiles to a JSON Schema injected into the structured outputs API. The LLM's output is validated against it before your code sees it. Every contract gets a content hash — a hash change means the compiled prompt changed and LLM behavior may have drifted.

### Retry is structured

```python
@infer(
    intent="Classify sentiment",
    ensure=lambda r: r.confidence > 0.7,
    retries=3,
)
def classify(text: str) -> SentimentResult: ...
```

On failure, the LLM receives:

```
Previous attempt failed:
  - ensure: result.confidence > 0.7 (actual: 0.42)
Fix this issue specifically.
```

Not a full prompt replay. The specific violation, nothing else.

### Flows are deterministic

```python
@flow(budget=Budget(ms=5000, usd=0.01))
async def process_ticket(ticket: SupportTicket) -> Resolution:
    category  = await classify(ticket.body)
    sentiment = await classify_sentiment(ticket.body)
    response  = await draft_response(ticket, category, sentiment)
    return response if rule_check(response) else escalate(ticket)
```

`@flow` compiles to normal control flow. You can read it, test it, and trace it. The orchestration is auditable even when individual steps are opaque LLM calls.

---

## Features

| Feature | Description |
|---|---|
| Structured retry | `ensure` postconditions drive retry with targeted failure feedback |
| Hard budget limits | Per-call and per-flow — `BudgetExceeded`, not a soft hint |
| `opaque[T]` | Field-level prompt injection protection |
| `await_human` | HITL as a first-class typed primitive — genuine suspension |
| `stratum.parallel` | Concurrent execution with `require: all/any/N/0` semantics |
| `quorum` | Run N times, require majority agreement |
| `stratum.debate` | Adversarial multi-agent synthesis with convergence detection |
| Full observability | Structured trace record on every call, OTel export built-in |
| One dependency | `litellm` only. No OTel SDK. Pydantic optional. |

---

## Specification

See [`SPEC.md`](SPEC.md) for the complete normative specification including:

- Full type system with schema compilation rules and content hash algorithm
- Exact decorator signatures and rules
- Complete execution loop for `@infer`, `@refine`, `@flow`
- Prompt compiler assembly order and `opaque[T]` handling
- Concurrency primitive semantics
- HITL execution sequence and `ReviewSink` protocol
- Budget inheritance chain
- Trace record schema and OTel attribute mapping
- Static analysis rules and error types
- Complete worked example

---

## Versioning

The spec is versioned independently from any implementation.

| Version | Status | Date |
|---|---|---|
| 1.0.0 | Draft | 2026-02-23 |

---

## License

[Apache 2.0](LICENSE)
