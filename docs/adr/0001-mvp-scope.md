# ADR 0001: MVP Scope

## Status

Accepted

## Context

The project needs to prove a complete Korean AI voice-call flow without becoming
a call-center product, telecom platform, or large enterprise integration stack.
It should also remain useful as a future MVP template for field delivery work.

## Decision

The MVP will focus on one end-to-end browser voice 상담 scenario:

- A user starts a call in Caller Lab.
- Browser audio connects through LiveKit.
- A Python voice runtime manages the realtime model session.
- The model may call explicit mock business tools through the Tool API.
- PostgreSQL stores call sessions, utterances, tool executions, events, and
  metrics.
- Supervisor Dashboard shows the live and completed call timeline.

The MVP will not implement PSTN/SIP, carrier integrations, Kubernetes, Kafka,
Redis, Neo4j, vector databases, generic RAG, `contexture-bridge`, multi-agent
orchestration, human voice takeover, custom model training, complex auth,
production billing, multi-tenancy, or real customer personal data.

## Consequences

- Work should proceed by small verified phases, starting with repository hygiene,
  scaffolding, Tool API persistence, mock tools, dashboard visibility, LiveKit
  media validation, then the realtime provider.
- PostgreSQL is the default persistence adapter, but application code should
  depend on repository and unit-of-work boundaries so another RDBMS can be
  considered for customer deployments.
- Logging must start as vendor-neutral structured JSON stdout logs, while
  domain audit events are stored in PostgreSQL for Supervisor and traceability.
- AWS demo deployment may use the existing public `aws-bastion` host with Caddy
  and ALB/ACM, avoiding default NLB or NAT additions unless network validation
  proves they are needed.
