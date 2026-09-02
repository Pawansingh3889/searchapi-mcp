# searchapi-mcp

Unofficial MCP server for [SearchApi](https://www.searchapi.io/): search
engines as typed tools for agents. Run it under any MCP client (Claude
Desktop, Cursor, and every MCP-compatible host) and give the model live
SERP data with strict input schemas, context-window-aware result
trimming, and an explicit boundary around untrusted result text.

Status: **in planning**. See [PLAN.md](./PLAN.md) for the architecture,
phases, and gates. Nothing ships until the drift gates and the envelope
test are green. License will be MIT.

## Why an unofficial one

There is no official SearchApi MCP server, and the community ones are
unmaintained: the `searchapi-mcp` npm package has a single version,
published 2 June 2025, with nothing since. The npm name here is
`mcp-searchapi`, because `searchapi-mcp` and `searchapi-mcp-server` are
taken.

This project is not built or endorsed by SearchApi. If they want it, the
repo moves under their org and the naming follows.

## Planned usage

```json
{
  "mcpServers": {
    "searchapi": {
      "command": "npx",
      "args": ["-y", "mcp-searchapi"],
      "env": { "SEARCHAPI_API_KEY": "your-key" }
    }
  }
}
```

## Design in one line

Every engine parameter lives once in a registry, tool schemas derive from
it, a checked-in dated snapshot of SearchApi's documented parameters gets
diffed weekly against the live docs, and results leave only inside an
untrusted-data envelope. SERP text is data, never instructions.
