# GIS layers and the eOSC+ meeting map

Implemented increment: eOSC+ Plan 2 (map), 2026-10-29, on branch `feat/eosc-map` (not yet merged; built on `feat/eosc-foundation`). Decision record: ADR-315. GIS is core capability C13; the meeting map belongs to eOSC+ (C10). This runbook covers replicating the MBSP ArcGIS layers into PostGIS, the legacy MJKP import and the settings the map needs.

## How it works

- The meeting map (`osc.meetings.map`) never calls ArcGIS. Lots, zoning and utilities are read from PostGIS and served as vector tiles (`gis.tiles`, zoom 12 to 22 only, throttled to 600 requests a minute per user). Agenda items, earlier tablings and past decisions come as GeoJSON from `osc.meetings.map-data`. Only the replication command talks to `spgis.mbsp.gov.my`.
- Returned agenda items are not on the map. Items not yet accepted into the agenda (proposed or endorsed) are drawn muted and labelled with their status in the list and the popup.
- An item with no drawn boundary, site boundary or matching lot, but with a positioned layout plan, is located by the plan's footprint (`geometry_source` `layout`).
- Layout plans are not drawn by default. Selecting an application (agenda item or past decision) offers **Lihat pelan susun atur**; switching it on shows that application's plan only (switching on another replaces it), and its image is fetched only then.
- Each layer is copied into a dated snapshot (`core.gis_layer_snapshots`). A layer points at its active snapshot. A failed import never replaces it, so the map keeps the last good copy.
- Reference data can be up to one refresh old. The `/osc` page lists the last snapshot time per layer for managers (see "Refresh button").
- Tiles are served only to active staff accounts that can reach eOSC+, Perancang PKM registration or a Bangunan workspace (`canUseGisTiles`); others get 403.

## Settings in `.env`

| Key | Purpose |
|---|---|
| `GIS_OGR2OGR` | Full path to `ogr2ogr`. Without it replication cannot run and the replication tests are skipped. |
| `GIS_QUEUE_RETRY_AFTER` | `retry_after` of the `gis` queue connection used by the refresh button, default 3900 seconds. Must stay above the one-hour job timeout. |
| `ARCGIS_PORTAL_URL` | Default `https://spgis.mbsp.gov.my/portal`. |
| `ARCGIS_USERNAME`, `ARCGIS_PASSWORD` | Optional. Queries work anonymously; use a token account only if MBSP later restricts the layers. Credentials stay in `.env`, never in git, and are redacted from logs and snapshot errors. |
| `ORTHOPHOTO_TILE_URL` | Tile URL template for the MBSP orthophoto basemap (see "Orthophoto tiles"). |
| `DISPLAY_TIMEZONE` | Times on the page are shown in Malaysian local time; the database stores UTC. |

On Windows the `GIS_OGR2OGR` path must be wrapped in single quotes in `.env`, otherwise the backslashes are read as escapes:

```
GIS_OGR2OGR='C:\Program Files\QGIS 3.40\bin\ogr2ogr.exe'
```

The GDAL build bundled with QGIS has no PostgreSQL driver. The service therefore asks `ogr2ogr` to write a PGDUMP SQL script and loads it with `psql`, looked up in the same folder as `ogr2ogr` and otherwise on the PATH. `psql` must be able to reach the iNDECS database with the connection in `.env`. After editing `.env` run `php artisan config:clear`.

## Layers

Layer definitions live in `config/gis.php` and are written to `core.gis_layers` by `GisLayerSeeder`:

| Layer id | Source (`spgis.mbsp.gov.my/geoportal/rest/services/...`) | Notes |
|---|---|---|
| `lots` | `Kadaster/KADASTER/FeatureServer/3` | Lot polygons. `match_attributes`: application `lot_number` to `lot`, `mukim` to `mukim`, `district` to `daerah`. |
| `zoning` | `Gunatanah/Gunatanah/FeatureServer/3` | Land-use zones. `properties`: `gunatanah1` (main class, 12 values such as Perumahan, Komersial), `gunatanah3` (detailed class, about 250 values) and `kod_gtn` (class code, e.g. TRM304), shown in the map popup. Zones are drawn in one colour; colouring by class would need more than the token palette. |
| `utilities` | `Utiliti/Utiliti/FeatureServer/0` | Points. |
| `legacy_mjkp` | `DEMARCATION/MJKP_eOSC/FeatureServer/1204` | Past OSC decisions (kind `legacy_papers`). Refreshed only by `osc:import-legacy --points`, which replicates it and rebuilds `core.osc_legacy_papers`; `gis:replicate --all` and the refresh button skip it, and `gis:replicate legacy_mjkp` is refused. |

Seed or refresh the definitions (safe to rerun; creates no snapshots):

```sh
php artisan db:seed --class=GisLayerSeeder --force
```

### Adding a layer

1. Add an entry to `layers` in `config/gis.php` with a unique lower-case `id`, `label`, `kind` (`reference` or `legacy_papers`), the FeatureServer layer URL as `source_url`, `where_clause` (`1=1` for everything), `is_active`, `sort_order` and `properties`. `properties` whitelists the lower-case attribute names encoded into vector tiles; list no personal data.
2. Set `match_attributes` only if the layer is used to match applications to features (as `lots` does). It maps an application column to a layer attribute; both sides are normalised before comparing. Use `null` otherwise.
3. Run the seeder, then `php artisan gis:replicate <id>`.
4. A new layer must also be added to the map page code to be drawn; replication alone only stores it.

## Replicating layers

```sh
php artisan gis:replicate lots        # one layer
php artisan gis:replicate --all       # every active reference layer, in sort order (legacy_papers layers are skipped)
```

Replication pages through ArcGIS in PHP, 2,000 features per page, writes them to a staging table through `ogr2ogr` (PGDUMP) and `psql`, then swaps the layer to the new snapshot in one transaction. The previous snapshot becomes `superseded`. Only the two most recent superseded snapshots keep their features; older ones are pruned (their snapshot rows stay as history). Plan the storage for the active snapshot plus two superseded copies of every layer; lots alone are about 266,000 features per copy.

A layer refuses to start while another import of the same layer began less than two hours ago (status `importing`). If a run was killed, wait two hours or mark the stale `importing` row `failed`.

Measured on the development database (anonymous, 2026-10-29): `lots`, 265,731 features in about 159 seconds; `zoning`, 262,114 features in about 171 seconds.

### Refresh button

A manager (a role holding `osc.meeting.manage`: `osc_secretary`, `osc_steno`, `system_admin`) sees a "Lapisan peta" panel on `/osc` with the last finished time (Malaysian local time) and the status of each active layer, and a "Kemas kini lapisan" button. The button posts to `osc.gis.refresh`, which queues `App\Jobs\Gis\ReplicateLayers` with the signed-in user as actor and shows "Kemas kini lapisan dimulakan." The job replicates every active reference layer one after another (timeout 1 hour, no retries); `legacy_papers` layers are skipped (see the legacy import), a layer that fails does not stop the others, and the job then ends as failed listing them.

The job runs on its own queue connection, `gis` (`config/queue.php`: database driver, queue `gis`, `retry_after` 3900 seconds from `GIS_QUEUE_RETRY_AFTER`). `retry_after` must stay above the one-hour job timeout, otherwise the queue releases the job while it is still running and a second copy starts; the default connection's 90 seconds is far too short. Run a dedicated worker, or the button has no effect:

```sh
php artisan queue:work gis --queue=gis --timeout=3600
``` Statuses: Siap (`ready`), Sedang berjalan (`importing`), Gagal (`failed`), Diganti (`superseded`). "Belum dikemas kini" means the layer has never been replicated. The panel shows the latest snapshot of each layer, so a recent failure appears even though the map still uses the earlier good copy.

Snapshot times written before this button was added (development data) were stored 8 hours early because the timestamp lost its offset; new snapshots are correct. Do not rely on the old dates.

### Yearly refresh

Source data changes slowly. Refresh at least once a year, and before a meeting when lots or zoning are known to have changed: run `php artisan gis:replicate --all` (or use the button) outside office hours, then open a meeting map and check the lots and the snapshot time. Each run writes an audit row (`gis_layer_replicated` or `gis_layer_replication_failed`) with the actor.

### Reading a failed snapshot

```sql
SELECT id, layer_id, status, feature_count, started_at, finished_at, left(error, 300) AS error
FROM core.gis_layer_snapshots ORDER BY id DESC LIMIT 10;
```

A `failed` row carries the reason in `error` (tokens are redacted). Common causes:

- `ogr2ogr gagal` or `psql gagal`: `GIS_OGR2OGR` unset or wrongly quoted, or `psql` not reachable or not authorised. Fix `.env`, run `config:clear`, rerun.
- `ArcGIS HTTP ...` or `ArcGIS ralat`: the portal is down, slow or rejected the query. The active snapshot is untouched and the map keeps working; rerun later.
- `sedang berjalan`: another import of the same layer is under two hours old, see above.

The snapshot in use is the one in `core.gis_layers.active_snapshot_id`.

## Spatial reference systems (SRIDs)

All stored geometry is EPSG:4326 and must fall inside the MBSP box `[100.33, 5.10, 100.60, 5.62]`. Two MBSP source systems are registered in `spatial_ref_sys`:

- **990001**, GDM2000 Cassini Pulau Pinang & Seberang Perai: the ArcGIS layer CRS and the default for the legacy import. It agrees with ArcGIS within 1 m.
- **990002**, legacy MJKP Kertau Cassini Penang (`towgs84=-11,851,5`).

Alignment check on 20 random MJKP papers that have both an ArcGIS point and a georeferenced plan: the point fell inside the plan in 17 of 20 (85 %) under both SRIDs, with the same three misses under both (3 to 38 km away, so bad legacy points or mismatched plans, not a projection error). The two SRIDs differ by about 11 m at plan scale (measured), and the 100 m containment guard cannot tell them apart. `--srid=990001` is the default until the go-live check below is done.

### Go-live check: choosing the legacy SRID

Before go-live (owner decision, recorded in ADR-315):

1. Pick 5 to 10 legacy plans that clearly show lot lines, spread over the area.
2. Import under 990001 (`php artisan osc:import-legacy --images --srid=990001`), open a meeting map near each plan at zoom 18 or more with the cadastre on, and measure the offset between the plan's lot lines and the cadastre lot lines with the measure tool.
3. Repeat with `--srid=990002` and measure again. Expect the two to differ by about 11 m.
4. Choose the SRID with the smaller offset and run `php artisan osc:import-legacy --images --srid=<chosen>`. Reruns converge: imported placements are recomputed, hand-placed ones are left alone. Record the measurements and the choice in ADR-315.

## Legacy MJKP import

`osc:import-legacy` loads past decisions and places the scanned layout plans on the map:

```sh
php artisan osc:import-legacy --points
php artisan osc:import-legacy --images --srid=990001 --dir="O:\Projects\mjkp\web\plan"
php artisan osc:import-legacy --points --images --srid=990001 --dir="O:\Projects\mjkp\web\plan"
```

- `--points` replicates the `legacy_mjkp` layer from ArcGIS into a new snapshot and copies it into `core.osc_legacy_papers`. It reports the records, those with a point and those without XY. This is the only way to refresh past decisions.
- `--images` reads the folder `--dir` (default `O:\Projects\mjkp\web\plan`), which must exist and hold files. A missing, unmounted or empty folder aborts the command before anything is written, so placements are never removed by mistake.
- For each paper the file is resolved from `pelan_tapak_scan` first, then from the `plan_` file name. The first named file that is usable wins; file names are not guessed.
- The four corners come from the world file and the SRID. A plan is placed only if it lies inside the MBSP box and within 100 m of the paper's point (containment guard).
- A file claimed by several papers is placed for all of them only when every claimant has a point and passes the guard, otherwise for none (shared-file rule).
- A rerun converges: papers whose plan fails for a content reason lose their imported placement, but the stored file is kept. Placements made by hand are never changed or deleted.

The command prints a table of outcomes: `Diletakkan` (placed) and the skip reasons `tiada XY`, `tiada fail`, `tiada world file`, `format TIFF belum disokong`, `di luar kawasan`, `kedudukan tidak sepadan`, `fail dikongsi`, `fail rosak` and `diletak secara manual`.

Results on the development data (2026-10-29): 5,407 legacy points; 2,800 layout images placed. Skipped: no file 107, no world file 1,140, TIFF 409, position mismatch 10, shared file 529. A further 1,389 world-file stems in the folder are not referenced by any paper, so they cannot be placed; that is not an error. TIFF plans are not supported yet.

## Orthophoto tiles

The map offers Esri imagery, Esri street map and "Ortofoto MBSP". The Esri URLs are built in. The orthophoto choice appears only when `ORTHOPHOTO_TILE_URL` is set to an XYZ template, for example `https://host/path/{z}/{y}/{x}.jpg`, supplied by the MBSP GIS unit (use the order of `{z}`, `{x}` and `{y}` that the provider documents). The browser fetches these tiles directly; if they fail the map shows a notice and the local layers keep rendering. Leave the value empty until MBSP provides a URL.

## Map use by staff

- **Boundary drawing:** officers who may register the application (the Perancang PKM register permission or the Bangunan `register` permission for that application type) draw the site boundary on the registration page. It is saved by `gis.applications.boundary` and must be a valid polygon inside the MBSP box.
- **Meeting map:** on a meeting page, anyone who can view the meeting opens "Peta" (`osc.meetings.map`), which shows each agenda item's site over lots, zoning, utilities and past decisions, with the layout plan in place.
- **Layout placement** (`osc.layout.edit`): a manager, the presenter or the proposer of an item uploads a JPEG or PNG layout plan (up to 20 MB and 36 million pixels) and marks three reference points (pixel position and map position). The three image points must span a triangle of at least 0.5 % of the image area. A JPEG whose EXIF orientation is set and not 1 is refused ("Imej mempunyai orientasi EXIF; simpan semula tanpa putaran."), because browsers rotate it but the placement uses raw pixels; save it again without rotation. An item has at most one layout plan (unique per owner); a new upload replaces it. The plan is stored with a four-corner polygon and drawn by affine transform; the image itself is not reprojected.

## Rollback

The GIS migration refuses to roll back while any GIS data exists. Use a reviewed forward fix instead. To free space, delete old `superseded` snapshots and their `core.gis_features` rows; never delete the active snapshot.
