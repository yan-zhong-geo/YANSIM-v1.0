# YANSIM v1.0 - a Multi-Hazard Simulation Tool

**Developer:** [Yan Zhong](https://sites.google.com/view/yanzhong-geo), [University of Geneva](https://c-cia.ch/)  
**E-mails:** yan.zhong@unige.ch | yan.zhong.geo@gmail.com  
**Toolbox file:** `Multi-Hazard Simulation Tool v1.0.atbx`  
**Last updated:** 13.03.2026  
**Language:** English  
**Coding language:** Python  
**Operating System:** Windows (ArcGIS 10.0 or later, including ArcGIS Pro)  
**Installation Time:** ~1 second  

---

## Overview

The **YANSIM v1.0 - a Multi-Hazard Simulation Tool** is an ArcGIS Pro Python Script Toolbox for automated runout simulation and hazard exposure assessment in high mountain regions, integrating **Rock-Ice Avalanche (RIA)**, **Glacial Lake Outburst Flood (GLOF)**, and **Landslide (LS)** models into a normalized composite Exposure Index.

Each hazard is modelled independently using empirical travel angle (Fahrböschung) constraints and flow-routing algorithms. Individual Exposure Index (EI) rasters are normalized to a common 0–1 scale and optionally combined into a composite **Multi-Hazard EI**.

All three AOI inputs are optional — the tool runs whichever hazards are provided and generates a composite EI only when two or more hazards are active.

---

## Inputs

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| DEM | Raster | Yes | — | Digital Elevation Model. |
| RIA AOI | Raster | Optional | — | Raster mask of RIA source areas. |
| GLOF AOI | Raster | Optional | — | Raster mask of glacial lakes. |
| LS AOI | Raster | Optional | — | Raster mask of landslide source areas. |
| Output Folder | Folder | Yes | — | Directory to save output rasters and temporary files. |
| Tana RIA | Float | Optional | 0.10 | Fahrböschung tangent for RIA runout stopping criterion. |
| Tana GLOF | Float | Optional | 0.05 | Fahrböschung tangent for GLOF runout stopping criterion. |
| Tana LS | Float | Optional | 0.19 | Fahrböschung tangent for landslide runout stopping criterion. |
| Buffer Cells | Integer | Optional | 2 | Number of pixels to buffer RIA and GLOF paths (compensates for D8 single-line width). |
| Depth RIA (m) | Float | Optional | 50 | Release depth used to estimate RIA volume for the alpha angle calculation. |

> **Note:** The DEM and AOI rasters are automatically reprojected to a meter-based UTM CRS if needed. Flow direction is computed internally — no pre-computed flow direction input is required.  
> **Note:** At least one AOI input must be provided.  
> **Expected run time:** Varies by study area size and number of source pixels. Landslide BFS simulation is the most computationally intensive step.

---

## Outputs

| Output | Type | Description |
|--------|------|-------------|
| `RIA_EI.tif` | Raster | Normalized RIA Exposure Index (0–1). |
| `GLOF_EI.tif` | Raster | Normalized GLOF Exposure Index (0–1). |
| `LS_EI.tif` | Raster | Normalized Landslide Exposure Index (0–1). |
| `Multi_EI.tif` | Raster | Normalized composite Exposure Index — sum of active hazard EIs, re-normalized to 0–1. Only generated when ≥2 hazards are active. |
| `MH_temp/` | Folder | Intermediate files. Can be deleted after a successful run. |

---

## Hazard Models

### RIA — Rock/Ice Avalanche
- **Algorithm:** D8 single-flow-direction tracking from each source pixel
- **Runout criterion:** Volume-dependent Fahrböschung angle based on Zhong et al., (2026): `α = atan(1.4082 − 0.1658 × log₁₀(V))`, where `V = Depth_RIA × connected_patch_area`
- **EI value:** Accumulated release depth along the runout path

### GLOF — Glacial Lake Outburst Flood
- **Algorithm:** D8 single-flow-direction tracking from the lowest boundary pixel of each lake
- **Runout criterion:** Fixed Fahrböschung angle `atan(Tana_GLOF)`, measured from lake surface elevation
- **EI value:** `lake_area × e^(−k × distance)` — larger lakes contribute higher values; EI decays with downstream distance

### Landslide (LS)
- **Algorithm:** Multi-directional BFS spreading to the top-2 steepest downslope neighbours at each step
- **Runout criterion:** Fixed Fahrböschung angle `atan(Tana_LS)`, measured from each source pixel
- **EI value:** Count of source pixels whose runout path covers each cell

---

## Workflow

1. Prepare a DEM and at least one AOI raster covering the study area.
2. Set the output folder and optionally adjust **Tana**, **Buffer Cells**, and **Depth RIA**.
3. Run the tool:
   - **PART 0** — Projection check: DEM is reprojected to UTM if in geographic CRS; AOI rasters are reprojected to match the DEM CRS.
   - **PART 1** — DEM preprocessing: `Int → Fill → FlowDirection`.
   - **PART 2** — Input arrays are read onto a consistent grid aligned to the filled DEM.
   - **PART 3** — Each active hazard model is run independently.
   - **PART 4** — EI rasters are buffered (RIA/GLOF), normalized, and saved. Multi-hazard EI is computed if ≥2 hazards are active.
4. Collect output rasters from the output folder.

> **Note:** Requires the **Spatial Analyst** extension and the `scipy` Python package in ArcGIS Pro.

---

## Tana Parameter Guide

`Tana` is the tangent of the Fahrböschung travel angle. A smaller value allows longer runout; a larger value stops the flow earlier.

| Hazard | Default Tana | Approx. angle | Reference |
|--------|-------------|---------------|-----------|
| RIA | 0.10 | 6° | Typical long-runout avalanche/debris flow |
| GLOF | 0.05 | 3° | Typical long-runout flood flow |
| LS | 0.19 | 11° | Typical long-runout landslides/debris flow |

---

## License

See the [LICENSE](LICENSE) file for full terms.
