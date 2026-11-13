# Postman Collections

Collections and environment for exercising a locally-deployed Mojaloop switch (see [`setup/MOJALOOP-AI-DEPLOY.md`](../setup/MOJALOOP-AI-DEPLOY.md) for deployment). Sourced from [mojaloop/postman](https://github.com/mojaloop/postman).

## Environment

Import [`environments/Mojaloop-Local-Docker-Compose.postman_environment.json`](environments/Mojaloop-Local-Docker-Compose.postman_environment.json) and select it in Postman before running any collection below.

## Collections, in run order

| Order | Collection | Purpose | Guide |
|---|---|---|---|
| 1 | [`collections/setup/MojaloopHub_Setup`](collections/setup/MojaloopHub_Setup.postman_collection.json) | Configures the hub: ledger accounts, notification emails, settlement model, oracle registration | [`collections/setup/README.md`](collections/setup/README.md) |
| 2 | [`collections/onboarding/MojaloopSims_Onboarding`](collections/onboarding/MojaloopSims_Onboarding.postman_collection.json) | Onboards a DFSP (participant): accounts, Net Debit Cap and initial position, callback URLs, funds-in | — |
| 3 | [`collections/golden_path/Golden_Path_Mojaloop`](collections/golden_path/Golden_Path_Mojaloop.postman_collection.json) | End-to-end P2P transfer: party lookup, quote, transfer, plus happy-path and error cases | — |

Each collection builds on the state created by the previous one, so run them in order against a freshly deployed stack.

## Notes

- `MojaloopHub_Setup` and `MojaloopSims_Onboarding` use the **admin APIs** (`/participants`, `/settlementModels`, `/oracles`), which are synchronous — the HTTP response is the final answer.
- `Golden_Path_Mojaloop` exercises the **FSPIOP API** (`/parties`, `/quotes`, `/transfers`), which is asynchronous — requests return `202 Accepted` and the real result arrives via callback, visible in the Testing Toolkit UI.
- See [`collections/setup/README.md`](collections/setup/README.md) for a full glossary of Mojaloop terms (DFSP, POSITION/SETTLEMENT accounts, settlement models, oracles, etc.) used across all three collections.
