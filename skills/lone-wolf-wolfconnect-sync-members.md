---
name: Sync brokerage members and transactions from WolfConnect
description: >-
  Read and write Lone Wolf member (agent/staff) records and transactions through the WolfConnect
  API, including its HMAC request-signing scheme, its OData query subset and its RowVersion
  optimistic-concurrency model. This is the API to use for keeping a third-party roster, CRM or
  intranet in step with the brokerage's system of record.
generated: '2026-07-26'
method: generated
api: openapi/lone-wolf-wolfconnect-api-openapi.yml
docs: https://apidocs.lwolf.com/doc/wolfconnect-api
base_url: https://api.globalwolfweb.com
sandbox_url: https://api-sb.globalwolfweb.com
operations:
  - getMembers
  - getMember
  - createMember
  - updateMember
  - getMemberProfileImage
  - setMemberProfileImage
  - getMemberInOutStatus
  - setMemberInOutStatus
  - getAllMembersInOutStatus
  - getTransactions
  - getTransaction
  - createTransaction
  - updateTransaction
  - finalizeTransaction
  - getClassifications
  - getPropertyTypes
  - getContactTypes
  - getConditions
  - getSourcesOfBusiness
---

# Sync brokerage members and transactions from WolfConnect

## Step 1 — sign every request

WolfConnect does not use bearer tokens. Each request carries two headers:

```
Authorization: LoneWolfToken [API Token]:[Client Code]:[Signature]:[Date]
Content-MD5: <base64 MD5 of the body>
```

- `Content-MD5` for a bodyless request is the constant `1B2M2Y8AsgTpgAmY7PhCfg==` (the MD5 of
  the empty string).
- The signature is `HMACSHA256(secretKey, "[HTTP Method]:[Resource URI]:[Date]:[Content-MD5]")`.
  `HMACSHA384` and `HMACSHA512` are supported by appending the algorithm to the scheme
  (`LoneWolfToken-HMACSHA512`).
- `Date` is a UTC string in the exact format the WolfConnect introduction specifies, and it is
  part of the signed material — clock skew breaks authentication.
- Account-level resources use the `LoneWolfKey [Consumer Key]:[Signature]:[Date]` variant.

Lone Wolf publishes no first-party SDK, so this scheme is hand-rolled in every integration; get
it wrong and you get an opaque failure. Test against `https://api-sb.globalwolfweb.com` first.

## Step 2 — pull the reference lists

`getClassifications`, `getPropertyTypes`, `getContactTypes`, `getConditions` and
`getSourcesOfBusiness` return the brokerage-configured lists that every transaction field keys
into. Cache them; refresh on a schedule, not per request.

## Step 3 — read members (`getMembers`, `getMember`)

`GET /wolfconnect/members/v1` supports the OData subset: `$filter`, `$orderby`, `$top`, `$skip`,
`$search`, `$expand`. Two behaviours will bite you if you assume standard OData:

- **Only logical, arithmetic and grouping operators are supported.** No `contains`,
  `startswith` or `endswith`.
- **`$expand` is inverted.** Omitting it returns *all* child objects and collections; passing an
  empty `$expand=` returns only the root object. For a Member that means addresses, email
  addresses, phone numbers, websites, MLS boards, public profiles and integrations all come back
  unless you suppress them.

The maximum page is 1,000 objects and `$top=1000` is assumed. `$orderby` must be set for
`$top`/`$skip` to work. If a page returns 1,000 rows you must page — and because each call is a
fresh query, records can shift between pages, so key on ids rather than offsets when
reconciling.

## Step 4 — incremental sync

Filter on a timestamp rather than re-reading the roster:

```
/wolfconnect/transactions/v1?$filter=CreatedTimestamp ge datetimeoffset'2014-04-01T00:00:00Z'
```

Store the high-water mark on your side; WolfConnect exposes no change feed and no webhooks.

## Step 5 — write safely (`updateMember`, `updateTransaction`)

Two rules, both from the WolfConnect introduction:

1. **Send only the fields you are changing.** On update, a property sent as `NULL` is *not*
   written — the current value is kept. On create, `NULL` means "use the column default". The
   exception is properties for which `NULL` is itself legal, where `NULL` is persisted.
2. **Use `RowVersion`.** Any object supporting concurrency checking carries a `RowVersion`
   property. Send the value you read; if it no longer matches, the write fails with `409` instead
   of silently overwriting a change someone made in the WOLFconnect UI. Sending `RowVersion` as
   `NULL` disables the check — do not do that on a shared record.

## Step 6 — profile images and presence

`getMemberProfileImage` / `setMemberProfileImage` / `deleteMemberProfileImage` manage the agent
photo (`getMemberPublicProfileImage` for the public one). `getMemberInOutStatus`,
`setMemberInOutStatus` and `getAllMembersInOutStatus` drive in/out board presence — the cheapest
useful integration WolfConnect offers.

## Step 7 — transactions

`getTransactions`, `getTransaction`, `createTransaction`, `updateTransaction` and
`finalizeTransaction` (`PUT /wolfconnect/transactions/v1/{transactionId}/finalize`) cover the
lifecycle. `finalizeTransaction` is a state change into the brokerage's accounting — treat it as
irreversible.

**`searchTransactions` is deprecated** (the only operation marked `deprecated: true` anywhere in
Lone Wolf's published estate). Use `getTransactions` with an OData `$filter` instead.

## Error handling

Failures return a `ClientError {Code, Message}` body carrying Lone Wolf's own registry:

| Code | Meaning |
|---|---|
| 1000 | Unknown error — contact Lone Wolf support |
| 1001 | Invalid parameter |
| 1002 | Missing parameter |
| 1003 | The request must be multipart |
| 1004 | No files sent |
| 1005 | Maximum files exceeded |
| 1006 | Validation failed — read `Details` |
| 1007 | Member authentication failed (deactivated, or password reset unfinished) |
| 1008 | Invalid operation — you probably lack permission |

For generic 400/401/404/409 responses `Code` simply repeats the HTTP status. **A failing
response with no `ClientError` body means your URL is wrong**, not that the record is missing —
that is documented behaviour, and it is the first thing to check.

## Rules

- Never send `RowVersion: null` on an update to a record a human also edits.
- Never PUT a full object you did not just read; send the delta.
- Do not use `searchTransactions`.
- Multipart uploads: honour codes 1003/1004/1005 rather than retrying blindly.
