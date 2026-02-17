# DGGS Integration via Plugin Architecture

A new **DGGSCube** data model is proposed to be integrated into `openeo-processes-dask` through a **separate plugin** (`openeo_dggs_plugin`), avoiding direct refactoring of the core backend.

#### Why a plugin approach?

* Preserve stability of `openeo-processes-dask`
* Keep DGGS functionality modular and extensible
* Allow iterative prototyping without impacting raster workflows
* Maintain clear separation between Raster and DGGS logic


### How the Plugin Interacts with `openeo-processes-dask`

The integration happens at two levels:

#### 1. Process Registration

The plugin registers new processes (e.g., `resample_dggs`) via the existing process registry mechanism. This means:

* No modification of core process definitions
* Backend execution framework remains unchanged
* DGGS functionality is invoked only when explicitly requested

---

#### 2. Data Model Extension

A new logical data model, **`DGGSCube`**, is introduced:

* `RasterCube`: `(time, bands, y, x)`
* `DGGSCube`: `(time, bands, cell_id)`

Initially, `DGGSCube` wraps xarray structures and reuses backend execution utilities where possible. Minimal, surgical extensions may be introduced only if required for dimension validation or cube typing.

Raster workflows remain unaffected.

---
### Entry Process: `resample_dggs`

A new process is introduced:

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

This performs:

```
RasterCube → DGGSCube
```

#### Supported methods

* **bin_mean**: pixel-to-cell aggregation
* **nearest**: nearest-neighbor interpolation
* **gauss**: Gaussian-weighted interpolation

The goal is to support both aggregation-based and interpolation-based resampling strategies.

## Process Compatibility

Many existing processes remain directly applicable:

* apply
* reduce_dimension
* aggregate_temporal
* masking
* merge_cubes (dimension-aligned)

Future work includes DGGS-aware spatial neighborhood and hierarchical operations.

### Multi-Level Handling

Supports:

* Single refinement level
* Multiple refinement levels (initially returned as separate cubes)

#### Future Option: `zuniq` Encoding

The `healpix-geo` library provides a **zuniq encoding**, which packs `(depth, ipix)` into a single `uint64` identifier.

This enables:

* Multi-resolution stacking in a single spatial dimension
* Hierarchical DGGS representation
* Avoidance of `(level, cell_id)` padding
* Cleaner storage in Zarr

This is not required for the initial prototype but is a viable future enhancement for hierarchical DGGS workflows.

Reference: `healpix-geo` documentation (zuniq encoding support).

### Key Dependencies Used by the Plugin

The DGGS plugin builds on the following core packages:

* **xarray** – data model and lazy computation
* **healpy** - basic math operations related to conversion 
* **healpix-geo** – HEALPix geometry operations:

  * lon/lat → cell mapping
  * target cell coverage restriction
  * cell center coordinate generation
  * optional hierarchical encoding (`zuniq`)
* **pyresample** – KD-tree based nearest and gaussian interpolation
* **numpy** – numerical operations
* **dask** (via xarray) – scalable execution

These are introduced at the plugin level and do not modify the dependency surface of the core backend.

---

## Plugin Repository Structure

```text
openeo_dggs_plugin/
├─ pyproject.toml
├─ README.md
├─ src/
│  └─ openeo_dggs_plugin/
│     ├─ __init__.py
│     ├─ plugin.py
│     │
│     ├─ data_model/
│     │  └─ dggs_cube.py
│     │
│     ├─ processes/
│     │  ├─ __init__.py
│     │  └─ resample_dggs.py
│     │
│     └─ specs/
│        └─ resample_dggs.json
│
└─ tests/
   ├─ test_registration.py
   ├─ test_resample_regular.py
   └─ test_resample_curvilinear.py
```

---

## What Each Important Part Does

### `data_model/dggs_cube.py`

Defines:

* `DGGSCube`
* Expected dimensions: `(time, bands, cell_id)`
* DGGS metadata (grid_name, level, indexing_scheme)

---

### `processes/resample_dggs.py`

Implements:

* `bin_mean`
* `nearest`
* `gauss`

Converts:

```
RasterCube → DGGSCube
```

---

### `specs/resample_dggs.json`

Defines the openEO process specification:

* arguments (`data`, `level`, `grid_system`, `method`)
* return type
* description
* parameter schema

This keeps:

* Process implementation
* Process spec

cleanly separated.

---

### `plugin.py`

Handles registration of new processes with the backend.

Example:

```python
from openeo_pg_parser_networkx import ProcessRegistry
from . import processes

def register(registry: ProcessRegistry | None = None):
    registry = registry or ProcessRegistry.default_registry
    registry.register_from_module(processes)
```

## How It Plugs into `openeo-processes-dask`

### Via Entry Point (Preferred)

In `pyproject.toml` of the plugin:

```toml
[tool.poetry.plugins."openeo_processes_dask.processes"]
openeo_dggs_plugin = "openeo_dggs_plugin.plugin:register"
```

When installed:

* `openeo-processes-dask` discovers the plugin
* Calls `register()`
* `resample_dggs` becomes available automatically

---