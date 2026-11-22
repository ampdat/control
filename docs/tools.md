# Tools

Every input is an Ethereum mainnet address: `0x` followed by 40 hex digits. Every tool is
annotated `readOnlyHint: true`, `destructiveHint: false`, `idempotentHint: true` and
`openWorldHint: true`.

| Tool | Price | Input | Returns |
| --- | --- | --- | --- |
| `get_coverage` | free | optional `address` | the chain; the covered families, with a count for each; the total number of contracts; the block and time the data is at; example addresses; links, including `method` (how answers are made). With `address`, it also says whether that address is covered, and by which family |
| `search_covered_contracts` | free | `query`; optional `limit` (default 10, max 25) | covered contracts whose name, asset symbol, family or address contains every word of the query: vaults, lending markets and their governance, coins and Safes (a Safe has no name, so find it by address). Each match has its address, name, asset, family and page link |
| `get_control_state` | free | `address` | a one-paragraph `summary`. `roles`: each role, its holder, and what the holder is. `powers`: each capability, the roles that hold it, the notice it gives, and who can revoke it. `pending`: each queued change, when it can take effect, and who can revoke it. Also `as_of` (block and time) and the link to the free page. More fields appear where they apply; see "What each kind of contract returns" |
| `get_control_evidence` | paid | `address`; optional `sections` (any of `state`, `control`, `records`) | the full answer for one contract, in the `subject/v1` shape. It is the same answer as `GET /v1/ethereum/{address}` |
| `get_change_records` | paid | optional `address`, `since` (a block), `kind`, `observed_status`, `family`, `limit` (default 25, max 100) and `cursor` | a page of change records, plus a `next_cursor`. With `address`, that contract's history; without it, the feed across every covered contract |

## How the tools behave

- **Name first, then address.** If the user names a vault, a market or a coin instead of giving its address, call
  `search_covered_contracts`, then `get_control_state` with the address it returns.
- **Uncovered addresses.** An address that is not covered is an answer, not an error:
  `{"covered": false, …}`. The result also says that this is not a finding about the contract,
  good or bad. One case differs: an address that belongs to a covered contract answered at
  another address returns `part_of` with that address, whose answer covers it. That is one of a
  coin's other contracts (USDC's MasterMinter, for example), or one the lending markets share
  (Compound's Governor or Configurator, answered by its governance).
- **Malformed arguments.** These return an error that names the argument.
- **Result size.** A free result is 1 to 5 KB. The full evidence answer can pass 50 KB; use
  `sections` to ask for only part of it.
- **As of a block.** Each answer is true at its `as_of` block, which is a few minutes behind the
  chain head.
- **Records do not change.** `observed_status` is a record's status when it was first observed,
  and a record never changes after that. So a cap that was submitted stays `pending` in its
  record even after it is accepted or revoked. For what is pending now, use the `pending` list
  in `get_control_state`.

## Reading notice

A power's notice is read along the whole chain behind it, not off the last link. It is how soon
the power could be used by the quickest holder.

| `notice_s` | `notice` | Meaning |
| --- | --- | --- |
| `0` | `none` | a known holder can act at once |
| seconds | `7 days` | exactly that wait |
| seconds | `at least 2 days`, and perhaps `, after a token vote` | a lower bound: a timelock's minimum, or a holder that may add a wait of its own |
| `null` | `not known: …` | the holder is a contract, DAO or unclassified account whose rules are not followed. It may add a delay. This is not "no notice" |

Where holders of one power wait differently, `notice` is the shortest and `by` has each group:
its `roles`, `holders`, `notice_s`, `notice` and `revocable_by`. A Compound market's `pause` is
the example: at once by the guardian Safe, at least two days by governance.

## What each kind of contract returns

`get_control_state` has the same shape for every covered contract. What fills it differs:

| Contract | `roles` and `powers` | Also |
| --- | --- | --- |
| A Morpho vault | owner, curator, guardian or sentinels, allocators; the notice each change gives | `subject.assets_held`. `depends_on` lists what the vault's funds rest on: its asset where that is a covered coin, and a Vaults V2 vault's adapters. A covered one has a `page` and an answer of its own |
| A lending market (Compound v3) | proxy admin, governor and pause guardian; `upgrade`, `pause`, `withdraw_reserves`, and `set_market_settings` (caps, rates and collateral, set in the Configurator for this market) | `governance`: the address to ask next. `staged_not_live`: settings changed and not yet deployed. `paused`: each switch. `pending`: queued calls that can still run |
| Compound's governance | the Governor, which queues and executes through the Timelock, and the Timelock's power over itself | `open_proposals`: what is under vote or waiting to be queued, with the voting blocks and the markets it names. These are not in `pending`, which holds queued calls only |
| A stablecoin | who can mint, burn, pause, freeze, upgrade the contract or appoint those roles | `subject.total_supply` (a coin has a supply, not assets held). `contracts`: every contract of the coin (the token, and for example USDC's MasterMinter or USD1's TokenGovernor), each with its capabilities and who holds them. `roles` and `powers` are the token's own, so read `contracts` for keys held one contract away |
| A Safe | its owners (`safe_owner`) and modules; the `summary` gives the threshold and says whether a guard can refuse a transaction | `holds_roles_on`: the vaults and roles it holds. `pending` is always empty, because a Safe changes in one transaction |

They connect. A vault's role holder may be a covered Safe, and its asset a covered coin:
call `get_control_state` again with that address to follow the chain.

More kinds will be added to this table (next, Aave v3 and SparkLend), in the same tools and the
same shape. `subject.kind` says which kind an answer is; a kind may bring a field of its own,
as those above do, and a field that does not apply is absent.

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
