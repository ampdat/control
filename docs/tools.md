# Tools

Every input is an Ethereum mainnet address: `0x` followed by 40 hex digits. Every tool is
annotated `readOnlyHint: true`, `destructiveHint: false`, `idempotentHint: true` and
`openWorldHint: true`.

| Tool | Price | Input | Returns |
| --- | --- | --- | --- |
| `get_coverage` | free | optional `address` | the chain; the covered families, with a count for each; the total number of contracts; the block and time the data is at; example addresses; links. With `address`, it also says whether that address is covered, and by which family |
| `search_covered_contracts` | free | `query`; optional `limit` (default 10, max 25) | covered contracts whose name, asset symbol, family or address contains every word of the query: vaults, coins and Safes (a Safe has no name, so find it by address). Each match has its address, name, asset, family and page link |
| `get_control_state` | free | `address` | a one-paragraph `summary`. `roles`: each role, its holder, and what the holder is. `powers`: each capability, the roles that hold it, the notice it gives, and who can revoke it. `pending`: each queued change, when it can take effect, and who can revoke it. Also `as_of` (block and time) and the link to the free page. Three more fields appear where they apply; see "What each kind of contract returns" |
| `get_control_evidence` | paid | `address`; optional `sections` (any of `state`, `control`, `records`) | the full answer for one contract, in the `subject/v1` shape. It is the same answer as `GET /v1/ethereum/{address}` |
| `get_change_records` | paid | optional `address`, `since` (a block), `kind`, `observed_status`, `family`, `limit` (default 25, max 100) and `cursor` | a page of change records, plus a `next_cursor`. With `address`, that contract's history; without it, the feed across every covered contract |

## How the tools behave

- **Name first, then address.** If the user names a vault or a coin instead of giving its address, call
  `search_covered_contracts`, then `get_control_state` with the address it returns.
- **Uncovered addresses.** An address that is not covered is an answer, not an error:
  `{"covered": false, …}`. The result also says that this is not a finding about the contract,
  good or bad. One case differs: an address that belongs to a covered contract answered at
  another address returns `part_of` with that address, whose answer covers it. Today that is
  one of a coin's other contracts (USDC's MasterMinter, for example); a lending market's other
  contracts will answer the same way.
- **Malformed arguments.** These return an error that names the argument.
- **Result size.** A free result is 1 to 5 KB. The full evidence answer can pass 50 KB; use
  `sections` to ask for only part of it.
- **As of a block.** Each answer is true at its `as_of` block, which is a few minutes behind the
  chain head.
- **Records do not change.** `observed_status` is a record's status when it was first observed,
  and a record never changes after that. So a cap that was submitted stays `pending` in its
  record even after it is accepted or revoked. For what is pending now, use the `pending` list
  in `get_control_state`.

## What each kind of contract returns

`get_control_state` has the same shape for every covered contract. What fills it differs:

| Contract | `roles` and `powers` | Also |
| --- | --- | --- |
| A Morpho vault | owner, curator, guardian or sentinels, allocators; the notice each change gives | `subject.assets_held`. `depends_on` lists what the vault's funds rest on: its asset where that is a covered coin, and a Vaults V2 vault's adapters. A covered one has a `page` and an answer of its own |
| A stablecoin | who can mint, burn, pause, freeze, upgrade the contract or appoint those roles | `subject.total_supply` (a coin has a supply, not assets held) |
| A Safe | its owners (`safe_owner`) and modules; the `summary` gives the threshold and says whether a guard can refuse a transaction | `holds_roles_on`: the vaults and roles it holds. `pending` is always empty, because a Safe changes in one transaction |

They connect. A vault's role holder may be a covered Safe, and its asset a covered coin:
call `get_control_state` again with that address to follow the chain.

More kinds will be added to this table (next, the lending markets and their governance), in the
same tools and the same shape. `subject.kind` says which kind an answer is; a kind may bring a
field of its own, as the three above do, and a field that does not apply is absent.

## Free and paid

Free answers say what is covered, and one contract's current state as its free page states it:
who holds each role and what that holder is, the notice each change gives, and what is pending
and who can revoke it. A Safe's own answer is free too, its owners, threshold and modules
included.

A vault's free answer names each Safe that holds a role and its threshold, not the Safe's
owners. To see them, call `get_control_state` with the Safe's address.

Paid answers give the whole, with the evidence behind it:

- the control chain followed up to three links in one answer (each Safe's owners, modules and
  guard, EIP-7702 delegates);
- the exact state: every market, cap and queue;
- the record, transaction and log behind each pending change;
- the history of changes, with who signed each Safe change, and the feed.
