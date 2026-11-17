# Ghana Elevation Maps: Process Notes

Step-by-step record of the whole workflow: data, QGIS maps, heightmap and texture, and the Blender 3D relief model. Last updated 17 November 2026.

**Pipeline in one line:** Copernicus DEM (30 m, degrees) > clip to Ghana (NoData -9999) > reproject to UTM 30N at 90 m > hillshade + contours > QGIS print layout. From the same 90 m DEM: 16-bit heightmap + aligned colour texture > Blender (Solidify + Displace) > render.

## Key numbers to remember

| Item | Value |
|---|---|
| Project CRS | EPSG:32630, WGS 84 / UTM zone 30N |
| Highest point | about 880 m (Mount Afadja) |
| Heightmap extent (E/N, metres) | W 471674.85, E 965234.85, S 523600.37, N 1238380.37 |
| Real size covered | 493.56 km wide x 714.78 km tall (ratio 0.6905) |
| Heightmap | 2000 x 2896 px, UInt16, about 247 m per pixel |
| Colour texture | 4936 x 7148 px, 100 m per pixel, same extent as the heightmap |
| Blender plane | about 82.85 x 119.97 units, 1 unit is about 5.96 km |
| Relief exaggeration | about 6.8 x Displace strength |

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
│   ├── maps/         exported PNG/PDF from print layouts
│   ├── textures/     heightmap and colour texture for Blender
│   └── renders/      final Blender renders
├── blender/
│   ├── ghana_3d.blend (and later versions)
│   └── scripts/      the .py helper scripts
└── docs/             notes.md, process_notes.md
```

### Git

- Repository managed with GitHub Desktop.
- `.gitignore` excludes large rasters (`data/raw/*`, `data/processed/*.tif`), large textures (`outputs/textures/*.tif`, `outputs/textures/*.png`), QGIS backups, Blender backups (`*.blend1`) and OS junk.
- `.gitkeep` files keep empty folders tracked.
- Committed: QGIS project, styles, notes, scripts, small vector data, the `.blend` file (about 11 MB), final exports and renders.
- Not committed: DEMs, heightmap and textures. GitHub rejects files over 100 MB and the first texture was 105 MB. Everything git-ignored can be regenerated from the steps below, and sources are recorded in `docs/notes.md`.
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
3. Colour ramp: Spectral, inverted so blue/green is low and red is high. A custom terrain ramp (greens to browns) works too, with more stops in the low range because most of Ghana is below 300 m.
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

## 10. Heightmap for Blender

A heightmap is a greyscale image where dark is low and bright is high. Blender pushes a flat grid up according to it. It must be **16-bit**, otherwise the terrain looks stepped.

1. Raster > Conversion > **Translate (Convert Format)**.
2. Input: `ghana_dem_utm30_90m`. Output data type: **UInt16**.
3. Advanced Parameters > Additional command-line parameters:

```
-scale 0 880 1 65535 -a_nodata 0 -outsize 2000 0
```

   - `-scale 0 880 1 65535` maps 0 m to the darkest value and 880 m to white.
   - `-a_nodata 0` makes the sea (NoData) pure black, so land always sits above it.
   - `-outsize 2000 0` shrinks the width to 2000 px and keeps the proportions. The full 90 m raster is far too large to displace in Blender.
4. Save to `outputs/textures/ghana_heightmap_16bit.tif`.
5. Result: 2000 x 2896 px, UInt16, pixel about 247 m, NoData 0. The displayed Min/Max in QGIS is an estimate, so ignore it. Note the extent and size from the Information tab (see Key numbers).

## 11. Colour texture for Blender (and the alignment lesson)

The texture must cover **exactly the same extent** as the heightmap. Blender stretches each image across the plane independently, so any difference in extent makes the colours slide against the terrain.

### Preparing the view

- Visible layers: the hillshade (Multiply) and the coloured DEM only. Turn off the boundary, contours and heightmap layer.
- Project > Properties > General > Background colour: pale blue `#bcd4e6`, so the sea is not transparent or black.
- Project CRS must be EPSG:32630.

### What did not work

- **Project > Import/Export > Export Map to Image.** The extent boxes were overridden by the scale setting. The export covered an area about 9 times too large, and Ghana filled only about 11% of the picture width.
- **Convert map to raster** (Processing Toolbox) with extent from the heightmap layer and 100 m per pixel. It worked, but it pads the output to whole 1024-pixel tiles: 5120 x 7168 px, reaching 18.4 km further east and 2 km further south than the heightmap.

### What worked

1. Run **Convert map to raster**: Extent > Calculate from Layer > heightmap, Map units per pixel 100, background `#bcd4e6`, transparent background off. Save as `outputs/textures/ghana_texture.tif`.
2. Run Raster > Extraction > **Clip Raster by Extent**: input `ghana_texture`, extent Calculate from Layer > heightmap, save as `outputs/textures/ghana_texture_aligned.tif`.
3. Check Properties > Information: the extent should match the heightmap's. Result: 4936 x 7148 px, within 40 m on the east edge and 20 m on the south edge.
4. Delete the earlier wrong exports so they are not loaded by mistake.

## 12. Blender scene

Files were saved by a Blender 5.x build. Menus can differ slightly between versions.

### Grid

1. Start a General project and delete the default cube (select, press X).
2. Add > Mesh > Grid. In Adjust Last Operation set X Subdivisions 501 and Y Subdivisions 725 (about 364,000 vertices, square cells).
3. Set Dimensions (N panel > Item) to about 82.85 x 119.97, or any size with the same 0.6905 ratio. Then Ctrl+A > Scale to apply it.

### Displace modifier (the terrain)

1. Modifiers tab > Add Modifier > Deform > Displace.
2. Texture > New > texture properties > Type **Image or Movie** > Open the heightmap.
3. Image colour space: **Non-Color** (the default would bend the greys and distort heights).
4. Texture Extension: **Extend** (not Repeat). Repeat wraps the left edge onto the right edge.
5. Displace settings: Direction **Z**, Texture Coordinates **UV** (UV map `UVMap`), Midlevel **0**, Strength about **4**.
6. If the grid has no UVs: Tab, A, then U > Reset, then Tab.

Exaggeration is about 6.8 x strength with this plane size, so strength 4 is about 27x. Ghana is very flat (880 m of relief over 715 km), so it needs it. Try 3 for subtle, 5 for clear, 8 for dramatic.

### Material (the colours)

1. Material tab > New. Base Color > connect an **Image Texture** and open `ghana_texture_aligned.tif`.
2. Colour space **sRGB** (the opposite of the heightmap), Extension **Extend**, Interpolation Linear.
3. Roughness **1.0**, Specular **0**, for a matte map look.
4. Switch the viewport to **Material Preview** (Z menu, or the third sphere button).

## 13. Making it a relief slab with a flat base

A solid slab with thick sides reads like a physical relief model. Order matters.

**What went wrong first:** the modifiers were Subdivision, Displace, Solidify. Solidify then thickened the already-raised terrain, so the underside copied the hills and bumps poked out from underneath.

**Fix:** the stack is now Subdivision, **Solidify**, then **Displace**.

1. **Subdivision Surface:** type Simple, Levels Viewport 0, Render 1. Smoother terrain in final renders while the viewport stays fast.
2. **Solidify:** Thickness 3, Offset -1 (grows downward, top stays at sea level), Even Thickness on, Rim Material Index Offset 1, and **Shell Vertex Group** set to a new group called `SlabBottom`.
3. **Displace:** Vertex Group `SlabBottom`, with the **invert arrow** on. This leaves the slab's bottom flat and raises only the top.
4. **Second material** `GhanaRim` (slot 2): dark brown, base colour about (0.16, 0.11, 0.08), roughness 0.9. Without it, the map colours smear down the sides.
5. If the sides show dark gradients along the top edge: Object > Shade Auto Smooth, angle about 35°.

The same setup is in `blender/scripts/fix_slab_v3.py`.

## 14. Lighting, camera and render

### Sun

- Sun light: strength 3, angle (shadow softness) 2°.
- Direction: from the **north-west**, the map convention and the same as the QGIS hillshade (azimuth 315). Rotation X = 90° minus the elevation (50° for a 40° elevation), **Rotation Z = 225°**.
- Earlier notes said Z 135°. That is wrong: it lights from the north-east and makes ridges look like valleys.

### World and colour

- World colour: soft light grey (about 0.93, 0.94, 0.96), strength 1.
- Render properties > Color Management > View Transform: **Standard**. Filmic or AgX wash out the map colours.

### Camera

- Location (0, -125, 85), Rotation X 55.8°, Y 0, Z 0, focal length 40 mm, Clip End 2000.
- Look through it with Numpad 0. This gives a three-quarter view over the whole slab. If Ghana is cut off, move the camera back or lower the focal length.
- Output: 3000 x 4000, EEVEE. Set Resolution % to 50 for quick previews, and F12 to render.
- Save the final image from the render window (Image > Save As) to `outputs/renders/ghana_3d_render.png`.

## 15. Blender scripts

Saved in `blender/scripts/`. Run each with Scripting tab > Text > Open > Alt+P. Each saves a new copy and leaves the original untouched. They were written without being able to run Blender, so check the console summary for any SKIP lines.

| Script | Purpose |
|---|---|
| `refine_ghana_blend.py` | First attempt. Flattened the outer rows of the grid to hide a strip along the east edge. It did not fix the strip, because the real cause was the misaligned texture. Superseded. |
| `improve_ghana_v2.py` | Relief strength, slab with dark sides, matte material, sun, camera, background and colour settings. Saves `ghana_3d_v2.blend`. |
| `fix_slab_v3.py` | Fixes the modifier order so the base is flat. Saves `ghana_3d_v3.blend`. |

Settings are at the top of each script and can be edited before re-running.

## 16. Problems met and how they were solved

| Problem | Cause | Fix |
|---|---|---|
| QGIS hung on Actual Min/Max | 414 million pixels to scan | Use User defined 0 and 880 |
| Max value reverted to 100 | Automatic Min/Max mode overwrote it | Select User defined first, or edit the colour table |
| NoData greyed out after clip | Clip run without a nodata value | Re-run with -9999 in Advanced Parameters |
| Clip options missing | Collapsed Advanced Parameters section | Expand it or enlarge the window |
| UTM 30N missing in dropdown | Short recent-CRS list | Use the globe icon and search the full list |
| Contours looked like a smudge | 50 m interval too dense at country scale | Use 100 m, or scale-dependent visibility |
| Map off-centre in layout | Layout copied an off-centre canvas | Re-centre the canvas and reset the map extent |
| Texture PNG showed Ghana tiny in a huge blue canvas | Export Map to Image scale box overrode the extent | Use Convert map to raster with the heightmap extent |
| Blender colours shifted left, raised land under sea colour | Texture extent larger than heightmap (1024 px tile padding) | Clip the texture to the heightmap extent |
| Raised strip along the right edge | Same misalignment, showing real Togo-border hills in sea colour | Aligned texture. Edge-flatten script was not the answer |
| Repeat wrap-around at the edges | Texture extension left on Repeat | Set Extension to Extend on both images |
| Bumpy underside, shapes poking out below | Solidify ran after Displace | Solidify before Displace, bottom shell excluded by vertex group |
| Terrain looked flat | 880 m over 715 km, strength too low | Displace strength 4 to 8 |
| Relief looked inverted or confusing | Sun from the north-east (Z 135°) | Sun Z 225° (north-west) |
| Texture rejected by GitHub | 105 MB, over the 100 MB limit | Ignore `outputs/textures/*.tif` in `.gitignore` |

## 17. Files and sources to remember

- DEM: Copernicus GLO-30 DSM via OpenTopography, job `rt1794826510842`, downloaded 2026-11-16. Credit: European Space Agency (2024), https://doi.org/10.5069/G9028PQB
- Boundary: geoBoundaries ADM0 Ghana, `data/external/ghana_boundary_adm0.geojson`.
- Processed (git-ignored): `ghana_dem_clipped.tif`, `ghana_dem_utm30_90m.tif`, `ghana_hillshade.tif`, `ghana_contours_100m.gpkg`.
- Blender inputs (git-ignored): `outputs/textures/ghana_heightmap_16bit.tif`, `outputs/textures/ghana_texture_aligned.tif`.
- Styles: `styles/elevation_hypsometric.qml`, `styles/contours.qml`.
- Blender: `blender/ghana_3d.blend` and later versions, scripts in `blender/scripts/`.

## 18. Rebuild checklist (from scratch)

1. Download the DEM (section 2) and the boundary (section 5).
2. Clip with NoData -9999, reproject to UTM 30N at 90 m (sections 5 and 6).
3. Style, hillshade and contours (sections 4, 7 and 8), then the layout (section 9).
4. Make the heightmap (section 10), then the texture and align it (section 11). Compare the two extents before opening Blender.
5. Build the Blender scene (sections 12 to 14), or run `improve_ghana_v2.py` and `fix_slab_v3.py`.
6. Render and commit (section 14 and the Git notes in section 1).

## 19. Open items and next steps

1. Confirm that `fix_slab_v3.py` gave a flat base, and settle the final relief strength.
2. Finish the legend middle value and the contour labels in QGIS.
3. Produce the three map variants by toggling layers: plain elevation, elevation with hillshade, and the full topographic map with contours.
4. Final Blender render saved to `outputs/renders/`.
5. Optional: more mesh detail (1000 x 1450 grid), or the BlenderGIS add-on for real-world scale. Compatibility of BlenderGIS with Blender 5.x was not confirmed from the official project, so test it on a copy first.
6. Update the README with the project goal, CRS (EPSG:32630) and a link to these notes.
