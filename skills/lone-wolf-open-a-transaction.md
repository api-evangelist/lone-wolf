---
name: Open a Lone Wolf transaction
description: >-
  Authenticate against the Lone Wolf gateway, resolve the acting user, create a transaction on
  the Transact Workflow service, then read it back and patch it. This is the entry point for
  every other Transact flow — nothing else in the API is reachable until you have a token, a
  subscription key and a userId.
generated: '2026-07-26'
method: generated
api: openapi/lone-wolf-transact-api-openapi.yml
docs: https://apidocs.lwolf.com/doc/transact-api
base_url: https://gateway.lwolf.com
operations:
  - getToken
  - getUsers
  - getOffices
  - createTransaction
  - getTransactions
  - getTransactionByKey
  - updateTransaction
  - deleteTransaction
---

# Open a Lone Wolf transaction

## Before you start

Lone Wolf does not issue credentials self-service. You need a client id, client secret, a Lone
Wolf client GUID (`lwt_client_id`) and a subscription key, all provisioned by the Lone Wolf
integrations team through the access-request form at
<https://www.lwolf.com/api-getting-started>. There is no sandbox you can sign up for.

## Step 1 — get an access token (`getToken`)

`POST /oauth/token` on `https://gateway.lwolf.com` with a JSON body:

```json
{
  "grant_type": "client_credentials",
  "client_id": "<your client id>",
  "client_secret": "<your client secret>",
  "audience": "https://api.lwolf.com",
  "lwt_client_id": "<your Lone Wolf client GUID>"
}
```

Take `access_token` from the response. Tokens are short-lived — request a new one on expiry
rather than caching indefinitely. No scope is requested or granted; Lone Wolf publishes no
API-permission scopes.

## Step 2 — send both credentials on every call

Every subsequent request needs **two** headers:

```
Authorization: Bearer <access_token>
lw-subscription-key: <your subscription key>
```

Diagnose failures by which one is missing:

- `401` — the bearer token is missing or expired. Re-run step 1 and retry once.
- `403` — the `lw-subscription-key` header is missing or invalid. Do not retry; fix the header.

## Step 3 — resolve the acting user (`getUsers`, `getOffices`)

Almost every Transact resource is addressed under `/users/{userId}/…`, where `userId` is the
GUID of the user on whose behalf you are acting. Call `getUsers`
(`GET /platform/v1/clients/{clientId}/users`) to list the users in your client, and `getOffices`
(`GET /platform/v1/clients/{clientId}/offices`) if you need office context. Cache the `userId`
alongside your own record for the agent — you will need it on every call.

## Step 4 — create the transaction (`createTransaction`)

`POST /transact-workflow/v1/Transactions`. Note this operation is **not** user-scoped: the
create endpoint sits at the service root while the read/update/delete endpoints are under
`/users/{userId}/`.

Property fields accepted on create (and on `updateTransaction`): `type`, `address1`–`address4`,
`locality`, `region`, `subRegion`, `postalCode`, `country`, `legalDescription`,
`propertyIncludes`, `propertyExcludes`, `taxNumber`, `mlsNumber`, `note`, `schoolDistrict`,
`zoningClass`, `yearBuilt`, `phase`, `propertyType`, `asking`, `deposit`, `taxes`,
`closingDate`, `listingExpiration`, `listingGoesLive`.

**There is no idempotency key.** If the POST times out you cannot safely retry it. Instead call
`getTransactions` with an OData `$filter` on a field you control (for example the `mlsNumber` or
your own reference in `note`) to check whether the transaction already exists before retrying.

## Step 5 — read it back (`getTransactions`, `getTransactionByKey`)

- `getTransactions` — `GET /transact-workflow/v1/users/{userId}/transactions`. Collection
  endpoints support OData `$filter` and `$expand` (for example
  `$filter=opportunityId eq <guid>`).
- `getTransactionByKey` — `GET /transact-workflow/v1/users/{userId}/transactions/{transactionId}`.

## Step 6 — update or delete (`updateTransaction`, `deleteTransaction`)

`updateTransaction` is a `PATCH` and accepts a partial representation — only the fields you send
are changed. `deleteTransaction` is destructive and has no undo; the Transact API publishes no
soft-delete or recovery operation (unlike Authentisign, which has `GetDeletedSignings` and
`RecoverDeletedSigning`).

## Error handling

The Transact definition shares four response objects across all 53 operations —
`BadRequest` (400), `Unauthorized` (401), `Forbidden` (403), `NotFound` (404) — with the
envelope `{error, message}` plus arbitrary additional properties. No 5xx contract is published,
so treat any 5xx as unknown and reconcile by reading before retrying. See
`errors/lone-wolf-problem-types.yml`.

## Rules

- Never retry a `POST` blindly — no Lone Wolf API supports idempotency keys.
- Always send both `Authorization` and `lw-subscription-key`.
- Do not hard-code `https://api.pre.lwolf.com`; that is the pre-production environment host.
- Rate limits are undocumented. Back off on any non-200 rather than looping.
