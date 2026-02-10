# HEALPix Learning: Resampling Methods When Converting Raster Data to HEALPix

There isn’t one **single best resampling method** for *all* variables when converting raster data to HEALPix. The correct choice depends on:

* **Grid type**: regular lat/lon vs curvilinear / native sensor grids
* **Variable type**:

  * continuous / intensive (e.g. temperature, radiance)
  * extensive / flux (e.g. precipitation totals, energy)
  * categorical (e.g. land cover)
  * bounded fractions (e.g. cloud fraction)
* **Goal**: ML feature engineering vs physically conservative, science-grade products

However, there *is* a best **overall strategy** for HEALPix conversion that applies to both regular and curvilinear grids:

> **Treat the operation as: source pixels/points → HEALPix target cells, with an explicit resampling kernel or conservative area overlap.**

This avoids thinking in terms of “reprojection” and instead frames the problem correctly as *spatial aggregation or interpolation on the sphere*.

Below is a practical, decision-oriented guide.

---

## The Three Main Approaches

### A) Binning / Pixel-to-Cell Aggregation

**How it works**

* Compute HEALPix `cell_id` for each source pixel using its latitude/longitude
* Aggregate all source pixels that fall into the same HEALPix cell using:

  * mean / median / min / max / mode

This is the simplest and most commonly used approach.

**Best for**

* ML feature engineering and prototyping
* Continuous fields where strict conservation is not mandatory
* Pipelines that value robustness and simplicity

**Regular grids**

* Use 1D lat/lon coordinates
* Build a meshgrid
* Convert to `cell_id` via `ang2pix`
* Group-by and aggregate

**Curvilinear grids**

* Use 2D `latitude(y,x)` / `longitude(y,x)` directly
* Convert each pixel to `cell_id`
* Group-by and aggregate

**Critical upgrade**

* Use **area-weighted binning** instead of naïve averaging
* Weight each source pixel by its physical area

This is the single most impactful improvement you can make without moving to full conservative remapping.

**Pros / Cons**

* ✅ Fast, robust, easy to implement with xarray
* ⚠️ Not strictly conservative unless carefully designed (and still approximate)

---

### B) Kernel / KDTree-Based Resampling

**How it works**

* Define HEALPix cell centers as target points
* For each target cell, find nearby source pixels
* Compute weighted averages using kernels such as:

  * nearest neighbor
  * inverse distance weighting
  * Gaussian kernels

This is closer to *interpolation* than aggregation.

**Best for**

* Native sensor grids (e.g. MSG geostationary)
* Swath-like or irregular sampling patterns
* Curvilinear grids with varying spacing
* Smooth physical fields (radiances, temperature, pressure)

**Strengths**

* Works for regular, curvilinear, and projected grids
* Handles irregular sampling gracefully
* Produces visually and statistically smooth results

**Limitations**

* ⚠️ Not conservative for extensive variables (e.g. precipitation totals)

**Typical tooling**

* `pyresample` (KDTreeResampler, GaussianResampler)
* Conceptually:

  * build `(lon,lat)` for source pixels
  * build `(lon,lat)` for HEALPix centers
  * run KDTree-based resampling

---

### C) Conservative Remapping by Area Overlap

**How it works**

* Treat source pixels as spherical polygons with known areas
* Treat HEALPix cells as exact spherical polygons
* Compute overlap fractions between source and target cells
* Remap values using overlap-weighted sums

**Best for**

* **Extensive variables** where conservation is mandatory:

  * precipitation totals
  * radiative energy
  * fluxes and emissions
* Producing science-grade, ARCO-quality datasets

**Pros / Cons**

* ✅ Physically correct and conservative
* ⚠️ Computationally expensive
* ⚠️ Complex to implement (spherical polygon intersections)

This is the gold standard, but not always necessary or practical.

---

## Best Choices for Common Grid Types

### 1) Regular Lat/Lon Grids

**Recommended default**

* **Area-weighted binning** (Approach A with weights)

**Why**

* Pixel boundaries are well defined
* Pixel areas are easy to compute (latitude-dependent)
* Results are stable and nearly conservative for averages

**Extensive variables**

* Conservative overlap (C) is ideal
* Area-weighted *sum* binning is a strong interim solution

---

### 2) Curvilinear Grids

**Recommended default**

* **Kernel / KDTree resampling** (Approach B)

**Why**

* Pixel shapes and spacing vary
* Equal-weight binning can bias results
* Distance-based kernels behave more like true resampling

**Alternative**

* If per-pixel areas are available:

  * area-weighted binning becomes competitive and robust

---

## Variable-Type Cheat Sheet

| Variable type                   | Best HEALPix strategy          | Notes                                      |
| ------------------------------- | ------------------------------ | ------------------------------------------ |
| Temperature, radiance, pressure | Kernel (B) or mean binning (A) | Interpolation-like behavior acceptable     |
| Precipitation totals, fluxes    | Conservative overlap (C)       | If unavailable → area-weighted sum binning |
| Fractions (0–1), cloud fraction | Area-weighted mean binning (A) | Preserves bounds                           |
| Categorical classes             | Mode / nearest (A)             | Never use bilinear or Gaussian             |


## One Crisp Answer to “Which Is Best?”

* **Regular grids**: area-weighted **binning** offers the best value-for-effort and strong scientific validity
* **Curvilinear grids**: **KDTree kernel resampling** is usually the most reliable practical choice unless accurate pixel areas are available

If the physical meaning of a variable is known (mean vs total vs fraction), the resampling method should always be chosen to respect that meaning.

---
