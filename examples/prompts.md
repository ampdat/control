# Example prompts

Each prompt lists the tools an assistant should call and what a good answer contains.

| Prompt | Tools | A good answer |
| --- | --- | --- |
| Who controls the Morpho vault 0x2371e134e3455e0593363cbf89d3b6cf53740618? | `get_control_state` | Names it as the Gauntlet WETH Prime vault (MetaMorpho v1.0). Says who holds the owner, curator, guardian and allocator roles, and what each holder is. Gives the timelock, any pending changes, and the block the answer is as of. Gives no safety score. |
| Find the Steakhouse High Yield Term USDC vault and tell me what changes are pending. | `search_covered_contracts`, then `get_control_state` | Resolves the name to 0xbeef1898443d2eb9c9c471e073f46a3928db21de, asking the user if several vaults match. Lists each pending change, when it can take effect and who can revoke it, or says nothing is pending. |
| Who can freeze or mint USDC? | `search_covered_contracts`, then `get_control_state` | Resolves USDC to 0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48. Says who can mint, freeze, pause and upgrade it and what each holder is, and whether a key change is pending. Gives no opinion on the issuer. |
| Who are the signers of the Safe 0x0000aeb716a0df7a9a1aad119b772644bc089da8, and what does it control? | `get_control_state` | Gives the threshold and the owners, says whether modules or a guard are set, and lists the vaults and roles it holds (`holds_roles_on`). |
| What does Ampdat Control cover? | `get_coverage` | Gives the chain, the covered families, how many contracts are covered, and the block the data is at. |
| Who controls 0x0000000000000000000000000000000000000001? | `get_control_state` | Says the address is not covered, and that this says nothing about the contract. Invents no role holders. |
| I'm about to deposit into 0xbeef01735c132ada46aa9aa4c54623caa92a64cb. How much notice would I get before the curator raises a supply cap, and who could stop it? | `get_control_state` | Gives the notice for a cap raise (the vault's timelock), the roles that can revoke a pending raise and what those holders are, and whether a raise is already pending. Does not say whether to deposit. |

## Prompts the tools should not answer

| Prompt | Why not |
| --- | --- |
| Send 100 USDC from my wallet to 0x2371…0618. | The tools only read; they never send a transaction. |
| How much is one ETH worth in dollars right now? | The tools have no market or price data. |
| Is this vault safe? | The tools report who controls a vault and what can change. They give no rating. The assistant should give those facts instead of a verdict. |
