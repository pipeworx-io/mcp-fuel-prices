# fuel-prices

What petrol, diesel and LPG cost across Europe — 28 countries, weekly, back to
2005.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1856+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

| Tool | Answers |
|---|---|
| `fuel_price` | "What does petrol cost in Portugal?" — current price per litre for one country. |
| `compare_fuel_prices` | "Where is diesel cheapest in Europe?" — every country ranked, within one week. |
| `fuel_price_history` | "How much has petrol gone up in Germany this year?" — weekly series plus the change. |
| `fuel_price_coverage` | What countries, weeks and fuels are in scope, and how fresh the newest week is. |

## Auth

None. No key, no account.

## Data sources

The European Commission's **Weekly Oil Bulletin**:

- Landing page: <https://energy.ec.europa.eu/data-and-analysis/weekly-oil-bulletin_en>
- Series: `Weekly_Oil_Bulletin_Prices_History_maticni_4web.xlsx`

EU Member States report retail prices into it every week under Council Decision
1999/280/EC, and the Commission publishes the consolidated series. Coverage is
the 27 EU states plus the United Kingdom, plus an EU-wide aggregate under the
code `EU`.

## Reading the numbers

- **Prices are per 1000 litres in the source.** Petrol, diesel, heating oil and
  LPG are quoted that way; a raw `2143` is €2.143 per litre. Every response
  carries `price_per_litre_eur` already converted, and the raw figure alongside
  it with its `unit`.
- **The two heavy fuel oils are quoted per tonne**, so they have no per-litre
  price at all. `price_per_litre_eur` is `null` for those rather than a
  fabricated conversion.
- **It is a national average for a week, not a live station price.** The newest
  week is a few days behind and any individual filling station will differ.
  Every response states its `week_ending` so a figure cannot be quoted as
  "today" by accident.
- **`EU` is an aggregate, not a country.** It is available to ask for directly
  but is excluded from `compare_fuel_prices` rankings.
- **Comparisons are pinned to one week.** A country that reports late would
  otherwise win a "cheapest" ranking on an older, lower figure.
- **Not covered:** the United States, Switzerland, Norway, Turkey, and
  everywhere else outside the EU and the UK. Asking for one returns an explicit
  `country_not_covered` refusal naming the reason rather than an empty result.

## Refreshing

Weekly, when the Commission republishes the bulletin:

```bash
node scripts/ingest-eu-oil-bulletin.mjs              # full history
node scripts/ingest-eu-oil-bulletin.mjs --weeks 8    # just the recent tail
```

The script parses the workbook with no dependencies — xlsx is a zip of XML, and
node ships zlib — and upserts by `(country_code, week_ending, fuel)`, so a week
that stops being republished keeps its last known value.

It refuses to load rather than warn in two cases, because both would otherwise
produce a plausible series that is wrong:

- the newest week it parses is more than 45 days old (the source is
  newest-first, so a layout change silently yields January 2005);
- any record loses its unit.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "fuel-prices": {
      "url": "https://gateway.pipeworx.io/fuel-prices/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/fuel-prices/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1856+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/fuel_price \
  -H 'Content-Type: application/json' \
  -d '{"country":"Portugal"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/fuel_price`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-fuel-prices` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/fuel-prices/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "fuel-prices": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-fuel-prices"]
    }
  }
}
```

Without a key it calls anonymously, capped at 50 calls a day. A free account
raises that to 200 a day: get a key at https://pipeworx.io/signup and add
`"env": { "PIPEWORX_API_KEY": "<your key>" }` to the server entry.
The key is sent as a Bearer token, the same way a hosted connection sends it,
so calls are metered and billed exactly as they are there.

It lists **only** this pack's tools by default. Set `PIPEWORX_META_TOOLS=1` to
also list `ask_pipeworx` and the other gateway-wide tools.

Behind an HTTPS proxy: Node's built-in `fetch` does not read `HTTPS_PROXY` by
default. Newer Node releases honour it when `NODE_USE_ENV_PROXY=1` is set; check
your Node version's documentation.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Fuel Prices data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
