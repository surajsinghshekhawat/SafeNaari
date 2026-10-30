# SafeNaari — system context

This document describes the repository as it exists in source at commit `73d3d76` (`chore: ignore local papers and templates`) on branch history that ends at `main`. It was written by reading the files in the tree. It does not restore deleted documents, and it does not describe behavior that is not in the code.

Earlier commits removed several markdown files (`FULL_PROJECT_REFERENCE.md`, `IMPLEMENTATION_TODO_DETAILED.md`, `RESEARCH_REPORT_CONTEXT.md`, `RESEARCH_READINESS_CHECKLIST.md`, `PROJECT_OVERVIEW_FOR_PPT.md`, `developer.md`, and a `docs/` directory). Those files are not in the current tree, so their contents are not summarized here.

Root `README.md` is a single line: `# SafeNaari`.

The product name used in the mobile UI is **SafeNaari**. Package descriptions, the Express process log, the FastAPI title, and the admin sidebar still say **Women Safety Analytics**.

Sections 1–12 are the file-by-file reference. Sections 13–16 are the same system written the way each role would have to hold it: architecture, then machine learning, data science, and fullstack. Those later sections cite the functions and the formulas that are actually in the tree. They do not add a model, a metric, or a service that the code does not contain.

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

---

## 13. Architecture, as the code is wired

This is the shape of the system an engineer has to keep in their head. It is the same four processes from section 1, drawn by ownership rather than by folder.

```
+-------------------+                     +------------------------+
| Expo app          | -- HTTP + Socket -> | Express  :3001         |
| frontend/mobile   |                     | backend/api            |
| no JWT, user123   |                     | auth, Google, fan-out  |
+-------------------+                     +-----------+------------+
                                                      |
+-------------------+                                 | axios / fetch
| Vite admin :5174  | -- /api proxy, Bearer JWT ----> |
| frontend/admin    |                                 v
+-------------------+                     +------------------------+
                                          | FastAPI  :8000         |
                                          | backend/ml             |
                                          | score, cluster, SQL    |
                                          +-----------+------------+
                                                      | psycopg2
                                                      v
                                          +------------------------+
                                          | PostgreSQL :5433       |
                                          | women_safety_db        |
                                          | incidents (+ PostGIS)  |
                                          +------------------------+

Google is called only from Express, never from the phone's route screen
and never from FastAPI:
  routes.googleapis.com/directions/v2:computeRoutes
  maps.googleapis.com/maps/api/place/autocomplete/json
  maps.googleapis.com/maps/api/geocode/json

The phone opens maps.google.com URLs itself for "view on map" and
"open in Google Maps". Those are links, not the Routes API.
The native map tiles use the key in app.json, separate from
GOOGLE_MAPS_API_KEY on the server.
```

### 13.1 Who owns which fact

| Fact | Owner | Everyone else |
| --- | --- | --- |
| Incident rows | PostgreSQL, written only by `backend/ml/app/db/storage.py` | Express never opens a database connection. The admin UI never talks to FastAPI directly. |
| Risk number 0–5 | `calculate_risk_score` / `_calculate_risk_score_from_incidents` in `backend/ml/app/ml/risk_scoring.py` | Mobile and admin only display it. Express forwards it. |
| Heatmap cells | `generate_heatmap` in `backend/ml/app/ml/heatmap.py` | Android/iOS draw circles. Admin prints a table. |
| Unsafe-zone clusters | In-process DBSCAN cache in `backend/ml/app/ml/clustering.py` | The `unsafe_zones` table is created by SQL and never inserted. |
| Driving geometry | Google Routes API, decoded in `backend/api/src/services/googleMapsService.ts` | FastAPI only receives waypoints and returns scores. |
| Admin identity | JWT signed in `backend/api/src/middleware/auth.ts` | FastAPI has no auth. Mobile sends no token. |
| Emergency contacts | AsyncStorage key `@emergency_contacts` on the phone | `PUT /api/auth/emergency-contacts` echoes the body and stores nothing. |
| Report prose and media | Phone memory until submit, then discarded | The database has no description column and no media column. |

### 13.2 Synchronous path versus the socket

Almost every user action is a request that finishes before the UI updates:

1. Phone or browser calls Express.
2. Express calls FastAPI and waits (axios timeout 120 seconds for the ML client; Google Routes timeout 15 seconds; Places timeout 10 seconds; geocode timeout 5 seconds).
3. FastAPI runs SQL and Python in the request.
4. Express maps the JSON and returns.

The socket is a side channel, not the source of heatmap geometry. `POST /api/reports/submit` calls `emitNewIncident` after the ML insert. Clients that joined `incidents:all`, and clients whose location room covers the point, receive `incident:new`. The Android map then waits 2 seconds and calls `fetchHeatmap` again over HTTP. `emitHeatmapUpdate` exists in `backend/api/src/websocket/socketHandler.ts` and has no caller, so the event `heatmap:update` is never produced. `emitRiskAlert` is the same: defined, never called, and no socket joins `user:<id>`.

Panic does not emit `incident:new`. A panic row is still inserted.

### 13.3 Failure policy is different per route

| Call | If the ML service errors |
| --- | --- |
| Heatmap (`GET /api/location/heatmap`) | HTTP 200, empty `cells`, warning string. The map stays up with nothing drawn. |
| Risk on location update | The update still returns 200. `riskAssessment` is null only if `getRiskScore` throws. The ML client usually returns a fallback score of `0` / `very_safe` instead of throwing. |
| Safe routes | HTTP 503 when `analyzeRoutes` returns `success: false`. |
| Report submit | HTTP 200 even when the ML client returns `success: false` inside the normal return, because the route checks the call did not throw. The fallback object does not throw. |
| Admin analytics | HTTP 502 when `getIncidentsAll` returns `success: false`. |
| Admin heatmap | HTTP 200 with whatever cells came back, plus a warning. |

FastAPI itself turns exceptions into HTTP 500 with `detail`. Express then applies the table above.

### 13.4 Process boundaries that matter when changing a feature

A change to the score formula is entirely inside `risk_scoring.py`, but every surface that shows a number will move together: mobile circles, admin top-20 table, route safety percent, dashboard nothing (the dashboard does not show risk), and the panic location response.

A change to what is stored on an incident has to touch, in order: `IncidentRequest` in `backend/ml/app/api/schemas.py`, the `INSERT` in `app/db/storage.py`, a migration, the `SELECT` column lists in `get_incidents` and `get_incidents_in_radius`, the Express body in `reports.ts` or `panic.ts`, and the mobile `fetch` body. Description never made it past step "Express body". The phone sends it. The handler requires it. The Pydantic model has no field for it, so it is dropped before the insert.

---

## 14. Machine learning, from the model owner's seat

This section is the system as the person responsible for `backend/ml/app/ml` has to understand it. The product copy says "ML-powered". The code is a deterministic scoring function plus one unsupervised clustering algorithm. There is no fitted regressor, no classifier, no saved weights, and no training loop that updates parameters from data.

`backend/ml/app/ml/models.py` is the file a reader would open looking for that loop. `train_models` reads `get_incident_count()` and returns `started: True` and the string `"5 minutes"`. `get_model_status` returns `loaded: True` and `last_training` set to `datetime.utcnow()` at the moment of the health check. Both functions contain TODO comments. `POST /ml/models/train` exposes that stub. `joblib` is pinned in `requirements.txt` and is not imported. `settings.model_path` (`./models`) is never read. `settings.auto_retrain_threshold` and `settings.min_retrain_interval` are never read.

The only scikit-learn call in the repository is `DBSCAN` in `backend/ml/app/ml/clustering.py`. It is fit on demand, in memory, and discarded except for the cluster summary cache.

### 14.1 What is being predicted

Nothing is predicted from a held-out label. For a query point `(lat, lng)` and a local hour, the service returns a scalar in `[0, 5]` and a string level. The scalar is a weighted blend of four quantities computed from incidents already stored within 1000 meters:

| Factor | Range before the final `* 5` | Weight in `Settings` | Meaning in the code |
| --- | --- | --- | --- |
| `incident_density` | `[0, 1]` | `0.5` | Log of a recency-and-hour weighted count, divided by the log of `max_expected_effective_incidents + 1` (default max 10). |
| `avg_recency` | `[0, 1]` | `0.25` | Distance-and-hour-weighted average of per-incident exponential decay. |
| `avg_severity` | `[0, 1]` | `0.15` | Weighted mean of `severity / 5`. |
| `time_pattern` | roughly `[0, 1]` | `0.1` | Mix of the clock prior and whether this hour matches the dominant historical bucket at this place. |

```python
risk_score = (
    settings.weight_incident_density * incident_density
    + settings.weight_recency * avg_recency
    + settings.weight_severity * avg_severity
    + settings.weight_time_pattern * time_pattern
) * 5.0
risk_score = max(0.0, min(5.0, risk_score))
```

That block is `_calculate_risk_score_from_incidents` in `backend/ml/app/ml/risk_scoring.py`. The weights in `backend/ml/app/config.py` sum to 1, so the theoretical maximum is 5 and the theoretical minimum, when incidents exist and every factor is 0, is 0. Rounding is to 2 decimal places on the way out.

Levels, applied only on this path (the empty-neighborhood path is different, below):

| Rounded score | `risk_level` |
| --- | --- |
| `<= 1.0` | `very_safe` |
| `<= 2.0` | `safe` |
| `<= 3.0` | `medium` |
| `<= 4.0` | `high` |
| otherwise | `very_high` |

Columns that exist on the row and are **not** inputs to this function: `type`, `category`, `verified`, `moderation_reason`, `user_id`, `id`. A rejected incident with `verified = false` still enters the sums. A panic and a community report with the same severity, place, and time contribute the same weight. Category strings never reach the formula.

### 14.2 Online inference, one point

`calculate_risk_score(lat, lng, query_timestamp=None, local_hour=None)` does two things:

1. `get_incidents_in_radius(lat, lng, 1000)` — PostGIS `ST_DWithin` on the `geography` column, radius in meters, no time filter, no verified filter.
2. Hand that list to `_calculate_risk_score_from_incidents` with `radius_meters=1000`.

The inner function filters again with haversine `<= 1000`. For the single-point API that filter is redundant with PostGIS. Heatmap calls the inner function with a pre-collected neighborhood, so the second filter is what keeps a cell's neighborhood round in meters rather than square in bins.

Query time defaults to `datetime.now(timezone.utc)`. Query hour defaults to `now.hour` when `local_hour` is omitted. `datetime.now()` without a timezone is the **server process** local hour, not the phone hour. Call sites that omit `local_hour`:

- `GET /ml/risk-score` when the query string has no `local_hour`. Express `getRiskScore` in `backend/api/src/services/mlService.ts` sends only `lat` and `lng`. Location updates therefore use the API host's clock.
- `get_clusters`, which calls `calculate_risk_score(center_lat, center_lng)` with no hour.
- `analyze_routes`, which calls `calculate_risk_score(mid_lat, mid_lng)` per segment with no hour.

Heatmap generation is the path that does pass the phone's `local_hour`, because `GET /api/location/heatmap` forwards it and `generate_heatmap` passes it into the inner scorer.

### 14.3 Kernels, with the numbers the code produces

Three kernels multiply. They are all in `risk_scoring.py`.

**Distance.** For each incident inside the 1000 m cutoff:

```python
distance_weight = math.exp(-distance_meters / 111.0)
```

The length scale is 111 meters. Values:

| Distance | Weight |
| --- | --- |
| 0 m | 1 |
| 111 m | `exp(-1)` ≈ 0.368 |
| 222 m | `exp(-2)` ≈ 0.135 |
| 500 m | `exp(-500/111)` ≈ 0.011 |
| 1000 m | `exp(-1000/111)` ≈ 0.00012 |

An incident at the edge of the search radius barely moves the weighted sums. It still counts in `raw_incident_count`. It does **not** use distance inside `effective_incident_count` (next paragraph).

**Recency.** `calculate_recency_weight`:

```python
decay_days = max(1.0, float(settings.recency_decay_days))  # default 30
return math.exp(-days_ago / decay_days)
```

This is an e-folding time, not a half-life. Half-life would be `decay_days * ln(2)` ≈ 20.8 days at the default.

| Age | Weight at 30-day scale |
| --- | --- |
| 0 | 1 |
| 7 days | ≈ 0.792 |
| 30 days | ≈ 0.368 |
| 90 days | ≈ 0.050 |
| 365 days | ≈ 5.2e-6 |

`days_ago` is `(query_timestamp - incident_timestamp).total_seconds() / 86400`. Future timestamps produce a negative age and a weight above 1. The code does not clamp that.

**Hour-of-day similarity.** Circular distance on 0–23, then a Gaussian:

```python
d = abs(a - b) % 24
dh = min(d, 24 - d)
w = exp(-(dh ** 2) / (2 * sigma ** 2))   # sigma default 3 hours, minimum 0.25
w = clip(w, floor, 1)                     # floor default 0.05
```

| Hour gap | Weight before the floor | After the 0.05 floor |
| --- | --- | --- |
| 0 | 1 | 1 |
| 1 | ≈ 0.946 | ≈ 0.946 |
| 3 | ≈ 0.607 | ≈ 0.607 |
| 6 | ≈ 0.135 | ≈ 0.135 |
| 9 | ≈ 0.011 | 0.05 |
| 12 | ≈ 0.0003 | 0.05 |

The floor means an incident from the opposite side of the clock never goes to zero. It keeps 5% of its recency weight in `effective_incident_count`.

The incident hour used for `dh` is `_incident_hour_local`: the stored `incident_local_hour` if it is an int in 0–23, else `timestamp + timezone_offset_minutes`, else `timestamp.hour`. If that still fails, the scorer substitutes the query hour, which makes `time_weight = 1` for that row.

**Combined weight** for severity and for the recency average:

```python
weight = distance_weight * recency_weight * time_weight
effective_incident_count += recency_weight * time_weight
```

`effective_incident_count` ignores distance. Ten brand-new, same-hour incidents 900 m away still add almost `10 * 1 * 1` to the effective count, while they add almost nothing to `weighted_severity` because distance weight is ~0.0003. Density can therefore read "busy" for a neighborhood whose nearby-in-meters incidents are actually far inside the 1 km disk. That split is what the code does. The comment in `config.py` says density is calibrated on effective count because old incidents decay. It does not say that distance is left out of the count on purpose; the addition line simply does not multiply by `distance_weight`.

### 14.4 Density, severity, recency aggregates

```python
max_expected = max(1.0, settings.max_expected_effective_incidents)  # 10
incident_density = log(effective_incident_count + 1) / log(max_expected + 1)
incident_density = clip(incident_density, 0, 1)
```

| Effective count | Density |
| --- | --- |
| 0 | 0 |
| 1 | `log(2)/log(11)` ≈ 0.289 |
| 3 | ≈ 0.578 |
| 10 | 1 |
| 30 | `log(31)/log(11)` ≈ 1.43, then clipped to 1 |

So the density term saturates once about ten "fresh, same-hour" incidents sit inside the 1 km disk. More incidents cannot raise the density contribution above `0.5 * 5 = 2.5` points of the final score.

Severity:

```python
avg_severity = (sum(severity_i * weight_i) / sum(weight_i)) / 5.0
```

A neighborhood of only severity-5 incidents has `avg_severity = 1`. Only severity-1 incidents have `0.2`. If `total_weight` is 0 the code uses 0. That happens when every incident was skipped by timestamp parsing.

Recency aggregate is not the mean of recency weights alone. It is

```python
avg_recency = sum(recency * distance * time) / sum(distance * time)
```

which is the distance-and-hour-weighted mean of recency. A fresh incident far away contributes little. A fresh incident on top of the query point contributes a lot.

### 14.5 The clock prior and the historical bucket

`calculate_time_of_day_factor(local_hour)` does not look at data:

| Local hour | Factor | Comment in the function |
| --- | --- | --- |
| 21, 22, 23, 0, 1, 2, 3 | 0.9 | night |
| 18, 19, 20 | 0.7 | evening |
| 4, 5, 6, 7 | 0.6 | early morning |
| 8 through 17 | 0.3 | daytime |

Separately, incidents that survived the 1 km filter are counted into buckets. These lists are **not** the same partitions:

| Bucket | Hours in the list |
| --- | --- |
| night | 21, 22, 23, 0, 1, 2, 3, **4** |
| evening | 18, 19, 20 |
| afternoon | 12, 13, 14, 15, 16, 17 |
| morning | 6, 7, 8, 9, 10, 11 |

Hour **5** is in none of the lists. An incident whose local hour is 5 increments no counter. Hour **4** is "early morning" in the clock prior (factor 0.6) and "night" in the historical bucket. The dominant bucket is `max(pattern_counts, key=pattern_counts.get)`. On a tie, Python's `max` keeps the first key with that value. Dict order is night, evening, afternoon, morning.

If the query hour sits in the dominant bucket:

| Dominant share | `location_time_pattern` |
| --- | --- |
| `> 0.5` | `0.9 + share * 0.1` (between 0.95 and 1.0) |
| `> 0.3` | `0.7 + share * 0.2` |
| any smaller match | `0.5 + share * 0.2` |
| query hour not in the dominant bucket | `current_time_factor * 0.6` |

Then:

```python
time_pattern = 0.7 * location_time_pattern + 0.3 * current_time_factor
```

If no incident had a parseable hour, `time_pattern` is just `current_time_factor`.

Worked night case. Query hour 22, clock factor 0.9. Dominant bucket is night with share 0.8. Match, so `location_time_pattern = 0.9 + 0.08 = 0.98`. `time_pattern = 0.7 * 0.98 + 0.3 * 0.9 = 0.956`. Contribution to the 0–5 score: `0.1 * 0.956 * 5 ≈ 0.48`.

Worked mismatch. Same place queried at hour 10 (factor 0.3) while nights dominate. `location_time_pattern = 0.3 * 0.6 = 0.18`. `time_pattern = 0.7 * 0.18 + 0.3 * 0.3 = 0.216`. Contribution ≈ 0.11. The Gaussian hour kernel will also down-weight the night incidents in the density and severity terms, which is the larger effect. The bucket logic is an extra 10% of the blend, not the whole time model.

### 14.6 Empty neighborhood

If the incident list is empty, or haversine filtering removes all of them, the function does **not** return score 0. It returns:

```python
base_risk = current_time_factor * 1.5
risk_level = "safe" if base_risk < 2.0 else "medium"
```

| Hour band | Factor | Score | Level from this branch | Level the five-way scale would have assigned |
| --- | --- | --- | --- | --- |
| Night | 0.9 | 1.35 | `safe` | `safe` |
| Evening | 0.7 | 1.05 | `safe` | `safe` |
| Early morning | 0.6 | 0.90 | `safe` | `very_safe` |
| Day | 0.3 | 0.45 | `safe` | `very_safe` |

No configured factor times 1.5 reaches 2, so this branch never returns `medium` with the current table. It also never returns `very_safe`. Factors in the payload are zeros plus `current_time_factor`. There is no `calculated_at` key on this branch; the incident branch includes `calculated_at`.

The mobile map drops cells with `risk_score <= 1.0`. An empty day cell at 0.45 would be hidden. An empty night cell at 1.35 would be drawn, in the yellow band, even though no incident was nearby. Heatmap generation does not emit a cell at all when a bin has no incidents (section 14.7), so this empty-neighborhood score is what `GET /ml/risk-score` and route midpoints see in quiet areas, not what the heatmap draws in empty ocean or empty wards.

### 14.7 Heatmap is repeated inference, not a second model

`generate_heatmap` in `backend/ml/app/ml/heatmap.py` does not score a full lattice. The comment in the function calls the approach option B: cells exist only where incidents exist.

Algorithm, in the order the function runs:

1. `radius_degrees = radius_meters / 111000`, same for longitude. The box is not corrected by `cos(latitude)`. At about 13° N, one degree of longitude is about `cos(13°) ≈ 0.974` of a degree of latitude in ground meters, so the degree box is slightly short east-west relative to the requested meter radius. Incident **fetch** uses PostGIS meters. Bin **indexes** use the degree box only as an origin. Points are not clipped to the box; a negative bin index is allowed.
2. If a naive full grid would exceed 3000 cells, `grid_size_degrees` is increased. The `grid_size` integer returned to the client stays the requested meter size. The mobile circle radius uses the requested `query.gridSize`, not the adjusted degree size.
3. `incidents_in_view = get_incidents_in_radius(center, radius_meters)`.
4. `incidents_context = get_incidents_in_radius(center, radius_meters + 1000)` so a cell near the edge of the view still sees its own 1 km disk.
5. Both sets are dropped into dicts keyed by `(i, j)` where `i = int((lat - lat_min) / grid_size_degrees)`.
6. For each view bin, the cell coordinate is the **mean lat/lng of incidents in that bin**, not the geometric center of the square.
7. `is_point_allowed(cell_lat, cell_lng)` can drop the cell. With `land_mask_enabled = False` this returns true immediately.
8. Neighborhood bins are gathered with `neighbor_range = ceil(1000 / grid_size_meters)`, then haversine-filtered to 1000 m, then scored with the same function as a single point, including `local_hour`.
9. The cell's `incident_count` is the number of incidents **in the bin**, not the number used in the score.
10. The loop stops at 3000 cells.

If the view radius contains no incidents, `cells` is `[]`. The comment in the function says a previous fallback put a marker on the viewport center and that marker followed pan and zoom, so it was removed. On an exception, the `except` builds one zero-risk cell at the center so the response is not empty.

`include_clusters` defaults to false. The mobile route passes false. The admin route passes true. When true, `get_clusters()` output is attached with **no geographic filter**. An admin query centered on Chennai still receives every cluster computed from the whole table.

`include_time_factor` is a query parameter on `GET /ml/heatmap` and is not read inside `generate_heatmap`. Time is applied whenever `local_hour` is passed, which the route always resolves to some integer.

### 14.8 DBSCAN, the only learned partition

`get_clusters` in `backend/ml/app/ml/clustering.py`:

```python
clustering = DBSCAN(
    eps=settings.dbscan_eps,          # 0.001
    min_samples=settings.dbscan_min_samples,  # 3
    metric="euclidean",
)
cluster_labels = clustering.fit_predict(coordinates)  # shape (n, 2) of [lat, lng]
```

Inputs are raw degrees, not projected meters, not a haversine metric. `eps = 0.001` degrees. The config comment says "~100 meters". At the equator, 0.001° of latitude is about 111 m. At 13° N, 0.001° of longitude is about 108 m. The epsilon neighborhood is therefore an ellipse in ground distance, about 111 m north-south and 108 m east-west. Points farther apart than that, even if they sit in the same named area in the mock generator, become separate clusters or noise.

`min_samples = 3`. Fewer than 3 incidents in the whole table yields `[]` without calling DBSCAN. Label `-1` is noise and is discarded. For each remaining label:

- Center is the arithmetic mean of member latitudes and longitudes.
- Radius is the max Euclidean distance in **degrees** from that mean, times `111000`. East-west extent is converted with the latitude scale, so the reported meter radius is a bit larger than haversine for an east-west chain.
- `incident_count` is the number of member rows.
- `risk_score` is a fresh `calculate_risk_score` at the center, with the server's current hour, not an aggregate of member severities.
- Id is `cluster_<sklearn label>`.

`update_clusters_if_needed`, called on every successful `POST /ml/incidents/process`, sets the module-level cache to `None` and returns `[]`. It does not refit. The next `get_clusters()` refits. The cache is per process. `affected_zones` in the process-incident response is that empty list. `model_updated` is hardcoded `False` in the route.

DBSCAN here is not a safety model. It is a grouping of points for the admin heatmap page and for the optional `clusters` array. The mobile map does not draw those circles. `HeatmapMap.android.tsx` has a comment that cluster markers were removed.

### 14.9 Routes are a reduction over the same point score

`analyze_routes` in `backend/ml/app/ml/route_analyzer.py` does not call Google. Express has already turned a Google encoded polyline into waypoints (`decodePolyline` in `googleMapsService.ts`, scale `1e-5`).

For each consecutive pair:

- Segment length is haversine with Earth radius `6371000` (the same formula as `app/utils/geospatial.py`, copied into this file).
- Risk is `calculate_risk_score` at the arithmetic midpoint `(lat1+lat2)/2`, `(lng1+lng2)/2`. That is not the geodesic midpoint. Each call issues its own `ST_DWithin` for 1000 m. A polyline with *N* vertices costs *N − 1* risk queries. `local_hour` is not passed.
- If segment risk `>= 3.5`, the segment is stored in `high_risk_segments`.

Route scores:

```python
avg_risk = weighted_risk_sum / total_distance          # weighted by segment length
safety_score = max(0.0, 1.0 - (avg_risk / 5.0))
safe_distance = total_distance                         # comment: "Simplified for now"
```

`safe_distance` is not the length of segments under a threshold. The API still returns it, and the mobile `SafeRoute` type still has `safeDistance`.

The recommended id is chosen in the FastAPI handler, not in the analyzer:

```python
recommended = max(routes_analysis, key=lambda r: r.safety_score).id
```

Ties keep the first maximum `max` finds, which is list order. Express builds route ids `route_primary`, then `route_alternative_1`, and so on, in Google response order. The primary route wins a tie.

If Google returns nothing or throws, `getRouteOptions` returns one synthetic route, `route_direct`, whose only waypoints are the start and the end. The analyzer then scores a single midpoint of the straight line. The user still gets a safety number. It is the score of that midpoint, not of a road.

### 14.10 Land mask is a filter in front of scoring, and it is off

`backend/ml/app/utils/land_mask.py` implements ray-casting point-in-polygon over every polygon in a GeoJSON feature collection, with a bounding-box reject before the ray cast, and with holes excluded. The file on disk, `backend/ml/app/resources/landmask/chennai_wards.geojson`, is a `FeatureCollection` with 201 features. If the file were missing, the loader would download the Datameet Chennai wards URL stored in settings.

`settings.land_mask_enabled` is `False`. `is_point_allowed` returns `True` before it opens the file. The module docstring says Shapely union was avoided because of a Shapely/NumPy failure in the environment they were using. `shapely` remains in `requirements.txt` and is not imported. If the flag is turned on and the file fails to parse, the function fails open (returns true) so heatmap generation continues.

Masking drops cells. It does not drop incidents from the database, and it is not applied inside `calculate_risk_score`. A point in the sea can still receive a risk score from the single-point API.

### 14.11 What happens at ingest, which is the only "learning"

`POST /ml/incidents/process` → `add_incident`:

- Inserts the row.
- Computes `incident_local_hour` as `(timestamp + timedelta(minutes=offset)).hour` when the client sent `timezone_offset_minutes`, otherwise `timestamp.hour`.
- Builds `location` with `ST_MakePoint(longitude, latitude)` cast to `geography`.
- Does not update weights.
- Invalidates the DBSCAN cache.

The next heatmap or risk call reads the new row if it falls inside the radius. That is the entire online update. There is no minibatch, no decay table, no feature store separate from `incidents`.

### 14.12 Complexity the service actually pays

For one heatmap request:

- Two `ST_DWithin` queries (view radius, and view radius + 1000 m).
- One Python score per occupied bin, each walking the incidents in neighboring bins. No extra SQL per cell, because the neighborhood is sliced from the context list. This is why heatmap cells call `_calculate_risk_score_from_incidents` directly instead of `calculate_risk_score`.
- If `include_clusters` is true, one additional full-table `SELECT` inside `get_incidents()` and one DBSCAN fit, plus one radius query per cluster center for that center's risk score.

For one route with *S* segments: *S* radius queries, run sequentially inside the async function. The function is `async` but the body does not `await` anything. The SQL calls are synchronous psycopg2.

For `GET /ml/incidents/all`: one unbounded `SELECT` of every incident, then a Python filter and slice. Admin analytics asks for `limit=1000`. The cap is applied after the full read.

### 14.13 Knobs, and the ones that are dead

Live knobs, all overridable by environment variables because `Settings` is a `BaseSettings` with `case_sensitive = False`:

| Env-style name | Default | Effect |
| --- | --- | --- |
| `WEIGHT_INCIDENT_DENSITY` | 0.5 | Blend |
| `WEIGHT_RECENCY` | 0.25 | Blend |
| `WEIGHT_SEVERITY` | 0.15 | Blend |
| `WEIGHT_TIME_PATTERN` | 0.1 | Blend |
| `MAX_EXPECTED_EFFECTIVE_INCIDENTS` | 10 | Density saturation |
| `RECENCY_DECAY_DAYS` | 30 | Age kernel |
| `TIME_OF_DAY_SIGMA_HOURS` | 3 | Hour kernel width |
| `TIME_OF_DAY_MIN_WEIGHT` | 0.05 | Hour kernel floor |
| `DBSCAN_EPS` | 0.001 | Cluster reach in degrees |
| `DBSCAN_MIN_SAMPLES` | 3 | Cluster minimum |
| `LAND_MASK_ENABLED` | false | Polygon filter |
| `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASSWORD`, `DB_SSL` | localhost, 5433, `women_safety_db`, postgres, empty, false | Storage |

Declared and unused: `MODEL_PATH`, `CACHE_TTL`, `DEFAULT_GRID_SIZE`, `DEFAULT_RADIUS`, `AUTO_RETRAIN_THRESHOLD`, `MIN_RETRAIN_INTERVAL`. Grid size and radius are request parameters, not these defaults. The heatmap route defaults are `radius=1000` and `grid_size=100` on the FastAPI signature itself.

### 14.14 Invariants an ML change has to preserve

These are consequences of the current callers, not extra requirements:

- Scores must stay inside `[0, 5]`. Pydantic `HeatmapCell` and `RiskScoreResponse` declare `risk_score` with `ge=0, le=5`. A formula that returns 5.01 fails the response model.
- `risk_level` strings the UI branches on are `very_safe`, `safe`, `medium`, `high`, `very_high`. The Android painter does not switch on the string. It switches on numeric bands at 1, 2, and 4, and it hides `<= 1`. Changing level names without changing numbers does not change the map. Changing numbers does.
- Cell objects need `lat`, `lng`, `risk_score`, `risk_level`. `incident_count` and `last_incident` are present. The phone's `HeatmapCell` TypeScript type only requires `lat`, `lng`, `risk_score`, `risk_level`.
- `safety_score` is documented in the schema as 0–1, higher safer. The phone multiplies by 100 for display and colors routes at 0.7 and 0.4.

---

## 15. Data science, from the person who has to trust the numbers

This section is the same pipeline read as a dataset and a measurement problem. The question the running code answers is: given the incidents stored so far, how elevated is this coordinate at this local hour, on a fixed 0–5 scale? It does not answer "what is the probability of a future incident", and it does not answer "was this report true".

### 15.1 The observation

One row in `incidents` is one observation. There is no separate label table. The score is a function of the rows, so the rows are simultaneously the history and the only supervision. Holding out incidents and asking whether the score "predicts" them is not implemented. Nothing in the repo computes precision, recall, ROC, log loss, or a calibration curve.

Fields a data scientist can use, because they are stored and returned by `get_incidents`:

| Field | How it is produced for a live phone report | How it is produced for `step2` mock rows |
| --- | --- | --- |
| `latitude`, `longitude` | User-picked marker or panic GPS | Random offset around a named Chennai center, then clamped to the generator bounds |
| `timestamp` | Phone `new Date().toISOString()` | `datetime.now(timezone.utc) - random days`, then `.replace(hour=chosen_hour)` |
| `timezone_offset_minutes` | `-getTimezoneOffset()` from the phone (minutes east of UTC; IST is +330) | Not set. The generator never passes the field. |
| `incident_local_hour` | Hour of `timestamp + offset` at insert | `timestamp.hour`, which is the hour the generator chose, stored on a UTC timestamp |
| `type` | `community_report`, or `panic_alert` for SOS | Function of severity: severity ≤ 2 always community; severity 3 is 20% panic; severity ≥ 4 is 40% panic |
| `severity` | User button 1–5, or 5 for panic | Function of area base severity, a time multiplier, noise, then extra resampling |
| `category` | One of five Title Case phone labels | Lowercase `harassment`, `assault`, `suspicious`, `other` |
| `verified` | `false` for reports, `true` for panics | `random.random() < 0.6` |
| `user_id` | The literal `user123` | `user_` plus an integer from 1 to 200 |
| `moderation_reason` | Null until an admin verify/reject | Null |

`description` is collected on the phone and required by Express. It is not a column. Free text never enters analysis. Media picked with `expo-image-picker` never leaves the device.

`verified` and `moderation_reason` change what the admin tables show. They do not change `get_incidents_in_radius`. A data scientist who treats "verified" as a clean analysis set has to filter in their own query. The scoring code will not.

### 15.2 Time, and the two clocks

Live path, the one that matches the comments in the mobile client:

1. The phone sends an ISO UTC timestamp and `timezone_offset_minutes = -Date.getTimezoneOffset()`.
2. It also sends `local_hour = Date.getHours()` on heatmap fetches.
3. Insert stores `incident_local_hour` from timestamp plus offset.
4. Later heatmap queries pass the viewer's current `local_hour`.
5. The Gaussian compares that hour to each row's `incident_local_hour`.

Mock path, `generate_chennai_incidents` / `_generate_incident_in_area`:

```python
base_time = datetime.now(timezone.utc) - timedelta(days=days_ago)
hour = _generate_risk_aware_hour(risk_level)
timestamp = base_time.replace(hour=hour, minute=random.randint(0, 59))
```

The chosen hour is written into the UTC timestamp. No offset is attached. After insert, `incident_local_hour` equals that chosen hour. Scoring that prefers the column will treat the generator's hour as "local". A human reading `timestamp` in IST will see a clock 5 hours 30 minutes later than the hour the generator thought it was assigning. The column and the UTC timestamp disagree with India local time by the offset the generator forgot to set. Analyses that recompute local hour as `timestamp + 330 minutes` will not match `incident_local_hour` for these rows. Analyses that trust the column will.

`HIGH_RISK_DAYS = [4, 5]` is defined in `chennai_mock_data.py` and is not read. Weekday is not a feature of the generator or of the scorer.

### 15.3 How the synthetic incidents are actually placed

The generator's own docstring says the named areas are assumptions for test data and that the scoring code does not receive the area names. That is true. The names exist only in the Python lists `HIGH_RISK_AREAS`, `MEDIUM_RISK_AREAS`, and `LOW_RISK_AREAS`. Once inserted, a row is indistinguishable from a phone row except by id prefix (`chennai_hr_`, `chennai_mr_`, `chennai_lr_`) and by the missing timezone offset.

Batch mix inside `generate_chennai_incidents`: `int(count * 0.40)` high, `int(count * 0.35)` medium, the remainder low. For 5000 that is 2000, 1750, and 1250. `step2_load_data.py` truncates the table and loads 5000. The printout divides by 426 and calls the result incidents per km². The number 426 is a literal in that print and in `load_chennai_mock_data_into_storage`. It is not computed from the ward GeoJSON.

Placement inside an area:

```python
radius_degrees = area["radius_km"] / 111.0
angle = random.uniform(0, 2 * 3.14159)          # computed, never used
distance_degrees = random.uniform(0, radius_degrees)
lat = area["lat"] + distance_degrees * random.uniform(-1, 1)
lng = area["lng"] + distance_degrees * random.uniform(-1, 1)
```

The unused `angle` means this is not a uniform disk. Each coordinate is perturbed by `U(-d, d)` with `d` uniform in `[0, radius]`. The point cloud is a fuzzy square around the center, denser near the center because small `d` is as likely as large `d` and then gets multiplied by another uniform. After that, latitude is clamped to `[12.85, 13.20]` and longitude to `[80.10, 80.35]`.

Named centers the high tier samples from, with the generator's radius and base severity:

| Name in the generator | Lat | Lng | Radius km | Base severity |
| --- | --- | --- | --- | --- |
| Central Railway Station | 13.0827 | 80.2707 | 0.5 | 4 |
| Egmore Railway Station | 13.0790 | 80.2606 | 0.4 | 4 |
| Marina Beach (Night Area) | 13.0475 | 80.2825 | 1.0 | 4 |
| Koyambedu Bus Stand | 13.0710 | 80.1980 | 0.6 | 4 |
| T. Nagar Market Area | 13.0418 | 80.2341 | 0.8 | 3 |

Medium tier: Anna Nagar (13.0850, 80.2100, 1.2 km, base 3), OMR IT Corridor (12.9716, 80.2200, 2.0 km, base 2), Guindy (12.9716, 80.2206, 1.5 km, base 2). Low tier: Adyar Residential (13.0067, 80.2206, 2.0 km, base 1), Besant Nagar (12.9990, 80.2640, 1.5 km, base 1).

Hour draw, `_generate_risk_aware_hour`:

| Tier | High hours 22, 23, 0, 1, 2, 3 | Medium hours 18–21 | Low hours 6–17 |
| --- | --- | --- | --- |
| high | 0.60 | 0.30 of the remainder is not quite 0.30; the code draws high with probability 0.60, then with probability 0.75 of the **remaining** draw picks medium. That is `0.40 * 0.75 = 0.30` medium and `0.10` low. | 0.10 |
| medium | 0.40 | 0.40 | 0.20 |
| low | 0.20 | 0.30 | 0.50 |

The medium-tier branch uses one `random.random()` and two thresholds, so its probabilities are exactly 0.40 / 0.40 / 0.20. The high-tier branch draws a second random after the first succeeded at `< 0.60`, so the comment "30% of remaining 40%" matches the arithmetic only in expectation if the second draw's 0.75 is applied to the complement. It is. The low tier is a single draw with cuts at 0.20 and 0.50.

Severity, `_generate_severity`: start from `base_severity`, multiply by 1.2 if the chosen hour is in the high-hour list, 1.0 if in the medium list, 0.8 otherwise, add `U(-0.5, 0.5)`, round, clamp to 1–5. Then, if the tier is high and the result is below 3, with probability 0.80 replace it by a draw from `{3, 4, 5}` with weights 0.3, 0.4, 0.3. If the tier is low and the result is above 3, with probability 0.70 replace it by `{1, 2, 3}` with the same weights. The medium tier has no second resampling.

Category given severity: severity ≥ 4 chooses harassment / assault / suspicious at 0.5 / 0.3 / 0.2. Severity 3 chooses harassment / suspicious / other at 0.4 / 0.4 / 0.2. Below that, suspicious / other / harassment at 0.4 / 0.4 / 0.2. The phone's categories are `Harassment`, `Suspicious Activity`, `Poor Lighting`, `Unsafe Area`, `Other`. The strings do not match. Admin "by category" will show both vocabularies if both sources have been loaded. The score ignores the string either way.

`app/data/mock_data.py` is a second generator centered on `(40.7128, -74.0060)` with a 5 km radius and three hand-picked cluster offsets. Sixty percent of its points are near those offsets. `load_mock_data_into_storage` inserts through the same `add_incident`. The step scripts do not call it. `app/data/storage.py` is an in-memory list the live routes do not import.

### 15.4 What gets measured, and what those scripts refuse to claim

There is no experiment tracker. The measurements that exist are print statements.

`step2_load_data.py` after the load: counts by `type`, counts by severity 1–5, verified versus not, min/max latitude and longitude, the 426 km² density print, and a bounding-box count around Central Station using `±0.009` degrees (the comment says ~1 km) through `get_incidents`, which is a lat/lng rectangle, not `ST_DWithin`.

`step3_test_ml_algorithms.py`:

- `get_clusters(force_recalculate=True)`.
- Prints the ten largest clusters by `incident_count`, then sum of member counts, mean cluster size, mean risk, and `incident_count - total_clustered_incidents` as "scattered". That subtraction treats noise points as scattered. It is not a second DBSCAN statistic; noise points were never given a cluster id.
- Scores five fixed coordinates with `calculate_risk_score` and prints the four factors.
- "Verifying model learning" takes the first five clusters and the test locations whose score is `>= 3`, and counts a match when Euclidean distance in degrees is `< 0.005` (the comment says ~500 m). It prints a success line if `matches > 0`. This is not a statistical test. It does not use a holdout. The locations it calls high-risk are the same coordinates the generator was aimed at.

`step4_test_heatmap.py` calls `generate_heatmap` with no `local_hour` for three centers: Marina Beach radius 1000 m grid 100 m, Central Station radius 800 m grid 100 m, Koyambedu radius 600 m grid 100 m. Because `include_clusters` defaults to false, the `clusters` list in that result is empty even though the script prints "Clusters found". The cell statistics it does compute, when cells exist, are mean, min, max of `risk_score`, and counts of the five `risk_level` strings, plus a list of cells with score `>= 3.5`.

`step5_test_routes.py` builds four hand-specified polylines (through Central Station, around it, through Marina, through Adyar/Besant Nagar) and prints safety, risk, distance, and segments with risk `>= 3.5`. The safety bands it prints (`>= 0.8` very safe, `>= 0.6` safe, `>= 0.4` moderate, `>= 0.2` high, else very high) exist only in that script. The product uses `safety_score` as a number. The phone's colors are 0.7 and 0.4, which are different cuts.

`test_risk_optimization.py` scores six coordinates, prints the factor breakdown, the min, max, mean, and a population standard deviation (divide by *n*, not *n − 1*), and then classifies the range `max - min`: above 2.0 "Good variation", above 1.0 "Moderate", otherwise "Low variation". Those words are not thresholds inside `risk_scoring.py`.

`test_full_integration.py` checks the pool, the count, clustering, one risk call, one heatmap, and one route analysis. It does not call Express. Its docstring says it tests the backend API. The body does not.

`test_mock_data.py` generates incidents in memory and does not need Postgres.

None of these scripts write a result file. Re-running `step2` deletes every live incident in the table (`clear_incidents` is `TRUNCATE`).

### 15.5 The admin aggregates are descriptive, and they are censored

`GET /api/admin/analytics` in `backend/api/src/routes/admin.ts` is the only dashboard-style measurement in the product.

- It asks FastAPI for 1000 incidents, newest first (`ORDER BY timestamp DESC` inside `get_incidents`).
- It keeps rows with `timestamp >= cutoff`, where cutoff is local server midnight-ish: `setDate(today - days)` then `setHours(0,0,0,0)` on the Node process clock.
- Day buckets are `toISOString().slice(0, 10)`, which is a **UTC** date. A report at 02:00 IST on Tuesday is still Monday UTC. The chart label parses that UTC date with `new Date(d.date)` in the browser, which interprets a date-only string as UTC midnight and can shift the label again in IST.
- `byHour` uses `incident_local_hour` only. Null hours are dropped, not bucketed.
- `byCategory` uses the raw string, or `uncategorized`.
- Verification rate is `verified / incidentsInPeriod`, one decimal. It is not a rate over a population of users. There is no user dimension in the chart.
- `totalIncidents` is the full SQL count. `incidentsInPeriod` cannot exceed 1000, and if more than 1000 incidents exist, older ones in the window are missing because the 1000 are the newest overall, not the newest inside the window. A 30-day window on a table of 5000 recent mock rows is computed from 1000 rows.

`GET /api/admin/dashboard` computes "today", panic count, and verified count on a page of 100, while `totalIncidents` is the unpaged total. Those cards are not population totals. Section 5.10 records the same fact; it matters here because the cards look like data-science outputs and are not.

`GET /api/admin/audit` is not an event log. It filters the newest 100 rows to those with a moderation reason or `verified === true`, keeps 50, and labels the action `verified` or `rejected` from the boolean. The timestamp shown is the incident time, not the time of the moderation update. `updated_at` is maintained by the database trigger and is not selected in `get_incidents`.

### 15.6 Leakage, in the narrow sense that applies here

There is no train/test split to leak across. The generator's area list is still a circular story if someone treats a high score at Central Station as evidence the model discovered Central Station. The score is high there when the inserted points are dense, recent, severe, and aligned in hour with the query. The generator put dense, severe, night-weighted points there. `step3` then checks whether clusters land near those same coordinates. A match supports "DBSCAN grouped the points we inserted close together". It does not support an independent geographic finding.

The ward polygons are not used as features while the mask flag is off. Turning the mask on changes which cells are returned. It does not change the stored sample.

### 15.7 A usable analysis frame, using only columns that exist

Someone continuing this work from the current schema can group by `incident_local_hour`, `severity`, `type`, `category`, `verified`, and by spatial bins they compute themselves. They cannot recover the typed report text. They cannot recover whether two `user123` rows are the same person, because every live phone row uses that id. They can separate mock rows by the `chennai_` id prefix until a truncate. They should not interpret `incident_local_hour` on mock rows as IST. They should not interpret admin "today" as a full-table count. They should not interpret `safe_distance` as a measured safe length.

The second mock generator's New York center will mix with Chennai rows if `load_mock_data_into_storage` is run against the same database. Nothing partitions by city.

---

## 16. Fullstack, from the person who ships the screens and the API

This section is the system as the person who owns `frontend/mobile`, `frontend/admin`, and `backend/api` has to walk it. FastAPI is a dependency they call. They do not reimplement the score. They do own the contract, the auth gate, the realtime fan-out, the map, and every place a failure is turned into a 200.

### 16.1 Runtime and repositories inside the repo

Three Node packages, one Python package. No workspace file ties them. Lockfiles are gitignored (`package-lock.json` and `yarn.lock` in `.gitignore`).

| App | Start | Dev URL the code expects |
| --- | --- | --- |
| `backend/ml` | `uvicorn app.main:app` or `python -m app.main` | `0.0.0.0:8000` |
| `backend/api` | `npm run dev` → `nodemon src/app.ts` | `0.0.0.0:3001` |
| `frontend/admin` | `npm run dev` | `localhost:5174`, proxy `/api` → `http://192.168.1.12:3001` |
| `frontend/mobile` | `npm start` → Expo | API base `http://192.168.1.12:3001` when `__DEV__` |

`backend/api/tsconfig.json` compiles `src` to `dist` as CommonJS, ES2020, `strict`, `exactOptionalPropertyTypes`, `noUncheckedIndexedAccess`. `npm start` runs `node dist/app.js`, so a production start needs `npm run build` first. `npm run dev` uses nodemon on TypeScript directly.

Admin TypeScript is bundler mode, `noEmit`, JSX `react-jsx`. Mobile TypeScript extends `expo/tsconfig.base` and sets `strict`.

Express 5 is the HTTP framework. The 404 handler is a non-error middleware with no path, registered after the routers. The error handler is four arguments and reads `err.status` and `err.message`.

### 16.2 Screen map and the calls each screen makes

Mobile tabs are declared only in `frontend/mobile/src/navigation/AppNavigator.tsx`. `App.tsx` renders that navigator and nothing else.

| Tab | Component | Network |
| --- | --- | --- |
| Home | `HomeScreen` → `HeatmapMap.android` or `.ios`, else `HeatmapMapFallback` | `GET /api/location/heatmap`. Socket join for the current center and radius. Listens for `incident:new` and refetches after 2 seconds. Also refetches every 60 seconds. |
| Panic | `PanicScreen` + `PanicButton` | `POST /api/panic/trigger` once. Then `POST /api/location/update` on a 30 second timer whose closure is described in section 6.10. Contacts are not a network call. |
| Report | `ReportsScreen` + `LocationPicker` | `POST /api/reports/submit`. Image picker is local. |
| Community | `CommunityReportsScreen` + `ReportDetailsScreen` | `GET /api/reports/all`. Socket `subscribe:incidents`, refetch after 1.5 seconds. "View on map" is a Google Maps URL via `Linking.openURL`, not a navigation to Home. |
| Routes | `RoutePlanningScreen` | `GET /api/location/search-places`, `GET /api/location/place-coordinates`, `GET /api/location/safe-routes`. Directions handoff is another Google Maps URL. |

`ContactsScreen` duplicates the contact editor and is not mounted. `SimpleMap` is not imported. The Home info button's `onPress` is empty. The address line under the map is the constant string `Connaught Place, New Delhi`. The map's first camera target, in `HeatmapMap.android.tsx`, is hardcoded Chennai `13.0827, 80.2707` with `latitudeDelta` 0.15 even after GPS succeeds. GPS is stored as a marker. It does not become the heatmap center unless the user pans or a `panToLocation` param arrives.

Admin routes are in `frontend/admin/src/App.tsx`. Login is `/`. Everything under `/admin` checks that `localStorage.admin_token` is non-empty. It does not decode the JWT. Expiry is enforced only when an API call returns 401, which clears the token and sets `window.location.href = '/'`.

| Page | Client method in `src/api/client.ts` | Express route |
| --- | --- | --- |
| Dashboard | `admin.dashboard()` | `GET /api/admin/dashboard` |
| Moderation | `admin.incidents`, `admin.verify`, `admin.reject` | `GET /api/admin/incidents`, `POST .../verify`, `POST .../reject` |
| Heatmap | `admin.heatmap(lat, lng, radius, 100)` | `GET /api/admin/heatmap` |
| Analytics | `admin.analytics('7d' or '30d')` | `GET /api/admin/analytics` |
| Audit | `admin.audit()` | `GET /api/admin/audit` |

### 16.3 Auth, and the hole next to it

Admin login, `POST /api/auth/admin-login`, compares plaintext to `ADMIN_EMAIL` / `ADMIN_PASSWORD` / `ADMIN_SECRET`, with defaults that accept user `admin` and password `admin`. `signAdminToken` uses `jsonwebtoken` and `JWT_SECRET`, falling back to `ADMIN_SECRET`, then to the string `change-me-in-production`. Payload is `{ sub, role }`, expiry `7d`. `requireAdmin` is mounted on the whole `/api/admin` router. It checks the `Bearer` prefix and `role === "admin"`.

`POST /api/auth/login` returns the literal token `mock_jwt_token_here`. The mobile app does not call it. No mobile request sets `Authorization`. Panic, reports, heatmap, and routes are open on the LAN the process binds (`0.0.0.0`).

`firebase-admin` is in `backend/api/package.json` and is not imported. `@react-native-firebase/auth`, `@react-native-firebase/app`, and `firebase` are in the mobile `package.json` and are not imported under `src`. `expo-secure-store` is an Expo plugin and is not imported. Contacts go through AsyncStorage, or through a process-memory object if that require throws (`frontend/mobile/src/utils/storage.ts`).

The user id sent by panic and by reports is the string `user123` in source. The report screen marks that with a TODO.

### 16.4 The JSON the phone sends, and the JSON that gets stored

Report submit body built in `ReportsScreen.tsx`:

```json
{
  "userId": "user123",
  "type": "community_report",
  "category": "Harassment",
  "description": "<typed text>",
  "severity": 3,
  "location": { "latitude": 13.08, "longitude": 80.27 },
  "timestamp": "<ISO>",
  "timezone_offset_minutes": 330
}
```

Express requires `description`, then builds an ML body without it: `id`, `latitude`, `longitude`, `timestamp`, `type`, `severity`, `category`, `verified: false`, `user_id`, and the offset if present. FastAPI `IncidentRequest` rejects a `type` outside `panic_alert|community_report`, a severity outside 1–5, and an offset outside −840..840.

Panic body from `triggerPanicAlert` in `api.ts`: `userId`, `location.{latitude,longitude}`, `emergencyContacts` as an array of phone strings, `timestamp`, `timezone_offset_minutes`. Express adds severity 5, category `emergency`, type `panic_alert`, `verified: true`. The phone numbers are counted for the response field `contactsNotified` and then dropped. No SMS library is in `package.json`.

Heatmap query from `fetchHeatmap`: `lat`, `lng`, `radius`, `grid_size`, `timestamp`, `local_hour`, `timezone_offset_minutes`. Express parses them and calls `getHeatmap(..., includeClusters=false)`. The ML query also sends `include_time_factor: true`.

Safe-route query: four numbers. Express adds Google waypoints and ML scores, then returns `safetyScore`, `riskScore`, `distance`, `safeDistance`, `highRiskSegments`, `waypoints`, `duration`, `instructions`, and `recommendedRoute`.

### 16.5 Map behavior the fullstack layer invented on top of the score

`HeatmapMap.android.tsx` decides **when** to ask, not how to score.

- Home passes `radius={10000}` and `gridSize={200}`. Inside the map those are a maximum radius and a coarsest grid.
- `onRegionChangeComplete` waits 450 ms, then sets the query center to 4 decimal places.
- Radius is half the longer screen edge in meters, times 1.1, rounded to the nearest 250 m, clamped to `[1000, 10000]`. Meter conversion uses `111000 * cos(latitude)` for longitude, which the Python heatmap box does not.
- Grid tier is 50, 100, or 200 with hysteresis. Leaving 50 m needs radius `> 3000`. Entering 50 m from 100 m needs radius `< 2500`. Entering 200 m needs radius `> 8000`. Leaving 200 m needs radius `< 7000`.
- A request is skipped when `lat:lng:radius:grid` is unchanged.
- Responses carry a sequence number. A late response from an older pan is ignored.
- Drawn circles: `risk_score > 1`, at most 1000 of them. Radius of each circle is `gridSize * 0.7` meters. Color is a hand-written RGB lerp: 1–2 yellow `#FACC15` toward amber `#EAB308`, 2–4 orange `#F97316` toward red `#EF4444`, 4–5 red toward `#B91C1C`. Opacity is `0.22 + (score/5)*0.48`.
- The fallback list (`HeatmapMapFallback.tsx`) uses different cuts: `>= 4` red, `>= 2.5` orange, `>= 1.5` yellow, else green. The mobile README's legend matches the fallback, not the native lerp.

Route polylines: selected route uses theme primary `#5B9BD5`. Others are green `#4CAF50` at safety `>= 0.7`, orange `#FF9800` at `>= 0.4`, else red `#F44336`. Opening Google Maps directions samples the polyline down to 23 intermediate points because the URL the screen builds is the public maps directions URL, not the Routes API.

Two Google keys exist and they are not the same channel. The app key is a literal in `frontend/mobile/app.json` for the native map tiles. The server key is `process.env.GOOGLE_MAPS_API_KEY`, default empty, used for Routes, Places Autocomplete, and Geocoding. An empty server key still returns the straight-line fallback from `getRouteOptions`, so route planning fails soft into a two-point segment rather than throwing inside the Google client. Places search returns `[]` on `REQUEST_DENIED`. The route screen then shows no suggestions.

### 16.6 Realtime, room by room

Server, `setupWebSocket`:

- Client emits `subscribe:location` with `{lat, lng, radius}`. Room name is `location:` + lat to 4 decimals + `:` + lng to 4 decimals + `:` + radius. The radius is stringified as the client sent it. The mobile client sends the integer meter radius.
- Client emits `subscribe:incidents` and joins `incidents:all`.
- `emitNewIncident` writes `timestamp` to "now", dropping the incident timestamp that the caller passed. It emits to `incidents:all` and to every location room whose center is within that room's radius of the point. Distance is haversine with radius `6371000`, duplicated again in this file rather than shared with Python.

Client, `websocket.ts`:

- `socket.io-client` is loaded with `require`. `metro.config.js` adds `mjs` and `cjs` and turns on package exports so that require resolves.
- Transports start at `polling` and may upgrade. Connect failure is logged once. The UI keeps using HTTP.
- `incident:new` is deduplicated for 10 seconds per `incidentId`, because the server can deliver the same event on the global room and on a location room.
- The heatmap's `subscribeToLocation` callback sets heatmap state from `data.heatmap`. The server never emits `heatmap:update`, so that callback does not run. Refresh of colors is the HTTP refetch.
- Home's heatmap effect does not disconnect the socket on unmount. The community screen's effect does call `disconnectWebSocket` on unmount. Whichever screen unmounted last wins. That is the code as written.

### 16.7 State the user can actually lose

| State | Where it lives | What clears it |
| --- | --- | --- |
| Admin JWT | `localStorage.admin_token` | Logout, or any 401 |
| Emergency contacts | AsyncStorage `@emergency_contacts` | Delete button, or a process restart if AsyncStorage failed to load and the memory fallback was used |
| Report draft | React state in `ReportsScreen` | Successful submit, or leaving the component |
| Heatmap cells | React state in the map component | Next successful fetch replaces them |
| Panic "active" | `useState` in `PanicScreen` | Cancel confirm, or remount. The server status route always returns `hasActivePanic: false`. |
| Selected route | React state in `RoutePlanningScreen` | A new search |

There is no global store (no Redux, no Zustand, no React Query). Each screen fetches in `useEffect`.

### 16.8 Error strings the UI will show

- Map load throws: Android overlay prints the error and the sentence `Make sure ML service is running: cd backend/ml && uvicorn app.main:app`, plus Retry.
- Location permission denied on the Android map: `Location permission denied. Using Chennai center.`
- Report validation: alerts for missing category, description, or location.
- Image picker missing: alert that a development build (`npx expo run:android`) is required.
- Route search under 2 characters: no request. Failure: `Failed to search locations...`.
- Admin pages: the `Error:` string from `data.error` or `data.message`, which is what Express puts in the JSON.

Express heatmap success with zero cells is still `success: true`. The map then draws nothing and does not treat it as the error overlay. Quiet data and a dead ML service look the same on the map, except the dead path also sets `warning` on the JSON, which the phone does not read. The phone only checks `response.success && response.heatmap`.

### 16.9 A change, and every file it touches

These are the paths that already exist, so a fullstack change stays honest about the blast radius.

**Show the typed description in the community list.** Today the list text is built in `backend/api/src/routes/reports.ts` from `category`. The column does not exist. The chain is `ReportsScreen` body (already sends it) → `reports.ts` (must keep it) → `IncidentRequest` → `INSERT` → migration → `get_incidents` select list → the mapper in `GET /api/reports/all` → `CommunityReport.description` → `ReportDetailsScreen`. Until that chain exists, editing only the React text will not persist.

**Stop scoring rejected rows.** The filter is not in Express. It is the SQL in `get_incidents_in_radius` and `get_incidents` inside `backend/ml/app/db/storage.py`. Admin verify/reject already sets the boolean.

**Make panic show up on the heatmap immediately.** Insert already affects the next score. The missing piece on the API side is that `panic.ts` does not call `emitNewIncident`. The heatmap's 60 second poll and the next pan will still pick the row up without a socket event.

**Notify the phones in `emergencyContacts`.** No file sends a message. The response field `actions.contactsNotified` is `emergencyContacts.length`. A real sender would be new code in `panic.ts` plus a provider dependency that is not in `package.json` today. The contact list itself is already saved on device in `PanicScreen`.

**Protect mobile routes.** `requireAdmin` is the only middleware of its kind, and it is mounted only at `/api/admin`. Mobile routes are registered in `app.ts` with no guard. Login would have to stop returning `mock_jwt_token_here` in `auth.ts` before a guard would have a real token to check. The phone would have to store that token and send it from `src/services/api.ts`, which currently sets only `Content-Type`.

**Point the stack at a machine that is not `192.168.1.12`.** The host is duplicated, not read from one config: `frontend/mobile/src/services/api.ts`, `frontend/admin/vite.config.ts`, `backend/api/src/services/mlService.ts`, `backend/api/src/routes/reports.ts` (a second ML URL, separate from the axios client), `backend/api/src/app.ts` log lines, and the two Node test scripts. The ML database host is a different setting, `db_host` in `backend/ml/app/config.py`, default `localhost`.

### 16.10 Build and native surface

`frontend/mobile/app.json`: portrait, `newArchEnabled: true`, iOS bundle and Android package `com.womensafety.mobile`, location usage strings, Android permissions `ACCESS_FINE_LOCATION`, `ACCESS_COARSE_LOCATION`, `CAMERA`, `READ_EXTERNAL_STORAGE`, `WRITE_EXTERNAL_STORAGE`. The image-picker plugin adds its own photo and camera purpose strings. `eas.json` builds Android APKs for `development` (dev client), `preview`, and `production`. EAS project id `a0688f1d-8023-4486-9a61-22c5aa9fa70b` is in `app.json` `extra.eas`.

Maps on Android use `PROVIDER_GOOGLE` from `react-native-maps` 1.20.1. iOS uses `HeatmapMap.ios.tsx`. Web and a failed native require use the list fallback. Expo SDK is 54, React Native 0.81.5, React 19.1.0.

Admin has no map library. Heatmap is inputs plus tables. Analytics bars are divs whose pixel height is `(count / maxCount) * 180`, with a minimum of 8 pixels when count is positive. There is no chart dependency in `frontend/admin/package.json`. The dependencies there are React, React DOM, and React Router. Vite is the dev server and the bundler.

Express security middleware is `helmet()` and `cors({ origin: "*", credentials: true })`. Socket.IO uses the same open origin. Body JSON is limited to 10 mb, which is larger than any payload the current clients send. The report body is small because media is not attached.

---
