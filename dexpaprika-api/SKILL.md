---
name: dexpaprika-api
description: Access the DexPaprika API, CLI, and streaming service to query DEX data including networks, pools, tokens, and trading activity. Use this skill when making HTTP requests to api.dexpaprika.com or streaming.dexpaprika.com, or when using dexpaprika-cli for blockchain DEX information.
---

# DexPaprika API Skill

Free DEX data API covering 34 blockchains, 213 DEXes, 30M+ liquidity pools, and 27.7M+ tokens. Built by the CoinPaprika team (operating since 2018). No API key, no registration. Free public tier: 10,000 requests/day. Enterprise tier (api-pro.dexpaprika.com): unlimited requests with API key.

- Documentation: https://docs.dexpaprika.com
- AI Agents showcase: https://agents.dexpaprika.com
- Agent skill files: https://dexpaprika.com/agents/skill.md (REST) and https://dexpaprika.com/agents/streaming/skill.md (SSE)
- GitHub: https://github.com/coinpaprika
- Support: support@coinpaprika.com

> **Field naming note:** URL paths use `network` and `token_address`, but JSON responses return `chain` and `id` for the same values.

---

## Integration options

### Option 1: CLI (recommended for agents)

Install and query in seconds. Best for agents that can run shell commands.

```bash
curl -sSL https://raw.githubusercontent.com/coinpaprika/dexpaprika-cli/main/install.sh | sh
```

Always use `--output json --raw` for machine-readable output. Run `dexpaprika-cli onboard` for an interactive quick-start guide.

Common commands:

```bash
# Search for a token
dexpaprika-cli search USDC --output json --raw

# Get token price
dexpaprika-cli token ethereum 0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2 --output json --raw

# Top pools on a network
dexpaprika-cli pools ethereum --limit 10 --output json --raw

# Historical OHLCV for a pool
dexpaprika-cli pool-ohlcv ethereum 0x88e6a0c2ddd26feeb64f039a2c41296fcb3f5640 --start 2025-01-27 --output json --raw

# Top tokens on a network (ranked, with multi-timeframe metrics)
dexpaprika-cli top-tokens ethereum --limit 20 --output json --raw
dexpaprika-cli top-tokens solana --order-by price_change --sort asc --output json --raw

# Filter tokens by volume, FDV, liquidity, txns
dexpaprika-cli filter-tokens ethereum --volume-24h-min 100000 --output json --raw
dexpaprika-cli filter-tokens solana --fdv-min 1000000 --liquidity-usd-min 50000 --output json --raw

# Filter pools by volume, liquidity, txns, creation date
dexpaprika-cli pool-filter ethereum --volume-24h-min 500000 --liquidity-usd-min 50000 --output json --raw

# Batch token prices
dexpaprika-cli prices ethereum --tokens 0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2,0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48 --output json --raw

# Stream real-time prices (~1s updates)
dexpaprika-cli stream ethereum 0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2

# API health check
dexpaprika-cli status
```

For the full CLI command reference, read `references/cli-reference.md`.

### Option 2: REST API

Base URL: `https://api.dexpaprika.com`

No authentication required. All responses are JSON.

```bash
curl -s "https://api.dexpaprika.com/networks/ethereum/tokens/0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2" | jq
```

#### Endpoint table

| Need | Endpoint |
|------|----------|
| List all networks | `GET /networks` (returns volume_usd_24h, txns_24h, pools_count per network) |
| DEXes on a network | `GET /networks/{network}/dexes` (returns volume_usd_24h, txns_24h, pools_count per DEX) |
| Top pools on network | `GET /networks/{network}/pools` |
| Filter pools | `GET /networks/{network}/pools/filter` (volume, liquidity, txns, creation date filters) |
| Advanced pool search (all chains) | `GET /frontend/v1/pools` (sort, multi-filter, cursor pagination, detailed tokens) |
| Advanced pool search (one chain) | `GET /frontend/v1/networks/{network}/pools` (same params, scoped to one chain) |
| Pool details | `GET /networks/{network}/pools/{pool_address}` |
| Pool OHLCV (charts) | `GET /networks/{network}/pools/{pool_address}/ohlcv` |
| Pool transactions | `GET /networks/{network}/pools/{pool_address}/transactions` |
| Token price + data | `GET /networks/{network}/tokens/{token_address}` |
| Pools containing token | `GET /networks/{network}/tokens/{token_address}/pools` |
| Filter tokens | `GET /networks/{network}/tokens/filter` (volume, liquidity, FDV, txns, creation date filters) |
| Top tokens on network | `GET /networks/{network}/tokens/top` (ranked by volume, price, liquidity, txns, or price change) |
| Batch token prices | `GET /networks/{network}/multi/prices?tokens={addr1},{addr2}` |
| Pools for a DEX | `GET /networks/{network}/dexes/{dex}/pools` |
| Search tokens/pools/DEXes | `GET /search?query={term}` |
| Platform statistics | `GET /stats` |

For the full OpenAPI 3.1 specification with all schemas, parameters, and response types, read `references/openapi.yml`.

### Option 3: MCP Server (for AI IDEs)

Hosted MCP server for Claude Desktop, Cursor, Windsurf, and any MCP client.

Add to `claude_desktop_config.json` or equivalent:

```json
{
  "mcpServers": {
    "dexpaprika": {
      "url": "https://mcp.dexpaprika.com/sse"
    }
  }
}
```

No API key needed. Provides tools for querying networks, pools, tokens, OHLCV, transactions, and search.

Documentation: https://docs.dexpaprika.com/ai-integration/hosted-mcp-server

### Option 4: Streaming API (real-time prices + pool reserves)

Base URL: `https://streaming.dexpaprika.com`

Two SSE feeds share one transport:
- `/sse/prices`: live token price updates (~1 s cadence per asset).
- `/sse/reserves`: block-level pool reserve updates with USD-denominated deltas.

**Limits:** 25 subscriptions per POST connection. 10 concurrent SSE streams per IP. A `ping` event lands every 15 s.

Single token price (GET):
```bash
curl --http1.1 -N "https://streaming.dexpaprika.com/sse/prices?method=token_price&chain=ethereum&address=0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2"
```

Multiple token prices (POST, up to 25):
```bash
curl --http1.1 -N -X POST "https://streaming.dexpaprika.com/sse/prices" \
  -H "Accept: text/event-stream" -H "Content-Type: application/json" \
  -d '[{"chain":"ethereum","address":"0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2","method":"token_price"}]'
```

Pool reserves (GET):
```bash
curl --http1.1 -N "https://streaming.dexpaprika.com/sse/reserves?method=pool_reserves&chain=ethereum&address=0x88e6a0c2ddd26feeb64f039a2c41296fcb3f5640"
```

`token_price` event fields: `address`, `chain`, `price` (USD as string), `timestamp`, `timestamp_price` (both unix seconds). The legacy `t_p` method emits a compact `{a, c, p, t, t_p}` shape on the deprecated `/stream` path only and should not be used in new code.

The reserves feed now emits **method-named events**: the old single `reserve_update` event is gone. Match on the two event names instead:

- `pool_reserves` event (one pool, nested tokens): `chain`, `pool_id`, `block` (string), `tokens[]` (each `token_id`, `reserve`/`delta` as strings, `price_usd`/`reserve_usd`/`delta_usd` as numbers), `total_reserve_usd`, `total_delta_usd`, `timestamp`, `block_timestamp` (both unix seconds).
- `token_reserves` event (one token across all its pools, flat): `chain`, `token_id`, `reserve`/`delta` (strings), `block` (string), `price_usd`, `reserve_usd`, `delta_usd`, `updated_at`, `timestamp` (both unix seconds).

Raw integer fields (`reserve`, `delta`, `block`) exceed `Number.MAX_SAFE_INTEGER`, parse with `BigInt`. A consumer tailing reserves must stop matching `reserve_update` and handle these two event names plus their new timestamp fields.

`request_id` correlation (optional): pass `request_id` as a `uint32` (0..4294967295) on GET via the query string, or per-asset in the POST body (it defaults to the asset's array index). The server echoes it back as a `request_id:` SSE line on data events only. `ping`, `warning`, and `error` events carry no `request_id`.

```bash
# GET with request_id; the value comes back on each pool_reserves event
curl --http1.1 -N "https://streaming.dexpaprika.com/sse/reserves?method=pool_reserves&chain=ethereum&address=0x88e6a0c2ddd26feeb64f039a2c41296fcb3f5640&request_id=12345"
```

**Important:** Streaming requires HTTP/1.1. Add `--http1.1` with curl. One invalid asset cancels the entire stream with HTTP 400. SSE parsers must buffer one message between blank-line boundaries before dispatching: both `event:`/`data:` orderings are valid and the server uses either, and a `request_id:` line can appear alongside `event:`/`data:`.

For the full streaming reference (events, errors, parser patterns), read `references/streaming-api.md`.

### Option 5: SDKs

| Language | Repository |
|----------|------------|
| Go | https://github.com/coinpaprika/dexpaprika-sdk-go |
| Python | https://github.com/coinpaprika/dexpaprika-sdk-python |
| TypeScript | https://github.com/coinpaprika/dexpaprika-sdk-ts |
| PHP | https://github.com/coinpaprika/dexpaprika-sdk-php |
| Rust | https://github.com/coinpaprika/dexpaprika-sdk-rust |

---

## Common workflows

### Get a token price

CLI:
```bash
dexpaprika-cli token ethereum 0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2 --output json --raw
```

curl:
```bash
curl -s "https://api.dexpaprika.com/networks/ethereum/tokens/0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2" | jq '.summary.price_usd'
```

Python:
```python
import requests
r = requests.get("https://api.dexpaprika.com/networks/ethereum/tokens/0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2")
token = r.json()
print(f"{token['symbol']}: ${token['summary']['price_usd']}")
```

### Search for a token

```bash
curl -s "https://api.dexpaprika.com/search?query=PEPE" | jq '.tokens[:5]'
```

Note: Search uses fuzzy name+symbol matching. "UNI" returns "Uniswap", "United Stables", etc. Filter by exact `symbol` match client-side.

### Get historical OHLCV for a pool

```bash
curl -s "https://api.dexpaprika.com/networks/ethereum/pools/0x88e6a0c2ddd26feeb64f039a2c41296fcb3f5640/ohlcv?start=2025-01-01&interval=1h&limit=24" | jq
```

OHLCV params: `start` (required), `end`, `interval` (`1m`|`5m`|`10m`|`15m`|`30m`|`1h`|`6h`|`12h`|`24h`), `limit` (max 366), `inversed` (boolean, inverts price ratio for USD-denominated prices from stablecoin-paired pools).

### Batch prices for multiple tokens

```bash
curl -s "https://api.dexpaprika.com/networks/ethereum/multi/prices?tokens=0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2,0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48" | jq
```

Returns an **ARRAY** (not a keyed object). Max 10 tokens per request.

### Advanced pool search

For ranked, multi-filter pool discovery across one chain or all of them, use the `/frontend/v1/pools` endpoints. These are richer than `/networks/{network}/pools/filter`: sortable on 24h/7d/30d volume, cursor pagination instead of page numbers, and an optional `detailed` flag that inlines per-token FDV and multi-timeframe trade stats.

Global (every chain):
```bash
curl -s "https://api.dexpaprika.com/frontend/v1/pools?limit=3&order_by=volume_usd_24h&sort=desc&price_usd_min=0.5&dex_name=uniswap_v3&detailed=true" | jq '.results[0] | {id, dex_name, chain, price_usd, volume_usd_24h, transactions_24h, liquidity_usd}'
```

Per-network (one chain):
```bash
curl -s "https://api.dexpaprika.com/frontend/v1/networks/ethereum/pools?limit=10&order_by=volume_usd_24h&sort=desc&txns_24h_min=500" | jq '.results[] | {id, dex_name, volume_usd_24h, transactions_24h}'
```

**Sort parameters, canonical vs wire.** Our SDKs and the MCP server accept the canonical names `sort_by` and `sort_dir`. The raw HTTP endpoint does **not**: on the wire you must send `order_by` and `sort`, and anything sent as `sort_by`/`sort_dir` is silently ignored (the endpoint falls back to its default sort, confirmed via the `query` echo in the response). When calling the REST API directly, translate:

| Canonical (SDK/MCP) | Wire (raw HTTP) | Values |
|---|---|---|
| `sort_by` | `order_by` | `volume_usd_24h` (default), `volume_usd_7d`, `volume_usd_30d`, `liquidity_usd`, `txns_24h`, `price_usd`, `price_change_percentage_24h`, `created_at` |
| `sort_dir` | `sort` | `asc`, `desc` (default `desc`) |

**Filters** (all optional, same names on canonical and wire):

| Filter | Type | Notes |
|---|---|---|
| `volume_24h_min` / `volume_24h_max` | number | 24h USD volume bounds |
| `volume_7d_min` / `volume_7d_max` | number | 7d USD volume bounds |
| `liquidity_usd_min` / `liquidity_usd_max` | number | liquidity (USD) bounds |
| `txns_24h_min` | int | minimum 24h transaction count |
| `price_usd_min` / `price_usd_max` | number | pool price (USD) bounds |
| `price_change_percentage_24h_min` / `price_change_percentage_24h_max` | number | 24h % change bounds (accepts negatives) |
| `dex_name` | string | DEX slug, e.g. `uniswap_v3`, `pancakeswap_v3` |
| `created_after` / `created_before` | int | **Unix epoch seconds only.** Date strings (`2024-01-01`, RFC3339) break the response: the body comes back with a null `results` array. Pass `date -d 2024-01-01 +%s` style epochs. |

**Cursor pagination.** Unlike the page-numbered list endpoints, these use opaque cursors. Read `next_cursor` from the response and pass it back as `?cursor=...` to fetch the next page. Stop when `has_next_page` is `false`.

```bash
# page 1
RESP=$(curl -s "https://api.dexpaprika.com/frontend/v1/pools?limit=50&order_by=volume_usd_24h&sort=desc")
CURSOR=$(echo "$RESP" | jq -r '.next_cursor')
# page 2
curl -s "https://api.dexpaprika.com/frontend/v1/pools?limit=50&order_by=volume_usd_24h&sort=desc&cursor=$CURSOR" | jq '.has_next_page'
```

**Detailed tokens.** Add `detailed=true` to inline full token objects. Without it, each entry in `tokens[]` carries only `id`, `chain`, `has_image`. With it, tokens also carry `name`, `symbol`, `decimals`, `total_supply`, `added_at`, `status`, `fdv`, and per-timeframe trade blocks keyed `1m`, `5m`, `15m`, `30m`, `1h`, `6h`, `24h` (each `{volume_usd, buys, sells, txns, last_price_usd_change}`).

**Response shape:**
```json
{
  "results": [ /* PoolRow objects */ ],
  "has_next_page": true,
  "next_cursor": "eyJjaGFpbiI6...",
  "query": { "limit": 3, "order_by": "volume_usd_24h", "sort": "desc", ... }
}
```

Each `PoolRow`: `id`, `dex_id`, `dex_name`, `chain`, `fee`, `created_at`, `created_at_block_number`, `price_usd`, `transactions_24h`, `volume_usd_24h`, `volume_usd_7d`, `volume_usd_30d`, `liquidity_usd`, `price_change_percentage_5m`, `price_change_percentage_1h`, `price_change_percentage_24h`, `tokens[]`.

**Treat every field as optional/nullable.** Live responses routinely return `fee: null` and `liquidity_usd: 0` on real pools, and `fdv` is absent on tokens the indexer hasn't priced yet. Do not assume any field is present or non-zero. Guard before you read.

### Stream real-time prices (Python)

```python
import requests, json

assets = [
    {"chain": "ethereum", "address": "0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2", "method": "token_price"},
    {"chain": "solana",   "address": "So11111111111111111111111111111111111111112",  "method": "token_price"}
]

r = requests.post("https://streaming.dexpaprika.com/sse/prices",
    headers={"Accept": "text/event-stream", "Content-Type": "application/json"},
    json=assets, stream=True)

# Buffer one SSE message at a time, then dispatch. Both `event:`/`data:`
# line orderings are valid SSE and the server has emitted either, so a
# line-by-line parser that assumes one order will silently mis-dispatch.
msg_lines = []
for line in r.iter_lines(decode_unicode=True):
    if line:
        msg_lines.append(line); continue
    event_type, data_str = "message", None
    for ml in msg_lines:
        if ml.startswith("event:"):
            event_type = ml.split(":", 1)[1].strip()
        elif ml.startswith("data:"):
            data_str = ml[5:].lstrip()
    msg_lines = []
    if event_type == "token_price" and data_str:
        d = json.loads(data_str)
        print(f"{d['chain']} {d['address']}: ${d['price']}")
```

---

## Common token addresses

Do not guess addresses. Use `search` to find tokens, or use these known addresses:

| Token | Chain | Address |
|-------|-------|---------|
| WETH | ethereum | `0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2` |
| USDC | ethereum | `0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48` |
| USDC | polygon | `0x2791bca1f2de4661ed88a30c99a7a9449aa84174` |
| USDC | solana | `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v` |
| SOL | solana | `So11111111111111111111111111111111111111112` |

## Common network IDs

Always lowercase: `ethereum`, `solana`, `bsc`, `polygon`, `arbitrum`, `base`, `avalanche`, `optimism`, `sui`, `ton`, `tron`.

Full list: `GET /networks` or `dexpaprika-cli networks`.

## Pagination

Most list endpoints support: `?page=1&limit=10&order_by=volume_usd&sort=desc`

Pages are 1-indexed (first page is `page=1`). Max 1000 pages. Available `order_by` values: `volume_usd`, `liquidity_usd`, `price_usd`, `transactions`, `last_price_change_usd_24h`, `created_at`. The `/networks/{network}/{...}/filter` endpoints use the canonical `sort_by`/`sort_dir` names.

The `/frontend/v1/pools` and `/frontend/v1/networks/{network}/pools` endpoints are different: they use **cursor pagination** (`next_cursor` + `cursor`, not `page`), and on the wire they take `order_by`/`sort` (the canonical `sort_by`/`sort_dir` names are honored only by the SDKs and MCP, which translate them). See the "Advanced pool search" workflow above for the full param and field list.

## Timestamps

All timestamps support Unix, RFC3339, or `yyyy-mm-dd` format. OHLCV data limited to 366 data points per request.

## Rate limits and errors

- Free tier: 10,000 requests/day. Enterprise (api-pro.dexpaprika.com): unlimited with API key.
- HTTP errors: `200` OK | `400` bad params | `404` not found | `429` rate limited | `500` server error
- **On 429 rate limit:** Wait a few seconds/minutes, then retry. Blocks are temporary. If persistent, contact support@coinpaprika.com.
- Check API health: `dexpaprika-cli status` or `GET https://api.dexpaprika.com/stats`
- Full docs: https://docs.dexpaprika.com
