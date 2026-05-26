# Flowise OTA Pressure Matrix

## Scope

- Repo: `FlowiseAI/Flowise`
- Contract: [`ota.yaml`](./ota.yaml)
- Evidence sources:
  - `README.md`
  - `CONTRIBUTING.md`
  - `.github/workflows/main.yml`
  - `docker/README.md`
  - `docker/docker-compose.yml`
  - `docker/docker-compose-queue-prebuilt.yml`
  - `docker/worker/README.md`
  - `docker/worker/docker-compose.yml`

## Coverage Matrix

| Surface | Repo truth | OTA mapping | Current pressure result |
| --- | --- | --- | --- |
| Install | `pnpm install` (README + CI) | `task: install` (`corepack pnpm install`) | Contract-valid; blocked locally by Node version policy (`^20.20.2`) |
| Lint | `pnpm lint` (CI) | `task: lint` | Contract-valid; blocked locally by Node version policy |
| Build | `pnpm build` (README + CI) | `task: build` | Contract-valid; blocked locally by Node version policy |
| Typecheck | No standalone root script; TypeScript compile is via package build scripts | `task: typecheck` (package build chain) | Contract-valid; blocked locally by Node version policy |
| Test | `pnpm test` | `task: test` | Contract-valid; blocked locally by Node version policy |
| Test Coverage | `pnpm test:coverage` (CI) | `task: test:coverage` | Contract-valid; blocked locally by Node version policy |
| CI Verify Sequence | `lint -> build -> test:coverage` (CI workflow) | `task: verify:ci`, `workflow: verify` | Contract-valid; blocked locally by Node version policy |
| Dev Startup (native) | `pnpm dev` (README/CONTRIBUTING) | `task: dev`, `workflow: app` | Contract-valid; blocked locally by Node version policy |
| Prod Startup (native) | `pnpm start` after build | `task: start`, `workflow: app:prod` | Contract-valid; blocked locally by Node version policy |
| Service Startup (docker default) | `cd docker && docker compose up` | `task: dev:docker`, `workflow: app:docker` | Runnable in dry-run; warning about `.ota/*` gitignore hygiene |
| Service Startup (docker queue) | `cd docker && docker compose -f docker-compose-queue-prebuilt.yml up` | `task: dev:docker:queue`, `workflow: app:docker:queue` | Runnable in dry-run; warning about `.ota/*` gitignore hygiene |
| Service Startup (docker worker) | `cd docker/worker && docker compose up` | `task: dev:docker:worker`, `workflow: worker:docker` | Runnable in dry-run; warning about `.ota/*` gitignore hygiene |
| Health Check (API) | `/api/v1/ping` returns `pong` | `surface: api` | Modeled and wired to run workflows |
| Health Check (app HTML) | `/` serves UI on 3000 | `surface: app` | Modeled and wired to run workflows |
| Health Check (dev UI) | Dev UI on `http://localhost:8080` | `surface: dev-ui` | Modeled and wired to `workflow: app` |
| Health Check (worker) | Worker `/healthz` on `WORKER_PORT` in worker compose | `surface: worker` | Modeled and wired to `workflow: worker:docker` |

## Pressure Findings (OTA Gaps)

Gap | Severity | Where | Observed | Expected | Impact | Suggested fix
--- | --- | --- | --- | --- | --- | ---
Global optional tool leakage into unrelated task previews | Medium | `ota run <non-docker-task> --dry-run` requirement rendering | `tool docker` appears in requirement lines for Node-only tasks (for example `build`, `verify:ci`) once `docker` is declared at contract level, even when task does not require Docker | Requirement output should be scoped to selected task/workflow requirements (plus explicit defaults), not every globally declared optional tool | Operators and agents can misread readiness blockers and think Docker is part of all native task paths | In plan rendering, filter optional global tools unless selected task/workflow directly or transitively requires them
Task-level tool requirements depend on global tool declarations | Medium | Contract validation (`ota validate`) | `requirements.tools.docker` on a task fails with `unknown tool requirement` unless `tools.docker` exists globally | Task-level requirement declaration should be self-describing or support inline tool references for narrow flows | Adds author friction and encourages overly broad global tool declarations that then leak into previews | Support task-local tool requirement blocks (or scoped declarations) while preserving canonical global registry behavior
No first-class readiness binding to internal container health when service port is not published | Medium | Queue prebuilt compose worker in `docker/docker-compose-queue-prebuilt.yml` | Worker has compose healthcheck but no host port mapping; current surface model cannot probe that worker health over host HTTP in `app:docker:queue` workflow | Ability to attach readiness to compose/container health state, not only host-reachable endpoints | Partial readiness proof for multi-service stacks where critical workers are healthy internally but not externally exposed | Add readiness probe kinds for container runtime health (for example compose service health/status) with deterministic backend support
Repeated `.ota/*` gitignore warning is manual to fix per-repo | Low | `ota doctor` warning path | Dry-run previews repeatedly warn on `.ota/state`, `.ota/receipts`, `.ota/proof` ignore hygiene unless repo is manually updated | A cleaner one-command adoption path for first pressure-test run | Extra friction/noise during repo pressure testing and onboarding | Provide a lightweight `ota init --hygiene` or doctor autofix profile that stages recommended ignore entries with explicit diff preview
