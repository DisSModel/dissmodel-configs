# dissmodel-configs

External registry of executor configurations for the
[DisSModel](https://github.com/DisSModel) ecosystem, consumed by
**dissmodel-platform**.

A TOML file merged here means the executor is available on the platform.
New executors enter the platform only through a PR to this repository.

---

## Purpose

`dissmodel-configs` decouples model parameterization from both the
executor packages and the platform itself. The platform reads each TOML
at experiment submission time, resolves the executor class from the
registered Python package, and injects the spec into the experiment
record.

---

## TOML structure

Use `brmangue_raster.toml` as the canonical reference. Every entry must
follow this structure:

```toml
[model]
# ── Registry metadata ────────────────────────────────────────────────
executor_module = "<python.module.path>"   # module that exports the executor
name            = "<executor_name>"        # must match ModelExecutor.name exactly
class           = "<executor_name>"        # same value as name (platform convention)
description     = "..."
package         = "git+https://github.com/DisSModel/<repo>@<ref>"
dissmodel       = ">=0.6.0,<0.7.0"        # compatible dissmodel range

# ── Model-level config (record.resolved_spec["model"]) ───────────────
# Fields read by the executor from spec go here directly under [model].
# Examples: land_use_types, cell_area, complementar_lu, annual_demand, …

[model.parameters]
# Runtime parameters (record.parameters).
# Fields read via params.get("key") go here.
# Examples: n_steps, resolution, demand_csv, end_time, …

# ── Executor-specific sub-tables ─────────────────────────────────────
# [[model.potential]], [[model.allocation]], [model.transition_matrix], …
```

Fields under `[model]` (excluding `parameters`) are exposed as
`record.resolved_spec["model"]`, and `[model.parameters]` is exposed
as `record.parameters`. Executors access these two paths independently,
so placing a field in the wrong section silently falls back to its
default.

---

## File naming convention

```
<model>_<substrate>.toml
```

| Segment | Convention |
|---|---|
| `model` | Short identifier for the scientific model or package (e.g. `brmangue`, `lucc_continuous`, `lucc_discrete`) |
| `substrate` | Computational substrate: `raster` or `vector` |

Examples: `brmangue_raster.toml`, `lucc_continuous_raster.toml`,
`lucc_continuous_vector.toml`, `lucc_discrete_vector.toml`.

---

## `name` / `class` convention

The `name` field **must match the `name` class attribute** of the
`ModelExecutor` subclass — not the Python class name.

```python
# disslucc_continuous/executors/clue_like_raster_executor.py
class LUCCRasterExecutor(ModelExecutor):
    name = "lucc_raster"          # ← this goes in the TOML
```

```toml
# lucc_continuous_raster.toml
[model]
name  = "lucc_raster"            # ← must match exactly
class = "lucc_raster"
```

The executor registry in each package (`EXECUTOR_REGISTRY` or
`[project.entry-points."dissmodel.executors"]` in `pyproject.toml`)
is the authoritative source. Always confirm the name from the source
code, never from documentation.

---

## Version compatibility — why `dissmodel>=0.6.0,<0.7.0`

`dissmodel` v0.6.0 introduced a **breaking change**: `end_time` in
`Environment` became **inclusive** (TerraME-style). Executors compiled
against `<0.6.0` compute one extra step when run on `>=0.6.0`, and vice
versa.

Every TOML in this registry must declare:

```toml
dissmodel = ">=0.6.0,<0.7.0"
```

The lower bound guarantees inclusive `end_time` semantics. The upper
bound protects against future breaking changes in the `0.7.x` series.

Both `disslucc-continuous` and `disslucc-discrete` already declare
`dissmodel>=0.6.0,<0.7.0` in their own `pyproject.toml`.

---

## How to add an executor

1. Implement `ModelExecutor` in your package, set a unique `name`
   attribute, and register it via `[project.entry-points."dissmodel.executors"]`
   in `pyproject.toml`.
2. Fork this repository and create `models/<model>_<substrate>.toml`
   following the structure above.
3. Confirm the `name` value from the source code (`executor/__init__.py`
   or the class attribute directly).
4. Declare `dissmodel = ">=0.6.0,<0.7.0"` (or the appropriate range
   once a new breaking change is released).
5. Open a PR. Merge = available on the platform.

---

## Current registry state

| File | Executor `name` | Substrate | Source package | Status |
|---|---|---|---|---|
| `brmangue_raster.toml` | `brmangue_raster` | Raster | `brmangue-dissmodel` | Registered |
| `lucc_continuous_raster.toml` | `lucc_raster` | Raster | `disslucc-continuous` v0.2.0 | Registered |
| `lucc_continuous_vector.toml` | `lucc_vector` | Vector | `disslucc-continuous` v0.2.0 | Registered |
| `lucc_discrete_vector.toml` | `clue_s_vector` | Vector | `disslucc-discrete` v0.1.0 | Registered |

### Executors intentionally not registered

| Executor `name` | Package | Reason |
|---|---|---|
| `lucc_benchmark` | `disslucc-continuous` | Internal benchmark/validation only; hardcoded Lab1 constants |
| `lucc_validation` | `disslucc-discrete` | Internal validation only; hardcoded Lab6 (cs_moju) constants |

These are used for CI validation and TerraME reference comparison. They
are not general-purpose simulation executors and should not be exposed
on the platform.
