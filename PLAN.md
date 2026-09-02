# searchapi-mcp: plan

An unofficial MCP server that exposes SearchApi engines as typed tools
for agents. One tool per engine family, strict input schemas,
context-window-aware result trimming, rate-limit-aware retries, and a
two-part drift gate that keeps the tool schemas in sync with what
SearchApi actually documents and serves.

Package name: `mcp-searchapi`. The `searchapi-mcp` name is taken on npm
(community package, one version, published June 2025, unmaintained), and
`searchapi-mcp-server` is gone too. `mcp-searchapi` was free as of
September 2026 and reads as what it is.

Status: this is an unofficial client. SearchApi does not own or endorse
it. If they want to adopt it, the repo transfers or mirrors under their
org and the naming discussion happens then.

## Why this exists

Agents increasingly reach live search through MCP clients (Claude Desktop,
Cursor, and every MCP-compatible host). There is no official SearchApi MCP
server, and the community ones are unmaintained: the `searchapi-mcp`
npm package has a single version, published 2 June 2025, with no release
since. This repo is the maintained, unofficial alternative.

## Distribution targets

1. npm package `mcp-searchapi`, runnable via `npx -y mcp-searchapi` with
   `SEARCHAPI_API_KEY` set.
2. Publish to the MCP Server Registry (registry.modelcontextprotocol.io)
   via its quickstart: a `server.json` in the repo, submitted through the
   registry's publish flow. The modelcontextprotocol/servers repo no
   longer accepts new server implementations, and its README list of
   third-party servers was retired in favour of the registry, so a PR
   there would be rejected on sight. The registry is the right door
   anyway: it is the discovery surface people actually browse.
3. A docs snippet and demo recipe offered to SearchApi (out of this repo,
   and offered as an unofficial project unless they say otherwise).

## Architecture

```
src/
  index.ts               entrypoint: stdio transport, env validation
  server.ts              MCP server setup, tool registration from registry
  registry.ts            single source of truth: engine params + shapes
  envelope.ts            untrusted-data envelope around every result
  trim.ts                result trimming: full | compact | answers_only
  searchapi/
    client.ts            HTTP boundary: timeouts, retries, rate limits
    types.ts             request/response types generated around registry
  tools/
    google_search.ts     first tool, mirrors the registry exactly
    index.ts             registration helper shared by all tools
docs/
  engines/
    google-search.md     dated snapshot of SearchApi's documented params
test/
  client.test.ts         HTTP client behaviour, mocked at the boundary
  schema-drift.test.ts   gate 1: registry vs derived tool schemas
  snapshot-drift.test.ts gate 2: registry vs the dated docs snapshot
  envelope.test.ts       results can only leave as quoted data
  tools.test.ts          per-tool behaviour incl. trimming modes
server.json              MCP Server Registry manifest (registry quickstart)
.github/workflows/
  ci.yml                 build, lint, typecheck, test on node 20 and 22
  docs-drift.yml         weekly: live docs vs snapshot, opens an issue
```

### The registry, the snapshot, and the two-part gate

Every engine parameter lives in `registry.ts` exactly once: name, type,
required or optional, description, and how it maps into the request. Tool
input schemas (zod) are derived from it.

The registry-to-schema comparison alone would be an internal consistency
test: both ends are files written in this repo, and they can agree with
each other while both disagree with SearchApi. If SearchApi renames a
parameter or adds a required one, that test passes and the server is
wrong. So the gate is deliberately two-part:

- **Gate 1, internal consistency** (`schema-drift.test.ts`): a registry
  param missing from a tool schema, or a schema param the registry does
  not know, fails CI. This catches local mistakes.
- **Gate 2, upstream drift** (`snapshot-drift.test.ts` plus the weekly
  scheduled job): `docs/engines/*.md` are dated snapshots of SearchApi's
  documented parameters, checked into the repo. The test fails CI when
  the registry and the snapshot disagree. A scheduled workflow re-fetches
  the live engine documentation weekly, diffs it against the snapshot,
  and opens an issue when they differ, so upstream changes arrive as a
  reviewable diff instead of a silently wrong server.

Gate 2 is the one that makes the headline claim true: the tool schemas
track SearchApi's parameter surface, not just themselves.

### Untrusted input boundary

Search results are attacker-controllable text. A page that ranks for a
query can carry "ignore previous instructions" and the trimmer would
otherwise deliver it faithfully into the agent's context. This server
treats SERP content as untrusted data on the same principle as the
pii-veil, query-warden and sql-steward work: input from outside gets a
boundary, not a shrug.

- Every tool result is wrapped in `envelope.ts` before it leaves: an
  explicit data envelope that marks the content as untrusted third-party
  data, not as instructions to the model.
- SERP text never reaches the model except as quoted data inside that
  envelope. Tool descriptions in each schema restate this, so the model
  is told at call time that result content is data to reason about, not
  directions to follow.
- The boundary is unit-tested: `envelope.test.ts` fails if any result
  path can emit content outside the envelope, including in `full` mode.
- The README documents the boundary, because a competing server that
  pipes ranked text straight into context is one prompt-injection demo
  away from embarrassing its users.

### Fail loudly

Missing `SEARCHAPI_API_KEY` is a startup error with a clear message, not a
runtime surprise. 4xx from SearchApi surfaces the upstream error body
rather than being swallowed. Retries apply only to 429 and 5xx, with
exponential backoff capped by a configurable ceiling, and honour
`Retry-After` when present. No silent fallbacks, no default-engine
shrugging.

### Quota surfacing

SearchApi bills per search, and an agent in a loop is a runaway bill.
When a response carries remaining-quota information (header or response
field, wherever SearchApi documents it for the plan tier), the client
passes it through as structured metadata on the tool result instead of
dropping it. When a response carries nothing, nothing is invented: the
metadata is simply absent. No local accounting, no estimates, no
caching of a number that could be stale. The value of this is that a
host or agent loop can watch the number it is given and stop, and it is
the kind of detail a search vendor notices in a client.

### Context-window economy

Agents pay per token, so trimming is a first-class feature, not an
afterthought:

- `full`: the raw organic results array, as-is, still inside the envelope.
- `compact`: position, title, url, snippet, date. Drops knowledge-graph
  bulk, ads, and related-searches.
- `answers_only`: answer box plus top 3 organic titles and urls.

Default is `compact`. The trimming layer is pure and unit-tested against
recorded response shapes.

### Secrets

The key is read from `SEARCHAPI_API_KEY` only. `.env.example` ships with a
placeholder, `.env` is gitignored, and no test ever needs a real key: the
HTTP client is mocked at its boundary so the whole suite runs offline.

### License

MIT, added in the scaffold phase, not deferred to publish. It blocks npm
publish and the registry submission, and there is no reason to sit on it.

## Phases

1. **Scaffold**: pnpm, TypeScript strict, vitest, eslint, CI on node 20
   and 22, MIT LICENSE. Client and types compile, env validation in place.
2. **google_search**: registry entry, dated docs snapshot, derived
   schema, tool, envelope, trimming modes, quota passthrough, mocked
   client tests, and the weekly docs-drift workflow. This is the vertical
   slice, and it includes gate 1 and gate 2 for this engine plus the
   scheduled job that keeps the snapshot honest.
3. **More engines**: google_news and google_shopping first, then
   google_maps, google_jobs, google_scholar, youtube search. Each is one
   registry entry, one snapshot, one thin tool file. The order is agent
   demand: news and shopping are what agent users ask for and what
   SearchApi itself leads with, maps and jobs are narrower verticals.
4. **Publish prep**: npm metadata under `mcp-searchapi`, `npx` smoke test
   in CI, README with a 30-second setup, `server.json` per the registry
   quickstart, then submission to registry.modelcontextprotocol.io.

## Non-goals (for now)

- No response caching layer: clients and hosts already do this better.
- No account management: key-only, no OAuth dance.
- No streaming: SearchApi responses are single-shot JSON.
- No local quota accounting: pass through what the response says, never
  estimate.
- No official status: this stays an unofficial client until SearchApi
  says otherwise.
- No modelcontextprotocol/servers PR: that repo is closed to new server
  implementations; the registry is the route.

## Verification gates before anything ships

- `pnpm lint`, `pnpm typecheck`, `pnpm test` all green in CI.
- Gate 1 proves registry and schemas cannot diverge locally.
- Gate 2 proves the registry tracks the dated docs snapshot, and the
  weekly job keeps that snapshot honest.
- The envelope test proves SERP text can only leave as quoted data.
- A recorded fixture pins the response shape the trimmer depends on.
- The quota passthrough is tested with both a quota-bearing and a
  quota-less fixture, and invents nothing in the latter.
- `server.json` validates against the registry's schema before phase 4
  closes.
