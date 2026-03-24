# CoinPaprika & DexPaprika Skills for AI Agents

Ready-to-use skills that give AI agents instant access to crypto market data. Install a skill and your agent immediately knows how to query prices, pools, tokens, exchanges, and more.

## Available skills

| Skill | What it does | Data source |
|-------|-------------|-------------|
| **[coinpaprika-api](./coinpaprika-api/)** | CEX market data: 12,000+ coins, 350+ exchanges, tickers, OHLCV, historical prices | [api.coinpaprika.com](https://api.coinpaprika.com) |
| **[dexpaprika-api](./dexpaprika-api/)** | DEX data: 34 chains, 30M+ pools, 27M+ tokens, real-time streaming | [api.dexpaprika.com](https://api.dexpaprika.com) |
| **[add-yaml-exchange](./.claude/skills/add-yaml-exchange/)** | Internal: Add new cryptocurrency exchanges via YAML configuration | CoinPaprika infrastructure |

## Install (Claude Code)

Clone this repo into your project's `.claude/skills/` directory:

```bash
# Clone into your project
cd your-project/
git clone https://github.com/coinpaprika/skills.git .claude/skills/coinpaprika

# Or clone just one skill using sparse checkout
git clone --filter=blob:none --sparse https://github.com/coinpaprika/skills.git .claude/skills/coinpaprika
cd .claude/skills/coinpaprika
git sparse-checkout set dexpaprika-api   # or coinpaprika-api, or both
```

Or simply copy the skill folder you need into `.claude/skills/` in your project. Then restart Claude Code — it picks up new skills automatically.

After installing, just ask your agent to query crypto data. It knows the APIs, CLI tools, endpoints, and response formats automatically.

## What's inside each skill

```
{skill-name}/
  SKILL.md              # Main instructions (loaded when skill triggers)
  references/
    openapi.yml         # Full API spec (loaded on demand)
    cli-reference.md    # CLI commands (loaded on demand)
    streaming-api.md    # SSE streaming docs (DexPaprika only)
```

## What agents can do with these skills

**CoinPaprika:**
- Get real-time prices for any coin (`GET /tickers/btc-bitcoin`)
- Historical OHLCV candlestick data
- Exchange and market data
- Search across 12,000+ coins
- Convert between currencies
- Look up tokens by contract address
- CLI: `coinpaprika-cli ticker btc-bitcoin`

**DexPaprika:**
- Get on-chain token prices across 34 blockchains
- Query liquidity pools, DEXes, and trading activity
- Historical OHLCV for any pool
- Batch price queries (up to 10 tokens)
- Real-time SSE price streaming (~1s updates)
- Search tokens, pools, and DEXes across all chains
- CLI: `dexpaprika-cli token ethereum 0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2`

## No API key needed

Both APIs have free tiers that work without authentication:
- **DexPaprika:** 10,000 requests/day, no key required
- **CoinPaprika:** 20,000 requests/month, no key required

## Other integration options

These skills are for Claude Code agents. For other AI tools:

| Integration | DexPaprika | CoinPaprika |
|-------------|-----------|-------------|
| MCP Server | [mcp.dexpaprika.com](https://mcp.dexpaprika.com/sse) | [mcp.coinpaprika.com](https://mcp.coinpaprika.com/sse) |
| CLI | [dexpaprika-cli](https://github.com/coinpaprika/dexpaprika-cli) | [coinpaprika-cli](https://github.com/coinpaprika/coinpaprika-cli) |
| Agent skills | [agents.dexpaprika.com](https://agents.dexpaprika.com) | - |
| LLM docs | [docs.dexpaprika.com](https://docs.dexpaprika.com) | [llms-full.txt](https://docs.coinpaprika.com/llms-full.txt) |

## SDKs

**DexPaprika:** [Go](https://github.com/coinpaprika/dexpaprika-sdk-go) | [Python](https://github.com/coinpaprika/dexpaprika-sdk-python) | [TypeScript](https://github.com/coinpaprika/dexpaprika-sdk-ts) | [PHP](https://github.com/coinpaprika/dexpaprika-sdk-php) | [Rust](https://github.com/coinpaprika/dexpaprika-sdk-rust)

**CoinPaprika:** [Go](https://github.com/coinpaprika/coinpaprika-api-go-client) | [Python](https://github.com/coinpaprika/coinpaprika-api-python-client) | [Node.js](https://github.com/coinpaprika/coinpaprika-api-nodejs-client) | [PHP](https://github.com/coinpaprika/coinpaprika-api-php-client) | [Swift](https://github.com/coinpaprika/coinpaprika-api-swift-client) | [Kotlin](https://github.com/coinpaprika/coinpaprika-api-kotlin-client)

## Links

- [DexPaprika Docs](https://docs.dexpaprika.com)
- [CoinPaprika Docs](https://docs.coinpaprika.com)
- [AI Agents Showcase](https://agents.dexpaprika.com)
- [CoinPaprika API Pricing](https://coinpaprika.com/api/pricing)

## License

MIT
