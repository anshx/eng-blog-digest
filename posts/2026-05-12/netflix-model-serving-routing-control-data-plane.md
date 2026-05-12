---
title: "State of Routing in Model Serving"
source: https://netflixtechblog.com/state-of-routing-in-model-serving-16e22fe18741
author: Netflix Technology Blog
company: Netflix
date_posted: 2026-05-02
date_digested: 2026-05-12
---

# State of Routing in Model Serving

## What's new to learn

1. **Control plane / data plane separation in ML routing** — the routing *decision* (which model version? which cluster shard?) can be made by a separate lightweight service (control plane) whose output is a single header token; the actual forwarding (data plane) is then a dumb O(1) lookup by an existing proxy like Envoy. This keeps the hot execution path free of business logic.

2. **Models as typed, self-contained workflows** — at Netflix, a "model" is not just a neural net weight file; it is a complete, packaged workflow that encapsulates pre-processing, feature retrieval, the trained component, and post-processing, all in a single unit with a standard interface. This packaging choice is what makes it safe to route traffic to a model without the router needing to know anything about the model's internals.

3. **Experimentation-aware routing** — A/B testing and model-version selection are solved as a *routing* problem, not as a model-inference problem. User cell allocation from the experimentation platform is consumed at request time and resolved into a routing key, so a single serving infrastructure can simultaneously run hundreds of model variants without any model-level awareness of experiments.

## Prerequisites

- **Envoy Proxy basics**: Envoy is a high-performance L7 proxy originally built by Lyft. It supports header-based routing, virtual IP (VIP) upstream clusters, and sidecar deployment patterns. You don't need to know its config in detail, but knowing it routes on headers is essential.
- **A/B testing / cell allocation**: The idea that each user is hashed into a "cell" at experiment enrollment time, and each cell is mapped to a treatment (here: a model version). Netflix calls this layer the Experimentation Platform.
- **Service mesh concepts**: the sidecar pattern (a proxy co-deployed with each service), VIPs (virtual IPs that front a cluster), and the general idea of a service mesh managing inter-service traffic.
- **ML serving basics**: knowing that model inference typically runs on dedicated GPU/CPU clusters, each cluster may host one or more model versions, and traffic splitting across versions is a core operational need.

## The core idea

Netflix runs a central ML platform that serves inference traffic for hundreds of model types at roughly **1 million requests per second**. Many model types have many live versions simultaneously (new candidate, champion, experiment variants). The platform needs to answer one question per request: *given this user and use-case, which cluster shard hosts the right model, and which model version should execute?*

The naive answer is a smart centralized proxy — Netflix called theirs **Switchboard**. Every client calls Switchboard, Switchboard queries the experimentation platform for the user's cell allocation, selects a model, and proxies the request to the appropriate cluster. This works, but it means Switchboard is in the execution critical path for every request: it adds latency, and a spike in slow or failing requests from one use-case can cascade into latency increases for all other use-cases sharing the same proxy cluster.

The evolved answer is **Lightbulb** — a service that produces only routing *metadata*, not routing *execution*. A client calls Lightbulb at the start of a request, receives back a `routingKey` token (an opaque identifier that maps to a cluster VIP) and an `ObjectiveConfig` blob (model id and serving parameters). The client attaches the `routingKey` as an HTTP header and puts `ObjectiveConfig` in the request body, then sends the request directly to the serving cluster. An Envoy sidecar at the serving cluster reads the header, maps the token to the right VIP, and forwards the request. Lightbulb never touches the actual inference payload.

The punchline: *routing decisions are separated from routing execution*. This is the control plane / data plane split that every network engineer knows from BGP + forwarding tables, applied to ML model serving.

## Mechanics

### Model abstraction

A Netflix "model" is a packaged, self-contained workflow unit. It owns:
- **Pre-processing**: feature retrieval, input transformation
- **Feature computation logic**: pulling from the feature store
- **The trained component** (optional — some "models" are rule engines)
- **Post-processing**: output transformation, ranking, filtering

This packaging means the serving platform can execute *any* model without understanding what it does. It also means a client calling the platform doesn't need to know which version of what model it's running — it just provides a use-case identifier (e.g., `homepage_ranking`) and the routing layer handles the rest.

### Switchboard (generation 1)

```
Client → Switchboard → [query exp platform for userId's cell]
                    → [select model version from A/B rule]
                    → [proxy request to serving cluster shard]
                    → Model Execution
                    → Response
```

Switchboard performs the full routing chain in-band:
1. **Allocation**: query the Experimentation Platform to get the user's cell
2. **Model Selection**: apply A/B test rule for this use-case → pick model id + version
3. **Request Routing**: forward to the cluster shard hosting that model

Problems that emerge at scale:
- Every request flows through Switchboard, so it becomes a **single point of failure**
- Diverse latency requirements (fast recommendation calls vs. expensive video understanding models) share the same cluster — one slow use-case backpressures others
- Error bursts from one tenant's misbehaving traffic can propagate through Switchboard to affect unrelated tenants
- Adding a new use-case or routing rule requires a Switchboard change/deploy

### Lightbulb (generation 2)

The key insight is that *routing metadata* (which cluster? which config?) is cheap to compute and tiny to transmit; the actual inference payload is large and expensive. Separating them lets you optimize each independently.

```
Client → Lightbulb: resolve(usecase, userId, context)
       ← routingKey: "homepage-v23-shard4"
       ← ObjectiveConfig: {model_id: "hprank_v23", params: {...}}

Client → Serving Cluster (with header: X-Routing-Key: "homepage-v23-shard4",
                           body includes ObjectiveConfig)
       → Envoy sidecar reads header → maps to VIP → forwards to shard
       → Model Execution (reads ObjectiveConfig from body)
       → Response
```

**Lightbulb's job** (control plane):
1. Consume minimal request context: use-case type, userId, optional hints
2. Query the Experimentation Platform for the userId's cell allocation
3. Apply the A/B rule for this use-case → select model id/version
4. Look up the cluster shard hosting that model version → derive `routingKey`
5. Package serving parameters into `ObjectiveConfig`
6. Return `{routingKey, ObjectiveConfig}` — nothing else

**Envoy sidecar's job** (data plane):
1. Intercept the inbound request
2. Read `X-Routing-Key` header
3. Lookup `routingKey → cluster VIP` in a local routing table (populated by Lightbulb or a config sync)
4. Forward request — the routing table lookup is O(1), no external calls

**Why the split on the response payload?**
- `routingKey` goes in the *header* so Envoy can act on it without touching the body (body parsing is expensive and defeats the point)
- `ObjectiveConfig` goes in the *body* because it may be rich (model parameters, contextual signals) and headers should stay small

### Request lifecycle summary

| Step | Who | What |
|---|---|---|
| 1. Allocation | Lightbulb ↔ Exp Platform | userId → cell |
| 2. Model Selection | Lightbulb | cell + rule → model id/version |
| 3. Metadata emission | Lightbulb → Client | `routingKey` + `ObjectiveConfig` |
| 4. Routing execution | Envoy | `routingKey` header → VIP lookup → forward |
| 5. Model Execution | Serving cluster | runs model workflow, returns result |

## Where it breaks

**Lightbulb is still a dependency.** Moving it off the execution hot path helps, but if Lightbulb is slow or unavailable, clients can't get routing metadata and requests fail before they even reach serving. The blast radius is smaller (Envoy can still route inflight requests that already have a `routingKey`) but cold requests are blocked.

**Routing table staleness.** Envoy's local routing table mapping `routingKey → VIP` must stay in sync with actual cluster state. If a shard goes down or a new model version is deployed and the routing table doesn't update fast enough, requests get misrouted. The post doesn't detail the sync mechanism but this is a real operational challenge.

**Header-body contract is a hidden coupling.** Lightbulb and Envoy are decoupled in theory, but they share an implicit contract: the header name (`X-Routing-Key`), the routing table format, and how `routingKey` tokens are structured. Changing any of these requires coordinated deploys.

**Homogeneous model interface required.** The whole architecture rests on models being self-contained, standardized workflow units with a uniform interface. Models that require bespoke client-side logic, streaming responses, or multi-stage round trips don't fit cleanly.

**Experimentation platform coupling.** Lightbulb queries the Experimentation Platform synchronously (implied). If the Exp Platform has elevated latency, Lightbulb's response time grows, and the benefit of decoupling from Switchboard is partially offset. Caching cell allocations helps but adds consistency complexity.

## Why it works

This is the **control plane / data plane separation** pattern from networking, verbatim.

In IP routing: the control plane (BGP, OSPF) runs distributed protocols to *compute* forwarding tables. The data plane just does a longest-prefix match on those tables for each packet — no protocol logic, no network-wide state, just a lookup. The genius is that the expensive computation (figuring out which path to take) is separated from the high-throughput execution (actually moving bits). Control plane can be slow and complex; data plane must be fast and simple.

At Netflix:
- **Lightbulb = control plane**: runs complex logic (experimentation queries, A/B rule evaluation, model-version selection), produces a compact routing token
- **Envoy = data plane**: does a fast O(1) lookup on that token, forwards traffic

The same principle appears in many distributed systems:
- **Kubernetes**: the scheduler (control plane) decides which node runs a pod; the kubelet and CNI (data plane) actually run it and wire up networking
- **Feature flags**: a flag evaluation service (control plane) resolves rules → variant; the application code just reads the resolved variant (data plane)
- **CDN**: the DNS/anycast layer (control plane) selects the PoP; the edge server (data plane) serves the cached bytes

The transferable mental model: **whenever routing decisions involve expensive lookups, business rules, or external dependencies, factor them into a control plane that emits a compact token, then let the data plane do a cheap token lookup.** The result is a system where the hot execution path has no external dependencies and scales independently from the decision-making logic.

## Going deeper

1. **Envoy's routing architecture** — Envoy's [documentation on route configuration](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto) explains how virtual hosts, route matchers, and header-based routing work. Understanding how Envoy evaluates route matches shows exactly why header-based routing is so cheap.

2. **Netflix's Experimentation Platform** — Netflix published ["It's All A/Bout Testing: The Netflix Experimentation Platform"](https://netflixtechblog.com/its-all-a-bout-testing-the-netflix-experimentation-platform-4e1ca458c15) which explains cell allocation, treatment assignment, and how they run hundreds of simultaneous experiments. Reading this alongside the routing post shows how the Exp Platform fits into Lightbulb's request flow.

3. **Metaflow (Netflix's ML workflow framework)** — the "models as self-contained workflows" abstraction described in the routing post is implemented, in part, through Netflix's open-source [Metaflow](https://metaflow.org/). Understanding how Metaflow packages steps into reproducible, versioned flows illuminates why a "model" at Netflix is a complete workflow unit rather than just weights.
