# Tools

Every input is an Ethereum mainnet address: `0x` followed by 40 hex digits. Every tool is
annotated `readOnlyHint: true`, `destructiveHint: false`, `idempotentHint: true` and
`openWorldHint: true`.

| Tool | Price | Input | Returns |
| --- | --- | --- | --- |
| `get_coverage` | free | optional `address` | the chain; the covered families, with a count for each; the total number of contracts; the block and time the data is at; example addresses; links. With `address`, it also says whether that address is covered, and by which family |
| `search_covered_contracts` | free | `query`; optional `limit` (default 10, max 25) | covered contracts whose name, asset symbol, family or address contains every word of the query. Each match has its address, name, asset, family and page link |
| `get_control_state` | free | `address` | a one-paragraph `summary`. `roles`: each role, its holder, and what the holder is. `powers`: each capability, the roles that hold it, the notice it gives, and who can revoke it. `pending`: each queued change, when it can take effect, and who can revoke it. Also `as_of` (block and time) and the link to the free page |
| `get_control_evidence` | paid | `address`; optional `sections` (any of `state`, `control`, `records`) | the full answer for one contract, in the `subject/v1` shape. It is the same answer as `GET /v1/ethereum/{address}` |
| `get_change_records` | paid | optional `address`, `since` (a block), `kind`, `observed_status`, `family`, `limit` (default 25, max 100) and `cursor` | a page of change records, plus a `next_cursor`. With `address`, that contract's history; without it, the feed across every covered contract |

## How the tools behave

- **Name first, then address.** If the user names a vault instead of giving its address, call
  `search_covered_contracts`, then `get_control_state` with the address it returns.
- **Uncovered addresses.** An address that is not covered is an answer, not an error:
  `{"covered": false, …}`. The result also says that this is not a finding about the contract,
  good or bad.
- **Malformed arguments.** These return an error that names the argument.
- **Result size.** A free result is 1 to 5 KB. The full evidence answer can pass 50 KB; use
  `sections` to ask for only part of it.
- **As of a block.** Each answer is true at its `as_of` block, which is a few minutes behind the
  chain head.
- **Records do not change.** `observed_status` is a record's status when it was first observed,
  and a record never changes after that. So a cap that was submitted stays `pending` in its
  record even after it is accepted or revoked. For what is pending now, use the `pending` list
  in `get_control_state`.

## Free and paid

Free answers say what is covered, and one contract's current state as its free page states it:
who holds each role and what that holder is, the notice each change gives, and what is pending
and who can revoke it.

Paid answers give the evidence behind those facts:

- who the signers of a Safe are, and its modules and guard;
- the exact state;
- the record, transaction and log behind each pending change;
- the history of changes, and the feed.

No Safe signer appears in a free result.
