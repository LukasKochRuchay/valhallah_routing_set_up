# valhallah_routing

Local [Valhalla](https://github.com/valhalla/valhalla) routing server (routes, isochrones, matrices) running in Docker.

## Setup

1. Choose the map region: in [docker-compose.yml](docker-compose.yml), set `tile_urls` to one or more
   OpenStreetMap `.osm.pbf` download links (separate multiple links with a space). Get them from
   [download.geofabrik.de](https://download.geofabrik.de), e.g.
   - Berlin (small, builds in a few minutes): `https://download.geofabrik.de/europe/germany/berlin-latest.osm.pbf`
   - Brandenburg incl. Berlin: `https://download.geofabrik.de/europe/germany/brandenburg-latest.osm.pbf`

   Routing only works inside the downloaded region. Larger regions need much more time, disk and RAM.
2. Start it:
   ```sh
   docker compose up -d
   docker compose logs -f   # watch progress
   ```
   On the first start the container downloads the `.pbf` and builds routing tiles into `custom_files/`.
   Later starts reuse those tiles and are ready within seconds.
4. Test it: `http://localhost:8002/status`

### Changing the region

`tile_urls` is only used when no tiles exist yet. After changing it, either delete the `custom_files/`
folder or set `force_rebuild=True` for one start, then run `docker compose up -d` again.

## Example request

`POST http://localhost:8002/isochrone` with a JSON body:

```json
{
  "locations": [{ "lat": 52.464704585910404, "lon": 13.386833350489944 }],
  "costing": "auto",
  "contours": [{ "time": 5 }, { "time": 10 }, { "time": 15 }],
  "polygons": true
}
```

## Public transport (GTFS)

1. Put an unzipped GTFS feed into its own subfolder, e.g. `gtfs_feeds/bvg/stops.txt`.
2. `build_transit=True` in [docker-compose.yml](docker-compose.yml) builds transit tiles on the next start if none exist
   (slow: the full VBB feed took ~3.5 h with `server_threads=2`).
3. **Gotcha:** the server reads `custom_files/valhalla_tiles.tar`, which is only re-packed if it is missing.
   After a transit build, delete that file and restart (`docker compose restart`), otherwise transit is ignored.
4. Use `"costing": "multimodal"` plus a departure time inside the feed's validity period:

```json
{
  "locations": [{ "lat": 52.464704585910404, "lon": 13.386833350489944 }],
  "costing": "multimodal",
  "date_time": { "type": 1, "value": "2026-09-29T08:00" },
  "contours": [{ "time": 15 }, { "time": 30 }],
  "polygons": true
}
```

API reference: https://valhalla.github.io/valhalla/api/
