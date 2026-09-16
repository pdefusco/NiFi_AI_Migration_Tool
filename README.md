# NiFi_AI_Migration_Tool

Agentic AI tool for migrating NiFi 1.x flows to NiFi 2.x on CDP (Cloudera Flow Management / Cloudera DataFlow).

## Status

Pre-build. Currently in the customer discovery phase.

## Documents

- **[DISCOVERY.md](DISCOVERY.md)** — customer discovery questionnaire, proposed architecture, and v1 success criteria. Working artifact for the discovery phase; comments and edits welcome.

## Dev environment

- **[dev/nifi1-local/](dev/nifi1-local/)** — one-command local Apache NiFi 1.28.1 + NiFi Registry via Docker Compose, for building and testing the migration tool against a real NiFi 1 REST API without any cloud infrastructure.
