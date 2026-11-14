# Connecting

Both endpoints use Streamable HTTP and need no sign-in. They are stateless: each POST gets one
JSON response, there is no session id, and there is no server-to-client stream.

| Endpoint | Tools | Use it for |
| --- | --- | --- |
| `https://control.ampdat.com/mcp` | all five | Claude, Claude Code, any MCP client |
| `https://control.ampdat.com/free/mcp` | the three free tools | ChatGPT, or any client that should never see a price |

The free endpoint's descriptions, instructions and results never mention a price or a purchase.

## Claude (web, desktop, mobile)

1. Open Settings, then Connectors, then Add custom connector.
2. Enter the URL `https://control.ampdat.com/mcp`.
3. Leave the OAuth fields empty.

## Claude Code

```bash
claude mcp add --transport http ampdat-control https://control.ampdat.com/mcp
```

Or, inside Claude Code, install the plugin from this repository. It adds both the server and the
`check-vault-control` skill:

```
/plugin marketplace add ampdat/control
/plugin install ampdat-control@ampdat
```

## ChatGPT

1. Turn on developer mode.
2. Add a connector with the URL `https://control.ampdat.com/free/mcp`.
3. Choose "No authentication".

## Python (official SDK, `mcp` 2.x)

```python
from mcp import Client

async with Client("https://control.ampdat.com/mcp") as client:
    result = await client.call_tool(
        "get_control_state", {"address": "0xbeef01735c132ada46aa9aa4c54623caa92a64cb"}
    )
    print(result.structured_content["summary"])
```

## MCP Inspector

```bash
npx @modelcontextprotocol/inspector --cli https://control.ampdat.com/mcp --transport http --method tools/list
```

## Without MCP

Every covered contract has a free page for people at `https://control.ampdat.com/ethereum/<address>`. The
same facts are available over HTTP; see [api.md](api.md).
