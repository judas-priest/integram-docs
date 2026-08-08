# Module: timeseries

**Path:** `src/api/v2/modules/timeseries/`
**Files:** `router.js`, `service.js`
**Base URL:** `/api/v2/:db/timeseries/...`
**Auth:** JWT required (`viewer` for reads, `editor` for writes). Note: JWT is implicit via `requireRole` which also checks JWT.

## Purpose

Universal time-series data attached to EAV objects. A "source" is any EAV object (e.g. a machine, sensor, or KPI tracker). Each point has a `source_id`, `metric` name, `value` (numeric) or `text_val`, and a timestamp.

## Endpoints

| Method | Path | Role | Description |
|--------|------|------|-------------|
| POST | `/timeseries` | editor | Ingest one point or array of points |
| GET | `/timeseries` | viewer | Query aggregated time series |
| GET | `/timeseries/sources` | viewer | List all sources with last value |

## Ingest Format

```json
{ "source_id": 42, "metric": "temperature", "value": 23.5 }
```

Or batch:
```json
[
  { "source_id": 42, "metric": "temperature", "value": 23.5 },
  { "source_id": 42, "metric": "humidity", "value": 65, "ts": "2025-01-01T12:00:00Z" }
]
```

Required fields: `source_id`, `metric`, and either `value` or `text_val`. `ts` defaults to server time.

## Query Parameters

| Param | Description |
|-------|-------------|
| `source_id` | Required. EAV object ID |
| `metric` | Required. Metric name |
| `from` | ISO datetime start |
| `to` | ISO datetime end |
| `bucket` | Time bucket for aggregation (e.g. `1 hour`, `1 day`) |
| `agg` | Aggregation function: `avg`, `sum`, `min`, `max`, `count` |
| `limit` | Max rows returned |

## TimescaleDB

Uses TimescaleDB hypertable when available (automatic time-based partitioning, `time_bucket` aggregation). Falls back to a regular PostgreSQL table if TimescaleDB extension is not installed.

## Automation Integration

`on_metric_threshold` and `on_metric_silence` automation triggers query this module to evaluate conditions.

## DB Tables

- `_v2_timeseries` (per-workspace, lazy-init) — `source_id`, `metric`, `value` (float8), `text_val`, `ts` (timestamptz). Hypertable if TimescaleDB available.
