# Mojaloop Hub Setup: API Guide and Glossary

A walkthrough of the `MojaloopHub_Setup` Postman collection from [mojaloop/postman](https://github.com/mojaloop/postman). It covers what each API call does, why it is needed, and the Mojaloop terms involved.

> **Environment used:** Mojaloop core services running locally via [`ml-core-test-harness`](https://github.com/mojaloop/ml-core-test-harness) (Docker Compose), with ports published through a `docker-compose.override.yml`.

---

## Table of contents

1. [Big picture](#1-big-picture)
2. [Prerequisites and environment variables](#2-prerequisites-and-environment-variables)
3. [Common request conventions](#3-common-request-conventions)
4. [API calls in detail](#4-api-calls-in-detail)
   - [Folder 1: Hub Account](#folder-1-hub-account)
   - [Folder 2: Settlement Models](#folder-2-settlement-models)
   - [Folder 3: Oracle Onboarding](#folder-3-oracle-onboarding)
5. [Verification calls](#5-verification-calls)
6. [Glossary of terms](#6-glossary-of-terms)
7. [Troubleshooting](#7-troubleshooting)
8. [What comes next](#8-what-comes-next)

---

## 1. Big picture

Mojaloop is a **switch (hub)** that connects **DFSPs** (banks, mobile wallets, microfinance providers) so they can send money to each other. A transfer has three phases:

| Phase | Hub service | FSPIOP API | Question answered |
|---|---|---|---|
| **Discovery** | account-lookup-service (ALS) + oracle | `GET /parties/{type}/{id}` | Which DFSP holds this phone number or account? |
| **Agreement** | quoting-service | `POST /quotes` | What will it cost, and do both sides agree? |
| **Transfer** | ml-api-adapter + central-ledger | `POST /transfers` | Reserve the funds, then commit them |

Before any of this can happen, the **hub operator** must configure the switch. That is what `MojaloopHub_Setup` does:

```
┌──────────────────── HUB SETUP (operator, admin APIs, synchronous) ─────────────────┐
│ 1. HUB_MULTILATERAL_SETTLEMENT account ─┐                                           │
│ 2. HUB_RECONCILIATION account ──────────┼─► central-ledger: hub can do bookkeeping  │
│ 3-5. Hub notification emails ───────────┘   (alerts for NDC / settlement events)    │
│ 6. Settlement model DEFERREDNET ──────────► "how & when DFSPs settle"               │
│ 7-9. Register oracles ────────────────────► account-lookup: "who to ask for an ID"  │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

**Execution order matters:** hub accounts first, then the settlement model, then the oracles.

---

## 2. Prerequisites and environment variables

### Service ports (local Docker)

`ml-core-test-harness` keeps services inside the Docker network by default. Publish them with `docker-compose.override.yml` in the harness folder:

```yaml
services:
  central-ledger:
    ports: ["3001:3001"]
  ml-api-adapter:
    ports: ["3000:3000"]
  quoting-service:
    ports: ["3002:3002"]
  account-lookup-service:
    ports: ["4001:4001", "4002:4002"]   # 4001 = admin API, 4002 = FSPIOP API
  central-settlement:
    ports: ["3007:3007"]
```

Then, from the harness folder:

```bash
docker compose --profile all-services up -d central-ledger ml-api-adapter quoting-service account-lookup-service central-settlement
```

### Postman environment variables used by this collection

| Variable | Value (local) | Purpose |
|---|---|---|
| `HOST_CENTRAL_LEDGER` | `http://localhost:3001` | Central-ledger admin API |
| `BASE_CENTRAL_LEDGER_ADMIN` | *(empty, but must be defined)* | Optional path prefix (used behind an ingress) |
| `HOST_ACCOUNT_LOOKUP_ADMIN` | `http://localhost:4001` | ALS admin API (oracles) |
| `HOST_SIMULATOR` | URL of the oracle, reachable **from inside Docker** | Oracle endpoint registered in ALS |
| `HUB_OPERATOR_BEARER_TOKEN` | `NOT_APPLICABLE` | Auth token. Not checked locally. |
| `WS02_OAUTH_ENABLED` | `false` | Skips the OAuth token fetch in the pre-request script |
| `HUB_OPERATOR_EMAIL` | any email, e.g. `hub@example.com` | Notification email address |
| `hub_operator` | `NOT_APPLICABLE` / `hub_operator` | `FSPIOP-Source` header value |
| `currency` | `USD` | Currency the hub is being configured for |

> ⚠️ The shipped environment file names the token `HUBOPERATOR_BEARER_TOKEN`, but the collection uses `HUB_OPERATOR_BEARER_TOKEN`. Add the second name.
>
> ⚠️ A variable that is **defined with an empty value** resolves to an empty string, which is fine. A variable that is **not defined at all** stays as the literal text `{{...}}` and breaks the URL.

---

## 3. Common request conventions

| Header | Value | Why |
|---|---|---|
| `Authorization` | `Bearer {{HUB_OPERATOR_BEARER_TOKEN}}` | In production an API gateway validates the operator's token |
| `FSPIOP-Source` | `hub_operator` | Who is calling. Used for audit logs. |
| `Content-Type` | `application/json` (central-ledger) or `application/vnd.interoperability.participants+json;version=1.0` (ALS) | ALS follows FSPIOP header rules. Central-ledger admin takes plain JSON. |

**Postman tip:** in each POST request, set **Body → raw → JSON** (not *Text*). Otherwise Postman also sends `Content-Type: text/plain` and the request carries two conflicting Content-Type headers.

**These are admin APIs.** They are **synchronous**: the response you see is the final answer. This is unlike the FSPIOP APIs (`/parties`, `/quotes`, `/transfers`), which return `202 Accepted` and send the real result later as an asynchronous **callback**.

**No Postman tests:** the collection has no assertions. Judge success by the status code:
- `200` / `201`: success
- "already exists" error: the item was created before (for example by TTK provisioning). This is harmless.

---

## 4. API calls in detail

### Folder 1: Hub Account

#### Request 1: Add Hub Account (HUB_MULTILATERAL_SETTLEMENT)

```http
POST {{HOST_CENTRAL_LEDGER}}{{BASE_CENTRAL_LEDGER_ADMIN}}/participants/Hub/accounts
Content-Type: application/json
FSPIOP-Source: hub_operator
Authorization: Bearer {{HUB_OPERATOR_BEARER_TOKEN}}
```

```json
{
  "type": "HUB_MULTILATERAL_SETTLEMENT",
  "currency": "{{currency}}"
}
```

| Field | Meaning |
|---|---|
| `Hub` (in the URL) | The participant that owns the account. `Hub` is the built-in participant representing the switch operator. |
| `type` | Ledger account type to create. Only hub account types are accepted here. |
| `currency` | ISO-4217 currency code. Accounts are created **per currency**. |

**What central-ledger does:**
1. Validates that participant `Hub` exists and the currency is valid.
2. Links participant `Hub` to the currency.
3. Creates a ledger account of type `HUB_MULTILATERAL_SETTLEMENT` with a balance of 0.
4. Returns **`201 Created`** with an empty body.

**Why it matters:** this account is the hub's side of every **settlement**. When a settlement window is settled, each DFSP's net amount is posted against this account.

---

#### Request 2: Add Hub Account (HUB_RECONCILIATION)

Same URL and headers as request 1.

```json
{
  "type": "HUB_RECONCILIATION",
  "currency": "{{currency}}"
}
```

**Why it matters:** this account mirrors the **real bank account** that holds DFSPs' collateral. Every **funds-in** (a DFSP deposits liquidity) and **funds-out** (a DFSP withdraws) is posted against it.

Example of a funds-in of 10,000 USD by `payerfsp`:

| Account | Debit | Credit |
|---|---|---|
| Hub `HUB_RECONCILIATION` | 10,000 | |
| payerfsp `SETTLEMENT` | | 10,000 |

---

#### Requests 3–5: Hub notification endpoints (emails)

```http
POST {{HOST_CENTRAL_LEDGER}}{{BASE_CENTRAL_LEDGER_ADMIN}}/participants/Hub/endpoints
Content-Type: application/json
FSPIOP-Source: hub_operator
```

| # | Body `type` | Fires when |
|---|---|---|
| 3 | `SETTLEMENT_TRANSFER_POSITION_CHANGE_EMAIL` | A settlement changes a DFSP's position |
| 4 | `NET_DEBIT_CAP_ADJUSTMENT_EMAIL` | A DFSP's Net Debit Cap is changed |
| 5 | `NET_DEBIT_CAP_THRESHOLD_BREACH_EMAIL` | A DFSP's position crosses its alarm threshold, so it is close to its NDC |

```json
{
  "type": "SETTLEMENT_TRANSFER_POSITION_CHANGE_EMAIL",
  "value": "{{HUB_OPERATOR_EMAIL}}"
}
```

**Notes:**
- `/participants/{name}/endpoints` is the **same API** DFSPs later use to register their **callback URLs** (e.g. `FSPIOP_CALLBACK_URL_TRANSFER_POST`). For the hub, the "endpoints" are operator email addresses.
- Emails are sent by the notification service, which is not running in a local setup. Locally these calls only **store configuration**.
- Sending a call again **updates** the stored value.
- ⚠️ Requests 4 and 5 ship with a hard-coded email address from the original author. Replace it with `{{HUB_OPERATOR_EMAIL}}`.

---

### Folder 2: Settlement Models

#### Request 6: Create settlement model DEFERREDNET

```http
POST {{HOST_CENTRAL_LEDGER}}/settlementModels
Content-Type: application/json
FSPIOP-Source: {{hub_operator}}
```

```json
{
  "name": "DEFERREDNET",
  "settlementGranularity": "NET",
  "settlementInterchange": "MULTILATERAL",
  "settlementDelay": "DEFERRED",
  "requireLiquidityCheck": true,
  "ledgerAccountType": "POSITION",
  "settlementAccountType": "SETTLEMENT",
  "autoPositionReset": true,
  "currency": "{{currency}}"
}
```

| Field | Value | Effect |
|---|---|---|
| `name` | `DEFERREDNET` | Unique model name |
| `settlementGranularity` | `NET` | Transfers are settled as a net total (alternative: `GROSS`, one by one) |
| `settlementInterchange` | `MULTILATERAL` | Each DFSP's net is computed against **all** other DFSPs through the hub (alternative: `BILATERAL`, pair by pair) |
| `settlementDelay` | `DEFERRED` | Transfers finish immediately. Money is settled later, when a **settlement window** is closed and settled (alternative: `IMMEDIATE`) |
| `requireLiquidityCheck` | `true` | Before reserving a transfer, the hub checks that `position + amount ≤ Net Debit Cap`. If not, the transfer is rejected (error `4001`, payer FSP insufficient liquidity). |
| `ledgerAccountType` | `POSITION` | DFSP account type this model tracks. Every transfer moves POSITION accounts. |
| `settlementAccountType` | `SETTLEMENT` | DFSP account debited or credited when settlement happens |
| `autoPositionReset` | `true` | Positions are automatically reset after a window is settled |
| `currency` | `USD` | Currency this model applies to. `null` means the default for all currencies. |

**Why it matters:** a settlement model does nothing by itself. When a DFSP is onboarded (`POST /participants` with a currency), central-ledger uses the matching model to create the DFSP's **POSITION** and **SETTLEMENT** accounts. It also decides how liquidity is checked and how settlement windows behave.

> If a model with `currency: null` (e.g. `DEFAULTDEFERREDNET`) already exists, it already covers USD, and this request may be rejected as a conflict. That is expected.

#### Worked example of multilateral net settlement

Transfers during one settlement window:

| From → To | Amount |
|---|---|
| payerfsp → payeefsp | 100 |
| payeefsp → payerfsp | 30 |
| payerfsp → testfsp1 | 50 |

Net positions at window close:

| DFSP | Sent | Received | Net position | At settlement |
|---|---|---|---|---|
| payerfsp | 150 | 30 | **+120** (owes) | Pays 120 |
| payeefsp | 30 | 100 | **−70** (is owed) | Receives 70 |
| testfsp1 | 0 | 50 | **−50** (is owed) | Receives 50 |

The nets sum to zero. Only 3 movements are needed instead of 3 separate gross transfers, and every movement goes through `HUB_MULTILATERAL_SETTLEMENT`.

---

### Folder 3: Oracle Onboarding

#### Requests 7–9: Register MSISDN / ACCOUNT_ID / ALIAS oracle

```http
POST {{HOST_ACCOUNT_LOOKUP_ADMIN}}/oracles
Content-Type: application/vnd.interoperability.participants+json;version=1.0
Accept: application/vnd.interoperability.participants+json;version=1
Date: Thu, 12 Nov 2026 12:00:00 GMT
```

```json
{
  "oracleIdType": "MSISDN",
  "endpoint": {
    "value": "{{HOST_SIMULATOR}}/oracle",
    "endpointType": "URL"
  },
  "currency": "{{currency}}",
  "isDefault": true
}
```

| # | `oracleIdType` | Resolves |
|---|---|---|
| 7 | `MSISDN` | Mobile phone numbers |
| 8 | `ACCOUNT_ID` | Bank account numbers / IDs |
| 9 | `ALIAS` | Aliases such as nicknames or merchant codes |

| Field | Meaning |
|---|---|
| `oracleIdType` | The party identifier type this oracle is responsible for |
| `endpoint.value` | URL ALS calls to ask the oracle. It must be reachable **from inside the ALS container** (Docker service name, **not** `localhost`). In ml-core-test-harness, the MSISDN oracle is `http://als-msisdn-oracle-svc:3000`. |
| `endpoint.endpointType` | Always `URL` |
| `currency` | Optional. Lets different currencies use different oracles. |
| `isDefault` | Use this oracle when no currency-specific oracle matches |

**How the oracle is used later (discovery phase):**

```
Payer DFSP ──GET /parties/MSISDN/27713803912──► ALS
ALS ──"who owns MSISDN 27713803912?"──► MSISDN oracle
MSISDN oracle ──"payeefsp"──► ALS
ALS ──GET /parties/MSISDN/27713803912──► payeefsp (its registered callback URL)
payeefsp ──PUT /parties/MSISDN/27713803912 {name, account…}──► ALS ──► Payer DFSP
```

**Header notes:**
- ALS validates FSPIOP headers. Sending plain `application/json` returns error `3102 Missing mandatory element`.
- The collection sends an **empty `Date` header**. Set a real HTTP date, or add this to the request's *Pre-request Script*:

  ```js
  pm.request.headers.upsert({ key: 'Date', value: new Date().toUTCString() });
  ```

---

## 5. Verification calls

After running the collection, check what was created:

| Check | Request | Expect |
|---|---|---|
| Hub accounts | `GET {{HOST_CENTRAL_LEDGER}}/participants/Hub/accounts` | `HUB_MULTILATERAL_SETTLEMENT` and `HUB_RECONCILIATION` for your currency |
| Hub emails | `GET {{HOST_CENTRAL_LEDGER}}/participants/Hub/endpoints` | The 3 email endpoint types |
| Settlement models | `GET {{HOST_CENTRAL_LEDGER}}/settlementModels` | `DEFERREDNET` (or a default model with `currency: null`) |
| Oracles | `GET {{HOST_ACCOUNT_LOOKUP_ADMIN}}/oracles` (with the ALS headers above) | `MSISDN`, `ACCOUNT_ID`, `ALIAS` |
| Service health | `GET {{HOST_CENTRAL_LEDGER}}/health` | `{"status":"OK", ...}` |

---

## 6. Glossary of terms

### Participants and roles

| Term | Meaning |
|---|---|
| **Hub / Switch** | The central Mojaloop system that routes messages and keeps the ledger between DFSPs |
| **Hub operator** | The organisation running the hub. It calls admin APIs (`FSPIOP-Source: hub_operator`). |
| **DFSP** | *Digital Financial Service Provider*: a bank, wallet or other provider connected to the hub |
| **Participant** | Any entity registered in central-ledger: each DFSP and the special `Hub` participant |
| **Payer / Payer FSP** | The person sending money and their DFSP |
| **Payee / Payee FSP** | The person receiving money and their DFSP |
| **Party** | An end user (person or business) identified by an ID type and value, e.g. `MSISDN / 27713803912` |

### Hub ledger account types

| Term | Owner | Meaning |
|---|---|---|
| **HUB_MULTILATERAL_SETTLEMENT** | Hub | Counter-account for settlement. When a settlement window is settled, each DFSP's net position is posted against it. Its balance returns to zero once all DFSPs have settled. |
| **HUB_RECONCILIATION** | Hub | Counter-account for **funds-in / funds-out**. It mirrors real money held at the settlement bank. Its balance equals the total liquidity DFSPs have deposited. |

### DFSP ledger account types (created during onboarding)

| Term | Meaning |
|---|---|
| **POSITION** | Running total of what a DFSP **owes** (positive) or **is owed** (negative) since the last settlement. Every transfer changes it. |
| **SETTLEMENT** | The DFSP's deposited collateral (liquidity) held for settlement. Funds-in increases it. Settlement draws on it. |
| **INTERCHANGE_FEE** | Optional account for interchange fees between DFSPs |

### Liquidity and risk

| Term | Meaning |
|---|---|
| **Net Debit Cap (NDC)** | Maximum net amount a DFSP may owe before its transfers are rejected. Normally ≤ its SETTLEMENT balance. |
| **Liquidity check** | Hub check at transfer *prepare*: `current position + reserved + amount ≤ NDC` |
| **Funds-in** | A DFSP deposits real money at the settlement bank. The hub credits its SETTLEMENT account. |
| **Funds-out** | A DFSP withdraws money. The hub debits its SETTLEMENT account. |
| **Initial position and limits** | Onboarding call that sets a DFSP's starting position and its NDC |
| **Threshold alarm** | Percentage of the NDC at which `NET_DEBIT_CAP_THRESHOLD_BREACH_EMAIL` fires |

### Settlement

| Term | Meaning |
|---|---|
| **Settlement** | The actual movement of real money between DFSPs to cover the transfers recorded by the hub |
| **Settlement model** | Rules for how and when settlement happens (granularity, interchange, delay, account types) |
| **Granularity: GROSS / NET** | Settle each transfer individually, or settle the net total |
| **Interchange: BILATERAL / MULTILATERAL** | Net between each pair of DFSPs, or net each DFSP against the hub as a whole |
| **Delay: IMMEDIATE / DEFERRED** | Settle in real time, or later in batches |
| **DEFERREDNET** | The common model: deferred, net, multilateral |
| **Settlement window** | A time period that collects transfers. It is *closed* and then *settled* by the operator via central-settlement. |
| **autoPositionReset** | After settlement, DFSP positions reset automatically |

### Account lookup

| Term | Meaning |
|---|---|
| **ALS (Account Lookup Service)** | Hub service that finds which DFSP owns a party identifier |
| **Oracle** | A directory service that maps identifiers (e.g. phone numbers) to DFSPs. ALS calls it. |
| **MSISDN** | *Mobile Station International Subscriber Directory Number*: a phone number in international format |
| **ACCOUNT_ID** | Identifier type for bank account numbers |
| **ALIAS** | Identifier type for aliases such as nicknames or merchant codes |
| **isDefault** | Marks the oracle used when no currency-specific oracle exists |

### API and protocol

| Term | Meaning |
|---|---|
| **FSPIOP API** | *FSP Interoperability API*: the asynchronous REST API DFSPs use (`/parties`, `/quotes`, `/transfers`) |
| **Admin API** | Synchronous operator APIs (`/participants`, `/settlementModels`, `/oracles`) |
| **Callback** | The async response: the hub or a DFSP sends `PUT /resource/{id}` to the URL the other side registered |
| **Endpoint (participant)** | A registered URL or email for a participant: DFSP callback URLs, or hub alert emails |
| **FSPIOP-Source / FSPIOP-Destination** | Headers naming the sending and receiving participant |
| **`202 Accepted`** | Immediate response to an FSPIOP request. The real result arrives by callback. |
| **ILP packet / condition / fulfilment** | Interledger cryptographic elements: the quote returns a *condition*, and the payee proves the transfer with a matching *fulfilment* |
| **Prepare / Fulfil / Commit** | Transfer phases: payer FSP *prepares* (funds reserved), payee FSP *fulfils*, hub *commits* (final) |

### Hub services

| Service | Port (local) | Role |
|---|---|---|
| **central-ledger** | 3001 | Participants, accounts, positions, transfers, settlement models |
| **ml-api-adapter** | 3000 | FSPIOP entry point for `/transfers` |
| **quoting-service** | 3002 | `/quotes` |
| **account-lookup-service** | 4001 (admin), 4002 (API) | `/parties`, `/participants`, `/oracles` |
| **central-settlement** | 3007 | Settlement windows and settlements (`/v2`) |
| **Testing Toolkit (TTK)** | 9660 (UI) | Simulators, test runner, reports |

---

## 7. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| URL contains literal `{{BASE_...}}` | Variable not defined in the selected environment | Add it with an empty value. Select the environment. |
| `getaddrinfo ENOTFOUND central-ledger.local` | `.local` hostnames do not resolve on Windows | Use `http://localhost:<port>` or add entries to the `hosts` file |
| `ECONNREFUSED 127.0.0.1:3001` | Port not published from the Docker container | Add `docker-compose.override.yml` and recreate the services |
| `socket hang up` | Container still starting after a restart | Wait about a minute and resend. Check `/health`. |
| Two `Content-Type` headers | Body type set to *Text* | Body → raw → **JSON** |
| `3102 Missing mandatory element` (ALS) | Missing or empty FSPIOP headers (`Content-Type`, `Date`) | Use the FSPIOP content type and a real `Date` |
| "Already exists" errors | Created earlier (e.g. by TTK provisioning) | Harmless. Verify with the GET calls. |
| `401` / `403` | `WS02_OAUTH_ENABLED` is not `false`, or wrong token variable name | Set it to `false` and add `HUB_OPERATOR_BEARER_TOKEN` |

---

## 8. What comes next

| Order | Collection | Builds |
|---|---|---|
| ✅ 1 | `MojaloopHub_Setup` | Hub accounts, alert emails, settlement model, oracles |
| 2 | `MojaloopSims_Onboarding` | For each DFSP: create the participant (POSITION + SETTLEMENT accounts), set the NDC and initial position, register callback URLs, funds-in, register parties in the oracle |
| 3 | `Golden_Path_Mojaloop` | End-to-end transfers: lookup → quote → transfer, plus negative cases |

**The rule that ties it together:** a DFSP can send money only up to its **Net Debit Cap**, which is backed by funds deposited through **funds-in** (recorded against `HUB_RECONCILIATION`). Each transfer raises the payer's **POSITION**. **Settlement** clears positions through `HUB_MULTILATERAL_SETTLEMENT` according to the **settlement model**.
