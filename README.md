# searchapi-mcp

MCP server for [SearchApi](https://www.searchapi.io/): search engines as
typed tools for agents. Run it under any MCP client (Claude Desktop,
Cursor, and every MCP-compatible host) and give the model live SERP data
with strict input schemas and context-window-aware result trimming.

Status: **in planning**. See [PLAN.md](./PLAN.md) for the architecture,
phases, and gates. Nothing ships until the drift gate and the recorded
fixture tests are green.

## Planned usage

```json
{
  "mcpServers": {
    "searchapi": {
      "command": "npx",
      "args": ["-y", "searchapi-mcp"],
      "env": { "SEARCHAPI_API_KEY": "your-key" }
    }
  }
}
```

## Design in one line

One registry entry per engine parameter, tool schemas derived from it, and
a CI test that fails if the two ever diverge.
