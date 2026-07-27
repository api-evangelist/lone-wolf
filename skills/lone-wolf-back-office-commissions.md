---
name: Record a brokerage transaction and its commission splits
description: >-
  Build the money side of a deal in Lone Wolf Back Office Online: create the transaction, add a
  tier (a side of the deal), attach agent commissions and their fees, add external-agent
  commissions for the co-operating side, and use the calculator before committing. This is the
  brokerWOLF lineage — the system of record for brokerage accounting.
generated: '2026-07-26'
method: generated
api: openapi/lone-wolf-back-office-online-api-openapi.yml
docs: https://apidocs.lwolf.com/doc/back-office-online-api
base_url: https://api.lwolf.com/backoffice
operations:
  - get-v1-transactions
  - post-v1-transactions
  - get-v1-transactions-transactionid
  - patch-v1-transactions-transactionid
  - get-v1-transactions-transactionid-summary
  - get-v1-classifications
  - get-v1-property-types
  - post-v1-transactions-transactionid-tiers
  - get-v1-transactions-transactionid-tiers
  - patch-v1-tiers-tierid
  - post-v1-tiers-calculate-tier
  - post-v1-transactions-transactionid-tiers-tierid-commissions
  - get-v1-transactions-transactionid-tiers-tierid-commissions
  - patch-v1-commissions-id
  - get-v1-transactions-transactionid-tiers-tierid-commissions-commissionid-fees
  - post-v1-transactions-transactionid-tiers-tierid-external-commissions
  - get-v1-employees
  - get-v1-offices
---

# Record a brokerage transaction and its commission splits

## The model you are building

A **Transaction** carries one or more **Tiers**. A Tier is a side/leg of the deal and holds the
**Commissions**; each Commission belongs to an **Employee** (the agent) and an **Office**, and
carries **CommissionFees**. Co-operating brokerages are modelled separately as
**ExternalAgents** with **ExternalCommissions** on the same tier. See
`data-model/lone-wolf-data-model.yml`.

The Back Office Online OpenAPI declares **no securitySchemes** — the definition does not publish
its auth model. Confirm credentials with the Lone Wolf integrations team who provision your
access; do not assume the Transact gateway's bearer + subscription-key pair applies here.

`operationId`s on this API are generated from method and path
(`post-v1-transactions-transactionid-tiers`), so read them as `POST /v1/transactions/{transactionId}/tiers`.

## Step 1 — resolve the reference data first

- `get-v1-classifications` → the `classificationId` you will set on the transaction and tier.
- `get-v1-property-types` → the `propertyTypeId`.
- `get-v1-employees` → the `agentId` for each commission.
- `get-v1-offices` → the `officeId`.

Never post a transaction with a guessed classification or property type; both are foreign keys
into brokerage-configured lists that differ per client.

## Step 2 — create the transaction (`post-v1-transactions`)

`POST /v1/transactions`. Read it back with `get-v1-transactions-transactionid`, or list with
`get-v1-transactions`. `get-v1-transactions-summary` and
`get-v1-transactions-transactionid-summary` return the lighter summary projections — prefer them
for list views, since the full Transaction expands tiers, conditions, deposits, external agents
and both contact collections.

## Step 3 — add the tier (`post-v1-transactions-transactionid-tiers`)

`POST /v1/transactions/{transactionId}/tiers` with the `classificationId`. Patch it later with
`patch-v1-tiers-tierid`.

## Step 4 — model the split before you commit it (`post-v1-tiers-calculate-tier`)

`POST /v1/tiers/calculate-tier` runs the commission calculation without persisting. Use it to
preview a split, show the agent what they will net, and validate your inputs — then write the
real records. This is the one operation in the Back Office API that is safe to call repeatedly.

## Step 5 — attach the commissions

- `post-v1-transactions-transactionid-tiers-tierid-commissions` — one commission per agent on
  that tier, referencing `agentId` and `officeId`.
- `get-v1-transactions-transactionid-tiers-tierid-commissions-commissionid-fees` — the fees
  deducted from that commission.
- `post-v1-transactions-transactionid-tiers-tierid-external-commissions` — the co-operating
  side, referencing an `externalAgentId`.
- `patch-v1-commissions-id` — adjust an existing commission.

## Error handling

Back Office is the only Lone Wolf API that separates **422 Validation failed** from **400
Invalid request**, and it declares 400/401/403 on all 69 operations plus 409 Conflict on the
nine create operations. The envelope is `ApiError {id, code, message, details[]}` where `code`
mirrors the HTTP status and `id` is a unique error identifier worth quoting to support.

- `422` — your payload is structurally fine but violates a brokerage rule. Read `details[]`.
- `409` — a duplicate. Look the existing record up rather than retrying.
- `400` — malformed request.

## Rules

- **This API moves money.** There is no idempotency key: a retried commission POST creates a
  second commission. Before any retry, `GET` the tier's commissions and check whether yours
  landed.
- Preview with `calculate-tier` before writing.
- Never delete a commission (`delete-v1-commissions-id`) to "fix" a value — patch it, so the
  accounting history survives.
- No rate limit is documented; keep concurrency low against an accounting system of record.
