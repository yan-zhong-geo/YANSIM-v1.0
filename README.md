# YANSIM v1.0: A Multi-Hazard Simulation Tool

**Developer:** [Yan Zhong](https://sites.google.com/view/yanzhong-geo), [University of Geneva](https://c-cia.ch/)  
**E-mails:** yan.zhong@unige.ch | yan.zhong.geo@gmail.com  
**Last updated:** 22.03.2026  
**Language:** English  
**Coding language:** Python  

---
## 1. Overview
 
**YANSIM v1.0** is a multi-hazard simulation framework designed for automated runout modelling and exposure assessment in high mountain regions. It integrates three major hazard types:
 
- Rock-Ice Avalanches (RIA)
- Glacial Lake Outburst Floods (GLOF)
- Landslides (LS)

This model accompanies the paper:  
> Zhong, Y., Allen, S., … Stoffel, M. (2026). *YANSIM v1.0: A Multi-Hazard Simulation Framework for Regional-Scale Exposure Assessment of Infrastructure to Mass Movement Hazards in High Mountain Regions*  

Please cite this publication when using the model.
 
### 1.1 Path Model
 
Each hazard is simulated using a unified **three-step D8 + BFS path model**. For each seed pixel (or lake outlet), the model:

1. **D8 main path** — walks downstream along the D8 flow direction, recording each node's exact cumulative path length (`path_len`). The path terminates when the average slope from the seed to the current position falls below the Fahrböschung threshold (`tana`). The cumulative length at termination is `total_len`.
2. **BFS lateral extent** — from each D8 node, a breadth-first search spreads laterally to the steepest downslope neighbours (up to `bfs_max_n`). A shared visited set across all D8 nodes of a single seed ensures each pixel is expanded only once. BFS collects spatial extent only — no values are computed during this step.
3. **Nearest D8 node assignment** — each pixel in the BFS extent is assigned the EI value of its nearest D8 node (Euclidean pixel distance). EI decays quadratically from seed/outlet to end of reach:

```math
EI = weight × (1 − path_len / total_len)²
```

where `weight = 1.0` for RIA and LS, and `weight = sqrt(log(lake_size + 1))` for GLOF.

This design ensures that slope termination and EI decay are always based on true flow-path distance, correctly navigating flat terrain, glaciers, and valley floors without premature path truncation. The separation of BFS (extent) from D8 (values) ensures that lateral spread pixels inherit spatially accurate EI values rather than artificially uniform ones.

### 1.2 Hazard Models

All three hazards use the same `sim_path()` core function and differ only in seed definition, `weight`, and AOI preprocessing:

| Parameter | RIA | LS | GLOF |
|---|---|---|---|
| Seed (`r0, c0`) | Each AOI source pixel | Each AOI source pixel | Lowest outlet pixel of each lake |
| `elev0` | Elevation of seed pixel | Elevation of seed pixel | Elevation of outlet pixel |
| `tana` | 0.10 | 0.19 | 0.05 |
| `weight` | 1.0 (fixed) | 1.0 (fixed) | sqrt(log(lake_size+1)) |
| `total_len` | path_len of last D8 node | path_len of last D8 node | path_len of last D8 node |
| EI formula | `(1−ratio)²` | `(1−ratio)²` | `sqrt(log(lake_size+1)) × (1−ratio)²` |
| Seed accumulation | Sum over all AOI pixels | Sum over all AOI pixels | Sum over all lakes |
| AOI preprocessing | Remove patches < `MIN_PATCH_RIA` | Remove patches < `MIN_PATCH_LS`; remove pixels overlapping RIA | No patch filtering; individual lakes identified via `scipy.label` |
| Final normalisation | min-max to 0–1 (99th percentile clip) | min-max to 0–1 (99th percentile clip) | min-max to 0–1 (99th percentile clip) |

#### 1.2.1 Rock-Ice Avalanche (RIA)

Each AOI pixel is a seed. EI at each reached pixel accumulates the quadratic decay contributions from all upstream seeds whose path passes through or near that location. Small AOI patches below a minimum size threshold are removed before simulation.

#### 1.2.2 Glacial Lake Outburst Flood (GLOF)

Each glacial lake is a seed. The outlet pixel (lowest elevation within a dilated search ring around the lake mask) is used as the starting point. EI decays quadratically with normalised flow-path distance and is weighted by lake size:

```math
EI = \sqrt{\ln(lake\_size + 1)} \times \left(1 - \frac{path\_len}{total\_len}\right)^2
```

Each lake's contribution is summed across all lakes before final normalisation. The `sqrt(log(...))` weighting compresses the difference between large and small lakes while preserving their relative ordering.

#### 1.2.3 Landslide (LS)

Same approach as RIA, with additional AOI pre-processing: small patches are removed, and pixels overlapping the RIA AOI are excluded to avoid double-counting.

### 1.3 Output
 
The resulting Exposure Index (EI) layers are normalized to a 0–1 scale and can be combined into a composite Multi-Hazard EI. The tool is flexible: users can run individual hazards or combine multiple hazards depending on data availability.
 
---
 
## 2. Available Versions
 
YANSIM is currently available in **three different implementations**, designed for users with different technical backgrounds and computational needs:
 
| Version | Platform | Best for | Status |
|---------|----------|----------|--------|
| ArcGIS Toolbox | ArcGIS Pro · Windows | Small- to medium-scale areas · GIS users | Released |
| Google Colab | Browser · Google Drive | Medium- to large-scale areas · Python users | Released |
| YANSIM Desktop | Windows standalone | Non-technical users | Under development |

> This recommendation is indicative. The actual applicability and computational speed depend on the pixel counts in the RIA AOI and LS AOI, as well as the number of lakes. Typically, within a single study area: Number of RIA AOI pixels > Number of LS AOI pixels > Number of lakes.

### 2.1 ArcGIS Toolbox Version
 
- **File:** `YANSIM v1.0 Toolbox.atbx`
- **Requirements:** ArcGIS Pro + Spatial Analyst extension + `scipy`
 
Best suited for small- to medium-scale study areas (e.g. watershed or basin scale) and users familiar with ArcGIS-based GIS workflows.
 
### 2.2 Google Colab Version
 
- **File:** `YANSIM v1.0.ipynb`
- **Requirements:** Google account + Google Drive
 
Best suited for medium- to large-scale study areas (basin to regional scale) and computationally intensive simulations. Runs on Google cloud infrastructure — independent of local computer performance. Results can be **visualized** interactively on Google Satellite basemaps via `folium`.
 
### 2.3 YANSIM Desktop *(Under Development)*
 
- **Platform:** Windows standalone application
- **License:** Free and open-source
 
User-friendly Graphical User Interface (GUI) with no coding or GIS software required, faster than the ArcGIS version, fully local execution.
 
> ⚠️ This version is currently under development.
 
---
 
## 3. General Notes
 
- All hazard inputs (RIA, GLOF, LS) are optional — the tool runs only the hazards provided
- A composite Multi-Hazard EI is generated only when **two or more hazards** are active
- DEM and AOI rasters are automatically reprojected to a meter-based UTM CRS if needed
 
---
 
## 4. Workflow — ArcGIS Toolbox Version
 
### 4.1 Inputs
 
| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| DEM | Raster | Yes | — | Digital Elevation Model. |
| RIA AOI | Raster | Optional | — | Raster mask of RIA source areas. |
| GLOF AOI | Raster | Optional | — | Raster mask of glacial lakes. |
| LS AOI | Raster | Optional | — | Raster mask of landslide source areas. |
| Output Folder | Folder | Yes | — | Directory to save output rasters and temporary files. |
| Tana RIA | Float | Optional | 0.10 | Fahrböschung tangent for RIA runout stopping criterion. |
| Tana GLOF | Float | Optional | 0.05 | Fahrböschung tangent for GLOF runout stopping criterion. |
| Tana LS | Float | Optional | 0.19 | Fahrböschung tangent for LS runout stopping criterion. |
| BFS max neighbours | Integer | Optional | 2 | Number of steepest downslope neighbours followed by BFS spread at each D8 node. Higher values produce wider lateral spread. Applied to all three hazard models. |
| Min patch pixels RIA | Integer | Optional | 22 | Minimum contiguous patch size (pixels) for RIA AOI. Patches below this threshold are removed before simulation. Default based on 30 m DEM. |
| Min patch pixels LS | Integer | Optional | 4 | Minimum contiguous patch size (pixels) for LS AOI. Patches below this threshold are removed before simulation. Default based on 30 m DEM. |
 
> **Note:** At least one AOI input must be provided.  
> **Expected run time:** Varies by study area size and number of source pixels. RIA and LS are the most computationally intensive steps.
 
### 4.2 Outputs
 
| Output | Description |
|--------|-------------|
| `RIA_EI.tif` | Normalized RIA Exposure Index (0–1). |
| `GLOF_EI.tif` | Normalized GLOF Exposure Index (0–1). |
| `LS_EI.tif` | Normalized Landslide Exposure Index (0–1). |
| `Multi_EI.tif` | Normalized composite EI — sum of active hazard EIs, re-normalized to 0–1. Only generated when ≥2 hazards are active. |
| `MH_temp/` | Intermediate files. Can be deleted after a successful run. |
 
### 4.3 Steps
 
1. Prepare a DEM and at least one AOI raster covering the study area.
2. Set the output folder and optionally adjust **Tana**, **BFS max neighbours**, **Min patch pixels RIA**, and **Min patch pixels LS**.
3. Run the tool:
   - **PART 0** — Projection check: DEM is reprojected to UTM if in geographic CRS; AOI rasters are reprojected to match the DEM CRS.
   - **PART 1** — DEM preprocessing: `Fill → FocalStatistics (3×3 MEAN, N passes) → Fill → FlowDirection`. The double-fill and smoothing pipeline removes single-pixel elevation noise that causes fragmented D8 flow directions on near-flat terrain, while preserving float precision throughout (no integer truncation).
   - **PART 2** — Input arrays are read onto a consistent grid aligned to the filled DEM.
   - **PART 3** — Each active hazard model is run independently using the shared `sim_path()` function. For RIA and LS, small AOI patches are removed before simulation; LS additionally excludes pixels overlapping the RIA AOI. For each seed pixel (or lake outlet for GLOF), the model walks the D8 path, collects BFS lateral extent, and assigns quadratic decay EI values via nearest D8 node lookup.
   - **PART 4** — EI rasters are normalized and saved. Multi-hazard EI is computed if ≥2 hazards are active.
4. Collect output rasters from the output folder.
 
> **Note:** Requires the **Spatial Analyst** extension and the `scipy` Python package in ArcGIS Pro.
 
---
 
## 5. Workflow — Google Colab Version
 
### 5.1 Setup
 
1. Upload all input `.tif` files to your Google Drive.
2. Open `YANSIM v1.0.ipynb` in [Google Colab](https://colab.research.google.com).
3. Set the runtime to **CPU**: *Runtime → Change runtime type → CPU*. GPU and TPU are not required and will not improve performance.
 
### 5.2 Inputs
 
Same parameters as the ArcGIS Toolbox Version — see the Inputs table above.
 
### 5.3 Steps
 
1. **Cell 1 — Install dependencies:** Installs `rasterio`, `pysheds`, `scipy`, and `folium` automatically.
2. **Cell 2 — Mount Google Drive:** Authorise Colab to access your Drive.
3. **Cell 3 — Set parameters:** Edit file paths and model parameters:
   ```python
   DEM_PATH  = '/content/drive/MyDrive/your_folder/dem.tif'
   RIA_PATH  = '/content/drive/MyDrive/your_folder/ria_prone.tif'  # None if not used
   GLOF_PATH = '/content/drive/MyDrive/your_folder/lakes.tif'      # None if not used
   LS_PATH   = '/content/drive/MyDrive/your_folder/ls_prone.tif'   # None if not used
   OUT_DIR   = '/content/drive/MyDrive/your_folder/output'
   ```
4. **Cell 4 — Load model code:** Loads all path model kernels. No edits needed.
5. **Cell 5 — DEM preprocessing:** Reprojects DEM to UTM if in geographic CRS, then runs `Fill → FocalStatistics → Fill → FlowDirection` via `pysheds` and `scipy.ndimage`.
6. **Cell 6 — Read AOI rasters:** Loads and reprojects all AOI rasters to match the DEM grid exactly.
7. **Cell 7 — Run models:** Runs RIA, GLOF, and LS independently using the shared `sim_path()` function.
8. **Cell 8 — Save outputs:** Normalizes and saves EI rasters to `OUT_DIR` on Google Drive.
9. **Cell 9 — Visualize:** Displays results interactively on a Google Satellite basemap using `folium`. Each hazard EI layer can be toggled on/off independently. A colour ramp from green (low) to red (high) indicates relative exposure.
 
### 5.4 Outputs
 
Same as the ArcGIS Toolbox Version. Files are saved directly to the specified Google Drive folder and remain accessible after the session ends.

> **Note:** Free-tier Colab sessions may disconnect after extended periods. For large study areas, consider **Colab Pro** for faster CPUs and longer session times.
 
---
 
## 6. Tana Parameter Guide
 
`Tana` is the tangent of the Fahrböschung travel angle. A smaller value allows longer runout; a larger value stops the flow earlier.
 
| Hazard | Default Tana | Approx. angle | Reference |
|--------|:-----------:|:-------------:|-----------|
| RIA | 0.10 | ~6° | Typical long-runout avalanche / debris flow |
| GLOF | 0.05 | ~3° | Typical long-runout glacial outburst flood |
| LS | 0.19 | ~11° | Typical long-runout landslide / debris flow |
 
---
 
## 7. License
 
See the [LICENSE](LICENSE) file for full terms.
