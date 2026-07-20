# Elevation data APIs

How an open source navigation host should talk to the three elevation sources used for rest stops and energy routing (try in this order):

1. Copernicus DEM GLO-30  
2. Viewfinder Panoramas (dem3)  
3. NASA SRTMGL1  

Downloads must support pause / resume / cancel, progress per region or entire-country job, and local tile cache lookup before network fetch. Void / missing elevation is commonly represented as `-32768` for HGT-style products.

---

## Table of contents

1. [Common host interface](#common-host-interface)
2. [Copernicus DEM GLO-30](#copernicus-dem-glo-30)
3. [Viewfinder Panoramas](#viewfinder-panoramas)
4. [NASA SRTMGL1](#nasa-srtmgl1)
5. [Fallback order and country jobs](#fallback-order-and-country-jobs)

---

## Common host interface

Suggested internal operations (language-agnostic):

| Operation | Purpose |
|-----------|---------|
| `tile_id(lat, lon)` | Map a point to a 1°×1° tile key (floor of lat/lon for HGT; Copernicus uses its own northing/easting labels). |
| `tile_exists(tile_id)` | True if a usable local file is in the elevation data directory. |
| `get_elevation(lat, lon)` | Read height from the preferred available source for that tile. |
| `download_tiles(bbox \| country)` | Queue tile downloads; track progress; allow pause/resume/cancel. |
| `source_priority` | Ordered list: Copernicus → Viewfinder → SRTMGL1. |

Tile index for HGT-family products (Viewfinder / SRTMGL1):

- `lon_idx = floor(lon)`, `lat_idx = floor(lat)`
- Filename stem: `N|S` + `dd` + `E|W` + `ddd` (e.g. `N61E009`, `S33W071`)
- Typical grid: 1 arc-second, 3601×3601 samples per 1° tile (16-bit big-endian heights in `.hgt`)

---

## Copernicus DEM GLO-30

### What it is

Global Digital Surface Model at ~30 m (1.0 arc-second class product). Public AWS open-data distribution as Cloud Optimized GeoTIFF (COG). Ocean areas may have no tiles (treat height as 0). A small set of land tiles may still be withheld from the public GLO-30 release.

### Access model

Not a JSON REST “query elevation” API. The interface is **anonymous HTTP(S) / S3 object GET** against a public bucket (no AWS account required when using unsigned access).

| Item | Value |
|------|--------|
| Product | GLO-30 Public |
| Bucket | `copernicus-dem-30m` |
| Region | `eu-central-1` |
| Object listing | `tileList.txt` in the bucket |
| Auth | Unsigned / `--no-sign-request` style public read |

### Path and naming

Directory (object prefix) pattern:

```text
Copernicus_DSM_COG_[resolution]_[northing]_[easting]_DEM/
```

| Token | Meaning |
|-------|---------|
| `resolution` | Arc seconds: `10` for GLO-30 (not “30 meters” in the name) |
| `northing` | e.g. `N61_00`, `S50_00` |
| `easting` | e.g. `E009_00`, `W125_00` |

Example prefix:

```text
Copernicus_DSM_COG_10_N61_00_E009_00_DEM/
```

Inside that prefix, download the DEM GeoTIFF object(s) for the tile (COG; DEFLATE; tiled internals such as 1024×1024 for GLO-30).

### Host operations

1. Convert `(lat, lon)` or a country/region bbox to the set of Copernicus tile prefixes.  
2. Optionally fetch `tileList.txt` to validate public availability.  
3. `HEAD`/`GET` the COG object (HTTPS endpoint for the bucket, or S3 API equivalent).  
4. Read elevation with a GeoTIFF reader (libtiff or similar); sample Int16/Float32 as provided.  
5. Cache locally under a stable tile key for offline use.

### List / download examples (conceptual)

```text
# List bucket (unsigned)
aws s3 ls --no-sign-request s3://copernicus-dem-30m/

# Fetch one tile prefix objects
aws s3 sync --no-sign-request \
  s3://copernicus-dem-30m/Copernicus_DSM_COG_10_N61_00_E009_00_DEM/ \
  ./srtm/copernicus/N61E009/
```

Equivalent: HTTPS GET of the same object URLs exposed by the public S3 website/API for that bucket.

### Notes for navigation hosts

- Prefer COG range requests when only a window is needed; full-tile download is fine for offline country packs.  
- Respect the Copernicus / open-data licence terms for redistribution.  
- If a tile is missing from the public set, fall through to Viewfinder, then SRTMGL1.

---

## Viewfinder Panoramas

### What it is

Community DEM product (dem3) providing 1°×1° HGT tiles, often used to fill gaps or as a second preference. Coverage varies by region; some tiles are absent.

### Access model

**HTTP download of ZIP archives** containing `.hgt` files, organized by map sheet / zone directories on the Viewfinder Panoramas dem3 tree. There is no official authenticated REST API; clients use predictable tile names and HTTP GET.

Optional helper pattern used by third-party indexes: request a logical tile name and follow a redirect to the ZIP URL, with a header naming the path inside the archive.

### Tile naming

Same HGT stem as SRTM:

```text
N47E011.hgt
S55W036.hgt
```

Lower-left corner of the 1° cell; north/south and east/west letters; latitude two digits, longitude three digits.

### Typical retrieval flow

1. Compute stem from `floor(lat)`, `floor(lon)`.  
2. Resolve the dem3 ZIP URL for that stem (coverage map / index / known directory layout under the dem3 tree).  
3. HTTP GET the ZIP (support resume with `Range` if the server allows).  
4. Extract the `.hgt` (or use `X-Zip-Path`-style hints when an index returns them).  
5. Store as `STEM.hgt` in the local elevation directory.

### Index helper behaviour (when used)

Request:

```text
GET /index.php/N47E011.hgt
```

Possible outcomes:

| HTTP | Meaning |
|------|---------|
| 302 | Tile exists; `Location` = ZIP URL; optional `X-Zip-Path` = path inside ZIP |
| 404 | No Viewfinder tile for that cell |

Hosts may also hardcode or ship a static mapping from stem → ZIP path if they do not want a live index.

### File format after extract

- Extension `.hgt`  
- 16-bit signed big-endian samples  
- Often 3601×3601 for 1″ dem3  
- Void: `-32768`

---

## NASA SRTMGL1

### What it is

NASA Shuttle Radar Topography Mission Global 1 arc-second (SRTMGL1), Version 3 / MEaSUREs family, archived by LP DAAC. ~30 m class elevation; files named like `N37W105.SRTMGL1.hgt` (or zipped equivalents).

### Access model

Two layers:

1. **Discovery** — NASA CMR (Common Metadata Repository) search API (no download of the raster itself).  
2. **Download** — Earthdata Login–authenticated HTTPS GET of granule URLs (Data Pool / Earthdata Cloud).

Anonymous public bulk download like Copernicus is **not** the normal path; hosts need Earthdata credentials (or a pre-staged local cache).

### Discovery (CMR)

Base search endpoint pattern:

```text
GET https://cmr.earthdata.nasa.gov/search/granules
```

Useful query parameters (examples):

| Parameter | Role |
|-----------|------|
| `short_name` | e.g. `SRTMGL1` |
| `version` | e.g. `003` |
| `bounding_box` | `lon_min,lat_min,lon_max,lat_max` |
| `page_size` / `page_num` | Pagination |
| `Accept` header | `application/json` or ATOM/XML as supported |

Response entries include granule metadata and **download / related URLs**. The host selects the HGT (or ZIP) link for each 1° tile covering the job bbox or country.

Collection DOI (reference): `10.5067/MEASURES/SRTM/SRTMGL1.003`

### Download (Earthdata)

1. Register / use NASA Earthdata Login.  
2. Authenticate downloads (`.netrc`, token, or `earthaccess`-style session).  
3. HTTP GET the granule URL from CMR (follow redirects).  
4. Store locally; unzip if the granule is `.zip` / `.hgt.zip`.  
5. Normalize filename to the host’s HGT stem convention for cache lookup.

Typical granule naming ideas:

```text
N26W077.SRTMGL1.hgt
N26W077.SRTMGL1.hgt.zip
```

### Host operations

| Step | API |
|------|-----|
| Find tiles for bbox/country | CMR granules search |
| Resolve URL | Parse granule links from CMR JSON/ATOM |
| Fetch bytes | Authenticated HTTPS GET (resume with `Range` when allowed) |
| Read height | Same HGT reader as Viewfinder |

### Licence / attribution

Follow NASA Earthdata / LP DAAC citation and redistribution rules for the product version you ship or cache.

---

## Fallback order and country jobs

1. For each required tile, try local cache.  
2. Else try Copernicus GLO-30 object GET.  
3. Else try Viewfinder dem3 HTTP ZIP.  
4. Else try NASA SRTMGL1 via CMR + Earthdata download.  
5. If all fail, mark tile missing; elevation queries return void.

**Region job:** download all tiles intersecting a bbox.  
**Country job:** expand country polygon/bbox to the tile set, then run the same queue with one progress counter for the whole country.

Pause / resume: persist the queue (tile id, source attempted, bytes received, etag/length). Cancel drops remaining queue items but keeps completed tiles on disk.
