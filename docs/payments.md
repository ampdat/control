# Paying for the evidence

`get_control_evidence` and `get_change_records` are paid per call, in USDC on Base. Payment uses
[x402's MCP transport](https://github.com/x402-foundation/x402/blob/main/specs/transports-v2/mcp.md).
There is no account and no API key.

## What happens on a paid call

1. **Without a payment**, the call returns a tool result with `isError: true` and charges
   nothing:
   - its `structuredContent` is x402's `PaymentRequired` object, which states the price;
   - it also includes a paragraph a model can read: the price, that nothing was charged, what
     the paid answer contains, the HTTP URL that sells the same answer, and the free tool to use
     meanwhile.
2. **A client that can pay** retries the same call with a signed payment in
   `params._meta["x402/payment"]`.
3. **The server** verifies the payment, returns the answer, and settles. The receipt is in
   `result._meta["x402/payment-response"]`.

A call is checked before any payment is asked for. An address that is not covered, or a
malformed query, is refused without a challenge. So you never pay for a request that cannot be
answered.

## Paying from an assistant today

Claude and ChatGPT do not pay x402 tool calls themselves. To buy the same answer, use a wallet
tool on the HTTP route, `GET https://control.ampdat.com/v1/ethereum/<address>`. Two options:

- **Coinbase's payments MCP server,** for Claude Desktop, Claude Code and other stdio clients:
  `npx @coinbase/payments-mcp`. It makes an x402 request to a URL and pays it in USDC on Base.
- **Coinbase's wallet CLI:**

  ```bash
  npx awal@latest x402 pay https://control.ampdat.com/v1/ethereum/<address>
  ```

An agent built on the x402 SDKs (for example `@x402/mcp` or the Python `x402` package) can pay
the MCP tool directly.

The `check-control` skill tells an assistant to state the price and ask the user before
paying, and never to pay on its own initiative.

## Prices

The price is set per call and is stated in every payment challenge, which is the authoritative
figure. The MCP tools cost the same as the HTTP routes that sell the same answer.

The HTTP API also gives each wallet a few free calls after Sign-In-With-X (SIWX). The MCP tools
do not: over MCP, the three free tools are the free tier.
