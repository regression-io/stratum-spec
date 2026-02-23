# Changelog

## [1.0.0] - 2026-02-23

Initial draft specification.

- Type system: primitives, `@contract`, schema compilation, content hash, `opaque[T]`, `Probabilistic[T]`, `HumanDecision[T]`, `HumanReviewContext`
- Decorators: `@contract`, `@compute`, `@infer`, `@refine`, `@flow` with full signatures and rules
- Execution semantics: `@infer` loop, retry context injection, `@refine` loop, `@flow` budget tracking
- Prompt compiler: assembly order, `opaque[T]` handling, compiled prompt hash
- Concurrency: `stratum.parallel`, `quorum`, `stratum.debate`, `stratum.race`
- HITL: `await_human`, `ReviewSink` protocol, `ConsoleReviewSink`
- Budget: enforcement, inheritance, per-call and per-flow
- Trace records: schema, OTel export attributes
- Caching: scopes and key definitions
- Static analysis rules and error types
- Configuration API and sync shim
- Module structure
