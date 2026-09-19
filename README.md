# @pipeworx/dubai-land

Dubai real-estate data from the Dubai Land Department — registered sale,
mortgage and gift transactions, Ejari rental contracts, official valuations,
development projects, land parcels, buildings, and the licensed broker and
developer registers.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1576+ live data sources.

## Tools

- `dld_search_transactions(from_date, to_date, ...)` — property transactions
  with transaction value, procedure, district, project, property type, room
  count and nearest metro/mall/landmark. Answers "what did this kind of
  property sell for in Dubai this month".
- `dld_search_rents(from_date, to_date, ...)` — Ejari-registered rental
  contracts with contract and annualised amounts, start/end dates, district and
  property size. Answers "what is the going rent".
- `dld_search_valuations(from_date, to_date, ...)` — official DLD valuations
  with appraised total value, actual area and property sub-type.
- `dld_search_projects(from_date, to_date, date_type, ...)` — registered
  development projects with developer, status, percent complete and dates.
- `dld_search_lands(...)` — registered land parcels with parcel/land number,
  district, zone, land type, area and project.
- `dld_search_buildings(...)` — registered buildings with building number,
  floors, rooms, unit counts, area, district and project.
- `dld_list_brokers(...)` — the licensed broker register: broker number and
  name, licence dates, brokerage office and the office phone the register
  publishes.
- `dld_list_developers(from_date, to_date, ...)` — the registered developer
  list with licence source and dates.
- `dld_list_areas()` — the 437 Dubai area (district) names in English and
  Arabic with their `AREA_ID` codes.

Every search tool takes `limit` (1-200, default 25), `offset`, and a `sort` of
the form `{COLUMN}_ASC` / `{COLUMN}_DESC` using any column name from the rows
(e.g. `TRANS_VALUE_DESC`). Each result carries `total`, the size of the whole
matching set, so `offset` can page through it.

## Auth

Keyless. No account, no registration, no quota — this is the same backend the
department's own public "Real Estate Data" search pages call.

## Data sources

- https://gateway.dubailand.gov.ae/open-data/transactions — sale, mortgage
  and gift transactions.
- https://gateway.dubailand.gov.ae/open-data/rents — Ejari rental contracts.
- https://gateway.dubailand.gov.ae/open-data/valuations — official valuations.
- https://gateway.dubailand.gov.ae/open-data/projects — development projects.
- https://gateway.dubailand.gov.ae/open-data/lands — land parcels.
- https://gateway.dubailand.gov.ae/open-data/buildings — buildings.
- https://gateway.dubailand.gov.ae/open-data/brokers — licensed brokers.
- https://gateway.dubailand.gov.ae/open-data/developers — registered developers.
- https://gateway.dubailand.gov.ae/open-data/carea-lookup — area/district codes.

### What the next person would otherwise rediscover

**Every declared parameter must be present in the body, even when unset.** An
omitted key is not a default — the gateway answers a 500 HTML error page. Each
tool here therefore sends its command's full `P_*` parameter set with `""` for
anything the caller left out. A body of `{}` returns
`responseCode 420 INVALID_REQUEST`, which is what makes the endpoint look
key-gated when it is not.

**Dates go out as `MM/DD/YYYY`.** Tools accept ISO `YYYY-MM-DD` and convert.

**Transactions, rents and valuations only cover the current calendar year.**
Measured 2026-09-07: the whole of 2026 returned 152,531 transactions, while
2025 and 2024 returned an empty array with `responseCode 200` — no error, no
warning. Do not read an empty result for an older range as an outage.

**`projects` returns nothing without `date_type`.** Dates alone give an empty
200; `date_type` plus dates gives rows. `buildings` and `lands` are the
opposite — they want no date range at all.

**The upstream `P_AREA_ID` filter does not work and is not exposed.** The
`AREA_ID` codes from `carea-lookup` (e.g. `A-292` for Al Barsha) return zero
rows on `transactions` and a 500 on `lands`, so the tools always send `""` for
it. Rows still carry `AREA_EN` / `AREA_AR`, so filter by district on the way
out. `zone_id` (`1` Deira, `2` Dubai) does work where the command declares it.

**Row noise.** Each row repeats a `TOTAL` column holding the size of the whole
matching set, plus an `RN` sequence number and a `DEFAULT_SORT` marker. `TOTAL`
is hoisted to the response's `total` and all three are stripped from the rows.
Bilingual columns come in `_EN` / `_AR` pairs; the Arabic twin is dropped unless
`include_arabic: true` (always kept for `dld_list_areas`, where it is the
point).

**`GENDER_EN` on broker rows is returned in Arabic** by the upstream
(`أنثى` / `ذكر`) despite the `_EN` suffix. The `gender` filter itself works:
`0` male, `1` female.

**No spoofed headers needed.** The gateway answers a plain request carrying our
own `User-Agent`; it does not check `Origin` or `Referer`.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "dubai-land": {
      "url": "https://gateway.pipeworx.io/dubai-land/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/dubai-land/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1576+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/dld_search_transactions \
  -H 'Content-Type: application/json' \
  -d '{"from_date":"2026-09-01","to_date":"2026-09-06","transaction_type":"1","limit":3,"sort":"TRANS_VALUE_DESC"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/dld_search_transactions`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "dubai-land": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-dubai-land"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-dubai-land
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Dubai Land data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
