# The HTTP API

The MCP tools are a second way in to the same answers. Over HTTP, the free routes are pages and
metadata. The paid routes use [x402](https://www.x402.org/): a request without a payment gets
`402 Payment Required` with the price. Every paid route answers or refuses (400, 404 or 503)
before any payment is asked for.

| Route | Returns | Price |
| --- | --- | --- |
| `GET /ethereum/{address}` | a page for people: who holds each role and what that holder is, the notice each change gives, and what is pending and who can revoke it | free |
| `GET /` | what is pending across every covered contract, and when it can take effect | free |
| `GET /vaults` | every covered vault, then the coins, the lending markets with their governance, and the Safes, each linked to its page | free |
| `GET /llms.txt` | what the service is and the free ways in, for an assistant | free |
| `GET /v1/coverage` | the covered families; every situation id a record can carry, each with `recognised`; the schemas, whether they are frozen and which parts are still provisional; an example address of each kind of contract | free |
| `GET /v1/schemas/{id}` | a published JSON schema (below) | free |
| `GET /v1/ethereum/{address}` | the subject envelope: what the contract is, its state, who controls it, and its recent and pending records | paid |
| `GET /v1/ethereum/{address}/records` | that contract's records, oldest first (query: `since`, `kind`, `observed_status`, `limit`, `cursor`) | paid |
| `GET /v1/records` | the feed across contracts (query: `since`, `family`, `kind`, `observed_status`, `limit`, `cursor`) | paid |

All routes are on `https://control.ampdat.com`. The OpenAPI document is at `/openapi.json`.

An address that is not covered is a 404 with `{error, detail, address}`. If it belongs to a
covered contract that is answered at another address (one of a coin's other contracts, or one the lending markets share, such as the Governor),
the body also has `part_of`, that address, and the free page redirects there. Every error is
`{error, detail}`. A malformed query is a 400, and so is a `family` that `/v1/coverage` does not
list (families are builds: `safe-1.4.1`, not `safe`).

Records pages are `{as_of, subject, records, next}`, oldest first; the last page has
`next: null`. To follow the feed, ask again with `since` set to the block of the last record you
hold and drop the ids you already have. A record above `as_of.finalised_block` can still be
dropped by a reorg.

## Records

A record's `situation.id` names the kind of change, in one form for every family:
`<rules>/<Event>[/<key>=<value>,...]`, for example `metamorpho/SubmitCap/…`, `vaultv2/SetName`
or `stablecoin-usdc/OwnershipTransferred/contract=token`. The build that ran is the record's
`family`. `situation.recognised` is true when a checked explanation covers that situation; a
record whose situation is not recognised is still published, with the same facts.

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
| `state/compound/v1` | A Compound v3 market: what its running code was built with, what is staged, its pause flags and the calls queued for it |
| `state/compound-governance/v1` | The governance the markets share: the Timelock and its queue, the Governor and its open proposals |
| `state/safe/v1` | A Safe's owners and threshold, modules, guards, fallback handler and singleton |

v1 is frozen (`GET /v1/coverage` says `"frozen": true`). The only additions allowed are optional
fields and new enum values, and no field's meaning changes. A breaking change becomes a new
version, served beside the old one for a deprecation period. So ignore a field you do not know.
A record's `id`, `situation.id` and `setting`, and a graph edge's `capability` and `role`, are
never renamed.

The freeze covers what the served families use. The parts of the schemas that none uses yet are
listed under `provisional` in `GET /v1/coverage` and may still change until one does.

New families bring new enum values: a unit, a status, a kind of target or of graph node. Treat a
value you do not know as opaque, and never read a number in an unknown unit as a plain quantity.
Compound v3 brought, among others, the status `proposed` for a change that is under vote and not
yet queued, and records of kind `bundle` for a proposal, with the records it made linked by
`caused_by`. Aave v3 and SparkLend are planned next.

A graph edge's `delay_s` is that edge's own delay. The notice a reader gets is the sum along the
path behind it: a market's proxy admin has no delay of its own and is owned by a Timelock with
two days, so an upgrade gives at least two days. Do not read one edge's 0 as "no notice".
