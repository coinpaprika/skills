# DexPaprika CLI Reference

DEX data from the terminal. 36 chains, 230+ DEXes, 33M+ tokens, 36M+ pools. Free tier, no API key needed to start.

- GitHub: https://github.com/coinpaprika/dexpaprika-cli
- Install: `curl -sSL https://raw.githubusercontent.com/coinpaprika/dexpaprika-cli/main/install.sh | sh`
- Quick start: `dexpaprika-cli onboard`

**For agent use:** Always append `--output json --raw` for machine-readable JSON output.

---

## All commands

| Command | Description | Example |
|---------|-------------|---------|
| `stats` | Global stats (networks, DEXes, pools, tokens) | `dexpaprika-cli stats` |
| `networks` | List all supported networks/chains | `dexpaprika-cli networks` |
| `dexes` | List DEXes on a network | `dexpaprika-cli dexes ethereum` |
| `pools` | List top pools on a network | `dexpaprika-cli pools ethereum --limit 10` |
| `pool-filter` | Filter pools by volume, liquidity, txns, price change (0.4.4+), creation date | `dexpaprika-cli pool-filter ethereum --volume-24h-min 100000` |
| `pool` | Get detailed info about a specific pool | `dexpaprika-cli pool ethereum 0x88e6...` |
| `dex-pools` | List pools on a specific DEX | `dexpaprika-cli dex-pools ethereum uniswap_v3` |
| `transactions` | Get recent transactions for a pool | `dexpaprika-cli transactions ethereum 0x88e6...` |
| `pool-ohlcv` | Get OHLCV data for a pool | `dexpaprika-cli pool-ohlcv ethereum 0x88e6... --start 2025-01-27` |
| `token` | Get detailed info about a token | `dexpaprika-cli token ethereum 0xc02a...` |
| `token-pools` | Get pools containing a token | `dexpaprika-cli token-pools ethereum 0xc02a...` |
| `filter-tokens` | Filter tokens by volume, liquidity, FDV, txns, 24h price change (0.4.4+), creation date | `dexpaprika-cli filter-tokens ethereum --volume-24h-min 100000` |
| `top-tokens` | Discover top tokens by volume (derived from pools) | `dexpaprika-cli top-tokens ethereum` |
| `prices` | Get batch prices for multiple tokens | `dexpaprika-cli prices ethereum --tokens 0xc02a...,0xa0b8...` |
| `search` | Search tokens, pools, DEXes across all networks | `dexpaprika-cli search USDC` |
| `stream` | Stream real-time token prices via SSE | `dexpaprika-cli stream ethereum 0xc02a...` |
| `stream-reserves` | Stream real-time pool/token reserves via SSE | `dexpaprika-cli stream-reserves ethereum 0x88e6... --method pool_reserves` |
| `status` | Check DexPaprika API health status | `dexpaprika-cli status` |
| `check-update` | Check for CLI updates | `dexpaprika-cli check-update` |
| `attribution` | Get attribution snippets for DexPaprika | `dexpaprika-cli attribution` |
| `shell` | Interactive shell mode (REPL) | `dexpaprika-cli shell` |
| `onboard` | Welcome message and quick start guide | `dexpaprika-cli onboard` |

---

## Search-backed commands and sort fields

`pools`, `pool-filter`, `token-pools`, `top-tokens`, and `filter-tokens` call the API's `/search` endpoints (`/networks/{network}/pools/search`, `/networks/{network}/tokens/search`) under the hood. `token-pools` passes its token argument as the `token_address` query parameter on the network-scoped pool search; the filter only exists there, so the command always needs a network argument. The command names did not change, and the CLI maps legacy sort-field values to the canonical names the search endpoints require:

| Legacy value | Canonical value sent to the API |
|--------------|--------------------------------|
| `volume_usd`, `volume_24h` | `volume_usd_24h` |
| `volume_7d` | `volume_usd_7d` |
| `volume_30d` | `volume_usd_30d` |
| `transactions`, `txns` | `txns_24h` |
| `last_price_change_usd_24h`, `price_change` | `price_change_percentage_24h` |
| `liquidity` | `liquidity_usd` |
| `fdv` | `fdv_usd` |

Canonical values pass through unchanged. Unknown values fall back to `volume_usd_24h`. Note: token search rejects `price_usd` ordering, so `top-tokens --order-by price_usd` falls back to volume. The search endpoints are cursor-paginated, so `--page` has no effect on these commands; use `--limit` and the CLI's next-cursor hint.

```bash
# Sort flags: pools/top-tokens use --order-by/--sort, pool-filter/filter-tokens use --sort-by/--sort-dir
dexpaprika-cli pools ethereum --order-by volume_usd_24h --sort desc
dexpaprika-cli top-tokens solana --order-by price_change_percentage_24h --sort asc
dexpaprika-cli pool-filter ethereum --volume-24h-min 500000 --sort-by liquidity_usd --sort-dir desc
dexpaprika-cli filter-tokens solana --fdv-min 1000000 --sort-by liquidity_usd
```

## Price-change windows

`pools` and `pool-filter` sort and filter on four price-change windows: 24h, 6h, 1h and 5m. The three short windows and all eight filter flags land in the CLI release after 0.4.4, so check your version before you script against them. In that release the sort values are the canonical API names, and they pass through the mapping table above unchanged. On 0.4.4 and earlier the three short ones are unknown to the CLI, so they hit the fallback and you get a volume-sorted page at exit 0:

- `price_change_percentage_24h`
- `price_change_percentage_6h`
- `price_change_percentage_1h`
- `price_change_percentage_5m`

The filter flags drop the `percentage` noise word, the way `--volume-24h-min` drops the `usd` from `volume_usd_24h_min`:

| Flag | API parameter |
|------|---------------|
| `--price-change-24h-min`, `--price-change-24h-max` | `price_change_percentage_24h_min`, `_max` |
| `--price-change-6h-min`, `--price-change-6h-max` | `price_change_percentage_6h_min`, `_max` |
| `--price-change-1h-min`, `--price-change-1h-max` | `price_change_percentage_1h_min`, `_max` |
| `--price-change-5m-min`, `--price-change-5m-max` | `price_change_percentage_5m_min`, `_max` |

There is no `--price-change-percentage-1h-min` spelling. The long form exits with `error: unexpected argument '--price-change-percentage-1h-min' found` and a tip pointing at the short one.

Values are signed percentages, and downside screens are the common case. The eight flags are declared with `allow_negative_numbers`, so `--price-change-24h-max -20` parses as written. The `=` form works too if you prefer to be explicit.

```bash
# Biggest 1h movers on ethereum
dexpaprika-cli pools ethereum --order-by price_change_percentage_1h --sort desc --limit 10

# Pools down 20 percent or more over the last 24h
dexpaprika-cli pool-filter ethereum --price-change-24h-max -20 --limit 10

# Pools up 50 percent or more in the last hour, deepest liquidity first
dexpaprika-cli pool-filter ethereum --price-change-1h-min 50 --sort-by liquidity_usd
```

Two things to know before you trust the output.

**Version.** On 0.4.4 and older, `pool-filter` has no price-change filter at all, not even 24h, and `--order-by price_change_percentage_1h` quietly falls back to `volume_usd_24h`. That returns a full result set ranked by volume rather than an error, which is the failure mode worth guarding against.

**Scope.** Only the three short windows are pool-only. `top-tokens` and `filter-tokens` hit `/networks/{network}/tokens/search`, which returns HTTP 400 for `price_change_percentage_{6h,1h,5m}` and silently ignores the matching `_min` and `_max` filters, so the CLI maps those three sort values back to `volume_usd_24h`. The 24h window behaves differently: it sorts and filters on the token side. `top-tokens --order-by price_change_percentage_24h` works, and `filter-tokens` takes `--price-change-24h-min` and `--price-change-24h-max` (0.4.4+), which map to `price_change_percentage_24h_min` and `_max`. Token rows carry `price_change_percentage_24h` and nothing equivalent for the short windows. `detailed=true` adds seven interval objects to each token row, `1m`, `5m`, `15m`, `30m`, `1h`, `6h` and `24h`, each holding `buys`, `sells`, `txns`, `volume_usd` and `last_price_usd_change`. None of them sort or filter, and `last_price_usd_change` came back null in all 20 ethereum rows I sampled on 2026-08-07, so do not build on it.

---

## Global flags

| Flag | Description |
|------|-------------|
| `-o, --output json` | Output as JSON (default: table) |
| `--raw` | Raw JSON without _meta wrapper (for scripts/piping) |
| `-h, --help` | Command help |
| `-V, --version` | Print version |

---

## Streaming commands

```bash
# Single token prices
dexpaprika-cli stream ethereum 0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2

# Multiple tokens: --tokens takes a PATH to a JSON file (max 25 entries),
# e.g. [{"chain": "ethereum", "address": "0xc02a..."}, {"chain": "solana", "address": "JUPy..."}]
dexpaprika-cli stream --tokens watchlist.json

# Limit number of events
dexpaprika-cli stream ethereum 0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2 --limit 10

# Reserves: one pool (fires pool_reserves events)
dexpaprika-cli stream-reserves ethereum 0x88e6a0c2ddd26feeb64f039a2c41296fcb3f5640 --method pool_reserves

# Reserves: one token across all its pools (fires token_reserves events)
dexpaprika-cli stream-reserves ethereum 0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48 --method token_reserves --limit 10

# Reserves: multiple subscriptions from a JSON file (max 25, methods can be mixed),
# e.g. [{"chain": "ethereum", "address": "0x88e6...", "method": "pool_reserves", "request_id": 1}]
dexpaprika-cli stream-reserves --subscriptions reserves.json
```

`stream-reserves` accepts `--request-id <0..4294967295>` on single-target streams; the server echoes it back on each data event. In the subscriptions file, `request_id` is per entry and defaults to the array index when omitted.

---

## Common token addresses

| Token | Chain | Address |
|-------|-------|---------|
| WETH | ethereum | `0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2` |
| USDC | ethereum | `0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48` |
| SOL | solana | `So11111111111111111111111111111111111111112` |
| USDC | solana | `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v` |
