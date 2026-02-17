### DGGS Integration via Plugin Architecture

DGGS support is proposed via a **separate plugin** (`openeo_dggs_plugin`), avoiding refactoring of `openeo-processes-dask`.

#### Why plugin-based?

* Preserve stability of `openeo-processes-dask`
* Keep DGGS functionality modular
* Allow iterative prototyping without impacting RasterCube workflows
* Maintain clear separation between raster and DGGS logic

---

## Integration Model

### 1. Process Registration

The plugin registers new processes (e.g. `resample_dggs`) through the existing process registry.

This means:

* No changes to core process definitions
* Backend execution framework remains unchanged
* DGGS functionality is opt-in

---

### 2. Data Model Extension

Introduce a new logical data model:

* `RasterCube`: `(time, bands, y, x)`
* `DGGSCube`: `(time, bands, cell_id)`

`DGGSCube` wraps xarray structures and reuses backend execution utilities where possible. Raster workflows remain unaffected.

---

## Entry Process: `resample_dggs`

```json
{
  "process_id": "resample_dggs",
  "arguments": {
    "data": RasterCube,
    "level": int | [int],
    "grid_system": "healpix",
    "method": "bin_mean" | "nearest" | "gauss"
  }
}
```

Performs:

```
RasterCube → DGGSCube
```

Supported methods:

* `bin_mean` (pixel aggregation)
* `nearest`
* `gauss`

---

## Multi-Level Handling

* Single level supported
* Multiple levels initially returned as separate cubes

Future option: use **zuniq encoding** (from `healpix-geo`) to represent `(depth, ipix)` in a single identifier for hierarchical multi-resolution stacking.

---

## Dependencies (plugin-level only)

* xarray (+ dask)
* healpy
* healpix-geo
* pyresample
* numpy

These are isolated to the plugin and do not alter core backend dependencies.