# Ghana Elevation Maps: Process Notes

Step-by-step record of the workflow so far, from folder setup to the finished print layout. Written 16 November 2026. Remaining work: the plain elevation and topographic map variants, then the 3D model in Blender.

## 1. Project setup

### Folder structure

Raw data, processed data and outputs are kept separate. Raw data is never edited.

```
ghana-elevation-maps/
├── README.md
├── .gitignore
├── data/
│   ├── raw/          untouched downloads
│   ├── processed/    clipped, reprojected, hillshade, contours
│   └── external/     boundaries and other sources
├── qgis/             ghana_elevation.qgz
├── styles/           saved .qml layer styles
├── outputs/
│   ├── maps/         exported PNG/PDF
│   └── textures/     images for Blender
├── blender/          ghana_3d.blend
└── docs/             notes.md
```

### Git

- Repository managed with GitHub Desktop.
- `.gitignore` excludes large rasters (`data/raw/*`, `data/processed/*.tif`), QGIS backups, Blender backups (`*.blend1`) and OS junk.
- `.gitkeep` files keep empty folders tracked.
- Committed: QGIS project, styles, notes, small vector data, final exports.
- Not committed: DEMs and other large GeoTIFFs. Their sources are recorded in `docs/notes.md` so they can be re-downloaded.
- In QGIS, Project > Properties > General > Save paths is set to **Relative**.

## 2. Getting the DEM

**Source:** Copernicus GLO-30 DSM (30 m), distributed by OpenTopography.

1. Create a free account at opentopography.org and log in.
2. Open the Copernicus GLO-30 dataset page and start a data download.
3. Enter the bounding box (Xmin/Xmax are longitude, Ymin/Ymax are latitude):
   - Xmin -3.4, Ymin 4.5, Xmax 1.3, Ymax 11.3 (whole of Ghana)
4. Skip the Raster Visualization step. Hillshade and colour products are made in QGIS instead.
5. Submit the job and wait for the email or check Raster Jobs. Job ID: `rt1794826510842`.
6. Download `rasters_COP30.tar.gz` (1.3 GB).
7. Extract in two stages with 7-Zip: `.tar.gz` gives `.tar`, then `.tar` gives the files.
8. Rename the DEM to `copernicus_glo30_ghana.tif` and move it to `data/raw/`.

**Facts about the file:** EPSG:4326, 16,920 x 24,480 pixels (about 414 million), 1 arc-second (about 30 m) pixels, Float32, no NoData value (the sea is stored as 0 m). GLO-30 is a surface model, so it includes treetops and buildings.

## 3. Loading and first look in QGIS

1. Project > New, saved as `qgis/ghana_elevation.qgz`.
2. Layer > Add Layer > Add Raster Layer, then select the DEM.
3. Check Properties > Information: CRS, extent, pixel size and value range.
4. Note: the displayed maximum (628 m) was only an estimate. The real high point, Mount Afadja, is about 880 m.

**Lesson:** setting Min/Max accuracy to Actual on 414 million pixels made QGIS appear to hang. Set the range by hand instead (0 to 880).

## 4. First colour map

1. Layer Properties > Symbology > Render type: **Singleband pseudocolor**.
2. Min/Max: choose **User defined** first, then type 0 and 880. If a value reverts, edit the last row of the colour table directly.
3. Colour ramp: Spectral, inverted so blue/green is low and red is high. A custom terrain ramp (greens to browns) was also planned, with more stops in the low range because most of Ghana is below 300 m.
4. Mode: Continuous or Linear.

## 5. Boundary and clipping

### Boundary

- Source: geoBoundaries, Ghana ADM0.
- Chosen file: the full-resolution `geoBoundaries-GHA-ADM0.geojson` (not the simplified version, and not the shapefile, which is several files that must stay together).
- Renamed to `data/external/ghana_boundary_adm0.geojson`, with `CITATION-AND-USE-geoBoundaries.txt` copied beside it.

### Clip

1. Raster > Extraction > **Clip Raster by Mask Layer**.
2. Input: the original DEM. Mask: the Ghana boundary.
3. Open **Advanced Parameters** (collapsed by default) and set:
   - Match the extent of the clipped raster to the extent of the mask layer: on
   - Assign a specified nodata value: **-9999**
4. Save to `data/processed/ghana_dem_clipped.tif`.
5. Check Properties > Information: NoData should read -9999, so the sea is genuinely empty rather than 0 m.

**Lesson:** the first clip had no NoData value. A display-only workaround is Transparency > Additional no data value = 0, but the proper fix is to re-run the clip with -9999.

## 6. Reprojection and downsampling

The DEM was in degrees but its heights are in metres, which breaks hillshade and slope calculations. It was converted to a metric CRS.

1. Raster > Projections > **Warp (Reproject)**.
2. Target CRS: **EPSG:32630, WGS 84 / UTM zone 30N**. It does not appear in the short dropdown. Click the globe icon and search the full list.
3. Resampling: **Bilinear**. NoData: -9999. Output resolution: **90 m** (about 9 times fewer pixels than 30 m).
4. Save to `data/processed/ghana_dem_utm30_90m.tif`.
5. Load the saved style, remove the old degree-based layer, and set the project CRS to EPSG:32630.
6. Save the style as `styles/elevation_hypsometric.qml`.

Ghana straddles the edge of UTM zones 30 and 31. Zone 30N is used for the whole country, with small distortion in the east.

## 7. Hillshade

1. Raster > Analysis > **Hillshade**.
2. Input: `ghana_dem_utm30_90m`. Z factor 1, azimuth 315, vertical angle 45.
3. Save to `data/processed/ghana_hillshade.tif`.
4. Place the hillshade above the coloured DEM. Symbology > Layer Rendering: blending mode **Multiply**, opacity about 65%.
5. If the relief is too subtle, re-run with Z factor 3 (up to 5) and save as a separate file to compare.

## 8. Contours

1. Raster > Extraction > **Contour**, input `ghana_dem_utm30_90m`, attribute name `ELEV`.
2. A 50 m interval looked like a brown smudge at country scale. Use **100 m** (`ghana_contours_100m.gpkg`) for the whole country, and keep the 50 m version only for zoomed views with scale-dependent visibility (minimum 1:500,000).
3. Style: dark brown (about `#6b4a2b`), width 0.15 mm, around 70% opacity. Check the width unit is Millimeters.
4. Index contours using rule-based styling: normal lines `"ELEV" % 500 != 0`, thicker index lines `"ELEV" % 500 = 0` (for a 100 m interval).
5. Optional smoothing: Vector > Geometry Tools > Smooth (2 iterations, offset 0.25).
6. Labels (not finished): Rule-based Labeling, filter on the index lines, Label with `ELEV`, white buffer, curved placement, 0 decimals. Revisit later.
7. Style saved to `styles/contours.qml`.

## 9. Print layout

Layout name: `ghana_elevation_a3`, A3 portrait (297 x 420 mm), 15 mm margins. Every item is positioned by typing exact values under Item Properties > Position and Size.

| Item | Key settings |
|---|---|
| Map | X 15, Y 45, 267 x 340 mm, scale about 1:2,400,000 |
| Title | `Elevation Map of Ghana`, bold 40 pt, X 15, Y 12 |
| Subtitle | `Copernicus GLO-30 DEM, 90 m resolution \| UTM zone 30N`, 14 pt grey |
| Legend | Auto update off, DEM entry only, bottom-left, white background |
| Scale bar | Single Box, kilometres, 4 segments of 100 km, bottom-right |
| North arrow | 18 x 18 mm, top-right |
| Credits | `Data: Copernicus GLO-30 DEM (ESA) via OpenTopography \| Boundaries: geoBoundaries \| Made with QGIS`, 9 pt |

One font family and one dark grey (`#333333`) are used throughout.

### Centering the map

If Ghana sits off-centre in the frame, use the Move item content tool, or zoom to the DEM in the main window and click Set Map Extent to Match Main Canvas Extent.

### Graticule

- On the map item: Grids > + > Modify Grid.
- CRS WGS 84 (EPSG:4326), interval 1° in X and Y.
- Line style light grey, 0.2 mm (or the Cross grid type for a cleaner look).
- Zebra frame, 2 mm. Coordinates on all four sides, decimal degrees with suffix, 0 decimals, 9 pt.

### Legend

- Untick Auto update, then rename the DEM entry to `Elevation (m above sea level)`.
- Hide the extra layer-name heading, or rename the layer itself.
- Item labels come from the DEM layer's colour table (Symbology > Label column). Label the first row `0 m (sea level)` and the last `880 m`.
- For a middle value on a continuous bar: look for a tick setting in Legend Settings, use five discrete rows (0, 220, 440, 660, 880), or add a manual label.
- Number format: 0 decimal places.

### Export

Layout > Export as Image or PDF at 300 dpi, saved to `outputs/maps/`.

## 10. Problems met and how they were solved

| Problem | Cause | Fix |
|---|---|---|
| QGIS hung on Actual Min/Max | 414 million pixels to scan | Use User defined 0 and 880 |
| Max value reverted to 100 | Automatic Min/Max mode overwrote it | Select User defined first, or edit the colour table |
| NoData greyed out after clip | Clip run without a nodata value | Re-run with -9999 in Advanced Parameters |
| Clip options missing | Collapsed Advanced Parameters section | Expand it or enlarge the window |
| UTM 30N missing in dropdown | Short recent-CRS list | Use the globe icon and search the full list |
| Contours looked like a smudge | 50 m interval too dense at country scale | Use 100 m, or scale-dependent visibility |
| Map off-centre in layout | Layout copied an off-centre canvas | Re-centre the canvas and reset the map extent |

## 11. Files and sources to remember

- DEM: Copernicus GLO-30 DSM via OpenTopography, job `rt1794826510842`, downloaded 2026-11-16. Credit: European Space Agency (2024), https://doi.org/10.5069/G9028PQB
- Boundary: geoBoundaries ADM0 Ghana, `data/external/ghana_boundary_adm0.geojson`.
- Processed (git-ignored): `ghana_dem_clipped.tif`, `ghana_dem_utm30_90m.tif`, `ghana_hillshade.tif`, `ghana_contours_100m.gpkg`.
- Styles: `styles/elevation_hypsometric.qml`, `styles/contours.qml`.

## 12. Next steps

1. Finish the legend (middle value) and contour labels.
2. Produce the three map variants by toggling layers: plain elevation, elevation with hillshade, and the full topographic map with contours.
3. Prepare Blender inputs: a 16-bit greyscale heightmap from the DEM (Raster > Conversion > Translate, UInt16, scaled 0 to 65535) and the styled map exported as a texture.
4. Build the 3D model in Blender with a subdivided plane and Displace modifier, or the BlenderGIS add-on, using vertical exaggeration since Ghana is fairly flat.
