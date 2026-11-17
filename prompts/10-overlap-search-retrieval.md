# Stage 10: Overlap Keyword Retrieval and Query Embedding

## Objective
Run query embedding while authorized keyword/database work proceeds, reducing hybrid search latency while preserving ranking, consistent reads and graceful keyword fallback.

## Inputs
- Read `AGENTS.md`, global rules in `prompts/00-start-here.md`, prompts 08–09 and their handoffs.
- Inspect `internal/search/{search,rank}.go`, `internal/postgres/queries/search.sql`, transaction helpers and scope/override resolution, query cache implementation, MCP/portal search adapters and retrieval/security tests.
- Read the retrieval scoring, RLS and vector index decisions; verify relevant Team Memory guidance.

## Work
1. Measure the current sequential critical path. Design the smallest change that overlaps embedding with scope resolution and keyword retrieval. Preserve current query validation and authenticate/authorize the scope before dispatching new provider work. Keep cache keys and coalescing behavior from stage 09 intact.
2. Do not call methods concurrently on the same pgx connection or transaction. Prefer one sequential database transaction while embedding runs separately. Inspect the actual transaction isolation and document snapshot behavior; keep authorization, overrides and candidate hydration at least as consistent as before. Do not silently introduce separate snapshots that can return revoked or superseded data.
3. After keyword retrieval, wait only for the remaining embedding budget, then perform vector retrieval and existing fusion if embedding succeeds. Bound the embedding operation by one deadline, not a renewed timeout at each phase. Preserve full-text-only and vector-only modes and their failure semantics. Hybrid search still waits for its bounded semantic attempt; early streaming or returning incomplete keyword results is outside this change.
4. On provider failure or timeout, return the authorized keyword results with truthful metadata. On database/authorization errors or caller cancellation, release resources and detach/cancel the request's embedding waiter without disrupting other requests sharing a cache miss. Never suppress a database failure as a provider fallback.
5. Preserve candidate limits, similarity thresholds, scope filtering, current revisions, superseded/archived rules, effective overrides, trust boosts and ranking determinism. Keep `pending_embeddings` reporting honest; do not report an unperformed count as proof of complete indexing.
6. Update tracing/metrics so overlapping durations are understandable: component durations may overlap and must not be summed as total latency. Avoid extra connection-pool pressure or holding an idle transaction for longer than the embedding budget without measuring and documenting the tradeoff.

## Validation and acceptance
- Use gated fake providers and database instrumentation to prove keyword work advances while embedding is blocked, and keyword-only mode makes no provider call.
- Compare outputs against the sequential baseline for healthy hybrid, full-text-only, vector-only, timeout, unavailable provider and cache-hit paths. Test cancellation and all error exits for goroutine/connection leaks under the race detector.
- Exercise concurrent memory lifecycle, approval/override and permission changes in adversarial security tests; preserve documented transaction semantics and prevent unauthorized result exposure.
- Measure sequential versus overlapped p50/p95 and connection-pool behavior with cache misses and hits. Demonstrate overlap, not a speedup obtained by dropping semantic results or changing ranking.
- Run relevant race-enabled tests, `make test`, `make test-security` and `make lint`. Regenerate sqlc if SQL changes and include generated output. Report skipped checks and performance regressions.

## Deliverables
Implementation and tests, a decision record for concurrency/transaction/cancellation choices, updated observability docs and Unreleased changelog, and `docs/handoff/10-overlap-search-retrieval.md` with output comparisons and measured results.
