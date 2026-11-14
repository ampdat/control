# The HTTP API

The MCP tools are a second way in to the same answers. Over HTTP, the free routes are pages and
metadata. The paid routes use [x402](https://www.x402.org/): a request without a payment gets
`402 Payment Required` with the price. Every paid route answers or refuses (400, 404 or 503)
before any payment is asked for.

| Route | Returns | Price |
| --- | --- | --- |
| `GET /ethereum/{address}` | a page for people: who holds each role and what that holder is, the notice each change gives, and what is pending and who can revoke it | free |
| `GET /vaults` | every covered vault | free |
| `GET /v1/coverage` | the covered families, the schemas and whether each is frozen, and example addresses | free |
| `GET /v1/schemas/{id}` | a published JSON schema (below) | free |
| `GET /v1/ethereum/{address}` | the subject envelope: what the contract is, its state, who controls it, and its recent and pending records | paid |
| `GET /v1/ethereum/{address}/records` | that contract's records, oldest first (query: `since`, `kind`, `observed_status`, `limit`, `cursor`) | paid |
| `GET /v1/records` | the feed across contracts (query: `since`, `family`, `kind`, `observed_status`, `limit`, `cursor`) | paid |

All routes are on `https://control.ampdat.com`. The OpenAPI document is at `/openapi.json`.

## Schemas

Each answer names its schema by id. Fetch a schema from
`https://control.ampdat.com/v1/schemas/<id>`.

| Schema | What it describes |
| --- | --- |
| `record/v1` | One change seen on chain: its before and after values, when it takes effect, who may execute and revoke it, and the evidence it rests on |
| `subject/v1` | The envelope for one address: what it is, its state, who controls it, and its recent and pending records |
| `control/v1` | The control graph around a contract at a block: who holds what, through which Safe, module or DAO |
| `state/metamorpho/v1` | A MetaMorpho vault's terms |
| `state/vaultv2/v1` | A Morpho Vaults V2 vault's terms |
| `state/stablecoin/v1` | A stablecoin's contracts, and who holds each capability on them |
| `state/safe/v1` | A Safe's owners and threshold, modules, guards, fallback handler and singleton |

`GET /v1/coverage` says whether each schema version is frozen. Once a version is frozen, the only
additions allowed are optional fields and new enum values, and no field's meaning changes. A
breaking change becomes a new version, served beside the old one for a deprecation period.
