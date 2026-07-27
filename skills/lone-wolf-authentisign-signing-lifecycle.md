---
name: Run an Authentisign signing end to end
description: >-
  Drive a Lone Wolf Authentisign e-signature from creation through participants, callback wiring,
  sending, reminders, rejection handling and final-document retrieval — including the recovery
  paths (copy, reset, force-complete, recover-deleted) that make Authentisign the most complete
  lifecycle surface Lone Wolf publishes.
generated: '2026-07-26'
method: generated
api: openapi/lone-wolf-authentisign-api-openapi.yml
docs: https://apidocs.lwolf.com/doc/authentisign-api
base_url: https://api.lwolf.com/authentisign
operations:
  - CreateSigning
  - GetSigning
  - GetSignings
  - AddParticipants
  - GetParticipants
  - UpdateParticipantEmail
  - AddCCs
  - GetCcs
  - ApplyLayout
  - GetLayouts
  - UpdateSigningCallback
  - SendSigning
  - ResendInvitation
  - GetSigningHistory
  - RejectSigning
  - ForceCompleteSigning
  - ResetSigning
  - CopySigning
  - SendFinalDocuments
  - GetFinalSigningDocument
  - GetFinalSigningDocumentUrl
  - GetLatestSignedDocument
  - GetSigningCertificate
  - GetDocumentCertificate
  - LinkTransaction
  - GetDeletedSignings
  - RecoverDeletedSigning
  - DeleteSigning
  - DesignSso
---

# Run an Authentisign signing end to end

The published Authentisign OpenAPI declares **no securitySchemes** — the definition does not
state its auth model. Confirm credentials with the Lone Wolf integrations team. Note that the
same Authentisign v3 surface is also routed through the Transact gateway under
`/authentisign/v3`, where the gateway's bearer + `lw-subscription-key` pair applies.

## Step 1 — create the signing (`CreateSigning`)

`POST` with `multipart/form-data`. The documented body fields include `Name`, `AccountId`,
`StatusId`, `IsOrdered`, `ExpirationDate`, `ReminderDay`, `ReminderHour`, `TransactionId`,
`TechnologyProvider`, `CallbackUrl`, `LayoutId`, `ApplyLayout`, `Files[]` and
`FilesWithStrikeoutsDisabled[]`.

Two fields decide the shape of the whole flow:

- **`IsOrdered`** — sequential vs parallel signing.
- **`CallbackUrl`** — the only event mechanism Lone Wolf publishes anywhere. Set it here or you
  are committed to polling.

An optional `externalId` header lets you carry your own reference onto the signing — use it, it
is your only safe deduplication handle.

## Step 2 — add participants and CCs (`AddParticipants`, `AddCCs`)

`AddParticipants` batches the signers. `AddCCs` adds copy recipients. `GetParticipants` reads
them back, and `UpdateParticipantEmail` fixes a typo'd address *before* sending — after sending,
use `ResendInvitation`.

## Step 3 — apply a layout (`GetLayouts`, `ApplyLayout`)

Layouts are reusable field placements. `GetLayouts` lists them; `ApplyLayout` stamps one onto the
signing's documents. `CreateLayout` / `CopyLayout` / `DeleteLayout` manage the library. Applying
a layout is how you avoid re-placing signature blocks on every deal that uses the same form.

## Step 4 — wire or rewire the callback (`UpdateSigningCallback`)

`PATCH /api/v1/signings/{id}` with `{"CallbackUrl": "https://newcallbackurl.com"}` (it is also
accepted as a query parameter). Use this when your endpoint moves, or to attach a callback to a
signing that was created without one.

Lone Wolf publishes **no** payload schema, event catalog, signature-verification scheme, retry
policy or IP range for these callbacks. Treat an inbound callback as a *hint to re-read*, never
as trusted data: on receipt, call `GetSigning` and `GetSigningHistory` and act on what the API
says.

## Step 5 — send it (`SendSigning`)

`SendSigning` dispatches invitations. `ResendInvitation` nudges a participant who has not acted.
Respect `ReminderDay` / `ReminderHour` — Authentisign already reminds; do not build a second
reminder loop on top of it.

For an in-product experience instead of email, `DesignSso` and `LayoutSso` return SSO links into
the Authentisign design and layout surfaces.

## Step 6 — track it (`GetSigningHistory`, `GetSignings`, `GetSigning`)

`GetSigningHistory` is the richest read — what actually happened, not just the current status. It
is the correct polling target when no callback is wired, and the correct audit source when one is.

## Step 7 — handle the unhappy paths

- `RejectSigning` — a participant declines. The signing is over; decide whether to `CopySigning`
  into a fresh one rather than resetting.
- `ResetSigning` — returns the signing to an unsigned state. Destructive to in-flight
  signatures; confirm with a human first.
- `ForceCompleteSigning` — completes despite outstanding signers. This is a legally significant
  action on a real-estate contract: require explicit human authorization, never do it
  automatically.
- `GetDeletedSignings` / `RecoverDeletedSigning` — the undo path for `DeleteSigning`. Authentisign
  is the only Lone Wolf API that publishes one.

## Step 8 — collect the artifacts

- `SendFinalDocuments` distributes the executed package.
- `GetFinalSigningDocument` / `GetFinalSigningDocumentUrl`, `GetLatestSignedDocument`,
  `GetOriginalDocument`, `GetFinalDocument` retrieve the files (or a URL to them).
- `GetSigningCertificate` and `GetDocumentCertificate` return the signing certificates — the
  audit artifact. Archive these on your side.
- `LinkTransaction` associates the signing with a TransactionDesk transaction, carrying a
  document checklist. This is the only explicit cross-product foreign key Lone Wolf publishes.

## Error handling

The definition declares 400/401/404 on only part of its 46 operations and leans on an
undocumented `default` response elsewhere. The envelope is
`ErrorResponse {code, message, details[]}`. A `ProblemDetails` schema exists in
`components.schemas` but is never served, so do not expect `application/problem+json`.

## Rules

- `ForceCompleteSigning`, `ResetSigning` and `DeleteSigning` require human confirmation.
- `CreateSigning` is not idempotent and sends real email to buyers and sellers — before retrying
  a timed-out create, list signings and match on your `externalId`.
- Never trust a callback payload; re-read with `GetSigning` / `GetSigningHistory`.
- Archive signed documents and certificates; do not depend on Lone Wolf retention.
