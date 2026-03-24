# DexPaprika Streaming API Reference

Stream live token prices via Server-Sent Events (SSE). ~1 second updates, 1-2,000 tokens per connection. No API key required.

Base URL: `https://streaming.dexpaprika.com`

---

## Single token (GET)

```
GET /stream?method=t_p&chain={network}&address={token_address}
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| method | string | yes | Always `t_p` (token price) |
| chain | string | yes | Network ID (e.g., `ethereum`, `solana`) |
| address | string | yes | Token contract address |

Example:
```bash
curl --http1.1 -N "https://streaming.dexpaprika.com/stream?method=t_p&chain=ethereum&address=0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2"
```

Python:
```python
import requests, json

r = requests.get("https://streaming.dexpaprika.com/stream",
    params={"method": "t_p", "chain": "ethereum", "address": "0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2"},
    stream=True)

for line in r.iter_lines():
    if line and line.startswith(b'data:'):
        data = json.loads(line[5:])
        print(f"{data['c']} {data['a']}: ${data['p']}")
```

---

## Multiple tokens (POST)

```
POST /stream
```

Headers:
- `Accept: text/event-stream`
- `Content-Type: application/json`

Body: JSON array of subscription objects (max 2,000):

```json
[
    {"chain": "ethereum", "address": "0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2", "method": "t_p"},
    {"chain": "solana", "address": "So11111111111111111111111111111111111111112", "method": "t_p"}
]
```

**One invalid asset cancels the entire stream.** Validate token addresses with the REST API `GET /search?query={term}` before streaming.

Example:
```bash
curl --http1.1 -N -X POST "https://streaming.dexpaprika.com/stream" \
  -H "Accept: text/event-stream" -H "Content-Type: application/json" \
  -d '[{"chain":"ethereum","address":"0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2","method":"t_p"},{"chain":"solana","address":"So11111111111111111111111111111111111111112","method":"t_p"}]'
```

Python:
```python
import requests, json

assets = [
    {"chain": "ethereum", "address": "0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2", "method": "t_p"},
    {"chain": "solana", "address": "So11111111111111111111111111111111111111112", "method": "t_p"}
]

r = requests.post("https://streaming.dexpaprika.com/stream",
    headers={"Accept": "text/event-stream", "Content-Type": "application/json"},
    json=assets, stream=True)

for line in r.iter_lines():
    if line and line.startswith(b'data:'):
        data = json.loads(line[5:])
        print(f"{data['c']} {data['a']}: ${data['p']}")
```

---

## Response format

Each SSE frame:
```
data: {"a":"0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2","c":"ethereum","p":"2739.79","t":1769778188,"t_p":1769778187}
event: t_p

```

Parsed JSON:
```json
{
    "a": "0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2",
    "c": "ethereum",
    "p": "2739.79",
    "t": 1769778188,
    "t_p": 1769778187
}
```

| Field | Type | Description |
|-------|------|-------------|
| `a` | string | Token contract address |
| `c` | string | Chain/network ID |
| `p` | string | Price in USD (string for precision — parse as decimal, not float) |
| `t` | integer | Server timestamp (Unix seconds) |
| `t_p` | integer | Price timestamp (Unix seconds) |

---

## HTTP/1.1 requirement

SSE streaming requires HTTP/1.1. HTTP/2 (curl's default for HTTPS) may cause issues.

- curl: add `--http1.1`
- Python requests: works by default (uses HTTP/1.1)
- Node.js: use `http` module or libraries that support HTTP/1.1

---

## Error codes

| Code | Meaning |
|------|---------|
| 200 | Connected, streaming |
| 400 | Bad parameters, unsupported chain, or asset not found |
| 429 | Capacity exceeded (retry with backoff) |

---

## Best practices

- Reconnect with exponential backoff on disconnect
- Use POST for 10+ assets (one connection instead of many)
- Parse `p` as string/decimal to preserve precision (not float)
- Validate all assets via REST `/search` before streaming — one bad asset kills the entire stream
- Use `--limit N` with the CLI to cap the number of events for testing

---

## CLI streaming

```bash
# Single token
dexpaprika-cli stream ethereum 0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2

# Multiple tokens
dexpaprika-cli stream ethereum --tokens 0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2,0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48

# Limit to N events
dexpaprika-cli stream ethereum 0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2 --limit 10
```
