# APIBP021 — AutoBuild

Course: API Based Products & SESAPZG504 | Instructor: Vinaya Sathyanarayana

## Team Details

| # | Name                     | Roll No      | GitHub ID            | Email                                        |
|---|--------------------------|--------------|----------------------|----------------------------------------------|
| 1 | Suvabrata Chowdhury      | 2024sl70020  | SuvabrataChowdhury   | 2024sl70020@wilp.bits-pilani.ac.in           |
| 2 | Vivian Samuel Rodrigues  | 2024sl70050  | Sam007Yeah           | 2024sl70050@wilp.bits-pilani.ac.in           |
| 3 | Meenakshi Sarkar         | 2024sl70030  | MeenakshiSarkar2692  | 2024sl70030@wilp.bits-pilani.ac.in           |
| 4 | Yash Khaitan             | 2024sl70032  | yash-khaitan         | 2024sl70032@wilp.bits-pilani.ac.in           |
| 5 | Swastika Naha            | 2024sl70003  | Swastikanaha15       | 2024sl70003@wilp.bits-pilani.ac.in           |
| 6 | Wusat Sami               | 2024sl70002  | WusatSami03          | 2024sl70002@wilp.bits-pilani.ac.in           |

- Team ID: `APIBP021` 
- Chosen Approach: `A` — REST-First + GraphQL Layer

## Quick start

```bash
git clone --recurse-submodules https://github.com/SAP2024-4Y/APIB021.git
cd APIB021/src/AutoBuild

# Build the application image
docker build -t pipeline-app .

# Start the application stack (backend + database + auth + feature flags)
docker compose --env-file .env.local up -d
```

> If you cloned without `--recurse-submodules`, run:
> ```bash
> git submodule update --init --recursive
> ```

Services available after startup:

| Service           | URL                       |
|-------------------|---------------------------|
| Server            | http://localhost:8080     |
| Auth (Keycloak)   | http://localhost:8180     |
| Feature flags     | http://localhost:8016     |
| GraphiQL UI       | http://localhost:8080/graphiql |

> **Note:** The GraphQL layer is currently under development. GraphiQL is only accessible without authentication when running with the `default` Spring profile. The `docker compose` stack activates the `demo` profile (Keycloak/JWT), so visiting `/graphiql` will return an authentication error. To explore GraphiQL locally without auth, start the backend directly with `SPRING_PROFILES_ACTIVE=default`.

## GraphQL schema

Current schema (under development — only read queries are implemented):

```graphql
type Query {
    getPipelineById(pipelineId: ID!): Pipeline
    getAllPipelines: [Pipeline!]!
}

type Pipeline {
    id: ID
    name: String!
    stages: [Stage!]!
}

type Stage {
    id: ID
    name: String!
    scriptType: String!
    command: String
    stageOrder: Int!
}
```

## API collection (Bruno)

All API endpoints can be tested using the [Bruno](https://www.usebruno.com/) API client. The collection is at `src/AutoBuild/apis/opencollection/pipelinecollection/` and covers three folders:

- **Keycloak Auth** — fetch an access token and verify auth flows
- **Pipeline CRUD** — create, read, update, and delete pipelines
- **Executor Ops** — trigger builds, stream live build state via SSE, read stage logs

### Setup

1. Open Bruno and select **Open Collection**, then point it at `src/AutoBuild/apis/opencollection/pipelinecollection/`.
2. Select the `local` environment — it pre-populates `host`, `keycloak_url`, credentials, and an empty `access_token` variable.
3. Run **Keycloak Auth → GetToken** first; it stores the token in `access_token`, which is injected as a bearer token into all subsequent requests.

## Demo credentials

Get an access token from Keycloak:

```bash
curl -s -X POST \
  "http://localhost:8180/realms/SpringBootRealm/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=pipeline-app&username=testuser&password=testpass&grant_type=password" \
  | jq -r '.access_token'
```

## Run tests

```bash
# Backend — from APIB021/src/AutoBuild/pipeline
mvn test

# Frontend — from APIB021/src/AutoBuild/pipeline-ui
npm test
```

## Sample REST calls

**List all pipelines:**
```bash
curl -H "Authorization: Bearer <token>" \
     "http://localhost:8080/api/v1/pipeline"
```

**Create a pipeline:**
```bash
curl -s -X POST "http://localhost:8080/api/v1/pipeline" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "my pipeline",
    "stages": [
      {
        "name": "build",
        "scriptType": "bash",
        "command": "#!/bin/bash\necho \"Hello World\"",
        "stageOrder": 0
      }
    ]
  }'
```

**Execute a pipeline:**
```bash
curl -s -X POST "http://localhost:8080/api/v1/execute/pipeline" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{ "pipelineId": "<pipeline-id>" }'
```

## UI

### Start the frontend

The UI requires the backend stack to be running (see [Quick start](#quick-start)).

```bash
# From APIB021/src/AutoBuild/pipeline-ui
npm ci
npm run local-e2e       # connects to live backend at http://localhost:8080
```

To develop against a mock server instead of the real backend:

```bash
npm run local-with-mock # starts Prism mock server + Vite against mock
```

> `local-with-mock` uses `.env.local-with-mock` and points the UI at the Prism mock
> server (port 4010) generated from `apis/openapi-spec/pipeline-service.yaml`.

### UI walkthrough

1. **Login** — Navigate to `http://localhost:5173`. You are redirected to the Keycloak login page. Sign in with `testuser` / `testpass`.

2. **Pipelines list** — After login you land on the Pipelines page, which lists all defined pipelines fetched via the generated API client.

3. **Create a pipeline** — Click **New Pipeline**, fill in the pipeline name and add stages (name, script type, bash command, order).

4. **Run a pipeline** — Open a pipeline and click **Execute**. The build appears in the Builds list with state `WAITING → RUNNING`.

5. **Monitor build progress** — Navigate to the build detail page. Stage states and logs update in real time via Server-Sent Events (`/api/v1/pipeline/build/sse/subscribe/{id}`).

6. **Live Demo** - Autobuild Live Walkthrough
   

https://github.com/user-attachments/assets/cc225502-21c2-43e8-bc23-5c8b887f7483



## OpenAPI-driven client code generation

The frontend never calls the backend directly with hand-written fetch calls. Instead:

1. **Source of truth** — `apis/openapi-spec/pipeline-service.yaml` is the single OpenAPI spec for the pipeline service.

2. **Code generation** — `openapitools.json` (in `pipeline-ui/`) configures `openapi-generator-cli` to produce a TypeScript-Axios client into `pipeline-ui/src/gen/`:

   ```bash
   # From APIB021/src/AutoBuild/pipeline-ui
   npm run generate:clean-client-code   # removes src/gen/ then regenerates
   # or, to regenerate without deleting first:
   npm run generate:client-code
   ```

   The generator uses the `typescript-axios` generator and reads the spec at
   `../apis/openapi-spec/pipeline-service.yaml`.

3. **Generated output** — `pipeline-ui/src/gen/` contains:
   - `api.ts` — typed API classes (`PipelineApi`, `PipelineBuildApi`, `StageBuildApi`)
   - `configuration.ts` — base URL and auth header configuration
   - `index.ts` — barrel export used by the rest of the app

4. **Usage in the app** — Service files under `pipeline-ui/src/services/` import from `src/gen/` and pass the configured Axios instance (with the Keycloak bearer token injected via an interceptor in `axiosInstance.ts`).

5. **Mock server** — The same spec drives the Prism mock server (`npm run mock`), so the UI can be developed and tested without a running backend.

> Re-run `generate:clean-client-code` any time `pipeline-service.yaml` changes.

## Monitoring stack

The full observability stack (Grafana, Loki, Prometheus, Alloy) can be started alongside the application using additional compose files:

```bash
# From APIB021/src/AutoBuild
docker build -t pipeline-app .
docker compose --env-file .env.local \
  -f docker-compose.yml \
  -f monitoring/docker-compose-monitoring.yml \
  -f monitoring/docker-compose-monitoring-integration.yml \
  up -d
```

Monitoring UIs available after startup:

| Service    | URL                       |
|------------|---------------------------|
| Grafana    | http://localhost:3000     |
| Prometheus | http://localhost:9090     |
| Alloy      | http://localhost:12345    |

To stop the full stack (app + monitoring):

```bash
docker compose -p autobuild down
```

## Stopping

```bash
# From APIB021/src/AutoBuild
docker compose down

# To also wipe all persisted volumes (database, logs, metrics):
docker compose down -v
```

## Bazel based CI

As AutoBuild is a monorepo with multiple toolchains (Maven, npm), Bazel provides a unified build and test interface with caching across all services. Each service delegates to its native tooling while Bazel wraps it consistently.

### Prerequisites

- [Bazel](https://bazel.build/install) installed
- `WORKSPACE_DIR` exported in your shell profile pointing to the AutoBuild submodule root:

  ```bash
  export WORKSPACE_DIR=/path/to/APIB021/src/AutoBuild
  ```

### Running tests

```bash
# From APIB021/src/AutoBuild

# Backend (Java/Maven)
bazel test //pipeline:test

# Frontend (Node/npm)
bazel test //pipeline-ui:unit_test
```

Use `--test_output=streamed` to see Maven/npm output live:

```bash
bazel test //pipeline:test --test_output=streamed
```

### Generating the OpenAPI spec

```bash
# From APIB021/src/AutoBuild
bazel build --config=openapi //pipeline:generate-openapi
```

The spec is written to `apis/openapi-spec/pipeline-service.yaml`. Commit it after any API change.

> See `src/AutoBuild/autobuild-docs/setup/bazel.md` for full details on caching, test output flags, and adding new services.

## Dependency management

Dependency updates across Maven, npm, GitHub Actions, and Docker base images are automated via [Renovate](https://docs.renovatebot.com/). Renovate was adopted after an initial period using Dependabot — it offers better monorepo support and finer-grained control over grouping and scheduling updates.

Renovate runs on a schedule and opens PRs automatically when new versions are available. The configuration lives in `renovate.json` at the repo root.
