# @pipeworx/noaa-ncei

The US climate archive of record — GHCN-Daily station observations back to the
1800s, global summaries, and the NCEI dataset catalogue — through NOAA NCEI's
keyless Access services.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1683+ live data sources.

## Tools

- `ncei_daily_summaries(stations, startDate, endDate, dataTypes?, units?, dataset?, limit?)` — daily max/min temperature, precipitation, snow, wind for named stations.
- `ncei_station_search(bbox, dataset?, startDate?, endDate?, dataTypes?, limit?)` — stations in an area with their period of record and elements.
- `ncei_search_datasets(text?, limit?)` — the NCEI dataset catalogue.

## Auth

Keyless. The older CDO v2 API (`/cdo-web/api/v2`) needs a token and is capped at
5 req/s and 10,000/day; these Access services answer the same questions with no
credential, so CDO is deliberately not used here.

## Data sources

- `https://www.ncei.noaa.gov/access/services/data/v1` — observation values.
- `https://www.ncei.noaa.gov/access/services/search/v1/datasets` — dataset catalogue.
- `https://www.ncei.noaa.gov/access/services/search/v1/data` — station/file discovery.

Things the next person would otherwise rediscover:

- **The search service's bounding box is `bbox` in NORTH,WEST,SOUTH,EAST order**
  — not the `minLon,minLat,maxLon,maxLat` used everywhere else in this catalog,
  and not the parameter name `boundingBox`, which is rejected with a bare
  `Invalid search options` naming no field. The pack takes the usual order and
  reorders it.
- **`/search/v1/collections` does not exist** (404). The catalogue is `.../datasets`.
- **Values come back as strings** (`"TMAX":"64"`) and their unit depends on
  `units=standard|metric`. Both are surfaced.
- **A window outside a station's period of record returns an EMPTY BODY with
  HTTP 200** — not an error, not `[]`. `ncei_daily_summaries` reports that as an
  explicit `no_data` with the station and window named, never as a bare success.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "noaa-ncei": {
      "url": "https://gateway.pipeworx.io/noaa-ncei/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/noaa-ncei/mcp` returns the tools in the table
above **plus the shared Pipeworx meta-tools** — `ask_pipeworx`,
`discover_tools`, `search_within`, `remember`/`recall` and the rest of the
gateway-wide set. So the tool count you see is larger than this table: a
single-pack endpoint currently lists roughly 30 shared tools alongside the
pack's own. The connection's `initialize` response states its exact scope, and
is the authoritative answer for a given day.

This is deliberate, not multiplexing by accident. The meta-tools are what let a
scoped connection answer a question this pack does not cover — via
`ask_pipeworx`, which routes across the whole catalog — without you adding a
second MCP server. There is currently no way to mount a pack endpoint without
them; if the extra schemas cost you more context than the routing is worth,
connect to the full gateway once rather than to several pack endpoints.

Or connect to the full Pipeworx gateway to get every pack's tools listed
directly, instead of just this one's:

```json
{
  "mcpServers": {
    "pipeworx": {
      "url": "https://gateway.pipeworx.io/mcp"
    }
  }
}
```

Both URLs reach the same gateway and the same 1683+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/ncei_daily_summaries \
  -H 'Content-Type: application/json' \
  -d '{"stations":"USW00023174","startDate":"2026-01-01","endDate":"2026-01-05","dataTypes":"TMAX,TMIN,PRCP"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/ncei_daily_summaries`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "noaa-ncei": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-noaa-ncei"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-noaa-ncei
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Noaa Ncei data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
