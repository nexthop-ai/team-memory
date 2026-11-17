# Stage 09: Cache Query Embeddings

## Objective
Avoid repeated provider calls for identical queries using a bounded in-memory cache of successful query embeddings. Keep authorization and search results fresh on every request.

## Inputs
- Read `AGENTS.md`, `prompts/00-start-here.md`, `prompts/08-search-provider-reliability.md`, and its handoff at `docs/handoff/08-search-provider-reliability.md`.
- Inspect the current search service, embedding provider/prefix helpers, CLI retrieval configuration, telemetry and retrieval/security tests.
- Check relevant Team Memory decisions and verify them against the repository. Implement against current code, not an assumed stage 08 patch.

## Work
1. Add a process-local, concurrency-safe query embedding cache with finite capacity, TTL, documented defaults and an explicit disable option. Use a straightforward eviction policy; do not add Redis or a database table.
2. Partition entries by authenticated principal to avoid cross-user cache timing reuse. Key entries by exact trimmed query and the effective embedding identity: provider/configuration namespace, model name, dimensions and query prefix. Do not lowercase, collapse internal whitespace or strip search syntax. Account for model replacement under the same name using expiry and a documented namespace/version change or invalidation procedure.
3. Cache only successful, validated vectors. Never store provider errors, cancellations, timeouts or malformed responses as successful entries. Copy vectors where needed so callers cannot mutate cached values. Do not persist query text or vectors, or expose them through logs or metrics.
4. Coalesce concurrent misses for the same key. Each waiter must be able to cancel independently; cancellation of the first request must not cancel work needed by other live waiters. Bound shared work with the configured embedding timeout and service lifecycle; avoid unbounded work when all waiters have left. Document how the timeout and cache interact.
5. Cache only query embeddings. Re-run scope resolution, membership/RLS checks, override resolution, candidate retrieval and ranking on every search. Changes to memories, approvals and permissions must take effect without waiting for cache expiry. Do not introduce a result cache or change asynchronous document indexing.
6. Expose bounded hit/miss/eviction/coalescing metrics and configuration through existing environment and Helm patterns. Preserve provider-call metrics so a cache hit is not reported as a new provider call. Include configuration validation and operator documentation.

## Validation and acceptance
- Use a counting fake provider and controllable time for hit, expiry, eviction, disabled mode, key separation, failure retry and vector mutation tests.
- Exercise simultaneous misses, independent cancellations, all-waiter cancellation and timeout under `go test -race`; use synchronization rather than fragile sleeps.
- Add adversarial integration tests proving a warm cache cannot bypass another principal's permissions, revoked membership, current revisions, archives or approved overrides.
- Compare uncached and cached latency and provider call counts on the same fixture. An identical warm request must avoid the provider and produce the same retrieval behavior against unchanged data.
- Run relevant tests, `make test`, `make test-security`, `make lint`, and `make helm-lint` for chart changes; report skips and failures honestly.

## Deliverables
Implementation and tests, documented cache identity/defaults/privacy/limits and invalidation procedure, Unreleased changelog, and `docs/handoff/09-query-embedding-cache.md` with measurements and wrapper/lifecycle details needed by stages 10 and 11.
