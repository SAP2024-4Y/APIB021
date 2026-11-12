# APIB021 — AutoBuild

Course: `<course code & name>` | Section: `<section>` | Instructor: `<instructor>`

> Replace every `<placeholder>` in this file. Deleted placeholders or mismatched names
> (vs `team.json`) cost marks. Remove this note before submitting.

## Team Details

| # | Name        | Roll No     | GitHub ID   | Email     |
|---|-------------|-------------|-------------|-----------|
| 1 | Member 1    | `<roll-no>` | `<gh-id>`   | `<email>` |
| 2 | Member 2    | `<roll-no>` | `<gh-id>`   | `<email>` |
| 3 | Member 3    | `<roll-no>` | `<gh-id>`   | `<email>` |
| 4 | Member 4    | `<roll-no>` | `<gh-id>`   | `<email>` |
| 5 | Member 5    | `<roll-no>` | `<gh-id>`   | `<email>` |

- Team ID: `APIB021` (must equal the repository name and `team.json.team_id`)
- Chosen Approach: `A` — REST-First + GraphQL Layer
  (must equal `team.json.approach`; change to `B` — GraphQL-First + REST Wrapper if chosen)

## Quick start

```bash
git clone --recurse-submodules https://github.com/<course-org>/APIB021
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

`<!-- TODO: add screenshot or screen recording of the login page -->`

1. **Login** — Navigate to `http://localhost:5173`. You are redirected to the Keycloak login
   page. Sign in with `testuser` / `testpass`.

   `<!-- TODO: add screenshot: Keycloak login screen -->`

2. **Pipelines list** — After login you land on the Pipelines page, which lists all defined
   pipelines fetched via the generated API client.

   `<!-- TODO: add screenshot: Pipelines list page -->`

3. **Create a pipeline** — Click **New Pipeline**, fill in the pipeline name and add stages
   (name, script type, bash command, order).

   `<!-- TODO: add screenshot or video: pipeline creation form -->`

4. **Run a pipeline** — Open a pipeline and click **Execute**. The build appears in the
   Builds list with state `WAITING → RUNNING`.

   `<!-- TODO: add screenshot: pipeline execution triggered -->`

5. **Monitor build progress** — Navigate to the build detail page. Stage states and logs
   update in real time via Server-Sent Events (`/api/v1/pipeline/build/sse/subscribe/{id}`).

   `<!-- TODO: add screenshot or video: live build progress and stage logs -->`

## OpenAPI-driven client code generation

The frontend never calls the backend directly with hand-written fetch calls. Instead:

1. **Source of truth** — `apis/openapi-spec/pipeline-service.yaml` is the single OpenAPI spec
   for the pipeline service.

2. **Code generation** — `openapitools.json` (in `pipeline-ui/`) configures
   `openapi-generator-cli` to produce a TypeScript-Axios client into `pipeline-ui/src/gen/`:

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

4. **Usage in the app** — Service files under `pipeline-ui/src/services/` import from
   `src/gen/` and pass the configured Axios instance (with the Keycloak bearer token
   injected via an interceptor in `axiosInstance.ts`).

5. **Mock server** — The same spec drives the Prism mock server (`npm run mock`), so the UI
   can be developed and tested without a running backend.

> Re-run `generate:clean-client-code` any time `pipeline-service.yaml` changes.

## Monitoring stack

The full observability stack (Grafana, Loki, Prometheus, Alloy) can be started alongside
the application using additional compose files:

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
