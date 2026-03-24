# DexPaprika CLI Reference

Free DEX data from the terminal. 33+ chains, 213+ DEXes, 27M+ tokens, 30M+ pools. No API key needed.

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
| `pool-filter` | Filter pools by volume, txns, creation date | `dexpaprika-cli pool-filter ethereum --volume-24h-min 100000` |
| `pool` | Get detailed info about a specific pool | `dexpaprika-cli pool ethereum 0x88e6...` |
| `dex-pools` | List pools on a specific DEX | `dexpaprika-cli dex-pools ethereum uniswap_v3` |
| `transactions` | Get recent transactions for a pool | `dexpaprika-cli transactions ethereum 0x88e6...` |
| `pool-ohlcv` | Get OHLCV data for a pool | `dexpaprika-cli pool-ohlcv ethereum 0x88e6... --start 2025-01-27` |
| `token` | Get detailed info about a token | `dexpaprika-cli token ethereum 0xc02a...` |
| `token-pools` | Get pools containing a token | `dexpaprika-cli token-pools ethereum 0xc02a...` |
| `top-tokens` | Discover top tokens by volume (derived from pools) | `dexpaprika-cli top-tokens ethereum` |
| `prices` | Get batch prices for multiple tokens | `dexpaprika-cli prices ethereum --tokens 0xc02a...,0xa0b8...` |
| `search` | Search tokens, pools, DEXes across all networks | `dexpaprika-cli search USDC` |
| `stream` | Stream real-time token prices via SSE | `dexpaprika-cli stream ethereum 0xc02a...` |
| `status` | Check DexPaprika API health status | `dexpaprika-cli status` |
| `check-update` | Check for CLI updates | `dexpaprika-cli check-update` |
| `attribution` | Get attribution snippets for DexPaprika | `dexpaprika-cli attribution` |
| `shell` | Interactive shell mode (REPL) | `dexpaprika-cli shell` |
| `onboard` | Welcome message and quick start guide | `dexpaprika-cli onboard` |

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
# Single token
dexpaprika-cli stream ethereum 0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2

# Multiple tokens
dexpaprika-cli stream ethereum --tokens 0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2,0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48

# Limit number of events
dexpaprika-cli stream ethereum 0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2 --limit 10
```

---

## Common token addresses

| Token | Chain | Address |
|-------|-------|---------|
| WETH | ethereum | `0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2` |
| USDC | ethereum | `0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48` |
| SOL | solana | `So11111111111111111111111111111111111111112` |
| USDC | solana | `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v` |
