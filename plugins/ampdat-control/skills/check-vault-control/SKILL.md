---
name: check-vault-control
description: Check who controls a Morpho vault before depositing. Use when the user is about to deposit into, is considering, or already holds a Morpho vault (MetaMorpho v1.0 or v1.1, or Morpho Vaults V2) on Ethereum, or asks who controls, owns or curates a vault, who can change its terms, how long its timelock is, who can veto, or what changes are pending. Looks up the vault's role holders (a single key, a Safe and its threshold, a DAO), the notice each change gives and every queued change with Ampdat Control, and reports them as facts with the block they are as of. Also use it when the user asks who can mint, freeze, pause or upgrade a major stablecoin (USDC, USDT, USDS, DAI, PYUSD, USD1), or who controls a Safe that holds a vault role. A title search, not a rating - it never produces a safety score or investment advice.
---

# Check who controls the vault before depositing

A vault's yield is set by people: an owner, a curator, allocators, and sometimes a guardian or
sentinels who can veto. Before a user puts money in, tell them who those people are, how much
warning the vault gives before its terms change, and what is already queued. This is a title
search, not a rating: you report what the chain says and the user decides.

The facts come from **Ampdat Control** (`https://control.ampdat.com`), which reads every Morpho
vault on Ethereum mainnet, the Safes that hold those vaults' roles, and six stablecoins' contracts,
and answers deterministically, at a stated block. Its MCP server is
`https://control.ampdat.com/mcp`.

## When to run this

- The user asks to deposit into a Morpho vault, or you are about to prepare a deposit for them.
  Run the check first, without being asked, and show the result before any transaction.
- The user asks who controls, owns or curates a vault, whether its terms can change, what its
  timelock is, or what is pending.
- The user already holds a vault position and asks whether anything is about to change.

## Steps

### 1. Get the vault's address

The check takes an address on Ethereum mainnet (chain id 1), not a name.

- If the user gave an address, use it.
- If they gave a name, resolve it. With Morpho's MCP server connected (`https://mcp.morpho.org/`),
  `morpho_query_vaults` takes a `search` string and returns each match's `address`, `chain_id`
  and `name`. Otherwise `search_covered_contracts` on Ampdat Control finds covered vaults by
  name, asset symbol or address.
- Anyone can deploy a vault under any name. If more than one vault matches, or the match is not
  the one the user clearly meant, show the candidates and ask which. Never pick silently.
- If the vault is on another chain (Base, Arbitrum and so on), say that this check covers
  Ethereum mainnet only and that you have not checked who controls it. Do not substitute a
  same-named vault on Ethereum.

### 2. Read the control state

Call `get_control_state` with the address. It is free.

- `covered: false` means Ampdat Control does not cover the address. Say exactly that. It is not
  a finding about the vault, good or bad. `get_coverage` says what is covered.
- If the Ampdat Control tools are not connected, read the same facts from the vault's free page,
  `https://control.ampdat.com/ethereum/<address>`.

### 3. Tell the user, in this order

Lead with the tool's `summary` sentence, then:

1. **Who can change the terms.** Each entry in `roles`: the role, and what its holder is
   (`holder_is`): a single key, an m-of-n Safe, a Safe with modules that can act without the
   signers, a DAO, a known contract, or not yet identified. Say what the role can do; `powers`
   lists each capability and the roles that hold it.
2. **How much notice there is.** For each capability in `powers`, `notice` is how long a change
   waits before it can take effect and `revocable_by` is who can cancel it while it waits. Name
   the changes that take effect with no notice.
3. **What is already queued.** Each entry in `pending`: the change, when it can take effect
   (`status`, and `can_take_effect_now`), and who can revoke it. If nothing is pending, say so.
4. **As of when.** Give `as_of.block` and `as_of.time`, and the `page` link so the user can
   read it themselves.

State these plainly when they are true, as facts and without a verdict:

- a role is held by a single key, or by a Safe that one signer can move (1-of-n);
- a Safe holding a role has modules, which can act without the signers;
- the timelock is zero, or a capability has no notice;
- there is no guardian or sentinel, so nobody but the curator and owner can revoke a change;
- a pending change can take effect now, or an ownership handover is waiting to be accepted;
- a holder is not yet classified or not identified: say it is unknown, never guess.

### 4. Then let the user decide

Ask whether they want to go ahead. Only then continue with the deposit flow, for example
Morpho's `morpho_prepare_deposit`, which returns transactions for the user to sign and never
signs or sends anything itself. If the user changes their mind, stop.

<!-- paid:start -->
## When the free answer is not enough

`get_control_state` says what each role holder is. The evidence behind it is the paid answer:

- `get_control_evidence`: who the signers of each Safe are, its modules and guard, EIP-7702
  delegates, the exact caps and queues, and for every pending change the transaction and log it
  rests on. Use it when the user wants to verify the answer or look behind a Safe, or is
  depositing an amount large enough to audit first.
- `get_change_records`: the vault's history of changes, or the feed across all vaults. Use it
  when the user asks what this curator has changed before, or wants to monitor a position.

Both are paid per call over x402 (USDC on Base). Called without a payment they charge nothing
and return the price and the HTTP URL that sells the same answer
(`GET https://control.ampdat.com/v1/ethereum/<address>`). Tell the user the price and ask before
paying; never pay on your own initiative. If you have an x402-capable wallet tool and the user
agrees, use it on that URL. If you cannot pay, tell the user what the paid answer contains and
where it is, and do not guess at its contents.
<!-- paid:end -->

## Stablecoins and Safes

The same tools answer for two more kinds of contract. Read them out the same way, as facts:

- **A stablecoin** (USDC, USDT, USDS, DAI, PYUSD or USD1). `roles` and `powers` say who can mint,
  burn, pause, freeze, upgrade the contract or appoint those roles, and what each holder is. When
  a vault's asset is one of these coins, offer to check the coin too: who controls the asset is
  part of who controls the deposit.
- **A Safe** that holds a vault role. The `summary` says how many owners must sign, whether
  modules can act with no signature, and whether a guard can refuse a transaction.
  `holds_roles_on` lists the vaults and roles it holds. A Safe changes in one transaction, so its
  `pending` is always empty. That does not mean it cannot change.

## Working beside Morpho's own tools

Morpho publishes an MCP server at `https://mcp.morpho.org/` (experimental; its tool names can
change, so trust the tool list you are given over this page). The two fit together:

| Step | Tool |
| --- | --- |
| Find the vault and its address, APY and curator name | Morpho: `morpho_query_vaults`, `morpho_get_vault` |
| Who holds each role and what that holder is, notice, pending changes and who can revoke them | Ampdat Control: `get_control_state` |
| Build the deposit transaction for the user to sign | Morpho: `morpho_prepare_deposit` |

`morpho_get_vault` (selector `<chainId>:<vaultAddress>`) also returns owner, curator, guardian
and timelock addresses. Ampdat Control adds what those addresses are, what is queued and who can
stop it. If the two disagree, report both with their block or time and do not pick one: Ampdat
Control's answer is as of `as_of.block`, a few minutes behind the chain head.

## Do not

- Do not give a safety score, a grade, a ranking or the words "safe" or "unsafe". Do not say
  whether to deposit. If asked "is it safe?", say this check answers who controls the vault and
  what can change, and give those facts.
- Do not present the answer as an audit of the vault's code, its markets, its oracle or its
  yield. It covers control and pending changes only.
- Do not fill gaps. An unclassified holder, an uncovered address or an error is reported as
  that.
- Do not sign, send or simulate a transaction as part of this check.
