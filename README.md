# Ampdat Control

**Who controls a DeFi contract, and what is about to change.**

Ampdat Control reads Ethereum mainnet and answers, for every covered contract:

- who holds each role (a vault's owner, curator, guardian and allocators; a lending market's
  governor and pause guardian; a coin's minters and freezers; a Safe's owners) and what that
  holder is: a single key, a Safe and its threshold, a DAO, or a known contract;
- how much notice each kind of change gives (the timelock), and who can revoke it while it waits;
- every change already queued, and when it can take effect.

Every answer is a fact read from the chain at a stated block. It is a title search, not a rating:
it gives no safety score and no investment advice, and it never prepares, signs or sends a
transaction.

Coverage is on Ethereum mainnet:

- every Morpho vault (MetaMorpho v1.0 and v1.1, and Morpho Vaults V2);
- Compound v3's six lending markets and the governance they share (the vote, the Timelock and
  its queue);
- the Safes that hold those contracts' roles;
- six stablecoins and their admin contracts: USDC, USDT, USDS, DAI, PYUSD and USD1.

These are one picture, not four products. A vault's answer names the Safe that holds each role
and the coin the vault is denominated in, a market's names its governance, its guardian Safe and
the coins it holds, and each covered one has an answer of its own.

The `get_coverage` tool and `GET /v1/coverage` list every covered family, with how many contracts
each has.

Next are Aave v3 and SparkLend. They are not covered yet, and an address of theirs answers as
any uncovered address does. They will arrive as Compound v3 did: in the same routes and the same
schemas, with a market's governance as a covered contract of its own.

This repository holds the documentation, the MCP Registry metadata, and a Claude Code plugin with
the `check-control` skill. The service itself is operated by Ampdat LLC and is not open
source.

## How it's made

Anyone can read a contract over RPC. What makes this an answer:

- **Verified source.** A contract is covered only once its published source, recompiled, matches
  the deployed bytecode exactly. Its roles and timelocks are read from that source, not from docs.
- **From the chain.** Roles, timelocks and queued changes are derived from on-chain events and
  state, not from anyone's summary or API. Every read is kept, so any answer can be traced back.
- **Checked.** What is derived is checked against the contracts' own getters.
- **Played forward.** The derivation for Morpho's vaults and for Compound v3 is tested on a fork
  of mainnet: the holders it names can make the call and everyone else is refused, and a queued
  change fails one second before the time it gives and goes through at that time. The
  stablecoins and Safes are not yet tested this way.
- **As of a block.** Data is refreshed every few minutes, and every answer gives the block it is
  as of.

### Check it yourself

You don't have to take our word for it. Take the `as_of` block from an answer and read the same
values from any Ethereum node at that block. For example, for a Morpho vault:

```
cast call <vault> "owner()(address)" --block <as_of block> --rpc-url <your node>
cast call <vault> "curator()(address)" --block <as_of block> --rpc-url <your node>
```

Etherscan's Read Contract tab gives the current values. `get_control_evidence` gives the
transaction and log behind each pending change, so a queued change can be checked on any block
explorer.

No independent party has reviewed the service yet. Reproducing answers is the way to check it.

## Connect

The MCP server is remote: there is nothing to install, and no sign-in.

| Endpoint | Tools |
| --- | --- |
| `https://control.ampdat.com/mcp` | three free tools and two paid ones |
| `https://control.ampdat.com/free/mcp` | the three free tools only |

**Claude (web, desktop, mobile):** Settings, Connectors, Add custom connector. Use the URL
`https://control.ampdat.com/mcp` and leave the OAuth fields empty.

**Claude Code:**

```bash
claude mcp add --transport http ampdat-control https://control.ampdat.com/mcp
```

Or install the plugin, which adds both the server and the skill. Run these inside Claude Code:

```
/plugin marketplace add ampdat/control
/plugin install ampdat-control@ampdat
```

**ChatGPT (developer mode):** add a connector with the URL
`https://control.ampdat.com/free/mcp` and "No authentication".

More clients, including the Python SDK and MCP Inspector, are covered in
[docs/connect.md](docs/connect.md).

## Tools

| Tool | Price | What it answers |
| --- | --- | --- |
| `get_coverage` | free | what is covered, and whether one address is |
| `search_covered_contracts` | free | a covered contract's address from its name, asset symbol or part of its address |
| `get_control_state` | free | who holds each role and what that holder is, the notice each change gives, and what is pending |
| `get_control_evidence` | paid | the full evidence-backed answer: the whole control chain in one answer, exact caps and queues, and the transaction and log behind each pending change |
| `get_change_records` | paid | one contract's change history, or the feed across all covered contracts |

The inputs, the outputs and how each tool behaves are in [docs/tools.md](docs/tools.md).

## Paying

The two paid tools are paid per call in USDC on Base, using the
[x402](https://www.x402.org/) protocol over MCP. If a call carries no payment, nothing is
charged: it returns the price and the HTTP URL that sells the same answer.
[docs/payments.md](docs/payments.md) explains how to pay from an assistant today.

## The skill

[`check-control`](plugins/ampdat-control/skills/check-control/SKILL.md) tells an
assistant to check who controls a Morpho vault or a Compound v3 market before a deposit, without
being asked. It also
covers questions about who controls a stablecoin or a Safe. It tells the assistant how to report
the answer, and what it must never say. It works alongside
Morpho's own MCP server (`https://mcp.morpho.org/`).

You can use it in either of these ways:

- Install the plugin above.
- Copy the skill's folder to `~/.claude/skills/`, or upload it as a skill in claude.ai.

## Also here

- [docs/api.md](docs/api.md): the HTTP API, and the published JSON schemas.
- [examples/prompts.md](examples/prompts.md): example prompts, with the tools each one calls.
- [server.json](server.json): the MCP Registry entry (`com.ampdat/control`).
- [docs/privacy-and-support.md](docs/privacy-and-support.md): what is logged, and how to reach us.

## Licence

The contents of this repository (the docs, the skill and the plugin) are under the
[MIT licence](LICENSE). The service and its data are governed by its terms of service:
<https://control.ampdat.com/terms>.
