# DexPaprika Streaming API Reference

Two SSE feeds share one transport. No API key required.

| Feed | Endpoint | Cadence | Use for |
|---|---|---|---|
| Token prices | `/sse/prices` | ~1s per asset | tickers, portfolios, alerts |
| Pool reserves | `/sse/reserves` | block-level | liquidity dashboards, MEV, real-time TVL |

Base URL: `https://streaming.dexpaprika.com`

---

## Quotas and limits

- **Subscriptions per POST connection:** 25. Larger arrays are rejected with HTTP 400 before any events flow.
  - `POST /sse/prices` rejects with `{"message":"too many assets, max 25 allowed"}`.
  - `POST /sse/reserves` rejects with `{"message":"too many subscriptions"}`.
- **Concurrent SSE streams per IP:** 10. The 11th connection returns `429 {"message":"ip stream limit exceeded"}`.
- **Ping interval:** 15 seconds. A `ping` event keeps idle connections open.

---

## Event types

Either feed can emit any of these:

| Event | Where | Payload shape |
|---|---|---|
| `token_price` | prices feed | `{address, chain, price, timestamp, timestamp_price, token_price}` |
| `reserve_update` | reserves feed | `{chain, pool_id, block, previous_block, tokens[], total_reserve_usd, total_delta_usd}` |
| `ping` | both | `{"time": <unix>}` |
| `warning` | both | `{"message": "..."}` (non-fatal notice, e.g. deprecation) |
| `error` | both | `{"message": "..."}` (stream-terminating error) |

The legacy `t_p` event and compact `{a, c, p, t, t_p}` shape exist on the deprecated `/stream` path only.

### Wire-format gotchas

- `block`, `previous_block`, `reserve`, `delta` arrive as **JSON strings**, not numbers. They routinely exceed `Number.MAX_SAFE_INTEGER`. Parse with `BigInt` for arithmetic, or `Number()` for display-only. The USD fields (`price_usd`, `reserve_usd`, `delta_usd`, `total_reserve_usd`, `total_delta_usd`) are regular JSON numbers.
- Both orderings of `event:` and `data:` lines within one SSE message are valid per the spec, and the server has used both during the rollout. Parsers must buffer one message at a time (split on blank line) and dispatch on the parsed event type, not on field order.
- `previous_block` is omitted from the payload on the first event after subscribing (`omitempty`).

---

## Token prices, single token (GET)

```
GET /sse/prices?method=token_price&chain={network}&address={token_address}
```

| Parameter | Required | Description |
|---|---|---|
| method | yes | `token_price` (current). `t_p` is deprecated. |
| chain | yes | Network ID (`ethereum`, `solana`, `bsc`, etc.) |
| address | yes | Token contract address |
| limit | no | Max number of messages before the server closes the connection |

Example:

```bash
curl --http1.1 -N "https://streaming.dexpaprika.com/sse/prices?method=token_price&chain=ethereum&address=0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2"
```

---

## Token prices, multiple tokens (POST)

```
POST /sse/prices
Accept: text/event-stream
Content-Type: application/json

[
  {"chain": "ethereum", "address": "0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2", "method": "token_price"},
  {"chain": "solana",   "address": "So11111111111111111111111111111111111111112",  "method": "token_price"}
]
```

Max 25 entries per request body. One invalid asset cancels the entire stream; validate addresses with REST `GET /search?query=...` first.

---

## Pool reserves, single pool or single token (GET)

```
GET /sse/reserves?method=pool_reserves&chain={network}&address={pool_address}
GET /sse/reserves?method=token_reserves&chain={network}&address={token_address}
```

- `method=pool_reserves`: subscribe to one specific pool. Events fire when that pool's reserves change.
- `method=token_reserves`: subscribe to one token across every pool it sits in (high event volume on major assets like USDC).

---

## Pool reserves, multiple entries (POST)

```
POST /sse/reserves
Accept: text/event-stream
Content-Type: application/json

[
  {"chain": "ethereum", "address": "0x88e6a0c2ddd26feeb64f039a2c41296fcb3f5640", "method": "pool_reserves"},
  {"chain": "ethereum", "address": "0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48", "method": "token_reserves"}
]
```

Entries can mix `pool_reserves` and `token_reserves`. Max 25 per request body.

---

## Example wire output

Token price event:
```
event: token_price
data: {"address":"0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2","chain":"ethereum","price":"2145.78","timestamp":1779110450,"timestamp_price":1779110449,"token_price":1779110449}
```

Reserve update event (block 25,122,344 on a Uniswap V3 USDC/WETH pool):
```
event: reserve_update
data: {"chain":"ethereum","pool_id":"0x88e6a0c2ddd26feeb64f039a2c41296fcb3f5640","block":"25122344","tokens":[{"token_id":"0xa0b8...","reserve":"61995526164300","delta":"103817802","price_usd":1.0000511,"reserve_usd":61998696.43,"delta_usd":103.82},{"token_id":"0xc02a...","reserve":"17706554631959222896552","delta":"-48362919581328866","price_usd":2145.78,"reserve_usd":37994383.59,"delta_usd":-103.77}],"total_reserve_usd":99993080.03,"total_delta_usd":0.04}
```

That second event captures one swap: ~$103.82 USDC came in, ~$103.77 of WETH went out, `total_delta_usd: 0.04` is the trading fee captured. No on-chain log decoding required.

---

## Python parser (correct event-by-blank-line buffering)

```python
import json, requests

url = "https://streaming.dexpaprika.com/sse/prices"
params = {"method": "token_price", "chain": "ethereum",
          "address": "0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2"}

with requests.get(url, params=params, stream=True) as r:
    r.raise_for_status()
    msg_lines = []
    for line in r.iter_lines(decode_unicode=True):
        if line:
            msg_lines.append(line); continue

        # Blank line: dispatch the buffered message.
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
        elif event_type == "warning" and data_str:
            print("[warning]", json.loads(data_str)["message"])
```

For reserves, swap `event_type == "reserve_update"` and access `d['pool_id']`, `d['block']`, `d['total_delta_usd']`. Use `int(d['tokens'][0]['reserve'])` for raw-amount arithmetic, not `float`.

---

## JavaScript parser (fetch + for-await)

```javascript
const url = 'https://streaming.dexpaprika.com/sse/prices';
const body = JSON.stringify([
  { chain: 'ethereum', address: '0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2', method: 'token_price' },
]);

const r = await fetch(url, {
  method: 'POST',
  headers: { Accept: 'text/event-stream', 'Content-Type': 'application/json' },
  body,
});

const decoder = new TextDecoder();
let buffer = '';

for await (const chunk of r.body) {
  buffer += decoder.decode(chunk, { stream: true });
  const messages = buffer.split('\n\n');
  buffer = messages.pop() ?? '';

  for (const msg of messages) {
    const lines = msg.split('\n');
    const eventLine = lines.find(l => l.startsWith('event:'));
    const dataLine  = lines.find(l => l.startsWith('data:'));
    if (!dataLine) continue;

    const eventType = eventLine ? eventLine.slice(6).trim() : 'message';
    if (eventType !== 'token_price') continue;   // skip ping/warning/error

    const d = JSON.parse(dataLine.slice(5).trim());
    console.log(`${d.chain} ${d.address}: $${parseFloat(d.price).toFixed(4)}`);
  }
}
```

`EventSource` does not support POST, so multi-asset subscriptions on the browser require this `fetch` + ReadableStream pattern. Single-asset GET subscriptions work with `EventSource` directly.

For reserves, use `BigInt(d.tokens[0].reserve)` for raw-amount arithmetic.

---

## HTTP/1.1 requirement

SSE streaming requires HTTP/1.1. HTTP/2 (curl's default for HTTPS) may not behave correctly with persistent text streams.

- curl: add `--http1.1`.
- Python `requests`: works by default.
- Node.js `fetch`: works by default.

---

## Error codes

| Code | Cause | Body |
|---|---|---|
| 200 | Connected, streaming | (SSE event stream) |
| 400 | Bad params, unsupported chain, asset not found, or one invalid asset in a batch | `{"message": "..."}` |
| 400 | Too many entries in POST body (26+) | `{"message":"too many assets, max 25 allowed"}` (`/sse/prices`) or `{"message":"too many subscriptions"}` (`/sse/reserves`) |
| 429 | IP stream limit exceeded | `{"message":"ip stream limit exceeded"}` |

In-stream errors arrive as `event: error` SSE messages. They terminate the stream.

---

## Deprecated paths

`/stream` and `/reserves/stream` are predecessors. `/stream` still works but emits a one-shot `warning` event on connect telling clients to migrate to `/sse/prices`. `/reserves/stream` was retired and now returns 404. New code must use `/sse/prices` and `/sse/reserves`.

---

## Best practices

- Reconnect with exponential backoff on disconnect. Don't tight-loop.
- Use POST for multi-asset subscriptions: one connection instead of many.
- Parse `price` as a string for decimal precision. Don't `parseFloat` and re-serialize.
- Filter on the `event:` line. Treat unknown events as no-ops so future server-side additions don't break the handler.
- Use `BigInt` for `reserve`, `delta`, `block`, `previous_block` when you need arithmetic.
- Open parallel connections if you need more than 25 subscriptions, up to the 10/IP cap.
- Validate all asset addresses via REST `/search` before streaming. One bad address kills the entire stream.

---

## CLI streaming

```bash
# Single token (CLI uses the deprecated /stream path under the hood until v0.2.0)
dexpaprika-cli stream ethereum 0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2

# Multiple tokens
dexpaprika-cli stream ethereum --tokens 0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2,0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48

# Cap event count for smoke-tests
dexpaprika-cli stream ethereum 0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2 --limit 10
```

Reserves streaming is not yet exposed in the CLI; use the direct SSE API above until CLI v0.2.0 ships.
