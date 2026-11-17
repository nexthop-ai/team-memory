# Stage 11: Fail Fast During Embedding Provider Outages

## Objective
Stop repeated query embedding calls during a provider outage, return keyword results promptly, and recover semantic search automatically when the provider recovers.

## Inputs
- Read `AGENTS.md`, global rules in `prompts/00-start-here.md`, prompts 08–10 and their handoffs.
- Inspect current query embedding/cache orchestration, provider error classification, search metadata, telemetry, CLI/Helm configuration and worker retry handling.
- Verify relevant Team Memory records and current repository behavior before making changes.

## Work
1. Implement a small process-local closed/open/half-open circuit breaker around actual query-provider attempts. Scope state to the effective provider/model configuration. Make failure threshold, cooldown and enable/disable behavior configurable and validated, with finite documented defaults. Document that replicas have independent breakers.
2. Put successful query-cache lookup before the breaker so cached embeddings remain useful while it is open. Coalesced requests must count as one provider attempt, not many failures. Document wrapper order and keep document embeddings/worker retries independent of the query breaker.
3. Count transient provider failures and genuine provider timeouts toward opening the circuit. Do not count caller cancellation, revoked access, database failures, skipped requests or cache hits. Treat dimension mismatch as a configuration error with truthful metadata, not as a transient outage hidden by retry. Review error wrapping so classification is reliable.
4. While open, skip uncached provider calls and continue authorized keyword retrieval. After cooldown, allow a bounded half-open probe through normal query traffic; concurrent requests must not stampede the provider. Success closes the breaker and failure reopens it. Guard against late completions from an earlier state incorrectly closing a newly opened circuit.
5. Preserve full-text-only and vector-only mode semantics. Expose a stable machine-readable circuit-open status and a safe public explanation; inspect MCP schemas, portal rendering, docs and tests for compatibility. Never leak provider URLs or raw errors. A cache hit should report successful semantic retrieval even when uncached calls are blocked.
6. Add bounded-cardinality state/transition/skipped-call metrics and recovery logs without query text or principal identifiers. Wire configuration through environment parsing, `.env.example`, Helm values/schema and docs. Keep provider outages out of API liveness; do not add retry loops to individual search requests.

## Validation and acceptance
- Use controllable time and a scripted provider for threshold opening, cooldown, exactly bounded half-open probes, successful recovery, failed recovery and stale in-flight completions. Cover cancellation and independent model/configuration state with race-enabled tests.
- Prove cache hits still use vectors while open; misses return keyword fallback without provider calls. Verify coalesced errors affect the threshold once and worker retries remain unchanged.
- Exercise metadata through MCP and portal, including vector-only failure. Add security tests for public error/telemetry redaction where affected.
- Measure healthy, outage and recovery searches against the stage 08 baseline. Beyond any in-flight attempts already admitted, open-circuit cache misses must incur no provider call or provider timeout wait. Do not promise fixed end-to-end latency independent of database/network conditions.
- Run relevant tests, `make test`, `make test-security`, `make lint`, and `make helm-lint`; report skips and failures. Re-run retrieval quality checks if ranking or candidate behavior changes unexpectedly.

## Deliverables
Implementation and tests, breaker/wrapper-order decision record, operator outage/recovery guidance, Unreleased changelog, and `docs/handoff/11-query-provider-circuit-breaker.md`. Include a final before/after summary for all four stages: provider availability evidence, warm/cold/cache latency, outage latency, recovery behavior, retrieval equivalence and outstanding deployment work. Do not deploy as part of this prompt.
