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

## Lab 3: images

Docker Desktop 29.9.3 on the same MacBook Pro (Docker VM: 8 CPUs, ~3.8 GB RAM).

| Image | Disk usage | Content size | Build time | Runs as | `leaked.env` inside? |
|---|---|---|---|---|---|
| `fraud-service:naive` (`Dockerfile.naive`) | 3.33 GB | 845 MB | 382 s (cold) | `root` | yes |
| `fraud-service:dev` (multi-stage) | 847 MB | 180 MB | 111 s | `appuser` | no |
| `fraud-service:dev`, rebuild after a one-line edit in `routes.py` | | | 2.0 s | | |

Build context: the naive build sent 479.5 MB (it copied `.venv/` and `.git/`); with `.dockerignore`
the context is a few kB.

## Lab 3: container under load

`python scripts/loadtest.py` against `docker compose up` (2 uvicorn workers, 512 MB memory limit):

| Setup | p50 | p99 | max | Throughput | non-2xx |
|---|---|---|---|---|---|
| Lab 2b, local, 1 worker | 47.4 ms | 152.6 ms | 220.4 ms | 463 req/s | 0 |
| Lab 3, container, as shipped | 90.2 ms | 780.1 ms | 1345.8 ms | 170 req/s | 0 |
| Lab 3, container, `OMP_NUM_THREADS=1` (one-off `docker run`) | 54.3 ms | 202.2 ms | 297.3 ms | 383 req/s | 0 |

The container does not set `OMP_NUM_THREADS`, so each of the 2 workers starts one math thread per
CPU (8) and they fight over the cores; that is what pushed p99 to 780 ms.

- `docker stats --no-stream`: `fraud-api` 460.7 MiB of its 512 MiB limit, `feature-cache` 7.5 MiB.
- `docker compose stop fraud-api`: 1.1 s, with a clean `shutdown complete` in the logs (graceful).

## Lab 4: test suite

Local, `OMP_NUM_THREADS=1`:

| Command | Result | Time (pytest) |
|---|---|---|
| `pytest -m unit` | 16 passed | 0.03 s |
| `pytest -m integration` | 46 passed (40 from `payloads/malformed/`) | 1.69 s |
| `pytest -m "behavioural and not slow"` | 5 passed | 0.74 s |
| `pytest -m slow` (golden file, 5,000 rows) | 1 passed | 7.1 s |
| `pytest -m "not slow" --cov` (per-commit gate) | 67 passed, 100% coverage | 1.34 s (3.0 s wall) |

Planting `hour <= 6` again was caught by `tests/unit/test_features.py::test_night_flag_boundaries[6-0]`
(unit, ~1 s) and by `test_golden_scores` (slow behavioural).
