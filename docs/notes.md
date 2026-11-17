# Ghana Elevation Maps: Data and Processing Log

Short reference for where the data came from, what was derived from it, and the numbers that must stay consistent. The full step-by-step is in `process_notes.md`.

## Data sources

### Elevation (DEM)

- Dataset: Copernicus GLO-30 DSM (30 m), a surface model, so it includes treetops and buildings.
- Distributed by: OpenTopography, https://portal.opentopography.org
- Job ID: `rt1794826510842`
- Downloaded: 2026-11-16 (archive `rasters_COP30.tar.gz`, 1.3 GB)
- Bounds requested: S 4.5, N 11.3, W -3.4, E 1.3
- Stored as: `data/raw/copernicus_glo30_ghana.tif` (never edited)
- Original CRS: EPSG:4326. 16,920 x 24,480 px, Float32, no NoData value (sea stored as 0 m).
- Citation: European Space Agency (2024). Copernicus Global Digital Elevation Model. Distributed by OpenTopography. https://doi.org/10.5069/G9028PQB. Accessed 2026-11-16.
- Licence text from the dataset page: © DLR e.V. 2010-2014 and © Airbus Defence and Space GmbH 2014-2018, provided under COPERNICUS by the European Union and ESA; all rights reserved.

### Boundary

- Dataset: geoBoundaries ADM0 (Ghana), GeoJSON, full resolution (not the simplified version).
- File: `data/external/ghana_boundary_adm0.geojson`
- Source: geoboundaries.org. See `data/external/CITATION-AND-USE-geoBoundaries.txt` for how to credit it.

## Coordinate systems

- Storage and download: EPSG:4326 (degrees).
- Working CRS for everything derived: **EPSG:32630, WGS 84 / UTM zone 30N** (metres). Ghana straddles the edge of zones 30 and 31, and 30N is used for the whole country with small distortion in the east.
- Layout graticule: EPSG:4326, 1° interval.

## Derived products

| File | Made from | CRS | Resolution | Notes | In git |
|---|---|---|---|---|---|
| `data/processed/ghana_dem_clipped.tif` | raw DEM + boundary | 4326 | 30 m | Clip by mask layer, NoData -9999 | No |
| `data/processed/ghana_dem_utm30_90m.tif` | clipped DEM | 32630 | 90 m | Warp, bilinear, NoData -9999 | No |
| `data/processed/ghana_hillshade.tif` | 90 m DEM | 32630 | 90 m | Z factor 1, azimuth 315, altitude 45 | No |
| `data/processed/ghana_contours_100m.gpkg` | 90 m DEM | 32630 | 100 m interval | Attribute `ELEV`, index lines every 500 m | Check size |
| `outputs/textures/ghana_heightmap_16bit.tif` | 90 m DEM | 32630 | about 247 m | UInt16, 0 to 880 m scaled to 0 to 65535, sea NoData 0 | No |
| `outputs/textures/ghana_texture_aligned.tif` | styled map, clipped to heightmap extent | 32630 | 100 m | 4936 x 7148 px, 3 bands | No |
| `outputs/maps/` | QGIS print layout | | 300 dpi | A3 portrait | Yes |
| `blender/ghana_3d*.blend` | heightmap + texture | | | About 11 MB | Yes |
| `outputs/renders/` | Blender | | 3000 x 4000 | EEVEE, Standard colour | Yes |

## Numbers that must match

- Heightmap extent (metres): W 471674.85, E 965234.85, S 523600.37, N 1238380.37.
- Real size covered: 493.56 km x 714.78 km.
- Heightmap and colour texture must cover the **same extent**. Verify in QGIS Properties > Information before using them in Blender. The first texture was 18.4 km too wide on the east side and caused the colours to slide.
- Heightmap scale: 0 m = darkest, 880 m = white. Highest point of Ghana is about 880 m (Mount Afadja).
- Blender plane: about 82.85 x 119.97 units, 1 unit is about 5.96 km. Relief exaggeration is about 6.8 x the Displace strength.

## Styling choices

- Colour ramp: Spectral, inverted (blue/green low, red high), range 0 to 880 m set by hand.
- Contour interval: 100 m for the national map (50 m too dense at country scale).
- Sea background colour in textures and Blender: `#bcd4e6`.
- Sun for hillshade and Blender: from the north-west (QGIS azimuth 315, Blender Z rotation 225°).

## Tools and versions

- QGIS (project file `qgis/ghana_elevation.qgz`, relative paths on).
- Blender 5.x (files saved by a 5.x build).
- 7-Zip for extracting the `.tar.gz` download.
- GitHub Desktop for version control.

## Large files policy

GitHub rejects files over 100 MB. DEMs, heightmaps and textures are git-ignored and can be regenerated from `process_notes.md`. Check the size of any `.gpkg` or `.blend` before committing.

## Credits for finished maps

`Data: Copernicus GLO-30 DEM (ESA) via OpenTopography | Boundaries: geoBoundaries | Made with QGIS and Blender`
