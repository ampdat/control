# Ampdat Control

**A title search for DeFi: deterministic control checks and audit trails before you—or your agent—deposit.**

Ampdat Control reads Ethereum mainnet and answers, for every covered contract:

- who holds each role (a vault's owner, curator, guardian and allocators; a lending market's
  governor and pause guardian; a coin's minters and freezers; a Safe's owners) and what that
  holder is: a single key, a Safe and its threshold, a DAO, or a known contract;
- how much notice each kind of change gives (the timelock), and who can revoke it while it waits;
- the changes queued in the contracts it reads, and when each can take effect.

Every answer is a deterministic fact read from the chain at a stated block, with no LLM in the
runtime loop. Use it as a pre-deposit guardrail, a block-pinned audit receipt (`subject/v1`,
`record/v1`), or a feed for custody drift while holding. Not a rating: it gives no safety score
and no investment advice, and it never prepares, signs or sends a transaction.

Coverage is on Ethereum mainnet:

- every Morpho vault (MetaMorpho v1.0 and v1.1, and Morpho Vaults V2);
- Compound v3's six lending markets and the governance they share (the vote, the Timelock and
  its queue);
- many of the Safes that hold those contracts' roles;
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

- **Known code.** How a contract's code is identified depends on what it is:
  - *Morpho vaults:* each version was recompiled from published source and matched to the
    deployed bytecode, apart from values set in the constructor. A new vault is accepted because
    that version's factory created it.
  - *Compound v3:* each market's implementation must match a build recompiled the same way. The
    proxies, factories and governance contracts are pinned by code hash, not recompiled.
  - *Safes:* the proxy and the code it runs are each compared with recompiled builds.
  - *Stablecoins:* the contracts and their implementations are pinned by code hash, not
    recompiled. Their rules were read from source verified by others, such as Etherscan.

  Covered does not always mean recognised code. A Safe with an unknown proxy, or a contract
  upgraded to code not yet listed, is still served: its page says so.
- **From the chain.** No LLM in the runtime loop: roles, timelocks and queued changes are derived
  by a deterministic state machine replayed from on-chain events and state, not from anyone's
  summary or API. What each refresh reads is kept, so an answer can be rebuilt.
- **Checked.** What is derived is compared with the contracts' own getters. For Compound v3 and
  the stablecoins, key values are compared at each refresh and any difference is in the answer.
  For vaults and Safes, the comparison is made at audits.
- **Played forward.** On a fork of mainnet, the main permissions and timelocks were tested for
  Compound v3 and for samples of Morpho vaults: the holders named could make the call, the other
  accounts tried were refused, and a queued change failed one second before the time given and
  opened at it. The stablecoins and Safes are not yet tested this way.
- **As of a block.** Events are read about every 10 minutes, and every answer gives the block it
  is as of. What kind of account a holder is gets re-read less often, usually daily.

### What it does not see

- **Safes outside the list.** The covered Safes are a fixed list, found by surveying role
  holders. A Safe given a role since then is named as a holder, and may have no answer of its own.
- **Queues in other contracts.** Sky's governance queue (behind USDS and DAI), an issuer's
  multisig submissions and the queues of some contracts that own vaults are not read. "Nothing
  pending" means nothing in what is read.
- **What logs nothing.** Some changes emit no event, such as a Safe changed by delegatecall or a
  transfer of USDT's ownership. They are seen only when the value is next compared, and a change
  undone before then is missed.
- **What is off chain.** Signatures being collected, and who stands behind a key: an MPC wallet
  looks like a single key.
- **The end of the chain.** A holder whose rules are not followed is named and left unresolved.

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

**OpenClaw, Nous Hermes and other MCP / skill harnesses:** point your MCP client at
`https://control.ampdat.com/mcp` (or `/free/mcp` for free tools only) and load the
[`check-control`](plugins/ampdat-control/skills/check-control/SKILL.md) skill as a pre-deposit
guardrail.

More clients, including the Python SDK and MCP Inspector, are covered in
[docs/connect.md](docs/connect.md).

## Tools

| Tool | Price | What it answers |
| --- | --- | --- |
| `get_coverage` | free | what is covered, and whether one address is |
| `search_covered_contracts` | free | a covered contract's address from its name, asset symbol or part of its address |
| `get_control_state` | free | who holds each role and what that holder is, the notice each change gives, and what is pending |
| `get_control_evidence` | paid | the full evidence-backed answer: the control chain as far as it is followed, exact caps and queues, and the transaction and log behind each pending change |
| `get_change_records` | paid | one contract's change history, or the feed across all covered contracts |

The inputs, the outputs and how each tool behaves are in [docs/tools.md](docs/tools.md).

## Paying & team access

Current control state is free on the web and over MCP. The two paid tools (`get_control_evidence`
and `get_change_records`) and the HTTP API (`subject/v1`, `record/v1`) are paid per call in USDC
on Base using the [x402](https://www.x402.org/) protocol — no account required. If a call carries
no payment, nothing is charged: it returns the price and the HTTP URL that sells the same answer.
[docs/payments.md](docs/payments.md) explains how to pay from an assistant today.

For API keys, watchlists, alert webhooks, or custom integrations (for allocators, risk teams,
yield copilots and observability rails), email <info@ampdat.com>.

## The skill

[`check-control`](plugins/ampdat-control/skills/check-control/SKILL.md) tells an
assistant or agent harness to check who controls a Morpho vault or a Compound v3 market before a
deposit, without being asked. It also
covers questions about who controls a stablecoin or a Safe. It tells the assistant how to report
the answer, and what it must never say. It works alongside
Morpho's own MCP server (`https://mcp.morpho.org/`).

You can use it in any of these ways:

- Install the plugin above (Claude Code).
- Load [`plugins/ampdat-control/skills/check-control/SKILL.md`](plugins/ampdat-control/skills/check-control/SKILL.md) into OpenClaw, Nous Hermes, or any harness that supports `SKILL.md` skills.
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
