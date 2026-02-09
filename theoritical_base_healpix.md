# Theoretical Understanding of HEALPix

This document consolidates my current theoretical and practical understanding of **HEALPix** (Hierarchical Equal Area isoLatitude Pixelization) extracted from my personal notes different resources . It is written as a *conceptual + scientific reference*, not as a tutorial, and is intended to support correct architectural and analytical decisions in Earth Observation (EO), climate science, and ML pipelines.

---

## 1. What HEALPix Is (and Is Not)

### What HEALPix *is*

HEALPix is:

* A **global, equal-area discretization of the sphere**
* A **hierarchical spatial index** (power-of-two refinement)
* A **projection-independent representation** of spherical data

Each HEALPix cell:

* Covers the same solid angle
* Has a unique integer index
* Can be recursively subdivided into four children

This makes HEALPix fundamentally different from raster grids or map projections.

### What HEALPix *is not*

HEALPix is **not**:

* A map projection
* A visualization grid (very pathetic for visualiation)
* A resampling or interpolation policy
* A replacement for CRS-based regional analysis (doesn't integrate so well even for simpler tasks like zonal statistics)

HEALPix does *not* attempt to preserve the shape, orientation, or adjacency of raster pixels. It preserves **area and hierarchy only**.

---

## 2. The Meaning of `nside` (Resolution Parameter) - 
PS: key parameter that often keeps me confusing

### Definition

HEALPix resolution is controlled by a single parameter:

```
nside = 2^k ,  k ≥ 0
```

Total number of pixels:

```
Npix = 12 × nside²
```

All pixels have equal area:

```
Ω = 4π / (12 × nside²)
```

### Physical interpretation

An equivalent angular pixel size is approximately:

```
θ ≈ √(π / 3) / nside   (radians)
```

On Earth (R ≈ 6371 km):

```
pixel_size_km ≈ R × √(π / 3) / nside
```

This gives the intuitive resolution table: Can be a naive-guidance chart, still has to refine my understanding here

| nside | Approx. km |
| ----: | ---------: |
|    64 |    ~115 km |
|   128 |     ~57 km |
|   256 |     ~28 km |
|   512 |     ~14 km |
|  1024 |      ~7 km |

---

## 3. Fundamental Principle: HEALPix Is a Resolution Choice
(This area is why i mentioned the chart above is naive and not one size fit for all)

Choosing `nside` is **not a geometric operation**, it is a **scientific resampling decision**.

> HEALPix does not infer resolution automatically because only the user understands the dataset’s *true information content*.

Two datasets with the same grid spacing may have very different effective resolution due to:

* Sensor PSF
* Oversampling
* Viewing geometry
* Temporal averaging

Therefore, HEALPix **cannot and should not** auto-parameterize `nside`. [key note]

---

## 4. Correct Rule for Choosing `nside`

### The golden rule 
(this area can be kind of helpful before taking decisions on parameterisation)

> **A HEALPix pixel should never be smaller than the dataset’s true spatial resolution.**

In other words:

```
HEALPix pixel area ≥ native pixel area
```

Violating this rule:

* Invents spatial detail
* Introduces artificial correlations
* Breaks statistical validity
* Causes ML overfitting 

(I mention ML here as i see the end-goal of the DEDL project is to built zarr cubes for multi-data fusion for ml applications)

---

## 5. Estimating Native Resolution (Automatic but Scientific)

### Regular lat/lon grids

For regular grids:

* Resolution varies with latitude
* Pixel area is not constant

Correct approach:

* Compute **actual geodesic distance** between neighboring coordinates
* Use the **smallest achievable distance** (best resolution)

This yields a physically meaningful ground resolution in km.

### Native / projected grids (e.g., geostationary)

For projected grids:

* Convert adjacent pixels to lon/lat
* Compute geodesic distances
* Use the **best (smallest) distance**, usually near nadir

This avoids relying on metadata or nominal resolutions.

---

## 6. Automatic but Defensible `nside` Selection

Once a target ground resolution `s_target` (km) is estimated:

```
nside ≈ √(π / 3) × R / s_target
```

Then:

* Round **up** to avoid coarsening
* Snap to the **next power-of-two**

This guarantees:

* No loss of spatial detail
* No artificial oversampling
* Hierarchical consistency

---

## 7. Why we should not let algo auto-choose `nside`

Deliberately avoiding this because it cannot know:

* Whether to preserve best-case or worst-case resolution
* Whether interpolation is allowed
* Whether the use case is science, ML, or visualization

Automatic `nside` choice would silently encode incorrect assumptions.

Correct design principle:

> **Libraries suggest; users decide.**

---

## 8. When HEALPix Is Worth Using

HEALPix is appropriate when **at least two** of the following are required:

1. Equal-area statistics
2. Global or future-global scope
3. Multi-sensor fusion (where i feel key-use of healpix is)
4. ML models spanning regions
5. Projection-independent indexing

### Ideal use cases

* Global reanalysis (ERA5)
* Geostationary imagery (MSG, GOES)
* Cross-region ML training (e.g., Greenland + South Africa) — regions located at vastly different latitudes, used to assess whether resolution divergence away from the equator introduces biases, and how an equal-area representation mitigates such effects
* Long-term scalable EO pipelines

---

## 9. When HEALPix Is *Not* Worth Using

HEALPix is often unnecessary for:

* Purely regional analysis
* Human-facing cartographic products
* Single-sensor, single-region studies
* CRS-native operations (zonal stats, local maps)

In these cases, traditional projected grids are simpler and clearer.

---

## 10. Regional Data on HEALPix

HEALPix is global by design, but regional datasets can still benefit when:

* Regions are disjoint (e.g., Greenland + South Africa)
* The same ML model is applied across regions
* The pipeline is expected to scale globally later

In HEALPix:

* Regions are **masks**, not separate grids
* Spatial consistency is preserved

---

## 11. Geostationary Data and HEALPix

Geostationary sensors are a natural match for HEALPix because:

* Native projection is view-dependent
* Resolution varies radially
* Earth coverage is partial

HEALPix cleanly separates:

* Coverage (physics)
* Resolution (nside)
* Geometry (spherical)

NaNs in HEALPix products represent **physical invisibility**, not missing data.

---

## 12. Practical Heuristics (Summary)

### Conservative science

```
Choose nside so HEALPix pixel ≈ 1–1.5 × native resolution
```

### ML pipelines

* Use one global `nside`
* Downsample high-res sensors
* Upsample low-res sensors cautiously

### Storage-aware design

Remember:

* Pixels ∝ nside²
* Even sparse global grids have costs

---

## 13. Key One-Line Principles

* HEALPix standardizes **space**, not **data quality**
* Choosing `nside` is a **scientific decision**
* Equal-area ≠ equal-shape
* NaNs are physics, not errors
* Use HEALPix when you want **one spatial index forever**

---

## 14. Final Mental Model

```
Native grid (sensor- or CRS-dependent)
        ↓
True physical resolution (km) or regular lat-lon grid
        ↓
Chosen nside (equal-area constraint)
        ↓
Sparse global HEALPix index
        ↓
Analysis / ML / fusion
```
---
