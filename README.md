# Mojaloop Learning

Notes, guides, and Postman collections for learning the [Mojaloop](https://mojaloop.io) open-source payment switch by running it locally and tracing real API calls through the system.

## Goal

Stand up the Mojaloop core switch on a local machine with Docker, then walk through the hub setup → DFSP onboarding → transfer flow using Postman, understanding what each API call does and why, end to end.

## Repo layout

| Path | Contents |
|---|---|
| [`setup/MOJALOOP-AI-DEPLOY.md`](setup/MOJALOOP-AI-DEPLOY.md) | Step-by-step (AI-agent-executable) guide to deploy [`ml-core-test-harness`](https://github.com/mojaloop/ml-core-test-harness) locally via Docker Compose, including known fixes, health checks, and a recovery table for common failures. |
| [`postman/`](postman/) | Postman collections and environment for exercising the running stack, plus written walkthroughs of each collection. |
| [`postman/environments/`](postman/environments) | `Mojaloop-Local-Docker-Compose.postman_environment.json` — environment variables for the local Docker setup. |
| [`postman/collections/setup/`](postman/collections/setup) | `MojaloopHub_Setup` collection: configures the hub itself — hub ledger accounts, notification emails, settlement model, oracle registration. See [`setup/README.md`](postman/collections/setup/README.md) for a full request-by-request guide and glossary. |
| [`postman/collections/onboarding/`](postman/collections/onboarding) | `MojaloopSims_Onboarding` collection: onboards a DFSP (participant) — creates accounts, sets Net Debit Cap and initial position, registers callback URLs, funds-in liquidity. |
| [`postman/collections/golden_path/`](postman/collections/golden_path) | `Golden_Path_Mojaloop` collection: end-to-end P2P transfer — party lookup, quote, transfer — plus related happy-path and error cases. |

## Learning path

1. **Deploy the stack** — follow [`setup/MOJALOOP-AI-DEPLOY.md`](setup/MOJALOOP-AI-DEPLOY.md) to get Mojaloop core services and the Testing Toolkit running locally.
2. **Configure the hub** — run `MojaloopHub_Setup` in Postman; read [`postman/collections/setup/README.md`](postman/collections/setup/README.md) alongside it to understand each call (hub accounts, settlement models, oracles) and the core Mojaloop vocabulary (DFSP, POSITION/SETTLEMENT accounts, Net Debit Cap, settlement windows, FSPIOP vs admin APIs).
3. **Onboard a DFSP** — run `MojaloopSims_Onboarding` to register a simulated participant, set its limits, register callbacks, and deposit liquidity.
4. **Run a transfer** — run `Golden_Path_Mojaloop` to trace a full discovery → quote → transfer flow between two DFSPs.

## What this is not

This is a personal learning and reference repo, not a production deployment guide or an official Mojaloop resource. Links to upstream Mojaloop repos (`mojaloop/ml-core-test-harness`, `mojaloop/postman`) are the source of truth for the actual software; the content here is notes and explanations built while learning it.
