---
name: add-yaml-exchange
description: Add a new cryptocurrency exchange using a YAML configuration file. Use when asked to "add exchange", "add yaml exchange", "new exchange", "implement exchange", or "create exchange yaml". This skill handles the complete workflow of creating a YAML exchange definition, testing it, and deploying it.
---

# Add a New YAML Exchange

Add a new exchange `$ARGUMENTS` by creating a YAML configuration file.

**IMPORTANT RULES:**
- NEVER modify Go code. The YAML handler supports a fixed set of features. If an exchange cannot be implemented using the available YAML fields, tell the user clearly: "This exchange cannot be implemented via YAML configuration because [reason]." and stop.
- Do NOT read Go source code during implementation. Only read existing YAML files in `<PROJECT_ROOT>/config/prod/exchanges/` for reference patterns.
- Do NOT search for API documentation yourself. The user MUST provide either the API documentation text or a link to it. If the user has not provided documentation, ask them: "Please provide the exchange's API documentation or a link to it so I can inspect the endpoints."
- Keep all output simple and beginner-friendly. Do NOT show raw API responses, internal field mappings, or technical details of the YAML structure to the user. Just show the final result.

## Tool Permissions

Before starting, tell the user:

> To avoid approving every tool call manually, run this skill with `--allowedTools` or press **a** (allow all) when prompted. This skill uses Read, Write, Edit, Bash, WebFetch, and Glob — all safe, local operations.

## Step 0: Locate the coinpaprika-services Project

All file operations and tests run against the `github.com/coinpaprika/coinpaprika_services` repository. Locate it using **only non-Bash tools** to avoid permission prompts:

1. First, check if the project path is already available in the **additional working directories** provided in the environment. If you see a path containing `coinpaprika-services/config/prod/exchanges`, derive the project root from it (strip the suffix).
2. If not found there, use the **Glob** tool to search for the checker binary: `Glob(pattern="**/cmd_local/config_ticker_checker/main.go", path=os.path.expanduser("~"))` and derive the project root from the result.
3. Only if both above fail, ask the user: "Where is the `coinpaprika_services` repository cloned on your machine?"

**Do NOT use Bash** (`test -f`, `find`, `git rev-parse`) to locate the project — these require shell permission approval.

Store the result as `PROJECT_ROOT` and use it for all file paths below (e.g., `<PROJECT_ROOT>/config/prod/exchanges/`).

**Finding reference YAML files:** When you need to search existing YAML files for patterns (e.g., finding a Pattern 3 example), use the **Grep** tool instead of `bash grep`. Example: `Grep(pattern="@symbol", path="<PROJECT_ROOT>/config/prod/exchanges/")`.

## Step 1: Get API Documentation from the User

**Do NOT use `WebSearch` to find documentation.** The user must provide one of:
- A link to the API documentation (use `WebFetch` to read it)
- The API documentation text directly
- Specific endpoint URLs to inspect (use `WebFetch` to check the JSON responses)

If the user has not provided any of the above, ask them before proceeding.

Once you have the documentation or endpoint URLs, use `WebFetch` to inspect the actual JSON response structure of each endpoint. Understand:
- Where the data array/object is in the response (for `data_path`)
- What fields contain: symbol/pair name, base currency, quote currency, last price, volume
- Whether the ticker endpoint contains full market info or if a separate markets endpoint is needed
- Whether tickers are returned in a single endpoint or per-market (one endpoint per pair)

You need to identify endpoints for:
1. **Tickers/Market data** - an endpoint that returns price and volume data
2. **Markets/Trading pairs** (sometimes needed) - an endpoint that lists available trading pairs with base/quote currency info
3. **Order books** (optional) - an endpoint that returns order book data (asks/bids)

**IMPORTANT:** When the user provides a documentation link, always browse the surrounding API documentation (parent pages, sibling endpoints, navigation menus) to check if an **order book endpoint** exists. Use `WebFetch` on the documentation index/parent page to discover all available endpoints. Order books are valuable and should be configured whenever available.

## Step 2: Determine the YAML Pattern

Based on the API structure, determine which pattern to use. There are 3 patterns for tickers:

### Pattern 1: All-in-one (tickers contain everything)
Use when the ticker endpoint returns symbol, base, quote, price, and volume in one response.

```yaml
internal_id: exchange_name
tickers:
  url: https://api.example.com/tickers
  data_path: "@this"          # or "data", "result", etc.
  field_paths:
    symbol: "pair_name"        # field with market symbol like "BTC/USDT"
    base_currency_symbol: "base"
    quote_currency_symbol: "quote"
    last_price: "last"
    base_volume: "volume"
```

### Pattern 2: Separate markets + tickers endpoints
Use when the ticker endpoint lacks base/quote currency info and a separate markets endpoint provides it.

```yaml
internal_id: exchange_name
markets:
  url: https://api.example.com/markets
  data_path: "data"
  field_paths:
    id: "symbol"              # field to join markets with tickers
    symbol: "symbol"
    base_currency_symbol: "baseCoin"
    quote_currency_symbol: "quoteCoin"
tickers:
  url: https://api.example.com/tickers
  data_path: "data"
  field_paths:
    id: "symbol"              # must match markets.id
    last_price: "close"
    base_volume: "baseVol"
```

### Pattern 3: Per-market ticker endpoints
Use when each trading pair has its own ticker endpoint. Requires a markets endpoint. Use `%s` as placeholder for the market symbol.

```yaml
internal_id: exchange_name
markets:
  url: https://api.example.com/symbols
  data_path: "data"
  field_paths:
    id: "pair"
    symbol: "pair"
    base_currency_symbol: "base"
    quote_currency_symbol: "quote"
tickers:
  url: https://api.example.com/ticker?pair=%s
  data_path: "data"
  field_paths:
    id: "pair"                # or "@symbol" if response has no symbol field
    last_price: "last"
    base_volume: "volume"
```

## Step 3: Available YAML Fields Reference

This is the **complete** list of all supported YAML fields.

### Top-level
- `internal_id` (required): Unique identifier. Lowercase, no spaces (use `_`). Usually the exchange name.

### `markets` section (optional - only needed for patterns 2 and 3)
- `url` (required): Markets endpoint URL
- `data_path` (required): JSON path to data. Use `"@this"` if data is at root level.
- `field_paths` (required):
  - `id` (required): Field to join with tickers. Use `"@key"` if the key of the JSON object is the id.
  - `symbol` (required): Market symbol field. Use `"@key"` if the key of the JSON object is the symbol.
  - `symbol_separator` (optional): Character separating base/quote in the symbol (e.g., `"_"`, `"-"`, `"/"`). Use this when `symbol` contains both base and quote joined by a separator.
  - `symbol_inverted` (optional): `true` if the symbol is QUOTE_BASE instead of BASE_QUOTE. Only relevant when `symbol_separator` is set.
  - `base_currency_symbol` (optional): Field with base currency symbol. Not needed if `symbol_separator` is used.
  - `quote_currency_symbol` (optional): Field with quote currency symbol. Not needed if `symbol_separator` is used.

### `tickers` section (required)
- `url` (required): Tickers endpoint URL. Use `%s` as placeholder for per-market endpoints (pattern 3). The `%s` is replaced with `symbol` from the `markets` section.
- `data_path` (required): JSON path to data. Use `"@this"` if data is at root level.
- `field_paths` (required):
  - `id` (optional): Field to join with markets. Use `"@key"` for JSON object key. Use `"@symbol"` when the per-market endpoint response has no symbol field (the symbol from the URL placeholder is used instead). Required when `markets` section exists.
  - `symbol` (optional): Market symbol field. Use `"@key"` for JSON object key. Used when no `markets` section exists.
  - `symbol_separator` (optional): Character separating base/quote in the symbol (e.g., `"_"`, `"-"`, `"/"`). Use this when `symbol` contains both base and quote joined by a separator.
  - `symbol_inverted` (optional): `true` if the symbol is QUOTE_BASE instead of BASE_QUOTE. Only relevant when `symbol_separator` is set.
  - `base_currency_symbol` (optional): Field with base currency symbol. Not needed if `symbol_separator` is used or if `markets` section provides this info.
  - `quote_currency_symbol` (optional): Field with quote currency symbol. Not needed if `symbol_separator` is used or if `markets` section provides this info.
  - `last_price` (required): Field with last trade price.
  - `last_price_reversed` (optional): `true` if the price in the data is quote/base instead of base/quote (the handler will invert it: `1.0 / price`).
  - `base_volume` (required*): Field with base volume. *Either `base_volume` or `quote_volume` must be provided.
  - `quote_volume` (required*): Field with quote volume. Use instead of `base_volume` when only quote volume is available. It will be auto-converted to base volume using the formula: `base_volume = quote_volume / last_price`. *Either `base_volume` or `quote_volume` must be provided.
  - `include_zero_volume` (optional): `true` to include tickers with zero volume (normally they are filtered out).

### `order_books` section (optional)
- `url` (required): Order book endpoint URL. Use `%s` as placeholder for market symbol. The `%s` is replaced with `symbol` from `markets` section (if exists) or from `tickers` section.
- `data_path` (optional): JSON path to data. Rarely needed — typically the nesting is handled via the `asks`/`bids` paths directly.
- `field_paths` (required):
  - `asks` (required): JSON path to asks array (e.g., `"asks"`, `"data.asks"`). Can include nested path to handle data nesting.
  - `bids` (required): JSON path to bids array (e.g., `"bids"`, `"data.bids"`). Can include nested path to handle data nesting.
  - `order_price` (required): Field/index for price within each order. Use `"0"` for array index (when orders are arrays like `["price", "amount"]`), or a field name like `"price"` when orders are objects.
  - `order_amount` (required): Field/index for amount within each order. Use `"1"` for array index, or a field name like `"quantity"` when orders are objects.

### Nested field access
Fields can be nested using dots: `"ticker.last"`, `"data.result.price"`. This works for all string field values including `data_path` and all `field_paths` entries.

### Special values
- `"@this"` — Use as `data_path` when data is at the root level of the JSON response.
- `"@key"` — Use as `id`, `symbol`, or other field when the value is the key of the JSON object (not a field inside it).
- `"@symbol"` — Use as `id` in `tickers.field_paths` when using per-market endpoints (pattern 3) and the response has no field identifying the market. The symbol from the URL placeholder is used.
- `%s` — URL placeholder replaced with the market `symbol`. Used in `tickers.url` (pattern 3) and `order_books.url`.

## Step 4: Create the YAML File

Write the YAML file to: `<PROJECT_ROOT>/config/prod/exchanges/<exchange_name>.yaml`

The `<exchange_name>` should be:
- Lowercase
- No spaces (use `_` instead)
- No numbers at the start
- Similar to the exchange name

## Step 5: Test with config_ticker_checker

Run the checker from the project root:

```bash
cd <PROJECT_ROOT> && go run cmd_local/config_ticker_checker/main.go config/prod/exchanges/<exchange_name>.yaml
```

### Expected successful output
The checker should print **3 sample ticks** and optionally **3 sample order books**. Each tick should have:
- `MarketSymbol` - non-empty
- `BaseCurrencySymbol` - non-empty
- `QuoteCurrencySymbol` - non-empty
- `LastPrice` - non-zero
- `Volume` - non-zero (unless `include_zero_volume` is set)

### If the test fails
1. Read the error message carefully
2. Re-check the API response structure by fetching the endpoint again with `WebFetch`
3. Adjust the YAML file (fix `data_path`, `field_paths`, etc.)
4. Re-run the checker
5. If after 3 attempts it still fails, inform the user with the specific error

## Step 6: Report Test Results

Keep the output simple and short. Do NOT show raw API responses, field mappings, or YAML internals.

### If test passed:

Show the user a simple summary table of 3 sample ticks so they can verify the data looks correct (price is price, volume is volume, etc.):

> **Test passed! Here are 3 sample ticks — please verify the data looks correct:**
>
> | Market | Base | Quote | Last Price | Volume |
> |--------|------|-------|------------|--------|
> | BTC_USDT | BTC | USDT | 65432.10 | 1234.56 |
> | ... | ... | ... | ... | ... |

If order books were also tested, show 1 sample order book:

> **Sample order book for BTC_USDT:**
>
> | | Price | Amount |
> |------|-------|--------|
> | Ask 1 | 65433.00 | 0.5 |
> | Ask 2 | 65434.00 | 1.2 |
> | Bid 1 | 65431.00 | 0.8 |
> | Bid 2 | 65430.00 | 2.1 |

Then ask: **"Does this look correct? If yes, do you want me to commit and push this exchange?"**

Wait for the user to confirm the data looks right before proceeding.

### If test failed:

Show:
> **❌ Exchange `<exchange_name>` could not be added.**
> - Error: `<short error summary>`

Remove the YAML file if the exchange does not work after all attempts. Stop here.

## Step 7: Commit and Push (only if user agrees)

If the user says yes, commit and push the new YAML file in the `coinpaprika-services` repository:

```bash
cd <PROJECT_ROOT> && git add config/prod/exchanges/<exchange_name>.yaml && git commit -m "[yaml-exchange-skill] Add <exchange_name> exchange" && git push
```

The commit message MUST start with `[yaml-exchange-skill]` to clearly indicate it was created by this skill.

### After successful commit and push, show:

> **Next steps:**
> 1. Map `<internal_id>` to the exchange in the **Coinpaprika admin panel**
> 2. Run `deploy-fix insert_raw_ticks` on the `#rollout` Slack channel

### If the user declines to commit and push:

Show:
> **⚠️ The YAML file is saved locally but NOT committed.** Remember to commit and push it before proceeding with the deployment steps.

## Troubleshooting Common Issues

| Error | Likely Cause | Fix |
|-------|-------------|-----|
| `market symbol is empty` | Wrong `symbol` or `@key` path | Check the API response for the correct symbol field |
| `base currency symbol is empty` | Missing `base_currency_symbol` or wrong `symbol_separator` | Add base/quote fields or fix separator |
| `last price is zero` | Wrong `last_price` path | Check the actual field name in the API response |
| `volume is zero` | Wrong volume path or exchange has no volume | Try `quote_volume` instead, or set `include_zero_volume: true` |
| `could not read the yaml file` | File path issue | Ensure the file exists and has `.yaml` extension |
| HTTP errors | API may be geo-restricted or rate-limited | Try again, check if API requires auth (if so, cannot use YAML) |
| `market id is empty` | Wrong `id` path in markets | Check which field uniquely identifies a market |
| `invalid tick: market symbol is empty` | `id` in tickers doesn't match `id` in markets | Ensure both `id` fields reference the same value |

## What CANNOT Be Done via YAML

If an exchange requires any of the following, tell the user it **cannot be implemented via YAML**:
- Authentication (API keys, signatures, tokens)
- WebSocket connections
- Pagination to get all markets/tickers
- POST requests (only GET is supported)
- Custom headers
- Rate limiting configuration
- Response formats other than JSON (XML, CSV, etc.)
- Complex data transformations beyond simple field mapping
- Multiple ticker endpoints that need to be merged
