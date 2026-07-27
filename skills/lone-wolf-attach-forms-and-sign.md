---
name: Attach forms to a transaction and send them for signature
description: >-
  The marquee Lone Wolf flow: browse a form library, add forms to a transaction's form package,
  pull the rendered PDF, then create an Authentisign signing over those documents, add
  participants and hand back an SSO link. Runs entirely through the Transact gateway, which
  routes both the forms services and Authentisign v3.
generated: '2026-07-26'
method: generated
api: openapi/lone-wolf-transact-api-openapi.yml
docs: https://apidocs.lwolf.com/doc/transact-api
base_url: https://gateway.lwolf.com
operations:
  - getLibraries
  - getForms
  - getLibraryForm
  - addFormsToTransaction
  - getTransactionForms
  - getTransactionFormPdf
  - moveTransactionForm
  - createDocumentForUser
  - createSigning
  - addSigningParticipants
  - getSignings
  - getSigningSsoLink
  - getSigningDocuments
  - getSignedDocuments
  - getSigningCertificates
---

# Attach forms to a transaction and send them for signature

Prerequisite: you have a transaction and a `userId` — see the *Open a Lone Wolf transaction*
skill. Every call carries `Authorization: Bearer <token>` and `lw-subscription-key`.

## Step 1 — find the forms (`getLibraries`, `getForms`, `getLibraryForm`)

- `getLibraries` — `GET /forms-design/api/users/{userId}/libraries` lists the form libraries the
  user has access to. In North American real estate these are usually association or MLS
  libraries the agent is entitled to through membership, so the list is user-specific: never
  cache one user's libraries against another user.
- `getForms` — `GET /forms-design/api/users/{userId}/forms` lists forms across libraries.
- `getLibraryForm` — `GET /forms-design/api/users/{userId}/forms/{libId}/{formId}` reads one
  form definition.
- `getFormPageImage` renders a page image if you need a visual preview.

## Step 2 — add forms to the transaction's form package (`addFormsToTransaction`)

`POST /forms-editor/api/v1/users/{userId}/FormPackages/{transactionId}/forms` — the
`transactionId` *is* the form-package key. Send the `libraryId` / `formId` pair(s) you selected.

Read the package back with `getTransactionForms`
(`GET /forms-editor/api/v1/users/{userId}/FormPackages/{transactionId}/forms`) and reorder or
re-parent an entry with `moveTransactionForm` (`PATCH …/forms/{formId}`, using `parentId`).

## Step 3 — pull the PDF (`getTransactionFormPdf`)

`GET /forms-editor/api/v1/users/{userId}/FormPackages/{transactionId}/forms/{formId}` returns
the rendered form. Binary responses on this API are `application/pdf`,
`application/octet-stream` or `image/*` — set your client to stream rather than parse JSON.

If you are signing a document that did not come from a form library, upload it first with
`createDocumentForUser` (`POST /transact-workflow/v1/users/{userId}/documents`) and use the
resulting document instead.

## Step 4 — create the signing (`createSigning`)

`POST /authentisign/v3/users/{userId}/signings`. This is a `multipart/form-data` request —
the files travel with the metadata, not as a separate upload step.

Set `CallbackUrl` on creation if you want signing events posted to you. This is the **only**
event mechanism anywhere in the Lone Wolf estate: there is no account-level webhook registry,
no event-type catalog and no published payload schema or signature verification. See
`asyncapi/lone-wolf-authentisign-webhooks.yml`. If you do not set it, you must poll.

## Step 5 — add the participants (`addSigningParticipants`)

`POST /authentisign/v3/api/v3/users/{userId}/participants/batch` adds the signers in one call.
Batch them — do not loop one participant per request.

## Step 6 — hand off and collect (`getSigningSsoLink`, `getSigningDocuments`)

- `getSigningSsoLink` — `GET /authentisign/v3/api/v3/users/{userId}/sso/signing/{signingId}`
  returns a link that drops the user straight into the signing session. This is how you keep the
  agent inside your own product instead of bouncing them to Authentisign's UI.
- `getSigningDocuments` — the documents attached to the signing.
- `getSignedDocuments` — the executed copies.
- `getSigningCertificates` — the signing certificates, which are the audit artifact a brokerage
  keeps for compliance. Fetch and store them; do not rely on Lone Wolf retention.
- `getSignings` lists the user's signings if you need to reconcile state without a callback.

## Reconciliation without callbacks

If no `CallbackUrl` was set, poll `getSignings` for the user and compare status. On the
Authentisign API proper, `GetSigningHistory` is the richest read — it returns what actually
happened to a signing rather than just its current state.

## Rules

- Never invent a `formId` or `libraryId`; always resolve them from `getLibraries` / `getForms`
  for that specific `userId`.
- `createSigning` is not idempotent. Before retrying a timed-out create, call `getSignings` and
  match on your own reference before issuing a second one — a duplicate signing sends duplicate
  invitation emails to real buyers and sellers.
- Treat signed documents and certificates as records to archive on your side.
