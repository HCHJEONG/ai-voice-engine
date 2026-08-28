# AI Voice Engine Agent Instructions

Treat `docs/ai-voice-engine-kor-codex-implementation-plan.md` as the source of
truth for this repository. If these instructions appear to conflict with that
document, follow the document and update this file later.

This project is an MVP demo for a Korean AI voice 상담 flow: a browser Caller
Lab connects through LiveKit to a Python voice runtime, calls explicit mock
business tools through a FastAPI Tool API, stores structured call history in
PostgreSQL, and shows the conversation, tool calls, events, latency, and outcome
in a Supervisor Dashboard.

## Work Scope

- Preserve the intended monorepo shape:
  - `apps/console` for the React 19 web app.
  - `services/voice-runtime` for the Python LiveKit voice runtime.
  - `services/tool-api` for FastAPI, tools, persistence, and dashboard APIs.
  - `contracts`, `prompts`, `evals`, `infra`, and `docs` for their planned
    responsibilities.
- Do not split the project into multiple repositories.
- Do not implement PSTN/SIP, Twilio or other phone carriers, Kubernetes, Kafka,
  Redis, Neo4j, vector databases, generic RAG, `contexture-bridge` integration,
  multi-agent orchestration, real agent voice takeover, custom STT/TTS/model
  training, complex auth, production billing, or multi-tenancy unless the
  implementation plan is explicitly changed.
- Build a working vertical slice before broadening features.
- Prefer simple, explicit code over broad abstractions, while keeping provider
  adapter boundaries clear.
- Keep model SDKs inside provider adapters. Do not scatter provider-specific
  details through domain or application code.
- The LLM must not read or write the database directly. It may only use
  explicitly registered tools whose arguments are validated.

## Implementation Order

- Follow the phases in `docs/ai-voice-engine-kor-codex-implementation-plan.md`.
- Keep only one phase `in_progress` at a time.
- Inside a phase, work in the smallest useful vertical units.
- Do not connect a realtime model before the LiveKit echo or fixed-response
  media round trip is verified.
- Do not treat placeholder commands, fake tests, or unverified external-provider
  behavior as completed work.
- At the end of each meaningful turn, report changed files, verification run,
  unverified items, and remaining risk.
- At phase completion, use the reporting format from the implementation plan:

```text
Phase N 완료

- 구현:
- 주요 파일:
- 검증:
- 수동 확인 필요:
- 알려진 한계:
- 다음 Phase:
```

## Local Runtime

- Docker Compose is the default full-runtime path.
- The final Compose stack should include:
  - `console`
  - `tool-api`
  - `voice-runtime`
  - `postgres`
  - `livekit`
- PostgreSQL is the authoritative persistent store for call sessions,
  utterances, events, tool executions, and metrics.
- PostgreSQL is the default persistence adapter, not the application boundary.
  Application code should depend on repository and unit-of-work ports; keep
  SQLAlchemy, PostgreSQL SQL, JSONB behavior, and dialect details inside the
  persistence adapter.
- Assume a future customer may require a different RDBMS. Keep core application
  logic portable across common relational databases where practical.
- Do not store realtime audio frames in PostgreSQL.
- Original voice recordings are out of scope for the MVP unless the plan is
  changed.
- Browser-to-backend calls should use the configured public API URL, such as
  `VITE_API_BASE_URL=http://localhost:8000`.
- Browser LiveKit connections should use the configured public LiveKit URL,
  such as `VITE_LIVEKIT_URL=ws://localhost:7880`.
- Server-side LiveKit credentials must stay server-side and must never be
  exposed to the browser bundle.

## Data And Events

- Store all important events with `call_id`, `event_id` or event UUID, and
  `occurred_at`.
- Store timestamps in UTC on the server and present them in local time in the
  UI.
- Keep the planned data concepts aligned with the implementation plan:
  `call_sessions`, `utterances`, `call_events`, `tool_executions`, and
  `call_metrics`.
- Keep call status and outcome values compatible with the plan unless a
  documented migration updates them.
- Record both successful and failed tool executions.
- Validate every external input, including model-produced tool arguments.
- Do not record API keys, secrets, real customer personal data, or browser
  secrets in logs, database rows, transcripts, or events.

## Logging And Observability

- Build structured JSON stdout logging into every service from the beginning.
- Distinguish operational logs from domain audit events:
  - operational logs are for debugging runtime behavior and should go to stdout;
  - domain audit events are product data stored in `call_events`, `utterances`,
    `tool_executions`, and `call_metrics`.
- Include correlation fields whenever available: `service`, `event`,
  `request_id`, `call_id`, `room_name`, `tool_execution_id`, `provider`,
  `duration_ms`, and `error_code`.
- Do not bind the template implementation directly to CloudWatch, Azure
  Monitor, Google Cloud Logging, or any other cloud-vendor-specific logging
  service. Customers may require AWS, Azure, GCP, on-prem, or hybrid targets.
- Vendor-specific log shipping may be documented as an optional deployment
  adapter later, but it must not be required by application code.
- Do not store raw audio, secrets, provider API keys, LiveKit secrets, real PII,
  or full env files in logs or audit events.

## API And Realtime Behavior

- Keep API routes aligned with the minimum contract in the implementation plan.
- Use Server-Sent Events for Supervisor realtime updates in the MVP. Do not add
  a separate WebSocket server for dashboard updates.
- Caller Lab must clean up room, track, and media resources after refresh,
  disconnect, or call end.
- Call start and end handling must be idempotent where practical.
- Ended calls must not move back to `active`.
- Provider failures should become call failure events instead of crashing the
  whole application.
- Barge-in is core MVP behavior: when customer speech starts during agent audio,
  cancel the provider response, stop playback as much as possible, record
  `agent.response_cancelled`, and keep the session usable.

## Tool And Prompt Rules

- Implement mock customer and billing tools as deterministic fixtures.
- Do not provide billing details before customer verification.
- Tool results should include both human-readable explanations and structured
  amount data.
- Korean 상담 prompts should use polite language, avoid guessing unknown
  details, disclose tool failures, and keep responses short enough for voice.
- If a case cannot be solved, end with `handoff_required` rather than inventing
  a resolution.

## Frontend Guidance

- Build the actual Caller Lab and Supervisor Dashboard workflows, not a
  marketing landing page.
- Prioritize clear operational state over decorative UI.
- The Caller Lab must show scenario title, start/end controls, microphone
  state, connection state, AI/customer speaking state, realtime transcript, and
  a warning not to enter real personal information.
- The Supervisor Dashboard must show active/completed calls, selected call
  status and outcome, speaker-separated transcript, tool execution cards, event
  timeline, latency metrics, and interruption count.
- Keep the dashboard readable at 1280px. Full mobile support is not required for
  the MVP, but layouts should not obviously break on narrower screens.
- Use API data rather than static frontend fixtures once the relevant backend
  phase exists.
- Clearly distinguish fixture/demo data from real customer data in the UI and
  README.

## Secrets And Runtime Files

- Commit `.env.example` only with safe placeholders or local-only defaults.
- Do not commit `.env`, `.env.local`, private keys, provider API keys, LiveKit
  secrets, or database passwords.
- Do not hardcode API keys or long-lived secrets in Compose files, source code,
  frontend bundles, logs, tests, or documentation examples.
- Keep runtime secrets in environment variables or local runtime files excluded
  from Git.
- External model API keys are required for real voice-provider verification;
  absence of a key must be reported as unverified, not silently passed.
- Before adding a provider default model identifier, confirm it against current
  official SDK documentation and avoid permanently pinning a preview model name
  as a code default.

## Deployment Boundaries

- The MVP target is local Docker Compose, not production cloud deployment.
- AWS deployment, when requested, is manual and Docker-based. Keep it separate
  from the core local MVP runtime.
- The default cost-conscious AWS demo target may be the existing public
  `aws-bastion` instance. This is not the ideal security separation for a
  bastion, but it is allowed for demo cost control when the user accepts that
  tradeoff.
- Do not add CI/CD, registry pushes, hosted deployment pipelines, Kubernetes, or
  automatic deployment hooks unless the user explicitly asks and the
  implementation plan is updated.
- Do not run remote push, deployment, or paid infrastructure creation unless the
  user explicitly requests it.
- Do not use Kubernetes as the default AWS deployment layer.
- Do not add NLB or NAT Gateway by default. First validate the existing ALB plus
  direct public EC2 port strategy for LiveKit media traffic.
- AWS load balancers or reverse proxies, if later added, must stay outside the
  application contract and must not require changes to the MVP API semantics.
- Keep application containers separate from data containers.
- A normal application redeploy must replace only application containers and
  must not recreate PostgreSQL volumes or runtime data.
- Build images from a clean source state when preparing a remote deployment.
- Image tarballs are transfer artifacts only. They must not be committed and
  should be deleted locally and remotely after the target host has loaded them.
- Runtime deployment must use Docker or Docker Compose on the target host.
- `t3a.micro` may be used for an initial single-user demo, but if memory
  pressure, swap use, container instability, or voice latency appears, move to
  `t3a.small` immediately at that point.
- Verify deployment with health checks after container replacement:
  console HTTP, Tool API health, voice-runtime process health where available,
  PostgreSQL connection, and LiveKit availability.
- Keep deployment commands target-specific. Script names should make the target
  obvious, such as `deploy-aws-demo.sh`.

## Manual AWS Deployment Strategy

- Place deployment scripts under `.fordeploy/`.
- Keep manual runbook documentation beside scripts in `.fordeploy/README.md`.
- Use `.fordeploy/aws-backup/` as the local staging and backup area for
  deployment-time files that must not live in the application source tree.
- Treat `.fordeploy/aws-backup/` as local runtime material, not application
  source.
- Real files under `.fordeploy/aws-backup/` must be ignored by Git by default.
  Commit only deliberate documentation or sanitized examples such as
  `.env.example`, `*.example`, or `README.md`.
- Docker build contexts must exclude `.fordeploy/` so env files, credentials,
  image archives, and local runtime material cannot be copied into images.
- For remote hosts behind a Bastion, prefer this path: build locally, save image
  tarballs, copy tarballs to the Bastion, copy them to the private host, load
  images on the private host, replace application containers, verify, then clean
  up transfer artifacts.
- When `aws-bastion` itself is the deployment target, copy image tarballs
  directly to `aws-bastion`, load them there, replace application containers
  there, verify, then clean up transfer artifacts there.
- Local Docker images for AWS deployment should be built locally from a clean
  clone or another explicitly selected source directory, saved as tarballs, and
  transferred through the Bastion/private-host path.
- Deployment scripts must treat `.fordeploy/aws-backup/` paths as local absolute
  paths before copying anything to AWS. Do not rely on the caller's current
  directory for secret or backup file resolution.
- Before copying any file from `.fordeploy/aws-backup/` to an AWS host, the
  script must print the source absolute path, destination host, and destination
  absolute path, then ask for terminal confirmation with a clear `y/N` prompt.
- If the operator answers anything other than `y` or `Y`, the script must skip
  that file transfer or abort the requested credential/bootstrap step without
  treating it as a deployment failure.
- Scripts must never print the contents of environment files or credential
  files. They may print filenames, byte sizes, checksums, timestamps, and target
  paths.
- AWS-side runtime files should live under a repository-specific runtime root,
  such as `/home/ubuntu/ai-voice-engine`.
- AWS-side image archives should live only temporarily under a
  repository-specific image root, such as
  `/home/ubuntu/docker_images/ai-voice-engine`.

## Script Deployment Strategy

- Keep scripts readable, conservative, and idempotent where practical.
- Use `set -euo pipefail` in shell scripts.
- Make image names, tags, container names, ports, remote paths, health check
  paths, Bastion host, private host, SSH user, and SSH key paths configurable
  through environment variables with safe defaults where possible.
- Print each major step before executing it: build, save, transfer, load,
  replace, health check, and cleanup.
- Never print secrets or full env files.
- Do not commit generated image archives, copied env files, credential files, or
  target-host backups.
- A deployment script may replace the `console`, `tool-api`, and
  `voice-runtime` application containers.
- A deployment script must not remove PostgreSQL volumes or LiveKit/PostgreSQL
  runtime state unless it is explicitly named and documented as a destructive
  development reset script.
- Scripts that need runtime env files should support an explicit bootstrap step
  rather than silently copying local files.
- A bootstrap step should read candidate files from `.fordeploy/aws-backup/`,
  resolve them to absolute local paths, show the planned AWS destination, and
  require `y/N` confirmation for each transfer.
- Scripts should separate app image deployment from secret/runtime-file
  transfer. Re-deploying an application image should not overwrite a target-host
  env file unless the operator explicitly confirms that overwrite.
- Scripts should fail fast when required runtime files are missing on AWS, but
  the failure message should describe the expected target path instead of
  dumping local secret contents.

## Verification

- Start each task by checking repository status and existing changes.
- Do not overwrite or delete user changes.
- Do not run `git commit` or `git push` unless the user explicitly asks for that
  specific action. The user normally commits and pushes manually.
- At the end of code or documentation work, suggest a concise commit message
  instead of committing.
- After file edits, run relevant tests, lint, type checks, or endpoint/browser
  checks when feasible.
- Run `git diff --check` at the end of each phase.
- Use the planned commands as they become available:
  - `make install`
  - `make lint`
  - `make typecheck`
  - `make test`
  - `make test-api`
  - `make test-web`
  - `make test-e2e-supervisor`
  - `make test-e2e-media`
- For Docker changes, verify `docker compose up --build` and health checks when
  feasible.
- For database changes, verify migrations from an empty database and test
  isolation.
- For frontend media behavior, use Playwright fake media where possible and
  document any required manual microphone/speaker checks.
- For external model behavior, separate tests that require paid/provider access
  from default CI-safe tests.
