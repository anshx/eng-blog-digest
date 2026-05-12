# Index

- 2026-05-11 — [How Discord Automates ScyllaDB Clusters at Scale](../posts/2026-05-11/discord-scylladb-cluster-automation.md) — How lifting safety invariants from runbooks into typed workflow conditions makes cluster automation structurally un-bypassable (SCP framework, zone-aware concurrency, resumable SQLite state).
- 2026-05-12 — [State of Routing in Model Serving](../posts/2026-05-12/netflix-model-serving-routing-control-data-plane.md) — How Netflix applied the classic control plane / data plane separation from networking to ML model routing: a metadata-only service (Lightbulb) resolves which model to run and emits an opaque header token; Envoy does a cheap O(1) lookup to forward the actual request, keeping A/B testing and model-version selection out of the hot path.
