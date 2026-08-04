# Lone Wolf Technologies (lone-wolf)

<!-- API-EVANGELIST-PROVENANCE:BEGIN -->
> ### About this repository
>
> **This is not our API.** This repository is an independent, third-party profile of a company's
> **publicly available** API surface, maintained by [API Evangelist](https://apievangelist.com).
> API Evangelist does not operate, host, resell, or support this company's APIs, and is not
> affiliated with or endorsed by the company unless stated on the profile.
>
> **Where the information came from.** Everything here is assembled from material a member of the
> public can reach with a browser and no credentials — the company's own website, developer portal
> and documentation, the specifications it publishes for public use (OpenAPI, AsyncAPI, JSON Schema,
> `apis.json`, `llms.txt` and similar), its public repositories, and its public status, pricing and
> changelog pages. **Nothing here is obtained by breaching a system, defeating an access control, or
> using credentials of any kind.**
>
> **The rating is an independent assessment.** The Kin Score and Agent Readiness rating are
> independently calculated scores of a company's *public* API artifacts, produced by API Evangelist
> against a published rubric. They are not certifications, endorsements, security assessments, or
> audits, and they score published artifacts — not the quality, safety, or security of the software.
>
> **Corrections, re-scores, and removal are free.** No partnership, contract, or purchase is
> required, and you do not need to justify the request.
>
> - **Something wrong?** Open an issue on this repository, or email
>   [info@apievangelist.com](mailto:info@apievangelist.com).
> - **Published something new?** Ask for a re-score and we will re-run the rating.
> - **Want the listing taken down?** Say so and we will honor it. The profile is reduced to your
>   company name, a factual description, and a link to your own site, and the company is recorded as
>   **unrated** — never scored zero for having asked.
>
> **Response times.** Acknowledgement within **one business day**; removal or restriction within
> **two business days**; corrections and re-scores within **five business days**.
>
> **On a security or compliance team?** Email
> [info@apievangelist.com](mailto:info@apievangelist.com) with *security* in the subject line and
> you will get a person, not a form. We will tell you exactly which public URLs this profile was
> built from so your team can see the same surface we did, and we will take the listing down on
> request while you work through it.
>
> Full detail: **[Where this data comes from](https://apievangelist.com/about/where-our-data-comes-from)**
<!-- API-EVANGELIST-PROVENANCE:END -->

Lone Wolf Technologies is the dominant back-office, transaction-management and forms vendor in North American residential real estate, headquartered in Dallas, Texas and backed by Stone Point Capital. Its software runs the paperwork and money side of the deal rather than the listing feed: brokerage accounting and commission processing (Back Office, the brokerWOLF lineage), transaction management in two editions (zipForm Edition and TransactionDesk Edition), Authentisign and Inkless e-signature, Cloud CMA comparative market analysis, Boost digital advertising (HomeSpotter), Spacio open-house lead capture, and Propertybase/Relationships CRM. On 2026-02-02 it launched a public API Portal for the Lone Wolf Foundation platform, and its documentation hub publishes seven complete, anonymously downloadable OpenAPI 3.0 definitions. Documentation is genuinely open; credentials are not.

**APIs.json:** [https://raw.githubusercontent.com/api-evangelist/lone-wolf/refs/heads/main/apis.yml](https://raw.githubusercontent.com/api-evangelist/lone-wolf/refs/heads/main/apis.yml)

## Tags

- Real Estate
- United States
- PropTech
- Transactions
- Transaction Management
- Brokerage Back Office
- Real Estate Accounting
- Commissions
- Forms
- zipForm
- TransactionDesk
- E-Signature
- CMA
- Valuation
- CRM
- MLS
- Real Estate Agents
- Brokers

## Timestamps

- **Created:** 2026-07-26
- **Modified:** 2026-07-26

## RESO Posture

**Not RESO certified.** No RESO Web API certification, no Data Dictionary certification, no `$metadata` document (`https://api.lwolf.com/$metadata` → HTTP 404), and no Universal Property Identifier. A full-text search of all seven harvested OpenAPI definitions for "RESO", "Data Dictionary" and "UPI" returned zero matches, and RESO's public certificates listing contains no Lone Wolf entry.

This is the correct and structural answer, not a governance failure. RESO certification governs MLS listing feeds; Lone Wolf occupies the transaction and back-office layer instead, consuming MLS data under MLS agreements (Cloud CMA documents RETS live queries) rather than redistributing it.

**Do not mistake OData for RESO.** The Transact and WolfConnect APIs document OData query options (`$filter`, `$expand`) applied to Lone Wolf's own proprietary entity model. That is query grammar, not the RESO Web API — there is no service metadata document and no Data Dictionary field alignment.

Lone Wolf is the clean control case for this sector's central question, and it inverts the usual pattern: where a RESO-certified MLS endpoint is a standardized contract you cannot reach without a licence, Lone Wolf offers a non-standardized, entirely proprietary contract that you can read completely and anonymously. Standardization and reachability are independent axes.

## Access Gate

**application-approval.** Read the docs freely, then complete the access-request form at [https://www.lwolf.com/api-getting-started](https://www.lwolf.com/api-getting-started), which states: *"Complete the access request form and a member of our integrations team will be in touch to get you access to the APIs you need."* There is no self-serve key issuance anywhere on the estate.

No MLS membership, board membership, IDX/VOW agreement or real-estate licence is required to read the documentation or download the specs — a material difference from the MLS-feed providers in this sector. The zipForm Partner API is separately licensed, described in its own specification as *"made available on a licensed basis to third-party application partners."*

**Open data:** none. Every endpoint requires issued credentials.

## Auth Model

Auth0-backed OpenID Connect at `gateway.lwolf.com`, with a live discovery document fetched anonymously ([openid-configuration](https://gateway.lwolf.com/.well-known/openid-configuration), HTTP 200) supporting `client_credentials`, `authorization_code`, `refresh_token` and token exchange. Only standard OIDC identity scopes are advertised — no product- or resource-level API scopes are publicly inspectable.

Five distinct schemes across the estate, tracing the acquisition roll-up:

| API | Scheme |
|---|---|
| Transact | `bearerAuth` + `subscriptionKey` |
| Deals | JWT Bearer via `POST https://authentication.api.lwolf.com/v1/login` |
| TransactionDesk | OAuth 2.0 authorization-code (one-time code, 10-minute expiry) |
| zipForm | `X-Auth-SharedKey` + `X-Auth-ContextId` / External Id |
| WolfConnect | `LoneWolfToken` signed (HMAC) Authorization header |
| Authentisign | *no securitySchemes declared* |
| Back Office | *no securitySchemes declared* |

## APIs

Seven OpenAPI 3.0 definitions, harvested 2026-07-26 from the Bump.sh hub at `apidocs.lwolf.com`, all confirmed to parse. **348 operations total.**

### Lone Wolf Transact API

Full transaction lifecycle — transactions, offers, contacts, folders, documents, share groups, form libraries and e-signature dispatch. Resources scoped under `/users/{userId}`. 37 paths / 53 operations.

- **Human URL:** [https://apidocs.lwolf.com/doc/transact-api](https://apidocs.lwolf.com/doc/transact-api)
- **Base URL:** `https://gateway.lwolf.com`
- [OpenAPI](openapi/lone-wolf-transact-api-openapi.yml)

### Lone Wolf Deals API

Partner-integration API (release 25.01.00) covering deal records across the Foundation platform. The largest published surface — 53 paths / 86 operations — but declares no servers block, so no base URL is asserted.

- **Human URL:** [https://apidocs.lwolf.com/doc/deals-api](https://apidocs.lwolf.com/doc/deals-api)
- [OpenAPI](openapi/lone-wolf-deals-api-openapi.yml)

### Lone Wolf Back Office API

Brokerage accounting — commissions, commission fees and tiers, deposits, conditions, classifications, employees, offices and contacts. The brokerWOLF lineage. 39 paths / 69 operations.

- **Human URL:** [https://apidocs.lwolf.com/doc/back-office-online-api](https://apidocs.lwolf.com/doc/back-office-online-api)
- **Base URL:** `https://api.lwolf.com/backoffice`
- [OpenAPI](openapi/lone-wolf-back-office-online-api-openapi.yml)

### Lone Wolf Authentisign API

E-signature — signings, signer roles, documents and status. Supports an optional `CallbackUrl` per signing plus a PATCH endpoint to update it: the only genuine webhook surface across the estate. 41 paths / 46 operations.

- **Human URL:** [https://apidocs.lwolf.com/doc/authentisign-api](https://apidocs.lwolf.com/doc/authentisign-api)
- **Base URL:** `https://api.lwolf.com/authentisign`
- [OpenAPI](openapi/lone-wolf-authentisign-api-openapi.yml)

### Lone Wolf TransactionDesk Partner API

Transaction summaries, documents, contacts, types, statuses, single sign-on and resource metadata. Full OAuth 2.0 authorization-code flow. 25 paths / 44 operations. Only a preproduction host is published.

- **Human URL:** [https://apidocs.lwolf.com/doc/transactiondesk-api](https://apidocs.lwolf.com/doc/transactiondesk-api)
- **Base URL:** `https://api.pre.transactiondesk.com`
- [OpenAPI](openapi/lone-wolf-transactiondesk-api-openapi.yml)

### Lone Wolf zipForm Partner API

The zipForm REST web service (v5.1) — transaction data, PDF forms, templates, teams and title integration. Licensed to third-party application partners. 22 paths / 28 operations.

- **Human URL:** [https://apidocs.lwolf.com/doc/zipform-api](https://apidocs.lwolf.com/doc/zipform-api)
- **Base URL:** `https://ws.zipformplus.com/api`
- [OpenAPI](openapi/lone-wolf-zipform-api-openapi.yml)

### Lone Wolf WolfConnect API

RESTful access to members, transactions, classifications, conditions, contact types, property types and sources of business. Documents OData support, concurrency checking and permissions. 15 paths / 22 operations, with a real sandbox host.

- **Human URL:** [https://apidocs.lwolf.com/doc/wolfconnect-api](https://apidocs.lwolf.com/doc/wolfconnect-api)
- **Base URL:** `https://api.globalwolfweb.com`
- [OpenAPI](openapi/lone-wolf-wolfconnect-api-openapi.yml)

## Sibling Product Documentation

Linked from the official API Portal but documented outside the Bump.sh hub, with no downloadable specification:

- [Cloud CMA Developers](https://cloudcma.com/developers) — CMA, Buyer Tour and Property Reports; documents MLS Security and RETS Live Queries
- [Boost / HomeSpotter API Reference](https://docs.homespotter.com/) — social advertising checkout; `openapi.json` and `swagger.json` both return HTML, not specifications
- [Spacio API Reference](https://spac.io/docs/api/) — open-house lead capture; `https://ws.spac.io/api/v1/`, apikey header, 12,000 calls/hour default
- [Propertybase / Front Office API Docs](https://apidocs.propertybase.com/) — CRM

## Notes

- Developer portal: [https://www.lwolf.com/api-portal](https://www.lwolf.com/api-portal) (HTTP 200, launched 2026-02-02)
- Documentation hub: [https://apidocs.lwolf.com/](https://apidocs.lwolf.com/) — with a live MCP server at `https://apidocs.lwolf.com/mcp` and a Markdown export at `?format=md`
- No SDKs, client libraries, CLI, Postman collection or GitHub organization were found.
- The API Portal's "Back Office — Review API docs" link is broken: it resolves to `bump.sh/doc/back-office-api` (HTTP 404). The working document is `/doc/back-office-online-api`.

See [review.yml](review.yml) for the full probe log, RESO evidence and harvest provenance.
