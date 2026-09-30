# DexPaprika Streaming API Reference

Four SSE feeds share one transport. **Access differs per feed, and only one of them is keyless.**

Streaming is metered the same way as REST: each update delivered counts as one credit. Updates are swap-driven, not clock-driven and not per block: they are pushed only when a swap moves the value, so a quiet chain can go minutes without emitting anything while a fast-moving one draws down quota like polling would. Connection caps: 25 subscriptions per connection; concurrent streams 10 keyless (per IP), 10 on a free key, 30 on Dev, 100 on Pro (per account). Current allowances are on https://dexpaprika.com/api/pricing; do not hard-code them in published copy.

| Feed | Endpoint | Access | When it fires |
|---|---|---|---|
| Token prices | `/sse/prices` | keyless on the 35 preview assets only, free key for any asset | when a swap moves the price |
| Pool reserves | `/sse/reserves` | any key, free included | when a swap changes the pool's reserves |
| Swap transactions | `/sse/transactions` | **Dev, Pro or Enterprise** | on every swap |
| Token OHLCV candles | `/sse/ohlcv` | **Dev, Pro or Enterprise** | when a candle bucket seals, and only if it saw a swap |

Measured 2026-09-28, keyless, all four on one pass:

| Request | Response |
|---|---|
| `/sse/prices` on WETH ethereum or SOL (preview assets) | `200`, `token_price` events |
| `/sse/prices` on USDC ethereum (not a preview asset) | `403 {"error":"preview_only","tier":"keyless","message":"keyless access is limited to preview streams ...","links":{...}}` |
| `/sse/reserves` | `403 {"error":"preview_only","tier":"keyless","message":"this stream requires an API key ...","links":{...}}` |
| `/sse/transactions`, `/sse/ohlcv` | `403 {"error":"plan_required","tier":"keyless","message":"this endpoint requires a Dev or Pro plan","required_tier":"dev","links":{...}}` |

**Since 2026-09-30** `/sse/transactions` is paid-only: keyless callers and free keys get `403` with `"error":"plan_required"` and `"required_tier":"dev"` before a stream opens, and `/sse/ohlcv` refusals carry the same body. Dev, Pro and Enterprise keys use `streaming-pro.dexpaprika.com` for both.

Match on the `error` field (`preview_only`) rather than the message text: the human sentence contains an em dash and gets reworded. The `links` object carries URLs for registering, the docs and pricing that you can show the user.

**Hosts.** Keyless and free keys use `https://streaming.dexpaprika.com`. Pro and Enterprise use `https://streaming-pro.dexpaprika.com`. They are not interchangeable, and `/sse/ohlcv` and `/sse/transactions` need the paid host. On the `-pro` hosts a request the edge does not recognise, one with no `Authorization` header for example, gets a `403` HTML page instead of JSON.

Base URL: `https://streaming.dexpaprika.com`

---

## Quotas and limits

- **Subscriptions per POST connection:** 25. Larger arrays are rejected with HTTP 400 before any events flow.
  - `POST /sse/prices` rejects with `{"message":"too many assets, max 25 allowed"}`.
  - `POST /sse/reserves` rejects with `{"message":"too many subscriptions"}`.
- **Concurrent SSE streams:** 10 keyless, counted per IP; 10 on a free key, 30 on Dev, 100 on Pro, counted per account whatever the IP. The next connection returns `429 {"error":"rate_limited","tier":...,"message":"Concurrent stream limit reached for your plan. Close an open stream before starting another.","links":{...}}`. Older copies quote `ip stream limit exceeded`; that body is no longer sent.
- **Ping interval:** 15 seconds. A `ping` event keeps idle connections open.

---

## Event types

Either feed can emit any of these:

| Event | Where | Payload shape |
|---|---|---|
| `token_price` | prices feed | `{address, chain, price, timestamp, timestamp_price, token_price}` |
| `pool_reserves` | reserves feed | `{chain, pool_id, block, previous_block, tokens[], total_reserve_usd, total_delta_usd, timestamp, block_timestamp}` |
| `token_reserves` | reserves feed | `{chain, token_id, reserve, delta, block, price_usd, reserve_usd, delta_usd, updated_at, timestamp}` |
| `pool` / `token` | transactions feed | one swap. Event name matches the `method` you subscribed with |
| `token_ohlcv` | ohlcv feed | `{chain, token_id, interval, timestamp, open, high, low, close, avg, volume_usd, txns}` |
| `ping` | all | `{"time": <unix>}` |
| `warning` | all | `{"message": "..."}` (non-fatal notice, e.g. deprecation) |
| `error` | all | `{"message": "..."}` (stream-terminating error) |

**Only data events are billed.** `ping`, `warning` and `error` are written without firing the metering callback, so an idle connection costs nothing.

`method=t_p` still answers on `/sse/prices` with the legacy compact `{a, c, p, t, t_p}` shape under the event name `t_p` (measured 2026-09-28). It is not in the spec; use `token_price` in new code.

**Reserves events were restructured.** The old single `reserve_update` event no longer exists. The server now emits one event named after the subscription method:

- `pool_reserves`: fired for a `method=pool_reserves` subscription. Carries a nested `tokens[]` array (one entry per token in the pool, each with `token_id`, `reserve`, `delta`, `price_usd`, `reserve_usd`, `delta_usd`) plus pool-level `block`, `previous_block`, `total_reserve_usd`, `total_delta_usd`, and the new `timestamp` and `block_timestamp` fields.
- `token_reserves`: fired for a `method=token_reserves` subscription. Flat, single-token shape: `token_id`, `reserve`, `delta`, `block`, `price_usd`, `reserve_usd`, `delta_usd`, and the new `updated_at` and `timestamp` fields. No nested array.

A consumer that previously matched `reserve_update` must be updated to match `pool_reserves` and `token_reserves` and to read the new timestamp fields.

### `request_id` correlation

An optional `request_id` lets you correlate events back to the subscription that produced them. It is a `uint32` (range 0..4294967295).

- **GET:** pass `request_id` as a query parameter.
- **POST:** set `request_id` per asset in the body. If omitted, it defaults to that asset's index in the request array.

The server echoes the value back as a dedicated `request_id:` SSE line on **data events only** (`token_price`, `pool_reserves`, `token_reserves`). It is **not** attached to `ping`, `warning`, or `error` events, so a parser must tolerate its absence. On GET, omitting the parameter still produces `request_id: 0` lines on data events, so a parser must also tolerate its presence when it never asked for one.

### Wire-format gotchas

- `block`, `previous_block`, `reserve`, `delta` arrive as **JSON strings**, not numbers. They routinely exceed `Number.MAX_SAFE_INTEGER`. Parse with `BigInt` for arithmetic, or `Number()` for display-only. The USD fields (`price_usd`, `reserve_usd`, `delta_usd`, `total_reserve_usd`, `total_delta_usd`) and the timestamp fields (`timestamp`, `block_timestamp`, `updated_at`) are regular JSON numbers.
- `previous_block` appears on `pool_reserves` events only (`token_reserves` has no such field) and is marked `omitempty` server-side, so parse it defensively rather than assuming it is always present.
- Both orderings of `event:` and `data:` lines within one SSE message are valid per the spec, and the server has used both during the rollout. A `request_id:` line may also appear in the same message. Parsers must buffer one message at a time (split on blank line) and dispatch on the parsed event type, not on field order.

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

## Swap transactions (GET and POST)

```
GET /sse/transactions?method=pool&chain={network}&address={pool_address}
GET /sse/transactions?method=token&chain={network}&address={token_address}
```

**Dev, Pro or Enterprise**, on `streaming-pro.dexpaprika.com`; keyless and free keys get `403` with `"error":"plan_required"`. `method=pool` subscribes to one pool; `method=token` subscribes to every pool the token trades in, which on a major asset is three orders of magnitude more traffic and therefore more credits. POST takes up to 25 subscriptions, same shape as the other feeds.

Wire-format traps, both of which hide on Solana and bite on 18-decimal EVM tokens:

- `amount_0` / `amount_1` are raw signed **JSON numbers**, not strings, and routinely exceed `Number.MAX_SAFE_INTEGER`. The `_usd` fields are ordinary safe floats. `block_number` is already a string.
- `token_0` is not necessarily the same token as REST's `token_0` for the same pool. Resolve by address, never by index.
- Direction is not a field. Derive it from the sign of `amount_0`.
- `volume_usd` is about half the notional, roughly one side of the trade. Do not sum it against a REST 24h volume figure.

Full write-up: https://docs.dexpaprika.com/streaming/transactions-streaming

---

## Token OHLCV candles (GET, Dev, Pro or Enterprise)

```
GET /sse/ohlcv?method=token_ohlcv&chain={network}&address={token_address}&interval=60s
```

**A paid plan (Dev, Pro or Enterprise) and the `streaming-pro.dexpaprika.com` host.** Keyless and free keys get `403` with `"error":"plan_required"` and `"required_tier":"dev"`. One subscription per connection; there is no POST form.

| Parameter | Required | Description |
|---|---|---|
| method | yes | `token_ohlcv` (the only value) |
| chain | yes | Network ID. Every indexed chain broadcasts candles |
| address | yes | Token contract address |
| interval | no | `1s`, `5s` or `60s`. **Defaults to `1s`, the most expensive choice** |
| request_id | no | uint32, echoed on data events |
| limit | no | Max event count **including backfill**, then the server closes |
| since | no | Unix seconds. Backfills candles before live updates. Must be within the last 15 minutes |
| `Last-Event-ID` | no | **Request header**, not a query param. Event timestamp to resume from |

```bash
# live
curl --http1.1 -N -H "Authorization: $DEXPAPRIKA_API_KEY" \
  "https://streaming-pro.dexpaprika.com/sse/ohlcv?method=token_ohlcv&chain=ethereum&address=0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48&interval=60s"

# with the last 5 minutes replayed first
curl --http1.1 -N -H "Authorization: $DEXPAPRIKA_API_KEY" \
  "https://streaming-pro.dexpaprika.com/sse/ohlcv?method=token_ohlcv&chain=solana&address=So11111111111111111111111111111111111111112&interval=5s&since=$(( $(date +%s) - 300 ))"
```

### The five behaviours that are not in the spec

Each one changes how a consumer is written.

1. **A candle exists only where a swap did.** A bucket is created by the first swap in the interval and is never sealed if it stayed empty. `interval=1s` is a *ceiling* of 60 events a minute, not a rate. The bound is `candles/hour <= min(3600 / interval_seconds, swaps/hour)`, so a quiet token costs almost nothing on any interval.
2. **Sealed candles get republished.** A swap landing after its bucket sealed recomputes and resends the candle under **the same `timestamp`**, with updated values, and bills again. Late windows: `1s` accepts for 2 min, `5s` for 5 min, `60s` for 10 min. Seal delay after the bucket closes is 1s / 2s / 3s. **Key your store on `timestamp` and overwrite.** Appending draws the same candle twice.
3. **`since` and `Last-Event-ID` diverge past 15 minutes, deliberately.** `since` older than the window is a `400`; `Last-Event-ID` older is **silently clamped** to 15 minutes ago, so a browser back from a long outage reconnects instead of failing. A future or unparseable `Last-Event-ID` is ignored and you get live only. `since` wins when both are sent.
4. **The resume boundary is inclusive** (`WHERE timestamp >= ?`), so resuming at the last `id:` you saw redelivers that candle. Harmless if you overwrite; pass `since` one second later to avoid paying for it.
5. **Backfill is billed and counts toward `limit`.** `since` plus a small `limit` can close the connection before a single live candle arrives.

Two more worth knowing: the `id:` line is the candle timestamp in unix seconds, on live events as well as backfilled ones, which is what makes browser `EventSource` resume work with no code. And **`avg` is an unweighted mean** of the price samples, not volume weighted, despite the upstream tick source being VWAP-derived.

Cost ceiling per subscription, arithmetic from the interval alone:

| Interval | Ceiling/min | Ceiling per 30 days |
|---|---|---|
| `1s` | 60 | 2,592,000 |
| `5s` | 12 | 518,400 |
| `60s` | 1 | 43,200 |

Full write-up: https://docs.dexpaprika.com/streaming/ohlcv-streaming

---

## Example wire output

Token price event:
```
event: token_price
data: {"address":"0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2","chain":"ethereum","price":"2145.78","timestamp":1779110450,"timestamp_price":1779110449,"token_price":1779110449}
```

Pool reserves event (block 25,445,005 on a Uniswap V3 USDC/WETH pool, `request_id=12345`):
```
event: pool_reserves
request_id: 12345
data: {"chain":"ethereum","pool_id":"0x88e6a0c2ddd26feeb64f039a2c41296fcb3f5640","block":"25445005","previous_block":"25445004","tokens":[{"token_id":"0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48","reserve":"20267554175020","delta":"312181314","price_usd":1.0000086018651697,"reserve_usd":20267728.51378833,"delta_usd":312.1839993415715},{"token_id":"0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2","reserve":"38185564889605351116310","delta":"-187888867058651209","price_usd":1661.0500503604078,"reserve_usd":63428134.48291959,"delta_usd":-312.0928120899326}],"total_reserve_usd":83695862.99670792,"total_delta_usd":0.09118725163892805,"timestamp":1782997443,"block_timestamp":1782997439}
```

That event captures one swap: ~$312.18 of USDC came in, ~$312.09 of WETH went out, `total_delta_usd` is the residual (fee + rounding). No on-chain log decoding required. `timestamp` is when the server emitted the event; `block_timestamp` is the block's own time.

Token reserves event (USDC across all its pools, `request_id=777`):
```
event: token_reserves
request_id: 777
data: {"chain":"ethereum","token_id":"0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48","reserve":"190167972986869","delta":"3225671508","block":"25445017","price_usd":0.9999378879562559,"reserve_usd":190156161.26541212,"delta_usd":3225.471154950191,"updated_at":1782997583,"timestamp":1782997585}
```

This is the flat single-token shape: no nested `tokens[]` array, and the aggregate `reserve`/`delta` is for that one token across every pool it sits in.

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

For reserves, match the two method-named events instead. For `event_type == "pool_reserves"` read `d['pool_id']`, `d['block']`, `d['total_delta_usd']`, `d['block_timestamp']`, and iterate `d['tokens']` (use `int(d['tokens'][0]['reserve'])` for raw-amount arithmetic, not `float`). For `event_type == "token_reserves"` the payload is flat: read `d['token_id']`, `d['reserve']`, `d['delta']`, `d['updated_at']`. To correlate, capture the `request_id:` line during buffering (it sits alongside `event:`/`data:` and is absent on `ping`/`warning`/`error`).

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

For reserves, match `pool_reserves` (nested `d.tokens[]`, use `BigInt(d.tokens[0].reserve)`) or `token_reserves` (flat, use `BigInt(d.reserve)`).

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
| 403 | Keyless on a feed or asset that needs a key | `{"error":"preview_only","tier":"keyless","message":"...","links":{...}}` |
| 403 | `/sse/ohlcv` or `/sse/transactions` without a paid plan | `{"error":"plan_required","tier":"keyless","message":"this endpoint requires a Dev or Pro plan","required_tier":"dev","links":{...}}` |
| 403 | `-pro` host, a request the edge does not recognise (no `Authorization` header, for example) | HTML block page, no JSON |
| 401 | Key present and rejected | `{"message":"api key verification has failed"}` |
| 404 | `/sse/ohlcv`, token not indexed on that chain | `{"message":"token not found: {chain}/{address}"}` |
| 429 | Concurrent-stream cap for the plan reached | `{"error":"rate_limited","tier":...,"message":"Concurrent stream limit reached ...","links":{...}}` |

`403` and `401` mean different things and the difference is diagnostic: `403` is "wrong plan or no key", `401` is "you sent a key and it was rejected". A `403` that is HTML rather than JSON came from the edge: check the header and the host first.

In-stream errors arrive as `event: error` SSE messages. They terminate the stream.

---

## Deprecated paths

`/stream` and `/reserves/stream` are predecessors and both are gone. Measured 2026-09-09: `/stream` returns
`410 {"code":410,"message":"endpoint removed","replacement":"/sse/prices"}`, so it no longer emits the old
one-shot `warning` event and no longer serves data. `/reserves/stream` returns 404. Use `/sse/prices` and
`/sse/reserves`.

`/stream` used the event name `t_p` and compact keys `{a, c, p, t, t_p}`, and `/sse/prices` still accepts
`method=t_p` for that shape. The current one is the event name `token_price` and `{address, chain, price, timestamp, timestamp_price,
token_price}`. Migrating a caller means changing the URL, the `method` value, the event name AND the field reads.

---

## Best practices

- Reconnect with exponential backoff on disconnect. Don't tight-loop.
- Use POST for multi-asset subscriptions: one connection instead of many.
- Parse `price` as a string for decimal precision. Don't `parseFloat` and re-serialize.
- Filter on the `event:` line. Treat unknown events as no-ops so future server-side additions don't break the handler.
- Use `BigInt` for `reserve`, `delta`, `block`, `previous_block` when you need arithmetic.
- On the reserves feed, match `pool_reserves` and `token_reserves`, not the retired `reserve_update`. Pass a `request_id` if you fan out subscriptions and need to route events back; read it from the `request_id:` line on data events.
- Open parallel connections if you need more than 25 subscriptions, up to the plan's concurrent-stream cap (10 keyless or free, 30 Dev, 100 Pro).
- Validate all asset addresses via REST `/search` before streaming. One bad address kills the entire stream.
- On `/sse/ohlcv`, key candles by `timestamp` and overwrite. Republished candles and the inclusive resume boundary both redeliver a timestamp you already hold.
- Prefer `Last-Event-ID` over computing a `since`. It is clamped rather than refused, so it cannot fail on a long outage.
- Pick the widest OHLCV interval the product tolerates. `60s` carries the same trades as `1s` for one sixtieth of the spend.
- Trust the 15 s heartbeat, not the socket. On the candle and transaction feeds a silent connection and a quiet asset look identical.

---

## CLI streaming

The CLI talks to `/sse/prices` and `/sse/reserves` directly (not the deprecated `/stream` path).

```bash
# Single token prices
dexpaprika-cli stream ethereum 0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2

# Multiple tokens: --tokens takes a PATH to a JSON file (max 25 entries),
# e.g. [{"chain": "ethereum", "address": "0xc02a..."}, {"chain": "solana", "address": "JUPy..."}]
dexpaprika-cli stream --tokens watchlist.json

# Cap event count for smoke-tests
dexpaprika-cli stream ethereum 0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2 --limit 10

# Reserves: one pool (fires pool_reserves events)
dexpaprika-cli stream-reserves ethereum 0x88e6a0c2ddd26feeb64f039a2c41296fcb3f5640 --method pool_reserves

# Reserves: one token across all its pools (fires token_reserves events), with request_id correlation
dexpaprika-cli stream-reserves ethereum 0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48 --method token_reserves --request-id 777 --limit 10

# Reserves: multiple subscriptions from a JSON file (max 25, methods can be mixed),
# e.g. [{"chain": "ethereum", "address": "0x88e6...", "method": "pool_reserves", "request_id": 1}]
dexpaprika-cli stream-reserves --subscriptions reserves.json
```
