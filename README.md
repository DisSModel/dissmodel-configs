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
package         = "<pip requirement>"      # optional; omit when the executor is in the worker image
dissmodel       = ">=0.6.0,<0.8.0"        # compatible dissmodel range

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
<model>.toml            # the executor works on any substrate (e.g. lucc_continuous.toml)
<model>_<substrate>.toml  # the executor is specific to one substrate (e.g. brmangue_raster.toml)
```

| Segment | Convention |
|---|---|
| `model` | Short identifier for the scientific model (e.g. `brmangue`, `lucc_continuous`, `lucc_discrete`) |
| `substrate` | `raster` or `vector`, only when the executor accepts just one |

Since DisSLUCC 0.5.0 a single executor reads a vector (rasterized on load) or
a GeoTIFF with named bands, so the DisSLUCC specs carry no substrate suffix.

---

## `name` / `class` convention

The `name` field **must match the `name` class attribute** of the
`ModelExecutor` subclass — not the Python class name.

```python
# disslucc/executors/continuous.py
class LuccContinuousExecutor(LuccExecutorBase):
    name = "lucc_continuous"      # ← this goes in the TOML
```

```toml
# lucc_continuous.toml
[model]
name  = "lucc_continuous"        # ← must match exactly
class = "lucc_continuous"
```

The executor registry in each package (`EXECUTOR_REGISTRY` or
`[project.entry-points."dissmodel.executors"]` in `pyproject.toml`)
is the authoritative source. Always confirm the name from the source
code, never from documentation.

---

## Version compatibility — why `dissmodel>=0.6.4,<0.8.0`

`dissmodel` v0.6.0 introduced a **breaking change**: `end_time` in
`Environment` became **inclusive** (TerraME-style). Executors compiled
against `<0.6.0` compute one extra step when run on `>=0.6.0`, and vice
versa. DisSLUCC 0.5.0 needs `>=0.6.4`.

Every TOML in this registry declares the `dissmodel` range its executor was
tested against. The platform compares it with the installed version and
logs a warning when it does not match.

```toml
dissmodel = ">=0.6.4,<0.8.0"     # DisSLUCC specs
```

The upper bound protects against future breaking changes. It was raised from
`<0.7.0` to `<0.8.0` after the DisSLUCC test suite and the platform runner
passed on dissmodel 0.7.0.

---

## How to add an executor

1. Implement `ModelExecutor` in your package, set a unique `name`
   attribute, and register it via `[project.entry-points."dissmodel.executors"]`
   in `pyproject.toml`.
2. Fork this repository and create `models/<model>.toml`
   following the structure above.
3. Confirm the `name` value from the source code (`executor/__init__.py`
   or the class attribute directly).
4. Declare the `dissmodel` range the executor was tested against.
   Leave out `package` when the executor is bundled in the worker image:
   with `package` set, the worker runs `pip install` before every job, and
   pip re-resolves `dissmodel` from that package's metadata.
5. Open a PR. Merge = available on the platform.

---

## Current registry state

| File | Executor `name` | Source package | Status |
|---|---|---|---|
| `brmangue_raster.toml` | `brmangue_raster` | `brmangue-dissmodel` | Registered |
| `lucc_continuous.toml` | `lucc_continuous` | `disslucc` 0.5.0 | Registered |
| `lucc_discrete.toml` | `lucc_discrete` | `disslucc` 0.5.0 | Registered |
| `lucc_saturation.toml` | `lucc_continuous_saturation` | `disslucc` 0.5.0 | Registered |

The three DisSLUCC specs are the ones in `examples/dissmodel-configs/` of the
[disslucc](https://github.com/DisSModel/disslucc) repository, where
`tests/test_executor_toml.py` checks that each one gives the same output as
the equivalent Python script. Keep the two in sync.

### Replaced

| File | Executor `name` | Replaced by |
|---|---|---|
| `lucc_continuous_raster.toml` | `lucc_raster` | `lucc_continuous.toml` |
| `lucc_continuous_vector.toml` | `lucc_vector` | `lucc_continuous.toml` |
| `lucc_discrete_vector.toml` | `clue_s_vector` | `lucc_discrete.toml` |

These pointed to `disslucc-continuous` and `disslucc-discrete`, which
predate DisSLUCC 0.5.0. The old `lucc_benchmark` and `lucc_validation`
executors (hardcoded Lab 1 / Lab 6 constants, CI only) stay out of the
registry.
