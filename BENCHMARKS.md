# Benchmarks

## Lab 2b: before warm-up

First three `POST /v1/predict` requests with `payloads/sample.json` after a fresh server start
(`curl -s -o /dev/null -w "%{time_total}s\n"`, MacBook Pro, `OMP_NUM_THREADS=1`):

| Request | Time |
|---|---|
| 1 | 0.031055s |
| 2 | 0.003089s |
| 3 | 0.002747s |

## Lab 2b: after warm-up

Same measurement after adding one warm-up prediction in the lifespan:

| Request | Time |
|---|---|
| 1 | 0.003230s |
| 2 | 0.002492s |
| 3 | 0.002508s |

## Lab 2b: load test

`python scripts/loadtest.py` (2,000 requests, 25 concurrent clients, `payloads/sample.json`),
server and load generator on the same MacBook Pro (8 cores), `OMP_NUM_THREADS=1`, no `--reload`:

| Server | p50 | p99 | max | Throughput | non-2xx |
|---|---|---|---|---|---|
| 1 worker | 47.4 ms | 152.6 ms | 220.4 ms | 463 req/s | 0 |
| `--workers 4` | 35.3 ms | 87.3 ms | 112.7 ms | 629 req/s | 0 |

A second run with 4 workers gave the same picture (p50 36.2 ms, p99 89.5 ms, 619 req/s).
