# NewEATools — ArcGIS Pro Python Toolbox

A custom ArcGIS Pro Python Toolbox (`.pyt`) for **Enumeration Area (EA) delineation**. It partitions administrative boundary polygons into spatially constrained clusters based on a numeric analysis field (e.g., household counts), using a hierarchical splitting strategy followed by spatially constrained multivariate clustering (SC-MVC).

---

## Table of Contents

- [Overview](#overview)
- [Requirements](#requirements)
- [Installation](#installation)
- [Tool Reference](#tool-reference)
  - [Parameters](#parameters)
  - [How It Works](#how-it-works)
- [Workflow Diagram](#workflow-diagram)
- [Usage Example](#usage-example)
- [Known Limitations](#known-limitations)
- [File Structure](#file-structure)
- [License](#license)

---

## Overview

The toolbox contains a single tool that automates the process of splitting large administrative polygons into Enumeration Areas (EAs) that satisfy a minimum and maximum population (or household) threshold.

The core logic per admin polygon is:

1. **Aggregate** — Count the total analysis field value (e.g., households) within the admin boundary via a spatial join.
2. **Pass-through** — If the total is already within the maximum threshold, write the polygon to the output as-is.
3. **Hierarchical splitting** — If the total exceeds the maximum, iteratively split the polygon using ordered hierarchy feature classes (e.g., roads → sub-district boundaries).
4. **Gap-fill** — Union the split result with the original admin boundary to ensure full spatial coverage with no gaps.
5. **SC-MVC clustering** — Apply ArcGIS's Spatially Constrained Multivariate Clustering to group sub-polygons into EAs that satisfy the `min_val` / `max_val` constraints.
6. **Adaptive retry** — If clustering fails, the tool automatically adjusts constraints and retries, and falls back gracefully by eliminating zero-value polygons.

---

## Requirements

| Requirement | Details |
|---|---|
| **ArcGIS Pro** | Version 3.x or later (tested with the built-in Python 3 environment) |
| **Spatial Analyst** or **Advanced license** | Required for `SpatiallyConstrainedMultivariateClustering` and `Eliminate` |
| **Python** | 3.x (bundled with ArcGIS Pro — no separate install needed) |
| **Extensions** | Spatial Statistics (for SC-MVC) |

> ⚠️ The `arcpy.stats.SpatiallyConstrainedMultivariateClustering` tool requires an **Advanced** ArcGIS Pro license.

---

## Installation

1. **Clone or download** this repository:

   ```bash
   git clone https://github.com/<your-username>/NewEATools.git
   ```

2. **Open ArcGIS Pro** and navigate to the **Catalog pane**.

3. In the Catalog pane, go to **Toolboxes → Add Toolbox**.

4. Browse to the cloned folder and select `NewEATools.pyt`.

5. The toolbox will appear under **Toolboxes** and is ready to use.

> Alternatively, you can drag and drop the `.pyt` file directly into the Catalog pane.

---

## Tool Reference

### Parameters

| # | Display Name | Type | Required | Description |
|---|---|---|---|---|
| 1 | **Admin Boundary** | Feature Layer (Polygon) | ✅ Yes | The administrative boundary layer. Each polygon is processed independently. |
| 2 | **Hierarchy Feature Classes (ordered)** | Feature Layer (Multi-value) | ✅ Yes | An ordered list of feature classes used to split admin polygons. Processed from first (coarsest) to last (finest). Supports both **Polyline** and **Polygon** geometry types. |
| 3 | **Point Layer** | Feature Layer (Point) | ✅ Yes | The point dataset containing the analysis values (e.g., building centroids, household survey points). |
| 4 | **Analysis Field (numeric)** | Field | ✅ Yes | A numeric field on the **Point Layer** to be summed within each polygon (e.g., `HH_COUNT`). Dependent on Parameter 3. |
| 5 | **Minimum Value** | Long (Integer) | ✅ Yes | The minimum acceptable sum of the analysis field per output EA polygon. |
| 6 | **Maximum Value** | Long (Integer) | ✅ Yes | The maximum acceptable sum of the analysis field per output EA polygon. |
| 7 | **Output Feature Class** | Feature Class (Polygon) | ✅ Yes | Path to the output polygon feature class where all EA boundaries will be written. |

---

### How It Works

#### Step 1 — HH_SUM Calculation (`calc_hh`)

For any input polygon layer, the tool performs a **spatial join** to the Point Layer and sums the Analysis Field into a new field called `HH_SUM`. `NULL` values are replaced with `0`.

This step is repeated throughout the workflow whenever the polygon geometry changes.

#### Step 2 — Admin-level routing

For each admin polygon (iterated via a `SearchCursor`):

- **Total ≤ max_val** → The polygon is appended to the output directly. No further processing.
- **Total > max_val** → Hierarchical splitting is triggered.

#### Step 3 — Hierarchical splitting

Each hierarchy feature class is clipped to the current admin extent, then processed based on its geometry type:

- **Polyline** — Used directly with `FeatureToPolygon` to cut high-HH_SUM polygons along lines.
- **Polygon** — Centroids are extracted, **Thiessen polygons** are created, then clipped to the admin boundary. The result is used for an `Intersect`-based split.

At each hierarchy level:
1. Polygons exceeding `max_val` are separated as **"high"** features.
2. Only the high features are split using the current hierarchy splitter.
3. The split high features are merged back with the **"low"** (already-valid) features.
4. `HH_SUM` is recalculated on the merged result.
5. The output is passed to the next hierarchy level.

#### Step 4 — Gap fill via Union

After all splitting levels are complete, the tool unions the result with the original admin boundary polygon to ensure **no spatial gaps** exist in the coverage.

#### Step 5 — SC-MVC Clustering (`run_sc_mvc`)

The post-split polygons are passed to `arcpy.stats.SpatiallyConstrainedMultivariateClustering` with:

- **Analysis field**: `HH_SUM`
- **Size constraint**: Attribute value
- **Spatial constraint**: Contiguity (edges only)
- **Min/Max constraint**: `min_val` / `max_val`

The resulting clusters are dissolved by `CLUSTER_ID`, summing `HH_SUM`.

#### Step 6 — Adaptive retry logic

If clustering fails (`arcpy.ExecuteError`):

1. **First failure** — The tool increases `max_val` by 5%. If the total sum still exceeds the new max, it sets max to the total. If `min_val` exceeds the maximum single feature, it reduces `min_val` by 5%.
2. **Second failure** — Zero-`HH_SUM` polygons are eliminated (merged into adjacent polygons using `arcpy.management.Eliminate`), and the result is returned as-is.
3. **Third failure** — The pre-clustering input is returned unchanged, and a warning is logged.

---

## Workflow Diagram

```
For each Admin Polygon
│
├─ Calculate HH_SUM
│
├─ HH_SUM ≤ max_val?
│   └─ YES → Append to Output ──────────────────────────────────┐
│                                                                │
└─ NO → Hierarchical Splitting                                   │
    │                                                            │
    ├─ For each hierarchy level (ordered):                       │
    │   ├─ Clip splitter to admin extent                        │
    │   ├─ Polyline? → FeatureToPolygon                         │
    │   ├─ Polygon? → Thiessen → Clip → Intersect               │
    │   ├─ Separate HIGH (> max_val) and LOW polygons           │
    │   ├─ Split HIGH using splitter                            │
    │   ├─ Merge LOW + split HIGH                               │
    │   └─ Recalculate HH_SUM → feed to next level             │
    │                                                            │
    ├─ Union with admin boundary (gap fill)                     │
    │                                                            │
    ├─ SC-MVC Clustering                                        │
    │   ├─ Success → Dissolve by CLUSTER_ID                     │
    │   ├─ Fail #1 → Adjust min/max → Retry                    │
    │   └─ Fail #2 → Eliminate zeros → Return as-is            │
    │                                                            │
    └─ Append clusters to Output ───────────────────────────────┘
```

---

## Usage Example

**Scenario**: You have a national admin boundary layer and a building footprint point layer with a `HH_COUNT` field. You want to delineate EAs where each EA has between 80 and 120 households, using roads (polyline) as the primary splitter and sub-district boundaries (polygon) as the secondary splitter.

| Parameter | Value |
|---|---|
| Admin Boundary | `Admin_Districts` (polygon layer) |
| Hierarchy Feature Classes | `Roads_National; SubDistrict_Boundaries` (in this order) |
| Point Layer | `Building_Footprints` |
| Analysis Field | `HH_COUNT` |
| Minimum Value | `80` |
| Maximum Value | `120` |
| Output Feature Class | `C:\GIS\Output\EA_Boundaries.gdb\EA_2024` |

> 💡 **Tip**: Order the Hierarchy Feature Classes from coarsest to finest spatial resolution. The tool processes them in sequence, so major roads should come before sub-district lines.

---

## Known Limitations

- **In-memory workspace** — Intermediate datasets are stored in `in_memory`. For very large datasets with hundreds of admin polygons, this may strain available RAM.
- **SC-MVC license** — The Spatially Constrained Multivariate Clustering tool requires an ArcGIS Pro **Advanced** license and the **Spatial Statistics** extension.
- **Contiguity constraint** — Clustering uses edges-only contiguity. Non-contiguous polygons within an admin boundary may cause clustering failures that trigger the adaptive retry.
- **Hardcoded field name** — The intermediate summary field is always named `HH_SUM`. Ensure no existing field with this name carries critical data, as it will be deleted and recreated.
- **Single-level clustering** — The `process_splitter_single_or_all` variable is currently hardcoded to `"single"`. The `"all"` mode (simultaneous multi-level splitting) is present in code but not exposed as a parameter.

---

## File Structure

```
NewEATools/
│
├── NewEATools.pyt        # Main Python Toolbox file
└── README.md             # This documentation
```

> If you add tool icons or supplementary scripts in the future, place them in a `resources/` or `scripts/` subfolder and update this section accordingly.

---

## License

This project is not currently under an open-source license. All rights reserved by the author unless otherwise stated. If you intend to share or adapt this tool, please add a `LICENSE` file to the repository (e.g., MIT, GPL-3.0).
