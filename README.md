# AI Voice Engine

Korean AI voice-call MVP template for a browser Caller Lab, Python voice
runtime, explicit business tools, durable call events, and a Supervisor
Dashboard.

## Goal

This repository builds a small but real demo where a user starts a Korean voice
consultation in the browser, talks to an AI agent through LiveKit, and reviews
the transcript, tool calls, status changes, latency, and final outcome in a
Supervisor Dashboard.

The project is also intended to be a reusable MVP template for future voice-call
projects. Keep the implementation small, but keep service boundaries,
persistence adapters, logging, and deployment habits production-minded.

## MVP Scope

- React 19 Caller Lab and Supervisor Dashboard.
- Browser microphone input and speaker output.
- LiveKit-based WebRTC session.
- Python voice runtime.
- One realtime voice model provider adapter.
- Korean sample 상담 prompt.
- Barge-in and AI response cancellation.
- Mock customer verification and billing lookup tools.
- FastAPI Tool API.
- PostgreSQL-backed call sessions, utterances, events, tool executions, and
  metrics.
- Structured JSON stdout logs and DB-backed domain audit events.
- Docker Compose local runtime.
- Manual AWS demo deployment path using Caddy, ALB/ACM, and clean local Docker
  image builds.

## Out Of Scope For MVP

- PSTN/SIP phone numbers and telecom carrier integrations.
- Kubernetes, Kafka, Redis, Neo4j, vector databases, and generic RAG.
- Real `contexture-bridge` integration.
- Multi-agent orchestration.
- Human agent voice takeover.
- Custom STT/TTS/model training.
- Complex authentication, organizations, RBAC, production billing, or
  multi-tenancy.
- Real customer personal data.

Out-of-scope items may be documented as extension points, but they should not be
implemented before the MVP is complete.

## Security Notes

- Do not commit `.env`, `.env.local`, private keys, provider API keys, LiveKit
  secrets, database passwords, image tarballs, or local deployment backup files.
- Browser-exposed variables must be public URLs only.
- Server-side secrets must stay in server/runtime environment variables.
- Do not store raw audio, real PII, API keys, or full env files in logs,
  transcripts, database rows, or audit events.

## Source Of Truth

Read the implementation plan before changing behavior:

- `docs/ai-voice-engine-kor-codex-implementation-plan.md`

Read agent and design guidance before implementation:

- `AGENTS.md`
- `DESIGN.md`
