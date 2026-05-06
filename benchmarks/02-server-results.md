# 02 — Server Load Test Results

## Test Configuration

- Model: tinyllama-1.1b-chat-v1.0.Q4_K_M.gguf
- Server: llama-server via llama-cpp-python on :8080
- Threads: 8, Context: 2048, Parallel: 4
- GPU layers: 99 (CPU inference on Intel i5-8250U)

## Load Test — 10 Users, 1 minute

| Metric | Value |
|--------|------:|
| Total requests | 7 |
| Failed requests | 0 (0%) |
| Avg response time | 21,928 ms |
| Min / Max | 6,478 / 37,642 ms |
| Median (P50) | 22,000 ms |
| P95 | 38,000 ms |
| P99 | 38,000 ms |
| Throughput | 0.16 req/s |

## Load Test — 50 Users, 1 minute

| Metric | Value |
|--------|------:|
| Total requests | 5 |
| Failed requests | 0 (0%) |
| Avg response time | 29,079 ms |
| Min / Max | 15,090 / 46,835 ms |
| Median (P50) | 28,000 ms |
| P95 | 47,000 ms |
| P99 | 47,000 ms |
| Throughput | 0.10 req/s |

## KV Cache Observation

llama-cpp-python 0.3.22 does NOT expose /metrics endpoint. KV cache behavior observed via:
- Server logs showing "prefix-match hit" indicating cache reuse
- Tokens processed logged via `llama_perf_context_print` per request

## Server Endpoints Verified

- `GET /v1/models` — model list OK
- `POST /v1/chat/completions` — completions OK  
- `GET /metrics` — NOT available (planned for future version)
