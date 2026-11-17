# Stage 08: Measure Search Latency and Improve Embedding Reliability

## Objective
Measure where search time goes, make embedding-provider failures diagnosable, and reduce model cold starts without making memory writes wait for embeddings.

## Inputs
- Read `AGENTS.md` and the global rules in `prompts/00-start-here.md`.
- Inspect `internal/search/search.go`, `internal/embeddings/{ollama,provider,indexer}.go`, `cmd/team-memory/{retrieval,ops}.go`, and `internal/telemetry/`.
- Read `docs/decisions/{embedding-models,vector-index-strategy,retrieval-scoring}.md`, `docs/eval/retrieval-report.md`, and the current deployment and observability runbooks.
- Inspect `.env.example`, the local Docker configuration, and `deploy/helm/team-memory/`, including values, schema, environment helpers, NetworkPolicies and probes.
- Search Team Memory in `personal/team-memory` for relevant decisions; verify any applicable record with `memory_get` and repository evidence.

## Context and boundaries
Saves already enqueue embedding jobs transactionally. The worker embeds each revision asynchronously; searches embed only the query and fuse keyword and vector candidates. Preserve that architecture and existing scope, RLS, review and override semantics.

A connected search on 2026-11-17 reported `fulltext_only` with an unavailable embedding provider. This is one observation, not a diagnosis of DNS, cold starts or database performance. Recheck the current state. Do not infer embedding coverage from `pending_embeddings=0` on a fallback response: the current search code only counts missing embeddings when query embedding succeeds.

This prompt authorizes local implementation and disposable local verification. Use existing authorization for external diagnostics; if live access is absent, produce reproducible operator commands and report the live diagnosis as unverified. Do not deploy or change cluster-wide DNS, networking, resource allocations or external Ollama configuration without authorization.

## Work
1. Trace and measure end-to-end search, query embedding and database retrieval separately. Reuse existing embedding histograms and tracing; add only missing bounded-cardinality measurements, including fallback reasons. Cover both MCP and portal search through their shared service. Do not emit queries, content, credentials, provider URLs or tenant identifiers in metric labels.
2. Add a reproducible local measurement procedure covering warm and cold provider calls, healthy hybrid search, provider timeout, provider error, and keyword-only search. Report p50/p95 with sample sizes, hardware, model, configuration and measurement boundaries. Keep historical index benchmarks separate from new measurements.
3. Diagnose provider availability by checking configuration, registered/enabled model and dimensions, actual `/api/embed` behavior, connectivity from API and worker paths, and worker queue health. Distinguish disabled, unreachable, timed out, missing model and dimension mismatch. Keep internal provider details out of public responses.
4. Verify current official Ollama documentation before adding configurable model residency via `keep_alive`. Preserve existing behavior when unset, validate configuration, and document memory/resource tradeoffs. Wire supported configuration consistently through CLI environment parsing, `.env.example`, Helm values/schema and documentation. Prefer explicit residency configuration over a permanent polling loop; any warm-up must be bounded, optional and must not prevent keyword search from starting when Ollama is down.
5. Keep optional-provider failure from becoming an API liveness dependency. Document API versus worker behavior, how to verify revision embeddings, and how to identify backlog or dead jobs. Fix only demonstrated local configuration/code defects; leave unresolved deployment causes clearly identified.

## Validation and acceptance
- Add provider HTTP tests for residency request serialization, unset behavior and invalid configuration; exercise healthy, slow and failing providers without requiring external Ollama.
- Confirm keyword fallback and existing dimension validation still work and save/edit retains the transactional job path.
- Verify telemetry does not expose sensitive data; add adversarial coverage in `integration/security/` if telemetry behavior changes.
- Run relevant Go tests with the race detector, `make lint`, and `make helm-lint` when chart files change. Use `make test` for database-dependent checks; report skipped checks explicitly.
- Supply measured baseline results, observed causes versus hypotheses, and exact operator verification steps. Do not claim a live outage is fixed without a successful live query embedding and hybrid search.

## Deliverables
Implementation, configuration and tests; updated operations docs and Unreleased changelog; `docs/handoff/08-search-provider-reliability.md` with measurements, unresolved access/diagnostic gaps, and the baseline for stage 09. Record significant design choices in `docs/decisions/`.
