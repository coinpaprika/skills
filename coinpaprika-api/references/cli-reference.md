# CoinPaprika CLI Reference

Crypto market data from the terminal. 8,000+ coins, real-time prices, OHLCV, exchanges. Free tier included.

- GitHub: https://github.com/coinpaprika/coinpaprika-cli
- Install: `curl -sSL https://raw.githubusercontent.com/coinpaprika/coinpaprika-cli/main/install.sh | sh`
- Quick start: `coinpaprika-cli onboard`
- Free vs paid: `coinpaprika-cli plans`

**For agent use:** Always append `--output json --raw` for machine-readable JSON output.

---

## API key setup

Free tier works without a key (20,000 calls/month, 2,000 assets). For paid tiers:

1. **Config file:** `coinpaprika-cli onboard` — saves to `~/.coinpaprika/config.json`
2. **Environment variable:** `export COINPAPRIKA_API_KEY=your-key`
3. **CLI flag:** `coinpaprika-cli --api-key your-key ticker btc-bitcoin`

Priority: CLI flag > env var > config file.

---

## All commands

| Command | Description | Example |
|---------|-------------|---------|
| `global` | Market overview (cap, volume, BTC dominance) | `coinpaprika-cli global` |
| `coins` | List all coins | `coinpaprika-cli coins --limit 10` |
| `coin` | Coin details | `coinpaprika-cli coin btc-bitcoin` |
| `coin-events` | Coin events | `coinpaprika-cli coin-events btc-bitcoin` |
| `coin-exchanges` | Exchanges listing a coin | `coinpaprika-cli coin-exchanges btc-bitcoin` |
| `coin-markets` | Trading pairs for a coin | `coinpaprika-cli coin-markets btc-bitcoin` |
| `tickers` | All tickers (real-time prices) | `coinpaprika-cli tickers --limit 20` |
| `ticker` | Single coin ticker | `coinpaprika-cli ticker btc-bitcoin` |
| `ticker-history` | Historical tickers [Starter+] | `coinpaprika-cli ticker-history btc-bitcoin --start 2024-01-01` |
| `ohlcv` | Historical OHLCV [Starter+] | `coinpaprika-cli ohlcv btc-bitcoin --start 2024-01-01` |
| `ohlcv-latest` | Last full day OHLCV | `coinpaprika-cli ohlcv-latest btc-bitcoin` |
| `ohlcv-today` | Today's OHLCV (partial) | `coinpaprika-cli ohlcv-today btc-bitcoin` |
| `exchanges` | List exchanges | `coinpaprika-cli exchanges --limit 10` |
| `exchange` | Exchange details | `coinpaprika-cli exchange binance` |
| `exchange-markets` | Markets on an exchange | `coinpaprika-cli exchange-markets binance` |
| `tags` | List tags/categories | `coinpaprika-cli tags` |
| `tag` | Tag details | `coinpaprika-cli tag defi` |
| `person` | Person details | `coinpaprika-cli person vitalik-buterin` |
| `search` | Search coins, exchanges, people, tags | `coinpaprika-cli search bitcoin` |
| `convert` | Currency conversion | `coinpaprika-cli convert btc-bitcoin usd-us-dollars` |
| `platforms` | List contract platforms | `coinpaprika-cli platforms` |
| `contracts` | List contracts on a platform | `coinpaprika-cli contracts eth-ethereum` |
| `contract-ticker` | Ticker by contract address | `coinpaprika-cli contract-ticker eth-ethereum 0xdac...` |
| `contract-history` | Contract history [Starter+] | `coinpaprika-cli contract-history eth-ethereum 0xdac... --start 2024-01-01` |
| `key-info` | API key info [Paid] | `coinpaprika-cli key-info` |
| `mappings` | ID mappings [Business+] | `coinpaprika-cli mappings` |
| `changelog` | Coin ID changelog [Starter+] | `coinpaprika-cli changelog` |
| `config show` | Show config | `coinpaprika-cli config show` |
| `config set-key` | Set API key | `coinpaprika-cli config set-key <KEY>` |
| `config reset` | Delete config | `coinpaprika-cli config reset` |
| `plans` | Free tier details and paid overview | `coinpaprika-cli plans` |
| `status` | API health check | `coinpaprika-cli status` |
| `attribution` | Attribution snippets | `coinpaprika-cli attribution` |
| `shell` | Interactive REPL | `coinpaprika-cli shell` |
| `onboard` | Setup wizard | `coinpaprika-cli onboard` |

---

## Global flags

| Flag | Description |
|------|-------------|
| `-o, --output json` | Output as JSON (default: table) |
| `--raw` | Raw JSON without _meta wrapper (for scripts/piping) |
| `--api-key <KEY>` | API key (overrides env var and config) |
| `-h, --help` | Command help |
| `-V, --version` | Print version |

---

## Coin ID format

CoinPaprika coin IDs follow the pattern `{symbol}-{name}` (lowercase, hyphens):

`btc-bitcoin`, `eth-ethereum`, `usdt-tether`, `sol-solana`, `ada-cardano`, `doge-dogecoin`

Use `coinpaprika-cli search {query}` to find the correct ID for any coin.
