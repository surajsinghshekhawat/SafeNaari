# SafeNaari — system context

This document describes the repository as it exists in source at commit `73d3d76` (`chore: ignore local papers and templates`) on branch history that ends at `main`. It was written by reading the files in the tree. It does not restore deleted documents, and it does not describe behavior that is not in the code.

Earlier commits removed several markdown files (`FULL_PROJECT_REFERENCE.md`, `IMPLEMENTATION_TODO_DETAILED.md`, `RESEARCH_REPORT_CONTEXT.md`, `RESEARCH_READINESS_CHECKLIST.md`, `PROJECT_OVERVIEW_FOR_PPT.md`, `developer.md`, and a `docs/` directory). Those files are not in the current tree, so their contents are not summarized here.

Root `README.md` is a single line: `# SafeNaari`.

The product name used in the mobile UI is **SafeNaari**. Package descriptions, the Express process log, the FastAPI title, and the admin sidebar still say **Women Safety Analytics**.

---

## 1. What the running system is

Four processes are intended to work together:

| Piece | Path | Stack | Default listen |
| --- | --- | --- | --- |
| Mobile app | `frontend/mobile` | Expo SDK 54, React Native 0.81.5, React 19.1.0 | Expo dev server |
| Admin web app | `frontend/admin` | Vite 7, React 19.2.4, React Router 7 | `http://localhost:5174` |
| HTTP + WebSocket API | `backend/api` | Express 5, Socket.IO 4, TypeScript 5.9 | `0.0.0.0:3001` |
| ML / data service | `backend/ml` | FastAPI 0.115, Uvicorn, scikit-learn, psycopg2 | `0.0.0.0:8000` |
| Database | configured in `backend/ml/app/config.py` | PostgreSQL, with PostGIS functions used by the storage layer | host `localhost`, port `5433`, database `women_safety_db`, user `postgres` |

There is no Docker Compose file, no CI workflow, and no `.env` example committed. Configuration is environment variables plus hardcoded fallbacks.

A development machine LAN address, `192.168.1.12`, is hardcoded as the default host for:

- the mobile API base URL in development (`frontend/mobile/src/services/api.ts`)
- the ML service URL fallback (`backend/api/src/services/mlService.ts` and `backend/api/src/routes/reports.ts`)
- the admin Vite proxy target (`frontend/admin/vite.config.ts`)
- the Express startup log lines (`backend/api/src/app.ts`)
- the Node integration scripts (`backend/api/test-integration.js`, `backend/api/test-full-system.js`)

Production mobile URL in the same file is the literal string `https://api.womensafety.com`. Nothing in the repo shows that host is deployed.

---

## 2. Request path

```
Mobile app  --HTTP-->  Express :3001  --HTTP-->  FastAPI :8000  --SQL-->  PostgreSQL
Admin app   --HTTP /api (Vite proxy)--> Express :3001 --HTTP--> FastAPI
Mobile app  --Socket.IO--> Express :3001
```

Express does not talk to PostgreSQL. All incident persistence is in the Python service.

Socket.IO is attached to the same HTTP server as Express. `app.locals.io` is set so route handlers can emit events.

CORS is `origin: "*"` on both servers. Helmet is enabled on Express. JSON body limit is `10mb`. Morgan request logging runs when `NODE_ENV` is not `production`.

Admin routes are mounted as `app.use("/api/admin", requireAdmin, adminRoutes)`. Every `/api/admin/*` call requires `Authorization: Bearer <jwt>` with `role === "admin"`. Mobile routes (`/api/panic`, `/api/location`, `/api/reports`, `/api/auth/register`, `/api/auth/login`) do not check a token.

---

## 3. Data that is actually stored

The live table is `incidents`. The schema files also create `unsafe_zones`, but no Python or TypeScript file inserts into `unsafe_zones`. Clusters are computed in memory and cached in `backend/ml/app/ml/clustering.py`.

### 3.1 `incidents` columns

From `backend/ml/migrations/001_initial_schema.sql`, plus later migrations:

| Column | Source | Notes |
| --- | --- | --- |
| `id` | client-generated string | Primary key. Panic ids look like `panic_<ms>`. Report ids look like `report_<ms>_<userId>`. Mock ids look like `chennai_hr_0`. |
| `latitude`, `longitude` | request | `DECIMAL(10,8)` / `DECIMAL(11,8)` |
| `location` | computed on insert | PostGIS `GEOGRAPHY(POINT, 4326)` via `ST_SetSRID(ST_MakePoint(lng, lat), 4326)::geography`. Present only in the PostGIS schema, not in `001_initial_schema_no_postgis.sql`. |
| `timestamp` | request | `TIMESTAMPTZ` |
| `timezone_offset_minutes` | optional client field | Minutes east of UTC. IST is `+330`. Added by migration 002. |
| `incident_local_hour` | computed on insert | `0–23`. If an offset is present, hour of `timestamp + offset`. Otherwise the timestamp's own hour. |
| `type` | request | Check constraint: `panic_alert` or `community_report` |
| `severity` | request | Integer `1–5` |
| `category` | optional | `VARCHAR(100)`. Free string. |
| `verified` | request, later updated by admin | Boolean, default false. Panic alerts are sent as `verified: true`. Community reports are sent as `verified: false`. |
| `moderation_reason` | admin verify/reject | `VARCHAR(255)`. Migration 003. |
| `user_id` | request | Optional string. Mobile hardcodes `user123`. |
| `created_at`, `updated_at` | database defaults | Triggers in the initial SQL update `updated_at` on UPDATE. |

There is no column for report description, photo, video, emergency-contact phone numbers, or a continuous location trail.

### 3.2 `unsafe_zones` (schema only)

Columns: `id`, `center_lat`, `center_lng`, `center_location` (PostGIS version only), `radius_meters`, `risk_score` (`0–5`), `incident_count`, `last_incident_at`, timestamps. Indexes exist. Nothing in `app/db/storage.py` writes this table.

### 3.3 Migrations

| File | What it does |
| --- | --- |
| `backend/ml/migrations/001_initial_schema.sql` | PostGIS extension, `incidents`, `unsafe_zones`, GIST indexes, `updated_at` triggers. Already includes the time-of-day columns. |
| `backend/ml/migrations/001_initial_schema_no_postgis.sql` | Same tables without `GEOGRAPHY` and without PostGIS. Lat/lng B-tree indexes instead of GIST. |
| `backend/ml/migrations/002_time_of_day_columns.sql` | Idempotent `ADD COLUMN IF NOT EXISTS` for `timezone_offset_minutes` and `incident_local_hour`, plus an index. |
| `backend/ml/migrations/003_moderation.sql` | `moderation_reason VARCHAR(255)`, index `idx_incidents_verified`. |

`backend/ml/run_migration_002.py` and `run_migration_003.py` apply those `ALTER`s through psycopg2 using `app.config.settings`. They do not read the `.sql` files; the SQL is duplicated in the Python.

`backend/ml/setup-database.ps1` is a Windows helper. It looks for `psql.exe` under PostgreSQL 14/15/16, creates database `women_safety_db` on port `5433`, runs `CREATE EXTENSION IF NOT EXISTS postgis`, and then tells the operator to run the migration SQL.

The insert path in `app/db/storage.py` always calls `ST_MakePoint` / `ST_DWithin`. A database created from the non-PostGIS schema cannot serve the current storage code.

---

## 4. ML service (`backend/ml`)

Entry point: `backend/ml/app/main.py`.

- FastAPI app titled `Women Safety Analytics - ML Service`, version `1.0.0`.
- Docs at `/docs` and `/redoc`.
- Router included at prefix `/ml`.
- Startup calls `init_connection_pool()`. Shutdown calls `close_connection_pool()`. These use the FastAPI `@app.on_event` hooks.
- `GET /` returns service name, version, status `running`.
- `GET /health` returns status `healthy`. This is separate from `GET /ml/health`.

Run as written in `main.py`: `uvicorn app.main:app` on `settings.host` / `settings.port` with reload.

### 4.1 Settings (`backend/ml/app/config.py`)

`pydantic_settings.BaseSettings`, env file `.env`, case-insensitive.

| Setting | Default in code | Used by |
| --- | --- | --- |
| `port` | `8000` | uvicorn |
| `host` | `0.0.0.0` | uvicorn |
| `log_level` | `INFO` | uvicorn |
| `model_path` | `./models` | declared only; no loader reads it |
| `cache_ttl` | `300` | declared only |
| `dbscan_eps` | `0.001` | DBSCAN epsilon in degrees. Comment in code: about 100 meters. |
| `dbscan_min_samples` | `3` | DBSCAN |
| `weight_incident_density` | `0.5` | risk formula |
| `weight_recency` | `0.25` | risk formula |
| `weight_severity` | `0.15` | risk formula |
| `weight_time_pattern` | `0.1` | risk formula |
| `max_expected_effective_incidents` | `10.0` | density log normalization |
| `recency_decay_days` | `30.0` | exponential decay |
| `time_of_day_sigma_hours` | `3.0` | Gaussian hour similarity |
| `time_of_day_min_weight` | `0.05` | floor on that Gaussian |
| `default_grid_size` | `100` meters | declared; heatmap callers pass grid size explicitly |
| `default_radius` | `1000` meters | declared |
| `land_mask_enabled` | `False` | `is_point_allowed` returns true immediately when false |
| `land_mask_geojson_path` | `resources/landmask/chennai_wards.geojson` | resolved relative to `app/` |
| `land_mask_source_url` | Datameet Chennai wards GeoJSON on GitHub | used only if the local file is missing |
| `auto_retrain_threshold` | `0.1` | declared only |
| `min_retrain_interval` | `3600` | declared only |
| `db_host` | `localhost` | psycopg2 |
| `db_port` | `5433` | psycopg2 |
| `db_name` | `women_safety_db` | psycopg2 |
| `db_user` | `postgres` | psycopg2 |
| `db_password` | `""` | psycopg2 |
| `db_ssl` | `False` | `sslmode` is `require` when true, otherwise `prefer` |

The four weights sum to `1.0`. The final score multiplies the weighted sum by `5.0` and clamps to `[0, 5]`.

### 4.2 Connection pool (`backend/ml/app/db/connection.py`)

`psycopg2.pool.ThreadedConnectionPool`, `minconn=1`, `maxconn=10`. `get_db_connection()` is a context manager: commit on success, rollback on exception, connection returned to the pool. `test_connection()` runs `SELECT 1`.

SQLAlchemy is listed in `requirements.txt` and is not imported by any Python file in the repo.

### 4.3 Storage functions (`backend/ml/app/db/storage.py`)

| Function | Behavior |
| --- | --- |
| `add_incident` | `INSERT` including PostGIS point and precomputed `incident_local_hour`. |
| `get_incidents` | Optional lat/lng bounding box and time range. `ORDER BY timestamp DESC`. Returns dicts including `moderation_reason`. |
| `get_incidents_in_radius` | `ST_DWithin(location, ST_MakePoint(lng, lat)::geography, radius_meters)`. |
| `get_incident_count` | `SELECT COUNT(*) FROM incidents`. Returns `0` on error. |
| `get_all_incidents` | alias of `get_incidents()` with no filters. |
| `update_incident_verification(id, verified, reason)` | `UPDATE` `verified`, `moderation_reason`, `updated_at`. Returns whether `rowcount > 0`. |
| `clear_incidents` | `TRUNCATE TABLE incidents RESTART IDENTITY CASCADE`. Used by `step2_load_data.py`. |

`backend/ml/app/data/storage.py` is an older in-memory list (`_incidents`). The API routes import `app.db.storage`, not this module. Nothing in the live request path calls the in-memory module.

### 4.4 HTTP routes (`backend/ml/app/api/routes.py`)

| Method and path | Handler | What it does |
| --- | --- | --- |
| `POST /ml/incidents/process` | `process_incident` | Validates `IncidentRequest`, `add_incident`, then `update_clusters_if_needed`. Response `model_updated` is always `False`. |
| `GET /ml/heatmap` | `get_heatmap` | Query: `lat`, `lng`, `radius` (100–50000, default 1000), `grid_size` (50–1000, default 100), optional `timestamp`, `local_hour` 0–23, `timezone_offset_minutes` −840–840, `include_time_factor` (accepted, not read inside `generate_heatmap`), `include_clusters` (default false). |
| `GET /ml/risk-score` | `get_risk_score` | `calculate_risk_score(lat, lng, local_hour=...)`. |
| `POST /ml/routes/analyze` | `analyze_routes` | Calls `app.ml.route_analyzer.analyze_routes`. Recommended route is the id with the highest `safety_score`. |
| `POST /ml/models/train` | `train_models` | Calls `app.ml.models.train_models`. That function does not fit a model. |
| `GET /ml/incidents/all` | `get_all_incidents` | Loads every incident, filters in Python by `verified` and `type`, then slices `[offset:offset+limit]`. `limit` max 1000, default 200. |
| `PUT /ml/incidents/{id}/verify` | `verify_incident` | `update_incident_verification(id, True, reason)`. 404 if no row. |
| `PUT /ml/incidents/{id}/reject` | `reject_incident` | `update_incident_verification(id, False, reason or "rejected")`. 404 if no row. |
| `GET /ml/health` | `ml_health` | `models_loaded` from `get_model_status()` (hardcoded `True`), `last_training` is `datetime.utcnow()` at call time, plus incident count. |

`timestamp` query parsing imports `dateutil.parser` inside a `try`. `python-dateutil` is not in `requirements.txt`. If the import or parse fails, the handler uses `datetime.now(timezone.utc)` and still uses `local_hour` when the client sent it.

Pydantic models are in `backend/ml/app/api/schemas.py`. `IncidentRequest.type` must match `panic_alert|community_report`. `RouteRequest.waypoints` requires at least 2 points. `HeatmapResponse.heatmap` is an untyped `dict`.

### 4.5 Risk score (`backend/ml/app/ml/risk_scoring.py`)

Public function: `calculate_risk_score(lat, lng, query_timestamp=None, local_hour=None)`.

It loads incidents within 1000 meters via `get_incidents_in_radius`, then calls `_calculate_risk_score_from_incidents`.

If there are no incidents inside the radius, the score is `time_factor * 1.5` and the level is `safe` when that value is `< 2.0`, otherwise `medium`. Density, recency, and severity factors are `0`.

Time-of-day factor from the query's local hour (`calculate_time_of_day_factor`):

| Local hour | Factor | Comment in code |
| --- | --- | --- |
| 21–23 or 0–3 | `0.9` | night |
| 18–20 | `0.7` | evening |
| 4–7 | `0.6` | early morning |
| 8–17 | `0.3` | daytime |

With no incidents, night therefore scores `0.9 * 1.5 = 1.35` (`safe`), and evening scores `1.05` (`safe`). The empty-area level only becomes `medium` if the factor times 1.5 is at least 2, which none of these factors reach. The code still contains the branch.

For each nearby incident the scorer computes:

1. Haversine distance in meters (`app.utils.geospatial.calculate_distance_haversine`, Earth radius `6371000`).
2. Distance weight `exp(-distance_meters / 111.0)`.
3. Age in days from the query timestamp (default now, UTC).
4. Recency weight `exp(-days_ago / recency_decay_days)` with decay at least 1 day. Default 30 days.
5. Incident local hour: `incident_local_hour` if present and in 0–23, else timestamp plus `timezone_offset_minutes`, else the timestamp hour.
6. Time-of-day similarity: circular hour distance on a 24-hour clock, then `exp(-(dh^2) / (2 * sigma^2))` with `sigma` at least `0.25` (default 3 hours), floored by `time_of_day_min_weight` (default `0.05`) and capped at 1.
7. Combined weight `distance_weight * recency_weight * time_weight`.
8. `effective_incident_count` accumulates `recency_weight * time_weight` (distance is not in this sum).

Aggregates:

- Density: `log(effective_count + 1) / log(max_expected + 1)`, clamped to `[0, 1]`. Default `max_expected` is 10.
- Average recency: distance-and-time-weighted mean of recency weights.
- Average severity: weighted mean of severity, then divided by 5 so it is in `[0, 1]`.

A second time feature buckets nearby incidents into night `[21,22,23,0,1,2,3,4]`, evening `[18,19,20]`, afternoon `[12–17]`, morning `[6–11]`. Note that hour 4 is in the night bucket here, while `calculate_time_of_day_factor` treats hour 4 as early morning (`0.6`). The dominant bucket is the one with the largest count. If the query hour falls in that dominant bucket, `location_time_pattern` is raised according to the bucket's share of incidents (`>0.5`, `>0.3`, or any match). Otherwise it is `current_time_factor * 0.6`. The factor stored as `time_pattern` is `0.7 * location_time_pattern + 0.3 * current_time_factor`. If there are no parseable hours, `time_pattern` is just the current time factor.

Final:

```
risk = (
  0.5 * density
  + 0.25 * avg_recency
  + 0.15 * avg_severity
  + 0.1 * time_pattern
) * 5
```

Clamped to `[0, 5]`, rounded to 2 decimals.

Levels:

| Score | `risk_level` |
| --- | --- |
| `<= 1.0` | `very_safe` |
| `<= 2.0` | `safe` |
| `<= 3.0` | `medium` |
| `<= 4.0` | `high` |
| else | `very_high` |

The response `factors` object includes `incident_density`, `recency`, `severity`, `time_pattern`, `current_time_factor`, `raw_incident_count`, `effective_incident_count`. The single-location function does not attach `nearest_cluster`. The route schema allows `nearest_cluster`, and the handler reads `risk_data.get("nearest_cluster")`, which is absent, so the field is null.

### 4.6 Heatmap (`backend/ml/app/ml/heatmap.py`)

`generate_heatmap(center_lat, center_lng, radius_meters, grid_size_meters, query_timestamp, local_hour, include_clusters)`.

It does not score a full rectangular grid. The comment in the function calls this "option B": cells exist only where incidents exist.

Steps:

1. Convert meters to degrees with `meters / 111000`.
2. Build a bounding box of `± radius` in both latitude and longitude (longitude is not scaled by `cos(latitude)`).
3. If the estimated full-grid cell count would exceed `3000`, increase cell size so the estimate stays at or under 3000. This adjustment changes `grid_size_degrees` used for binning. The `grid_size` value returned to the client stays the requested meter size.
4. `get_incidents_in_radius(center, radius)` for bins that should appear.
5. `get_incidents_in_radius(center, radius + 1000)` as neighborhood context.
6. Bin both sets by integer cell index from the bounding box.
7. For each view bin, the cell coordinate is the mean latitude and mean longitude of incidents in that bin, not the geometric cell center.
8. Drop the cell if `is_point_allowed` is false (no-op while land mask is disabled).
9. Collect incidents from neighboring bins. Neighbor range is `ceil(1000 / grid_size_meters)`.
10. Keep only those within 1000 m of the cell centroid.
11. Score that set with `_calculate_risk_score_from_incidents`.
12. `incident_count` on the cell is the count in the bin, not the neighborhood count.
13. Stop after `max_cells` (3000).

If the view contains no incidents, `cells` is an empty list. A previous fallback that placed a cell at the viewport center was removed; the comment says that fallback produced a circle that followed pan and zoom.

On an exception, the function returns one cell at the center with risk `0` and level `very_safe`.

Clusters are attached only when `include_clusters` is true. They are not filtered to the requested radius; `get_clusters()` returns every cluster from all incidents.

### 4.7 Land mask (`backend/ml/app/utils/land_mask.py`)

Purpose, from the module docstring: keep heatmap cells inside a polygon so water is not drawn.

`land_mask_enabled` defaults to `False`, so `is_point_allowed` returns `True` and the GeoJSON is not consulted.

When enabled:

- Load `app/resources/landmask/chennai_wards.geojson` (resolved from `app/utils/../resources/...`).
- The file in the repo is a GeoJSON `FeatureCollection` with 201 features, about 565521 bytes.
- If the file is missing, download `land_mask_source_url` and write it.
- Point-in-polygon is a ray-cast, with a per-polygon bounding box check, and holes excluded.
- A point is allowed if it lies in any polygon.
- If loading fails, the mask fails open (returns true) so heatmap generation continues.

The module docstring says Shapely `unary_union` is intentionally not used because of a Shapely/NumPy incompatibility in the environment they were running. `shapely` is still listed in `requirements.txt` and is not imported.

### 4.8 Clustering (`backend/ml/app/ml/clustering.py`)

`update_clusters_if_needed` sets the in-process cache to `None` and returns `[]`. It does not recompute clusters and does not return affected zone ids. `ProcessIncidentResponse.affected_zones` is therefore always empty, and `model_updated` is always false.

`get_clusters(force_recalculate=False)`:

- Returns the cache if present.
- If incident count `< dbscan_min_samples` (3), returns `[]`.
- Builds a NumPy array of `[latitude, longitude]`.
- `sklearn.cluster.DBSCAN(eps=0.001, min_samples=3, metric="euclidean")` on degree coordinates. Euclidean distance in degrees is not meters. The eps comment says "~100 meters".
- Label `-1` (noise) is discarded.
- Cluster center is the mean lat/lng of member points.
- Radius is the max Euclidean distance in degrees from that center, times `111000`, reported as meters.
- Risk score is `calculate_risk_score(center_lat, center_lng)` with no `local_hour`, so the hour is the server's current hour.
- Id format: `cluster_<dbscan label>`.
- Result is cached in module globals until the next `update_clusters_if_needed` or `force_recalculate=True`.

The cache is per process. A new incident invalidates it only in the worker that handled `POST /ml/incidents/process`.

### 4.9 Route analysis (`backend/ml/app/ml/route_analyzer.py`)

`analyze_routes` walks each route's consecutive waypoints.

For each segment:

- Haversine length (same formula as geospatial utils, duplicated in this file).
- Risk is `calculate_risk_score` at the arithmetic midpoint. That call hits the database for incidents within 1000 m of the midpoint. A route with many vertices issues many queries. `local_hour` is not passed.
- Segments with risk `>= 3.5` are appended to `high_risk_segments`.

Route aggregates:

- `risk_score` = distance-weighted average of segment risks.
- `safety_score` = `max(0, 1 - avg_risk / 5)`, rounded to 3 decimals.
- `total_distance` = sum of segment lengths.
- `safe_distance` is set equal to `total_distance`. The comment in the function says "Simplified for now". It is not the length of segments under a risk threshold.

The recommended route is chosen in the API handler as the maximum `safety_score`, not in this module.

### 4.10 Model training (`backend/ml/app/ml/models.py`)

`get_model_status` returns `loaded: True` and `last_training` equal to the current UTC time. The comment says the loaded flag is a TODO.

`train_models` reads the incident count and returns `started: True`, `estimated_time: "5 minutes"`, and the count. The comment says actual training is a TODO. No joblib file is written. `joblib` is in `requirements.txt` and is not imported.

### 4.11 Mock data

`backend/ml/app/data/chennai_mock_data.py` builds synthetic `IncidentRequest` objects and inserts them with `app.db.storage.add_incident`.

The generator's own comments say the named areas are assumptions for test data and that scoring code does not receive those area names. That matches the code: risk scoring only sees stored incidents.

Fixed places used only by the generator:

- City center labeled Marina Beach: `13.0475, 80.2825`.
- Bounds: north `13.20`, south `12.85`, east `80.35`, west `80.10`.
- High-risk named points: Central Railway Station `13.0827, 80.2707` (radius 0.5 km, base severity 4), Egmore `13.0790, 80.2606`, Marina Beach night area, Koyambedu `13.0710, 80.1980`, T. Nagar `13.0418, 80.2341`.
- Medium: Anna Nagar, OMR IT Corridor, Guindy.
- Low: Adyar Residential, Besant Nagar.
- Hour lists: high `[22, 23, 0, 1, 2, 3]`, medium `[18, 19, 20, 21]`, low `6–17`.
- `HIGH_RISK_DAYS = [4, 5]` is defined and not referenced by any function in the file.

Distribution of a generated batch: 40% high-risk areas, 35% medium, the rest low. Each point is a random offset inside the area radius, then clamped to the city bounds. Timestamps span `start_date_days_ago` (default 90) with an hour chosen from the risk-level weights in `_generate_risk_aware_hour`. Severity is derived from `base_severity`, a time multiplier (`1.2` / `1.0` / `0.8`), noise, and extra resampling so high-risk areas lean toward 3–5 and low-risk areas lean toward 1–3. Type is `community_report` for severity under 3; severity 3 is 20% panic; severity 4–5 is 40% panic. Categories are `harassment`, `assault`, `suspicious`, or `other`. About 60% are `verified=True`. `user_id` is `user_1` through `user_200`. The generator does not set `timezone_offset_minutes`.

`load_chennai_mock_data_into_storage(count=2000)` inserts the batch. `step2_load_data.py` truncates the table first and loads 5000. The step 3, 4, and 5 scripts and `test_risk_optimization.py` load 5000 only when the table is empty. They do not truncate when rows already exist.

`backend/ml/app/data/mock_data.py` is a separate generator centered on `40.7128, -74.0060` with a 5 km radius and three synthetic clusters. `load_mock_data_into_storage` also inserts through `app.db.storage`. The step scripts do not call it.

### 4.12 Python scripts that are not the server

| File | Role |
| --- | --- |
| `step2_load_data.py` | Truncate incidents, insert 5000 Chennai mock rows, print a sample. |
| `step3_test_ml_algorithms.py` | Print DBSCAN clusters and risk scores for fixed Chennai coordinates. Loads 5000 rows if the table is empty. |
| `step4_test_heatmap.py` | Calls `generate_heatmap` for named Chennai locations. |
| `step5_test_routes.py` | Builds `RouteRequest` polylines and calls `analyze_routes`. |
| `test_db_connection.py` | Pool, `SELECT` count, and an insert. |
| `test_full_integration.py` | Connection, count, clusters, risk, heatmap, routes. Exits docstring says it tests the backend API; the body only imports Python ML modules. |
| `test_mock_data.py` | Generates Chennai incidents in memory and prints samples. Does not require the database. |
| `test_risk_optimization.py` | Prints risk scores for six named coordinates. |
| `run_migration_002.py`, `run_migration_003.py` | Schema alters. |

There is no `pytest` test module. `pytest` and `pytest-asyncio` are in `requirements.txt` only.

`requirements.txt` pins: fastapi 0.115.0, uvicorn 0.32.0, pydantic 2.9.2, pydantic-settings 2.5.2, scikit-learn 1.5.2, numpy 2.1.1, pandas 2.2.3, geopy 2.4.1, shapely 2.0.5, httpx 0.27.2, cachetools 5.4.0, joblib 1.4.2, pytest 8.3.3, pytest-asyncio 0.24.0, black 24.10.0, flake8 7.1.1, python-dotenv 1.0.1, psycopg2-binary 2.9.9, sqlalchemy 2.0.35.

Imports actually used by application code: FastAPI, pydantic, pydantic-settings, sklearn `DBSCAN`, numpy, httpx (land-mask download), psycopg2. pandas, geopy, shapely, cachetools, joblib, and sqlalchemy are not imported.

---

## 5. Express API (`backend/api`)

`package.json` name: `women-safety-analytics-api`, version `1.0.0`, MIT, main `dist/app.js`.

Scripts: `build` = `tsc`, `start` = `node dist/app.js`, `dev` = `nodemon src/app.ts`, `test` = a placeholder that exits 1.

TypeScript (`tsconfig.json`): target ES2020, module commonjs, `strict`, `noImplicitAny`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `outDir` `dist`. `noImplicitReturns` is false.

Dependencies: axios, cors, dotenv, express 5, firebase-admin, helmet, jsonwebtoken, morgan, socket.io. `firebase-admin` is not imported anywhere under `backend/api/src`.

### 5.1 Process (`src/app.ts`)

- Port `process.env.PORT` or `3001`, listen `0.0.0.0`.
- Socket.IO: CORS `*`, transports websocket and polling, `allowEIO3: true`, `pingTimeout` 60000, `pingInterval` 25000, `connectTimeout` 45000.
- `GET /health` returns status `OK`, service name `Women Safety Analytics API`, version `1.0.0`.
- Mounts: `/api/auth`, `/api/panic`, `/api/location`, `/api/admin` (with `requireAdmin`), `/api/reports`.
- Unknown paths return 404 JSON `{ error, path }`.
- Error handler returns `err.message` unless `NODE_ENV === "production"`, in which case the message is `Internal server error`.

### 5.2 Admin JWT (`src/middleware/auth.ts`)

`JWT_SECRET` is `process.env.JWT_SECRET` or `process.env.ADMIN_SECRET` or the string `change-me-in-production`.

`signAdminToken` signs `{ sub, role }` with expiry `7d`.

`requireAdmin` requires `Authorization: Bearer ...`, verifies the token, and rejects any role other than `admin` with 403. The decoded payload is stored on `req.admin`.

### 5.3 Auth routes (`src/routes/auth.ts`)

| Route | Implemented behavior |
| --- | --- |
| `POST /api/auth/admin-login` | Body `email` or `username`, plus `password`. Succeeds when the id equals `ADMIN_EMAIL` (default `admin@womensafety.local`) and the password equals `ADMIN_PASSWORD` or `ADMIN_SECRET` or `admin`, or when the id is the string `admin` and the password equals `ADMIN_SECRET` or that same password. Returns a real JWT, `user: { id: "admin", email: loginId, role: "admin" }`, `expiresIn: "7d"`. There is one admin. No user table. |
| `POST /api/auth/register` | Validates email, password, name. Does not hash, store, or check duplicates. Returns `user_<timestamp>`. Comments list the missing steps. |
| `POST /api/auth/login` | Validates email and password. Returns the literal token `mock_jwt_token_here` and a fixed user `user_123` / `Test User` / `+1234567890`. |
| `GET /api/auth/profile/:userId` | Returns a fixed profile for whatever id was in the path, including two fake emergency contacts. |
| `PUT /api/auth/emergency-contacts` | Requires `userId` and `emergencyContacts`. Logs the count and echoes the list. Does not store it. |

The mobile app does not call register, login, profile, or this emergency-contact endpoint. Contacts live in on-device storage.

### 5.4 Panic (`src/routes/panic.ts`)

`POST /api/panic/trigger` requires `userId`, `location`, `emergencyContacts`. It builds an incident id `panic_<Date.now()>`, type `panic_alert`, severity `5`, category `emergency`, `verified: true`, and calls `processIncident`. Optional `timezone_offset_minutes` is forwarded. If the ML call throws, the handler still returns HTTP 200. `processIncident` itself catches ML failures and returns `{ success: false, ... }` rather than throwing, so the route's catch is a second layer.

The JSON `actions` object always reports `contactsNotified: emergencyContacts.length`, `authoritiesAlerted: true`, and `locationTrackingStarted: true`. The handler does not send SMS, push, email, or a call, and it does not start tracking. Those three fields are constants in the response, not the result of a notification provider.

`POST /api/panic/cancel` checks `userId` and `panicId`, logs, and returns success. It does not update the database. The comment lists the missing steps.

`GET /api/panic/status/:userId` always returns `hasActivePanic: false` and `lastPanicTime: null`.

### 5.5 Reports (`src/routes/reports.ts`)

`POST /api/reports/submit` requires `userId`, `type`, `category`, `description`, `location.latitude`, `location.longitude`. Severity, if present, must be 1–5; default 3.

The ML payload does not include `description`. The description is validated and then dropped. Media is not part of the schema.

Type is forced to `community_report` unless the body type is exactly `panic_alert`. Id is `report_<ms>_<userId>`. `verified` is false.

On ML success the handler calls `emitNewIncident` with id, coordinates, type, severity, and timestamp. Then it returns `reportId` and `mlProcessed`.

If `processIncident` throws, the route still returns HTTP 200 with `warning: "ML service temporarily unavailable"`. As with panic, `processIncident` usually returns a fallback object instead of throwing.

`GET /api/reports/user/:userId` returns `reports: []`. The comment says a database fetch is not implemented. The `userId` path parameter is unused.

`GET /api/reports/all` fetches `GET {ML_SERVICE_URL}/ml/incidents/all` with the global `fetch` (Node's fetch, not axios). Default ML URL `http://192.168.1.12:8000`. It keeps rows whose `type === "community_report"`. The description sent to the phone is not the text the user typed. It is `` `${category} incident reported` `` when `category` is set, otherwise a sentence containing the timestamp. Panic alerts are counted in a log line and omitted from the list. On ML failure the handler still returns HTTP 200 with an empty list.

### 5.6 Location (`src/routes/location.ts`)

`POST /api/location/update` requires `userId`, `latitude`, `longitude`. Rejects coordinates outside world bounds. Calls `getRiskScore`. `isHighRisk` is `risk_score >= 4.0`. The location is not written anywhere. The comments say storage and high-risk alerts are not implemented. `timezone_offset_minutes` may be present in the mobile body; this handler does not read it and does not pass `local_hour` to `getRiskScore`.

`GET /api/location/heatmap` requires `lat` and `lng`. Optional `radius` (default 1000), `grid_size` (default 100), `timestamp`, `local_hour` (default the server's `Date.getHours()`), `timezone_offset_minutes`. Invalid `local_hour` returns 400. It calls `getHeatmap(..., includeClusters=false)`. If the ML client returns `success: false`, the handler still returns HTTP 200 with an empty cell list and a warning string. The response adds `next_update_recommended` 30 seconds in the future and `metadata.time_based` / `considers_recent_reports`, both hardcoded true.

`GET /api/location/safe-routes` requires `startLat`, `startLng`, `endLat`, `endLng`. It calls Google Routes via `getRouteOptions(..., alternatives=true)`, then `analyzeRoutes` on the ML service. If ML returns `success: false`, this route returns HTTP 503 (unlike heatmap, which returns an empty 200). Each analyzed route is joined back to the Google option by `id` so the response can include waypoints, duration, and turn instructions.

`GET /api/location/search-places` requires `query`. Optional `lat` and `lng` are passed as a location bias. Returns `{ places }`.

`GET /api/location/place-coordinates` requires `placeId`. 404 when geocoding returns nothing.

### 5.7 Google Maps client (`src/services/googleMapsService.ts`)

`GOOGLE_MAPS_API_KEY` is `process.env.GOOGLE_MAPS_API_KEY` or `""`.

`getRouteOptions` POSTs to `https://routes.googleapis.com/directions/v2:computeRoutes` with:

- travel mode `DRIVE`
- `routingPreference: TRAFFIC_UNAWARE`
- `computeAlternativeRoutes` from the caller
- `polylineQuality: OVERVIEW`, `polylineEncoding: ENCODED_POLYLINE`
- `languageCode: en-IN`, `units: METRIC`
- header `X-Goog-Api-Key`
- field mask for duration, distance, encoded polyline, legs, localized values, and step navigation instructions
- timeout 15 seconds

Polylines are decoded with the standard Google encoded-polyline algorithm (1e-5). Route ids are `route_primary`, then `route_alternative_1`, and so on. Duration strings such as `165s` are parsed to seconds. If the HTTP call fails or returns no routes, the function returns one route `route_direct` whose waypoints are only the start and end points. It does not throw.

`searchPlaces` uses the legacy Places Autocomplete JSON endpoint. No `types` filter. If `location` is passed, `radius` is 50000 meters. Non-OK statuses return `[]`. The code logs a warning specifically for `REQUEST_DENIED`.

`getPlaceCoordinates` uses the Geocoding API with `place_id`.

### 5.8 ML client (`src/services/mlService.ts`)

Axios instance, base URL `ML_SERVICE_URL` or `http://192.168.1.12:8000`, timeout `ML_SERVICE_TIMEOUT` or 120000 ms.

| Function | Upstream | Fallback on error |
| --- | --- | --- |
| `checkMLServiceHealth` | `GET /ml/health` | `false` |
| `processIncident` | `POST /ml/incidents/process` | `{ success: false, affected_zones: [], model_updated: false }` |
| `getHeatmap` | `GET /ml/heatmap` with `include_time_factor: true` | empty cells and clusters, `success: false` |
| `getRiskScore` | `GET /ml/risk-score` | score `0`, level `very_safe` |
| `analyzeRoutes` | `POST /ml/routes/analyze` | empty routes, `recommended_route: null` |
| `getIncidentsAll` | `GET /ml/incidents/all` | empty list |
| `verifyIncident` | `PUT /ml/incidents/:id/verify` | rethrows on HTTP 404; otherwise `{ success: false }` |
| `rejectIncident` | `PUT /ml/incidents/:id/reject` | same pattern |

### 5.9 WebSocket (`src/websocket/socketHandler.ts`)

On connection:

- `subscribe:location` with `{ lat, lng, radius }` joins room `location:<lat to 4 decimals>:<lng to 4 decimals>:<radius>`.
- `unsubscribe:location` leaves that room.
- `subscribe:incidents` joins `incidents:all`.

`emitNewIncident` (called from report submit):

1. Emits `incident:new` to `incidents:all`.
2. Walks `io.of("/").adapter.rooms`. For every room id that parses as a location room, computes haversine distance from the room center to the incident. If the distance is `<=` the room radius, emits `incident:new` to that room too.

The payload's `timestamp` is replaced with `new Date().toISOString()` inside `emitNewIncident`, so the incident's original timestamp is not what Socket.IO clients receive.

`emitHeatmapUpdate` and `emitRiskAlert` are exported and are not called from any other file. Nothing emits `heatmap:update`. Nothing joins a `user:<id>` room, which is the room `emitRiskAlert` would target. The mobile client still listens for `heatmap:update`.

### 5.10 Admin routes (`src/routes/admin.ts`)

All of these sit behind `requireAdmin`.

`GET /api/admin/dashboard`

- Calls ML health and `getIncidentsAll({ limit: 100, offset: 0 })`.
- `totalIncidents` is `result.total` when present (the full table count).
- `incidentsToday`, `panicAlerts`, and `verifiedIncidents` are counted only on the returned page (up to 100 rows, newest first).
- `pendingVerification` is `total - verifiedCount`, mixing the full total with the page's verified count.
- `recentIncidents` is the first 10 of that page.

`GET /api/admin/incidents`

- Query `status`: `verified` sets `verified=true`, `pending` sets `verified=false`, anything else is unfiltered. Rejected rows are stored as `verified=false` plus a reason, so they match the pending filter.
- Query `type`: only `panic_alert` or `community_report` is forwarded.
- `limit` is capped at 200, default 50.
- Each row's `status` string is `verified` or `pending`. There is no `rejected` status string.

`POST /api/admin/incidents/:incidentId/verify` and `.../reject` read optional `reason` from the JSON body and proxy to the ML service. Success is 200. A thrown 404 becomes 404. Other failures are 502.

`GET /api/admin/heatmap` requires `lat`, `lng`. Default radius 3000, grid 100. Calls `getHeatmap` with `includeClusters=true`. Does not forward `timezone_offset_minutes` (the call passes `undefined`). `local_hour` defaults to the server hour.

`GET /api/admin/analytics`

- `period=30d` uses 30 days; any other value uses 7.
- Fetches up to 1000 incidents (the ML route's max) and filters by timestamp in memory. Incidents older than the page, or beyond 1000, are absent.
- Builds `incidentsByDay` for each calendar day in the window (UTC date strings from `toISOString().slice(0, 10)`), `bySeverity` for 1–5, `byCategory` (missing category becomes `uncategorized`), and `byHour` from `incident_local_hour` only (rows with a null hour are skipped).
- Verification rate is `verified / inRange.length * 100`, rounded to one decimal.

`GET /api/admin/audit`

- Takes the newest 100 incidents.
- Keeps rows where `moderation_reason != null` or `verified === true`.
- Caps at 50.
- `action` is `verified` when `verified` is true, otherwise `rejected`.
- `timestamp` is the incident timestamp, not an update time. There is no audit table. The route comment calls this a stub.

### 5.11 Node scripts

`backend/api/test-integration.js` and `test-full-system.js` are axios scripts aimed at `http://192.168.1.12:8000` and `http://192.168.1.12:3001`. They are not wired to `npm test`.

---

## 6. Mobile app (`frontend/mobile`)

Expo app. `app.json`: name and slug `mobile`, version `1.0.0`, portrait, new architecture enabled, iOS bundle id and Android package `com.womensafety.mobile`.

Permissions and usage strings:

- iOS location when-in-use and always, with the strings in `app.json`.
- Android: fine location, coarse location, camera, read and write external storage.
- `expo-image-picker` plugin strings for photos and camera.

`eas.json` defines `development`, `preview`, and `production` Android APK builds. `development` uses a dev client. `extra.eas.projectId` is `a0688f1d-8023-4486-9a61-22c5aa9fa70b`.

`app.json` contains a Google Maps API key literal for both `ios.config.googleMapsApiKey` and `android.config.googleMaps.apiKey`. The value was changed in commit `0c0b9e4`. It is not repeated in this document. The server-side Routes and Places calls use `GOOGLE_MAPS_API_KEY` from the environment and do not read `app.json`.

Entry: `index.ts` registers `App.tsx`, which renders `AppNavigator` only.

### 6.1 Navigation (`src/navigation/AppNavigator.tsx`)

Bottom tabs, headers hidden, bar height 64, active tint `colors.primary` (`#5B9BD5`).

| Tab name | Label | Screen | Icon name passed to `TabBarIcon` |
| --- | --- | --- | --- |
| `Home` | Home | `HomeScreen` | `home` |
| `Panic` | Panic | `PanicScreen` | `alert-circle` |
| `Reports` | Report | `ReportsScreen` | `document-text` |
| `Community` | Community | `CommunityReportsScreen` | `people` |
| `Routes` | Routes | `RoutePlanningScreen` | `map` |

`ContactsScreen` is not registered. Emergency contacts are edited inside `PanicScreen`.

### 6.2 Theme

`src/theme/colors.ts`: light background `#F5F7FA`, primary `#5B9BD5`, danger `#ef4444`, warning `#f59e0b`, success `#10b981`. Risk legend colors are translucent red, amber, yellow, and green.

`src/theme/spacing.ts`: `xs 4`, `sm 8`, `md 16`, `lg 24`, `xl 32`, `xxl 48`.

`src/components/AppIcons.tsx` draws SVG icons (shield, info, location, navigate, alert, call, trash, document, map, open, close, videocam) and the tab icons. It does not use `@expo/vector-icons` even though that package is a dependency. `AppNavigator`'s file comment mentions Ionicons; the implementation uses `TabBarIcon`.

### 6.3 API client (`src/services/api.ts`)

`API_BASE_URL` is `http://192.168.1.12:3001` when `__DEV__` is true, otherwise `https://api.womensafety.com`.

| Function | Call |
| --- | --- |
| `fetchHeatmap` | `GET /api/location/heatmap` with lat, lng, radius, grid_size, ISO timestamp, `local_hour` from `Date.getHours()`, and `timezone_offset_minutes` = `-getTimezoneOffset()` (JavaScript's offset is minutes behind UTC, so the negation yields minutes east of UTC). |
| `fetchCommunityReports` | `GET /api/reports/all` |
| `updateLocation` | `POST /api/location/update` with user id, coordinates, timestamp, timezone offset, optional accuracy |
| `triggerPanicAlert` | `POST /api/panic/trigger` |
| `searchPlaces` | `GET /api/location/search-places` |
| `getPlaceCoordinates` | `GET /api/location/place-coordinates` |
| `getSafeRoutes` | `GET /api/location/safe-routes` |

No auth header is sent.

### 6.4 WebSocket client (`src/services/websocket.ts`)

Loads `socket.io-client` with `require` because of Metro ESM resolution. `metro.config.js` adds `mjs` and `cjs` to `sourceExts`, sets `unstable_enablePackageExports`, and sets `resolverMainFields` to `react-native`, `browser`, `main`. The file comment says `require` is what avoids the ESM issue.

`initWebSocket` connects to `API_BASE_URL` with polling first, then upgrade to websocket, and reuses an existing socket. Connect errors are logged once. The app is written to keep working over HTTP if the socket fails.

Handlers are bound once per socket (`_wsaHandlersBound`):

- `heatmap:update` forwards to the latest heatmap callback. The server never emits this event.
- `incident:new` drops duplicate `incidentId`s seen within 10 seconds, and caps the dedup map at 200 entries.
- `subscribed` logs a room once.

`subscribeToLocation` emits `unsubscribe:location` for the previous room, then `subscribe:location`. `subscribeToIncidents` emits `subscribe:incidents`.

### 6.5 On-device storage (`src/utils/storage.ts`)

Tries to `require('@react-native-async-storage/async-storage')`. If that throws, get/set/remove use a process-memory object. Panic contacts use the key `@emergency_contacts`.

`expo-secure-store` is a plugin and a dependency. No screen imports it.

Firebase packages are in `package.json` (`@react-native-firebase/app`, `@react-native-firebase/auth`, `firebase`). No file under `frontend/mobile/src` imports them. The mobile README's "Next Steps" still lists "Integrate Firebase Auth" as unchecked. That matches the source.

### 6.6 Home (`src/screens/HomeScreen.tsx`)

Header title `SafeNaari`, subtitle `Your safety companion`. The info button's `onPress` is an empty function.

On Android and iOS it `require`s `HeatmapMap.android` or `HeatmapMap.ios`. If the require throws, or the map reports `onError`, it renders `HeatmapMapFallback`.

Props passed to the map: `radius={10000}`, `gridSize={200}`, `panToLocation` from route params.

The card under the map always shows the text `Connaught Place, New Delhi`. It does not read GPS or reverse-geocode. "Plan Safe Route" navigates to the `Routes` tab.

Default map center, used when the map component has no GPS yet, is `13.0827, 80.2707` (the same coordinates the Chennai mock generator labels Central Railway Station).

### 6.7 Android heatmap (`src/components/HeatmapMap.android.tsx`)

Uses `react-native-maps` with `PROVIDER_GOOGLE`.

Zoom behavior:

- The `radius` prop is treated as a maximum query radius (Home passes 10000).
- The `gridSize` prop is the coarsest grid (Home passes 200).
- On `onRegionChangeComplete`, the region is debounced 450 ms.
- Viewport radius is half the larger of width and height in meters, times 1.1, rounded to a 250 m step, then clamped to `[1000, maxRadius]`. Meters-per-degree longitude uses `111000 * cos(latitude)`.
- Grid tier is 50, 100, or 200, with hysteresis so it does not flip at a single threshold: leaving 50 m requires radius `> 3000`; entering 50 m from 100 m requires radius `< 2500`; entering 200 m requires radius `> 8000`; leaving 200 m requires radius `< 7000`.
- A query is skipped when the key `lat4:lng4:radius:gridSize` is unchanged.

Fetching: `fetchHeatmap`. Initial load sets `loading`. Later loads set `refreshing` and do not block the map. Requests are sequenced with `requestSeqRef` so a slow response cannot overwrite a newer one. Cells with `risk_score <= 1.0` are not drawn. At most 1000 circles are rendered.

Color interpolation in `interpolateColor` (scores at or below 1 are not in the drawn set):

- 1 to 2: `#FACC15` toward `#EAB308` (yellow to amber)
- 2 to 4: `#F97316` toward `#EF4444` (orange to red)
- 4 to 5: `#EF4444` toward `#B91C1C`

Opacity is `0.22 + (score/5) * 0.48`. Circle radius is `gridSize * 0.7` meters so neighboring circles overlap. Cluster markers are not drawn. The comment says they were removed.

Location: asks `expo-location` for foreground permission and centers once. User marker uses `pinColor={colors.primary}`. `showsUserLocation` is also true.

WebSocket: `initWebSocket` on mount, `subscribeToIncidents`, and `subscribeToLocation` for the current query. An `incident:new` event triggers another heatmap fetch. The socket is not disconnected on unmount; the disconnect call is commented out. `heatmap:update` is subscribed in the client, and the server function that would emit it is unused.

`panToLocation` animates the map to a report coordinate. If the user pans more than about 0.0045 degrees (the comment says ~500 m) away from that point, `showReportedLocation` is cleared.

Map errors set `mapError` and call `onError`, which makes Home switch to the fallback list.

### 6.8 iOS heatmap (`src/components/HeatmapMap.ios.tsx`)

Same props, same grid/radius helpers, same circle rendering rule (`risk_score > 1.0`), Apple/Google provider usage differs in that this file is the iOS build. The file header says Android and web should use the fallback. Home still loads this file when `Platform.OS === 'ios'`.

### 6.9 Heatmap fallback (`src/components/HeatmapMapFallback.tsx`)

List UI, not a map. Fetches the heatmap and renders cells with a discrete color function: `>= 4` red, `>= 2.5` orange, `>= 1.5` yellow, else green. It also lists clusters. This component does not implement the zoom-tier logic.

`HeatmapMap.tsx.backup` is an unused backup file. `SimpleMap.tsx` renders a placeholder with coordinates and is not imported by any screen.

### 6.10 Panic (`src/screens/PanicScreen.tsx` and `src/components/PanicButton.tsx`)

Contacts are `{ id, name, phone, relationship? }`, stored under `@emergency_contacts`. Add and delete are local. The screen reloads contacts on focus.

The button is a `TouchableOpacity` `onPress`. The label reads `HOLD FOR EMERGENCY` when inactive and `ACTIVE` when `isActive` is true. There is no press-and-hold timer. Press vibrates `[0, 500, 200, 500]`, runs a scale animation, and opens an alert: `CANCEL` (logs only; the component comment says cancellation is a TODO) or `CONFIRM`, which calls `onPanicTrigger`.

`handlePanicTrigger`:

- Requests foreground location permission.
- Reads current position.
- `triggerPanicAlert('user123', latitude, longitude, phone strings)`.
- Sets `isActive` true.
- Calls `startLocationTracking`.
- Alerts "Your location has been shared with emergency contacts." The server does not send those messages. The alert text on the card also says contacts are notified by SMS and a call notification. No SMS or telephony code is in the app.

`startLocationTracking` starts a 30-second `setInterval` that posts `updateLocation('user123', ...)`. The interval closure reads `isActive` from the render in which `handlePanicTrigger` was created. `setIsActive(true)` has not re-rendered yet, so that closure still sees `isActive === false`, and the first tick clears the interval. The cleanup function returned by `startLocationTracking` is not used. Cancel on the screen sets `isActive` false locally and does not call `POST /api/panic/cancel`.

### 6.11 Report form (`src/screens/ReportsScreen.tsx`)

Categories: `Harassment`, `Suspicious Activity`, `Poor Lighting`, `Unsafe Area`, `Other`.

Severity buttons 1–5 with labels Very Low, Low, Medium, High, Very High. Default 3.

Description is a required text field.

Location opens `LocationPicker`: a modal `MapView` with a draggable marker. Initial position is the current GPS fix when permission is granted, otherwise `13.0827, 80.2707`.

Media: `expo-image-picker` is required inside try/catch. Library and camera can attach one image or a video up to 30 seconds, quality 0.8. If the native module is missing, the UI says a development build is required. The selected media is kept in component state and is not included in the POST body.

Submit body:

```
userId: "user123"
type: "community_report"
category, description, severity
location: { latitude, longitude }
timestamp
timezone_offset_minutes: -getTimezoneOffset()
```

`description` is sent. The Express handler requires it and does not persist it. The community list later shows a sentence built from `category`, not this text.

Success alert offers "View on Map" (navigates to Home with `panToLocation`), "View Reports" (Community tab), or OK, then resets the form.

### 6.12 Community list (`src/screens/CommunityReportsScreen.tsx`)

Loads `fetchCommunityReports` on mount and on focus. Subscribes to `incident:new` and reloads after 1.5 seconds. Unmount calls `disconnectWebSocket()`, which is a different lifecycle from the heatmap (the heatmap leaves the socket up).

Filters, applied on the device: category (unique values from the loaded list), exact severity, and date range `all | today | week | month`. The subtitle says "in your area". The request is `GET /api/reports/all`, which returns every community report, not a radius around the user.

A row opens `ReportDetailsScreen` as a modal: category, description, severity label, coordinates, time, verified flag. "View on map" opens `https://www.google.com/maps?q=lat,lng` with `Linking.openURL`. It does not switch to the Home heatmap. Home's `panToLocation` is used by the report form's success alert, not by this button.

### 6.13 Routes (`src/screens/RoutePlanningScreen.tsx`)

On mount, requests location and sets the start point to "Current Location".

Start and end can be set by:

- current location
- Places autocomplete (`searchPlaces` after 2 characters, biased to the current fix when available), then `getPlaceCoordinates`

`handleFindRoutes` calls `getSafeRoutes`. The recommended id is selected, or the first route. The map fits all waypoints.

Polyline color: selected route uses `colors.primary`. Others are `#4CAF50` when `safetyScore >= 0.7`, `#FF9800` when `>= 0.4`, otherwise `#F44336`.

The screen formats distance, duration, and safety as a percent. `openRouteInGoogleMaps` builds `https://www.google.com/maps/dir/?api=1` with origin, destination, `travelmode=driving`, and up to 23 intermediate waypoints sampled from the polyline (Google's cap is handled as 25 total points).

High-risk segments from the API are available on the `SafeRoute` type. The screen draws full route polylines rather than only those segments.

### 6.14 `ContactsScreen.tsx`

A standalone contacts UI with the same storage key `@emergency_contacts`. It is not mounted by the navigator. The Panic screen has its own copy of the contact list logic.

### 6.15 Mobile README versus the code

`frontend/mobile/README.md` still lists these as future work: route planning visualization, real-time location tracking, push notifications, Firebase Auth, community reporting.

In the current tree, route planning, community reporting, and a location-update call from the panic screen exist. Push notifications and Firebase Auth do not. The README's heatmap color bands (red `4.0+`, orange `2.5–4.0`, yellow `1.5–2.5`, green `0–1.5`) match `HeatmapMapFallback`, not the Android circle gradient, which hides scores at or below 1.0 and does not draw green.

---

## 7. Admin web app (`frontend/admin`)

Vite React app. `npm run dev` serves port `5174` and proxies `/api` to `http://192.168.1.12:3001`.

`src/main.tsx` wraps the tree in `BrowserRouter` and `AuthProvider`.

Token: `localStorage` key `admin_token`. `api()` in `src/api/client.ts` sends `Authorization: Bearer`. HTTP 401 clears the token and sets `window.location.href = '/'`.

Routes (`src/App.tsx`):

| Path | Page | Guard |
| --- | --- | --- |
| `/` | `Login` | none |
| `/admin` | redirects to `/admin/dashboard` | `ProtectedRoute` checks `isAuthenticated` (token presence only; it does not decode the JWT) |
| `/admin/dashboard` | `Dashboard` | same |
| `/admin/incidents` | `Incidents` | same |
| `/admin/heatmap` | `Heatmap` | same |
| `/admin/analytics` | `Analytics` | same |
| `/admin/audit` | `Audit` | same |

`Layout.tsx` sidebar brand text is `Women Safety Analytics`. Links: Dashboard, Moderation, Heatmap, Analytics, Audit. Logout clears the token and navigates to `/`.

Login form (`Login.tsx`) defaults the email field to `admin`. The hint on the page says `Default: admin / admin` and mentions `ADMIN_EMAIL`, `ADMIN_PASSWORD`, and `ADMIN_SECRET`. That matches `auth.ts`.

Dashboard: four cards (total, today, panic, pending) plus ML health and a 10-row recent table. Numbers are whatever `/api/admin/dashboard` returns, including the page-limited counts described in section 5.10.

Incidents: filters for status (all / pending / verified) and type (all / panic / community). Verify and Reject buttons show only when `verified` is false. Reject sends `rejectReason` or the string `rejected`. `rejectReason` state exists and is cleared after reject; the page JSX does not render an input bound to `rejectReason`, so the posted reason is `rejected` unless that state is set some other way. Location links open Google Maps. Pagination display is "Showing N of total". The load function always requests `offset: 0` and limit 50, so it does not page further.

Heatmap page: default center `13.0827, 80.2707`, default radius 3000, grid size fixed at 100 in the client call. It is a table of clusters and the top 20 cells by risk, plus an "Open in Google Maps" link. It does not draw a map or circles.

Analytics: period select `7d` or `30d`. Metric cards and CSS bar charts (no chart library) for incidents by day, severity, category, and hour. Empty state text tells the operator to submit incidents from the mobile app. The by-day chart heading in a comment says "Line Chart"; the markup is bars.

Audit: table of time, entity id, action, reason from `/api/admin/audit`.

`frontend/admin/README.md` matches this layout and says the API must be on port 3001 and the ML service on port 8000, and that migration 003 should be applied.

---

## 8. End-to-end behavior that is wired

### 8.1 Community report

1. Reports screen validates category, description, and a picked coordinate.
2. `POST /api/reports/submit` with `userId` `user123`.
3. Express checks fields, drops the description, and `POST`s `/ml/incidents/process`.
4. FastAPI inserts a row, including local hour from the timezone offset, and clears the cluster cache.
5. Express emits `incident:new` on Socket.IO.
6. The community screen and the Android heatmap, if connected, refresh.
7. The list text the user later sees is rebuilt from `category`, not from the typed description.
8. Photos and videos stay on the phone.

### 8.2 Panic

1. Confirm on the SOS button.
2. `POST /api/panic/trigger` with the GPS fix, severity forced to 5, type `panic_alert`, `verified: true`.
3. The row is inserted the same way as a report.
4. This route does not call `emitNewIncident`. A panic does not push a Socket.IO event. The community list filters to `community_report`, so the panic row would not appear there anyway. It does appear in the admin incident list.
5. The HTTP body says contacts and authorities were notified. No notifier runs.
6. The follow-up location posts call `GET`-style risk scoring through `POST /api/location/update` and are not stored. The interval implementation is described in section 6.10.

### 8.3 Heatmap

1. Map region change builds a center, radius, and grid tier.
2. `GET /api/location/heatmap` forwards local hour and timezone offset.
3. FastAPI bins incidents in view and scores each bin from incidents within 1 km, with recency decay and hour-of-day similarity.
4. Cells at or below risk 1 are omitted on the native map.
5. Admin heatmap is the same generator with `include_clusters=true`, then DBSCAN over all incidents.

### 8.4 Safe route

1. Google Routes API returns one or more driving polylines, or a straight two-point fallback if the call fails.
2. FastAPI scores each segment midpoint.
3. The highest `safety_score` is `recommended_route`.
4. The phone draws the polylines and can open Google Maps directions.

### 8.5 Moderation

1. Admin login compares plaintext credentials to environment variables and signs a 7-day JWT.
2. Verify sets `verified=true` and an optional reason.
3. Reject sets `verified=false` and a reason. It does not delete the row, and it does not remove the incident from future risk calculations. `get_incidents_in_radius` has no `verified` filter, so rejected and unverified rows still affect scores.
4. The audit page is a filtered view of recent incidents, not a separate log.

---

## 9. File inventory

Line counts below are `wc -l` on the current tree. Asset bytes that are images are not counted as source.

### 9.1 Repository root

| Path | Lines | What it is |
| --- | --- | --- |
| `README.md` | 1 | `# SafeNaari` |
| `.gitignore` | — | Ignores `node_modules`, env files, Python caches, `dist`, `*.md` except `README.md` and this file, `research/`, `scripts/`, `ref_paper/`, pdf and office documents, lockfiles |
| `fraud_vs_genuine_fingerprint.svg` | 71 | Diagram added in commit `7169da7`. Not imported by app code. Visible labels include "Genuine emergency - what it looks like", "Fraud ring - what it actually looks like", "500 claims, same tile", and timing/device comparison text. |
| `signal_fusion_stack.svg` | 75 | Same commit. Labels include "Claim arrives", "GPS coherence", "Sensor fusion", "Network delta", "Weather ground truth", "Platform telemetry", "Coordinated fraud detector", "Risk score". |
| `three_path_claim_routing.svg` | 83 | Same commit. Labels include "Risk score computed", "Score threshold check", thresholds `score < 0.25`, `0.25 – 0.65`, `score > 0.65`, and paths "Auto-approve", "Soft challenge", "Human review". |

Those three SVGs are not referenced by TypeScript, Python, or the admin app. No route implements claim payout, device-fingerprint entropy, weather checks, or the 0.25 / 0.65 thresholds drawn in the SVGs. The risk score the services actually compute is the 0–5 incident formula in section 4.5.

### 9.2 `backend/api`

| Path | Lines | Role |
| --- | --- | --- |
| `package.json` | 42 | Dependencies and scripts |
| `tsconfig.json` | 29 | Compiler options |
| `src/app.ts` | 136 | Express and Socket.IO bootstrap |
| `src/middleware/auth.ts` | 41 | Admin JWT |
| `src/routes/auth.ts` | 235 | Admin login plus stub user auth |
| `src/routes/panic.ts` | 171 | Panic trigger, cancel stub, status stub |
| `src/routes/reports.ts` | 240 | Submit, empty per-user list, all community reports |
| `src/routes/location.ts` | 405 | Location update, heatmap, routes, places |
| `src/routes/admin.ts` | 425 | Dashboard, moderation, heatmap, analytics, audit |
| `src/services/mlService.ts` | 220 | Axios client for FastAPI |
| `src/services/googleMapsService.ts` | 301 | Routes, Places, Geocoding |
| `src/websocket/socketHandler.ts` | 172 | Rooms and `incident:new` |
| `test-integration.js` | 248 | Manual axios checks |
| `test-full-system.js` | 190 | Manual axios checks |

### 9.3 `backend/ml`

| Path | Lines | Role |
| --- | --- | --- |
| `requirements.txt` | — | Python pins |
| `setup-database.ps1` | — | Windows database bootstrap |
| `app/main.py` | 77 | FastAPI app |
| `app/config.py` | 88 | Settings |
| `app/api/routes.py` | 330 | `/ml` routes |
| `app/api/schemas.py` | 138 | Pydantic models |
| `app/db/connection.py` | 95 | psycopg2 pool |
| `app/db/storage.py` | 274 | SQL for incidents |
| `app/ml/risk_scoring.py` | 369 | 0–5 score |
| `app/ml/heatmap.py` | 269 | Incident-binned heatmap |
| `app/ml/clustering.py` | 116 | DBSCAN cache |
| `app/ml/route_analyzer.py` | 107 | Segment risk |
| `app/ml/models.py` | 43 | Training stub |
| `app/utils/geospatial.py` | 62 | Haversine and degree conversion |
| `app/utils/land_mask.py` | 206 | Optional polygon mask |
| `app/data/chennai_mock_data.py` | 440 | Chennai synthetic incidents |
| `app/data/mock_data.py` | 86 | New York–centered synthetic incidents |
| `app/data/storage.py` | 102 | Unused in-memory store |
| `app/resources/landmask/chennai_wards.geojson` | — | 201-feature ward polygons, used only if the mask flag is turned on |
| `migrations/001_initial_schema.sql` | 66 | PostGIS schema |
| `migrations/001_initial_schema_no_postgis.sql` | 63 | Non-PostGIS schema |
| `migrations/002_time_of_day_columns.sql` | 12 | Time columns |
| `migrations/003_moderation.sql` | 5 | Moderation column |
| `run_migration_002.py` | 65 | Apply time columns |
| `run_migration_003.py` | 36 | Apply moderation column |
| `step2_load_data.py` | 134 | Load 5000 rows |
| `step3_test_ml_algorithms.py` | 214 | Cluster and score smoke script |
| `step4_test_heatmap.py` | 185 | Heatmap smoke script |
| `step5_test_routes.py` | 190 | Route smoke script |
| `test_db_connection.py` | 72 | DB smoke script |
| `test_full_integration.py` | 96 | Python-side integration script |
| `test_mock_data.py` | 60 | Generator smoke script |
| `test_risk_optimization.py` | 102 | Score printout |

`app/__init__.py`, `app/api/__init__.py`, `app/data/__init__.py`, `app/db/__init__.py`, `app/ml/__init__.py`, and `app/utils/__init__.py` are package markers (1–9 lines).

### 9.4 `frontend/mobile`

| Path | Lines | Role |
| --- | --- | --- |
| `package.json` | 45 | Expo dependencies |
| `app.json` | 66 | Expo config, permissions, Maps key |
| `eas.json` | 29 | EAS build profiles |
| `index.ts` | 8 | Expo root registration |
| `App.tsx` | 16 | Renders the navigator |
| `metro.config.js` | 24 | socket.io-client resolution |
| `tsconfig.json` | 6 | Extends Expo base, `strict` |
| `README.md` | 172 | Setup notes; feature checklist is behind the code |
| `src/navigation/AppNavigator.tsx` | 83 | Five tabs |
| `src/services/api.ts` | 376 | HTTP client |
| `src/services/websocket.ts` | 323 | Socket.IO client |
| `src/utils/storage.ts` | 60 | AsyncStorage or memory |
| `src/theme/colors.ts` | 47 | Palette |
| `src/theme/spacing.ts` | 19 | Spacing scale |
| `src/components/AppIcons.tsx` | 100 | SVG icons |
| `src/components/HeatmapMap.android.tsx` | 707 | Google map circles, zoom tiers |
| `src/components/HeatmapMap.ios.tsx` | 552 | iOS map variant |
| `src/components/HeatmapMapFallback.tsx` | 380 | List fallback |
| `src/components/HeatmapMap.tsx.backup` | — | Unused backup |
| `src/components/LocationPicker.tsx` | 317 | Draggable marker modal |
| `src/components/PanicButton.tsx` | 215 | SOS confirm button |
| `src/components/SimpleMap.tsx` | 69 | Unused placeholder |
| `src/screens/HomeScreen.tsx` | 250 | Map shell |
| `src/screens/PanicScreen.tsx` | 348 | SOS and contacts |
| `src/screens/ReportsScreen.tsx` | 730 | Report form |
| `src/screens/CommunityReportsScreen.tsx` | 730 | Report list and filters |
| `src/screens/ReportDetailsScreen.tsx` | 282 | Modal detail |
| `src/screens/RoutePlanningScreen.tsx` | 950 | Places search and safe routes |
| `src/screens/ContactsScreen.tsx` | 371 | Not mounted |
| `assets/icon.png`, `adaptive-icon.png`, `splash-icon.png`, `favicon.png` | — | Expo image assets |

### 9.5 `frontend/admin`

| Path | Lines | Role |
| --- | --- | --- |
| `package.json` | 23 | Vite React |
| `index.html` | — | Title `Women Safety Analytics — Admin` |
| `vite.config.ts` | 15 | Port 5174 and `/api` proxy |
| `tsconfig.json` | 27 | Strict bundler TypeScript |
| `README.md` | 31 | How to run the dashboard |
| `src/main.tsx` | 16 | Router and auth provider |
| `src/App.tsx` | 39 | Routes |
| `src/index.css` | 46 | Global styles |
| `src/api/client.ts` | 67 | Token and admin methods |
| `src/context/AuthContext.tsx` | 46 | Login state |
| `src/components/Layout.tsx` | 44 | Sidebar |
| `src/components/Layout.module.css` | 67 | Sidebar styles |
| `src/pages/Login.tsx` | 65 | Admin sign-in |
| `src/pages/Login.module.css` | 85 | |
| `src/pages/Dashboard.tsx` | 94 | Overview |
| `src/pages/Dashboard.module.css` | 116 | |
| `src/pages/Incidents.tsx` | 164 | Moderation table |
| `src/pages/Incidents.module.css` | 138 | |
| `src/pages/Heatmap.tsx` | 158 | Cluster and cell tables |
| `src/pages/Heatmap.module.css` | 119 | |
| `src/pages/Analytics.tsx` | 359 | CSS bar charts |
| `src/pages/Analytics.module.css` | 316 | |
| `src/pages/Audit.tsx` | 57 | Audit table |
| `src/pages/Audit.module.css` | 53 | |

---

## 10. Explicit TODOs left in the source

These strings are in the files. They are the author's notes, not an inference.

| Location | Note in the file |
| --- | --- |
| `backend/api/src/routes/auth.ts` | Register: no email or password checks, no duplicate check, no hash, no database user, no real JWT, no email. Login: no credential check. Profile: mock. Emergency contacts: not stored. |
| `backend/api/src/routes/panic.ts` | Trigger does not verify a user, store beyond the ML insert, notify contacts, notify authorities, or start tracking. Cancel does not verify ownership, update status, stop tracking, or notify. Status does not query active alerts. |
| `backend/api/src/routes/location.ts` | Location update is not stored. High-risk alerts (`risk_score >= 4.0`) are not sent. |
| `backend/api/src/routes/reports.ts` | `GET /user/:userId` does not read the database. |
| `backend/ml/app/ml/models.py` | Model-loaded check and training are TODOs. |
| `backend/ml/app/data/storage.py` | Comment says replace in-memory storage with a database. The database module exists and is what the API uses. |
| `frontend/mobile/src/components/PanicButton.tsx` | Cancel path inside the confirm dialog is a TODO. |
| `frontend/mobile/src/screens/ReportsScreen.tsx` | `userId: "user123"` is marked TODO for auth. |

Related behavior that is visible without a TODO comment:

- `safe_distance` equals `total_distance`.
- `update_clusters_if_needed` returns an empty list.
- `emitHeatmapUpdate` and `emitRiskAlert` have no callers.
- Rejected incidents remain in radius queries.
- Report description and media are not stored.
- Dashboard "today", panic, and verified counts are computed from at most 100 rows.
- Analytics reads at most 1000 rows.
- `GET /ml/health` reports `last_training` as the current time.
- Land mask code is present and the flag defaults to off.
- `ContactsScreen` and `SimpleMap` are not in the navigator.
- Home's address line is the fixed string `Connaught Place, New Delhi`.
- Admin and mobile clients, and the ML URL fallback, point at `192.168.1.12`.

---

## 11. Git history that produced this tree

Times below are the commit timestamps, already in `+0530` (Asia/Calcutta).

| Commit | When (IST) | Subject |
| --- | --- | --- |
| `2e98782` | 20 Jan 2026, 07:00 | Initial commit: Women Safety Analytics - Real-Time Monitoring & Proactive Crime Prevention |
| `1360879` | 31 Jan 2026, 14:39 | Snapshot: research docs + security cleanup |
| `5f28fa5` | 1 Feb 2026, 18:08 | Backup snapshot before zoom-based heatmap resolution |
| `d76889a` | 5 Feb 2026, 22:33 | Add zoom-adaptive heatmap, room-targeted realtime, and research README |
| `1097b58` | 6 Feb 2026, 10:32 | Update README.md |
| `09c7093` | 2 Mar 2026, 14:28 | Save UI before SafeNaari Figma redesign - revert point if new UI fails |
| `a65d457` | 2 Mar 2026, 16:25 | Heatmap legend removal, report details, routes UI, icons, gitignore (markdown ignored except README; `research/` and `scripts/` excluded) |
| `0c0b9e4` | 2 Mar 2026, 22:41 | Update Google Maps API key fallback value |
| `7169da7` | 20 Mar 2026, 23:44 | Update SafeNaari anti-spoofing submission. Adds the three root SVGs and edits README plus a few URL strings. Does not add anti-spoofing code under `backend/` or `frontend/`. |
| `43c7316` | 31 Mar 2026, 17:02 | Revise README by removing team info and content. Root README is left as the title only. |
| `73d3d76` | (parent of the working tree's `main`) | `chore: ignore local papers and templates`. Ignores `ref_paper/`, `*.pdf`, `*.doc`, `*.docx`. |

Intervening commits delete `docs/`, task files, `.cursor/rules`, `update-ip.ps1`, and the research markdown files named at the top of this document. `research/` and `scripts/` are gitignored and are not in the tracked tree.

Remote: `github.com/surajsinghshekhawat/SafeNaari`, branch `main` at `73d3d76` when this file was written.

---

## 12. How to run what the repo defines

These commands are the scripts in the manifests, not a verified run from this document.

1. PostgreSQL reachable at the settings in `backend/ml/app/config.py` (default `localhost:5433`, database `women_safety_db`). Apply `001_initial_schema.sql` if the tables do not exist (PostGIS required for the insert and radius SQL). Then `python run_migration_002.py` and `python run_migration_003.py` from `backend/ml` if those columns are missing. `001_initial_schema.sql` already contains the time columns; `003` is still required for `moderation_reason`.
2. Optional: `python step2_load_data.py` from `backend/ml`. This truncates `incidents` and inserts 5000 synthetic Chennai rows.
3. ML service: from `backend/ml`, with dependencies from `requirements.txt`, `uvicorn app.main:app --host 0.0.0.0 --port 8000` (or `python -m app.main`, which enables reload).
4. API: from `backend/api`, `npm install`, `npm run dev`. Listens on port 3001. Set `ML_SERVICE_URL` if the ML service is not at `http://192.168.1.12:8000`. Set `GOOGLE_MAPS_API_KEY` for routes and place search. Set `JWT_SECRET`, `ADMIN_EMAIL`, and `ADMIN_PASSWORD` if the built-in admin defaults should not be used.
5. Admin: from `frontend/admin`, `npm install`, `npm run dev`, open `http://localhost:5174`. The Vite proxy target is `192.168.1.12:3001`; change `vite.config.ts` if the API is elsewhere.
6. Mobile: from `frontend/mobile`, `npm install`, `npm start`. The dev API host is hardcoded in `src/services/api.ts`. Native maps need the Google key already placed in `app.json` and a build that includes `react-native-maps`. `eas.json` is set up for EAS APK builds.

`npm test` in `backend/api` is the placeholder that exits 1. The Python step scripts and the two Node axios scripts are manual.
