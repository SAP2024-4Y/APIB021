# Architecture Decision Record — AutoBuild API

**Team:** APIB021 | **Date:** 2026-11-13 | **Status:** Accepted

---

## 1. Business Problem & Consumers

Software teams lose significant time configuring and maintaining CI/CD infrastructure across disparate tools. AutoBuild provides a self-hosted platform where developers define, trigger, and monitor build pipelines through a UI or API with no vendor lock-in. The pipeline engine runs user-defined bash scripts and emits real-time state updates and logs.

**Primary consumers:**
- **Developer UI (`pipeline-ui`)** — React SPA using the generated TypeScript-Axios client (`src/gen/`).
- **Monitoring systems** — Prometheus scrapes `/actuator/prometheus`; `/health` is polled by Grafana and the Docker Compose health-check chain.
- **Programmatic integrations** — scripts or services listing builds, reading logs, or deleting stale records via REST.

---

## 2. Business Capabilities

1. **Pipeline definition management** — create, read, and delete named pipelines of ordered bash-script stages.
2. **On-demand and event-driven execution** — trigger runs manually or on SCM events, with an immediate `203` acknowledgement so callers are never blocked.
3. **Real-time build observability** — stream live stage state changes and logs via Server-Sent Events.
4. **Audit and history** — persist every `PipelineBuild` and `StageBuild` for querying past runs and managing storage.
5. **Feature-flagged evolution** — gate in-flight capabilities (e.g., `ENABLE_EDIT_PIPELINE`) behind OpenFeature-compatible flags via flagd.

---

## 3. Resource Model

Four core business resources map 1-to-1 to `pipeline-service.yaml` schemas and will be mirrored in `schema.graphql`.

```
Pipeline ──< Stage
    └──< PipelineBuild ──< StageBuild
```

| Resource | Description | Key Attributes |
|---|---|---|
| **Pipeline** | Reusable named build specification | `id`, `name`, `stages[]` |
| **Stage** | One ordered step within a Pipeline | `id`, `name`, `scriptType`, `command`, `stageOrder` |
| **PipelineBuild** | Single execution record of a Pipeline | `id`, `pipelineId`, `pipelineName`, `currentState` (WAITING/RUNNING/SUCCESS/FAILED), `stageBuilds[]` |
| **StageBuild** | Execution record for one Stage | `id`, `stageId`, `stageName`, `stageOrder`, `currentState` (WAITING/RUNNING/SUCCESS/FAILED/STOPPED) |

`Stage` objects are owned by their `Pipeline` (composition). Executing a `Pipeline` produces one `PipelineBuild` with a `StageBuild` per stage. Logs are on a separate sub-resource (`/stage/build/logs/{id}`) to avoid inflating `PipelineBuild` responses.

---

## 4. Approach Decision

**Chosen: Approach A — REST-First with a GraphQL Layer**

- The React UI already consumes a generated TypeScript-Axios client from the OpenAPI spec; pivoting to GraphQL-first would discard that investment.
- The GitHub webhook hard constraint (< 10 s acknowledgement) is best served by `POST /api/v1/execute/pipeline` returning a `203` immediately. A pure GraphQL mutation without a subscription would force polling.
- Spring Boot Actuator's `/health` — consumed by the Docker Compose health-check and Prometheus — is intrinsically REST.

**Schema impedance mismatches:**

| Mismatch | REST | GraphQL |
|---|---|---|
| Pending execution result | `203 Non-Authoritative` with `currentState: WAITING` | Mutation returns same; client subscribes for updates |
| Real-time state | SSE `/pipeline/build/sse/subscribe/{id}` | GraphQL Subscription over WebSocket backed by same SSE source |
| Large log payloads | Separate sub-resource endpoint | Field-level lazy resolution — `stageBuild { logs }` |
| List caching | `GET` cacheable via `ETag`/`Cache-Control` | `POST /graphql` not cacheable; requires persisted queries |
| HTTP status semantics | `201`, `203`, `204` carry domain meaning | Always `200`; domain outcome in `errors[]` or payload |

---

## 5. Domain Pattern Mapping

- **List** — `GET /api/v1/pipeline` and `GET /api/v1/pipeline/build` are canonical list resources; in GraphQL: `Query.pipelines` and `Query.pipelineBuilds`. Cursor-based pagination (`after`/`limit`) planned for v2.
- **Map** — `Stage.stageOrder` is an ordered integer key mapping position to stage. The execution engine runs stages from lowest to highest `stageOrder`; GraphQL exposes this as a sorted `[Stage!]!`.
- **Calendar** — Builds are stamped with the start time. A planned `Schedule` resource (cron + `pipelineId`) will model recurring runs: `POST /api/v1/pipeline/{id}/schedule` in REST, `Mutation.schedulePipeline` in GraphQL.

---

## 6. Composite Payload Comparison

**Scenario:** Build detail page — needs `PipelineBuild` + all `StageBuild` records + logs for each stage (4 stages).

**REST (N+1 for logs):**
```
GET /api/v1/pipeline/build/{buildId}          → ~420 bytes
GET /api/v1/stage/build/logs/{id}  × 4 stages → ~200 bytes each

Total: 5 requests, ~1,220 bytes
```

**GraphQL (1 round-trip):**
```graphql
query BuildDetail($id: ID!) {
  pipelineBuild(id: $id) {
    id pipelineName currentState
    stageBuilds { id stageName stageOrder currentState logs { stdout stderr } }
  }
}
# Total: 1 request, ~890 bytes
```

| | REST | GraphQL |
|---|---|---|
| Round-trips (4 stages) | 5 | 1 |
| Approx. bytes | ~1,220 | ~890 |
| Cacheable | Yes (GET) | No (POST) |
| Over-fetching | Yes (`pipelineId` always sent) | No |

GraphQL eliminates the N+1 log-fetch problem. REST retains HTTP caching on stable sub-resources.

---

## 7. Error Catalogue

All REST errors use `application/problem+json` (RFC 9457). All GraphQL errors carry `extensions.code`.

| # | Code | Cause | HTTP Status | GraphQL `extensions.code` |
|---|---|---|---|---|
| 1 | `PIPELINE_NOT_FOUND` | No pipeline for the given UUID | 404 | `PIPELINE_NOT_FOUND` |
| 2 | `PIPELINE_NAME_CONFLICT` | Pipeline name already exists | 409 | `PIPELINE_NAME_CONFLICT` |
| 3 | `PIPELINE_VALIDATION_ERROR` | Missing `name` or empty `stages` | 400 | `BAD_USER_INPUT` |
| 4 | `STAGE_ORDER_DUPLICATE` | Two stages share the same `stageOrder` | 400 | `BAD_USER_INPUT` |
| 5 | `STAGE_SCRIPT_TYPE_UNSUPPORTED` | `scriptType` not in supported set | 422 | `STAGE_SCRIPT_TYPE_UNSUPPORTED` |
| 6 | `BUILD_NOT_FOUND` | No `PipelineBuild` for the given UUID | 404 | `BUILD_NOT_FOUND` |
| 7 | `BUILD_ALREADY_RUNNING` | Concurrent run not permitted | 409 | `BUILD_ALREADY_RUNNING` |
| 8 | `STAGE_BUILD_NOT_FOUND` | No `StageBuild` for the given UUID | 404 | `STAGE_BUILD_NOT_FOUND` |
| 9 | `EXECUTION_DISPATCH_FAILED` | Execution engine failed to queue the build | 503 | `EXECUTION_DISPATCH_FAILED` |
| 10 | `UNAUTHENTICATED` | Missing or expired JWT | 401 | `UNAUTHENTICATED` |
| 11 | `FORBIDDEN` | Valid token, insufficient role | 403 | `FORBIDDEN` |
| 12 | `PIPELINE_DELETE_REJECTED` | Pipeline has a `RUNNING` build | 409 | `PIPELINE_DELETE_REJECTED` |

---

## 8. Design Decisions

**Decision 1 — `203` for pipeline execution.** `POST /api/v1/execute/pipeline` returns `203 Non-Authoritative Information`, not `202` (nothing to return) or `200` (work done). The server creates a `PipelineBuild` (`currentState: WAITING`) and returns it immediately, signalling the representation is non-final. The client navigates to the build detail page using the returned `id` without a second lookup.

**Decision 2 — OpenAPI spec as single source of truth for UI.** The frontend generates its entire API client from `pipeline-service.yaml` via `openapi-generator-cli` (`typescript-axios`, output `src/gen/`). Types stay in sync with the backend; breaking changes surface as compile errors; the same spec drives the Prism mock server for offline UI development.

**Decision 3 — Observer pattern for build state propagation.** The execution engine (`executor/execution/observer/`) decouples the state machine from consumers (SSE emitter, DB persistence, flag-gated notifications). Adding a new consumer requires only a new `Observer` — no changes to the state machine or existing observers.

---

## 9. Future Evolution

1. **Scheduled triggers (v2)** — A `Schedule` resource (cron + `pipelineId`) enables recurring runs. REST: `POST /api/v1/pipeline/{id}/schedule`; GraphQL: `Mutation.schedulePipeline`.
2. **Multi-tenant workspaces** — A `Workspace` above `Pipeline` enables team isolation and per-workspace RBAC. REST: `/api/v1/workspace/{id}/pipeline/...`; GraphQL: `Workspace` as top-level query entry.
3. **GraphQL subscriptions** — Replace the SSE endpoint with `Subscription.pipelineBuildUpdated(buildId: ID!): PipelineBuild` for real-time updates consumable by any GraphQL client without SSE support.
4. **Microservice Separation** - Replace the existing monolith executor module with a `go` based service which uses `dagger` based clients to orchestrate the pipeline dynamically.

---

## 10. Business Value

1. **Reduced time-to-pipeline** — Teams define and run pipelines in minutes without CI YAML or external agents. OpenAPI-driven codegen keeps the UI and API in sync automatically.
2. **Operational visibility out of the box** — The monitoring stack (Grafana, Loki, Prometheus, Alloy) ships as a compose overlay. `/health` is wired into the Docker Compose dependency graph and Prometheus scrape config from day one.
3. **Low-friction extensibility** — Feature flags (flagd / OpenFeature) enable incremental rollout without redeployment. The observer-pattern engine and planned GraphQL layer let new consumers and capabilities be added without breaking the existing REST contract.

