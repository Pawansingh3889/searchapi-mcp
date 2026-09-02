# searchapi-mcp: plan

An MCP server that exposes SearchApi engines as typed tools for agents.
One tool per engine family, strict input schemas, context-window-aware
result trimming, rate-limit-aware retries, and a CI gate that keeps the
tool schemas in sync with the SearchApi parameter surface.

## Why this exists

Agents increasingly reach live search through MCP clients (Claude Desktop,
Cursor, and every MCP-compatible host). LangChain and LlamaIndex already
wrap SearchApi for in-process use, but there is no maintained official MCP
server. This repo is that server.

## Distribution targets

1. npm package, runnable via `npx searchapi-mcp` with `SEARCHAPI_API_KEY`.
2. PR to the modelcontextprotocol servers directory once the server is real.
3. A docs snippet and demo recipe on the SearchApi side (out of this repo).

## Architecture

```
src/
  index.ts               entrypoint: stdio transport, env validation
  server.ts              MCP server setup, tool registration from registry
  registry.ts            single source of truth: engine params + shapes
  trim.ts                result trimming: full | compact | answers_only
  searchapi/
    client.ts            HTTP boundary: timeouts, retries, rate limits
    types.ts             request/response types generated around registry
  tools/
    google_search.ts     first tool, mirrors the registry exactly
    index.ts             registration helper shared by all tools
test/
  client.test.ts         HTTP client behaviour, mocked at the boundary
  schema-drift.test.ts   the drift gate: registry vs tool schemas
  tools.test.ts          per-tool behaviour incl. trimming modes
.github/workflows/ci.yml build, lint, typecheck, test on node 20 and 22
```

### The registry is the contract

Every engine parameter lives in `registry.ts` exactly once: name, type,
required or optional, description, and how it maps into the request. Tool
input schemas (zod) are derived from it. The drift test fails CI if a
registry param is missing from a tool schema, or if a schema carries a
param the registry does not know. Adding a new engine means adding one
registry entry and one thin tool file; everything else is generated.

### Fail loudly

Missing `SEARCHAPI_API_KEY` is a startup error with a clear message, not a
runtime surprise. 4xx from SearchApi surfaces the upstream error body
rather than being swallowed. Retries apply only to 429 and 5xx, with
exponential backoff capped by a configurable ceiling, and honour
`Retry-After` when present. No silent fallbacks, no default-engine
shrugging.

### Context-window economy

Agents pay per token, so trimming is a first-class feature, not an
afterthought:

- `full`: the raw organic results array, as-is.
- `compact`: position, title, url, snippet, date. Drops knowledge-graph
  bulk, ads, and related-searches.
- `answers_only`: answer box plus top 3 organic titles and urls.

Default is `compact`. The trimming layer is pure and unit-tested against
recorded response shapes.

### Secrets

The key is read from `SEARCHAPI_API_KEY` only. `.env.example` ships with a
placeholder, `.env` is gitignored, and no test ever needs a real key: the
HTTP client is mocked at its boundary so the whole suite runs offline.

## Phases

1. **Scaffold**: pnpm, TypeScript strict, vitest, eslint, biome or prettier
   per repo taste, CI on node 20 and 22. Client and types compile, env
   validation in place.
2. **google_search**: registry entry, derived schema, tool, trimming
   modes, mocked client tests. This is the vertical slice.
3. **Drift gate**: the schema-drift test plus a recorded-fixture test that
   catches upstream response shape changes.
4. **More engines**: google_maps, google_jobs, google_scholar, youtube
   search, each one registry entry plus thin tool file.
5. **Publish prep**: npm metadata, `npx` smoke test in CI, README with a
   30-second setup, then the MCP directory PR.

## Non-goals (for now)

- No response caching layer: clients and hosts already do this better.
- No account management: key-only, no OAuth dance.
- No streaming: SearchApi responses are single-shot JSON.

## Verification gates before anything ships

- `pnpm lint`, `pnpm typecheck`, `pnpm test` all green in CI.
- Drift test proves registry and schemas cannot diverge.
- A recorded fixture pins the response shape the trimmer depends on.
