# Python Tools and Input Datasets

Almost every CTSM run depends on offline-generated input datasets:

- An `fsurdat` (surface dataset) describing per-PFT, per-soil-type, urban,
  glacier, and lake fractions.
- An ESMF `mesh` file for both atmosphere and land grids (NUOPC needs no
  domain files but it does need meshes).
- A `paramfile` (PFT and physics parameters).
- Stream files for transient land use, atmospheric forcing, N deposition,
  CO2, etc.

CTSM ships a Python toolchain to create, subset, and modify these. All of
the tools live under `tools/` and a sibling Python package under
`python/ctsm/`. They share a single conda environment, **`ctsm_pylib`**.

---

## Setting up `ctsm_pylib`

Create or refresh the environment from the top of the CTSM checkout:

```bash
./py_env_create
conda activate ctsm_pylib
```

This installs `python/conda_env_ctsm_py.yml` (Python 3.13.2 in current
versions). On Derecho/Casper you may need `module load conda` first.

If you have an old environment (`ctsm_pylib` from before `ctsm5.3.040`),
preserve it with:

```bash
./py_env_create -r ctsm_pylib_old
```

The `python/` directory is the source of truth for the tools; entries in
`tools/` are shell wrappers that call into `python/ctsm/`.

---

## `tools/site_and_regional/subset_data` (the easy path)

Subsets an existing global `fsurdat` (and optionally a mesh file) to a
single point or a regional rectangle. This is the recommended tool for
single-site and regional runs on machines other than Derecho.

```bash
cd tools/site_and_regional
./subset_data point \
    --lat 45.5 --lon 275.6 \
    --site BondvilleIL \
    --create-surface \
    --inputdata-dir $DIN_LOC_ROOT
```

For a region:

```bash
./subset_data region \
    --lat1 30 --lat2 50 --lon1 230 --lon2 290 \
    --reg WesternUS \
    --create-surface --create-mesh \
    --inputdata-dir $DIN_LOC_ROOT
```

Outputs land in the current directory by default. Use `--outdir` to
override.

Single-point cases need only `fsurdat` (no mesh). Regional cases need
`fsurdat` plus a mesh file.

---

## `tools/mksurfdata_esmf` (the heavy path)

Distributed-memory parallel program (MPI plus ESMF for regridding plus PIO
for I/O) that creates `fsurdat` files at any resolution from the raw
input datasets (PFT, LAI, soil color, urban, lake, glacier, fire).

> **Important note:** As of `ctsm5.4.x`, `mksurfdata_esmf` is fully
> validated only on Derecho. On other CIME machines it may build, but
> regridding has not been recently regression-tested. See
> https://github.com/ESCOMP/CTSM/issues/2341.

Workflow on Derecho:

```bash
cd tools/mksurfdata_esmf

# 1. Build the executable (uses CIME)
./gen_mksurfdata_build

# 2. Generate a target namelist
conda activate ctsm_pylib
./gen_mksurfdata_namelist --res 0.9x1.25 --start-year 1850 --end-year 1850

# 3. Build a single-job batch script
./gen_mksurfdata_jobscript_single \
    --number-of-nodes 2 --tasks-per-node 128 \
    --namelist-file target.namelist

# 4. Submit
qsub mksurfdata_jobscript_single.sh
```

For multiple datasets in one submission (e.g., all years for a transient
landuse series):

```bash
./gen_mksurfdata_jobscript_multi --number-of-nodes 2 --scenario global-present
qsub mksurfdata_jobscript_multi.sh
```

For NEON or PLUMBER2 sites, use the wrappers under
`tools/site_and_regional/`:

```bash
./neon_surf_wrapper        # creates fsurdat for every NEON site
./plumber2_surf_wrapper    # creates fsurdat for every PLUMBER2 site
```

### Two gotchas

- **Raw input files must not be NetCDF4.** Convert with
  `nccopy -k cdf5 oldfile newfile`.
- **The LAI raw dataset must have `time` as an unlimited dimension.** Use
  `ncks --mk_rec_dmn time file_with_time_equals_12.nc -o file_with_time_unlimited.nc`.

---

## `tools/modify_input_files/fsurdat_modifier`

In-place modifier of an existing `fsurdat`. Reads the original, applies a
set of changes specified in a config file, writes a copy. Useful for
sensitivity studies (set FMAX to zero, force a uniform snowpack, override
a single PFT fraction at a site, etc.).

```bash
conda activate ctsm_pylib
cd tools/modify_input_files
cp modify_fsurdat_template.cfg my_run.cfg
$EDITOR my_run.cfg
./fsurdat_modifier my_run.cfg --verbose
```

Current scope is limited to the simplest CLM-SP variables. BGC, fire,
urban, VIC hydrology, lake, transient, and crop-related fields in the
`fsurdat` are not yet modifiable through this tool. For those, regenerate
through `mksurfdata_esmf` or use `subset_data`.

Difference vs `subset_data`:

| `fsurdat_modifier` option | `subset_data` option |
|--------------------------|----------------------|
| `std_elev` (user-set STD_ELEV) | `--uniform-snowpack` (sets STD_ELEV = 20) |
| `max_sat_area` (user-set FMAX) | `--cap-saturation` (sets FMAX = 0) |

---

## `tools/modify_input_files/mesh_mask_modifier`

Modifies a mesh file's element mask (turns gridcells on/off). Used for
removing problematic gridcells without regenerating the mesh:

```bash
conda activate ctsm_pylib
cp tools/modify_input_files/modify_mesh_template.cfg my_mesh.cfg
$EDITOR my_mesh.cfg
./mesh_mask_modifier my_mesh.cfg --verbose
```

---

## `tools/site_and_regional/mesh_maker` and `mesh_plotter`

`mesh_maker` builds a SCRIP-format ESMF mesh file from a NetCDF grid file
with `lat` and `lon` coordinate variables. `mesh_plotter` renders an
existing mesh as a sanity-check map. Both are wrappers around
`python/ctsm/mesh_maker.py` and `python/ctsm/mesh_plotter.py`.

```bash
conda activate ctsm_pylib
cd tools/site_and_regional
./mesh_maker --input grid.nc --output grid_mesh.nc \
             --lat lat --lon lon --mask landmask
./mesh_plotter grid_mesh.nc
```

---

## `tools/crop_calendars/`

Tools to regrid and process the GGCMI sowing-and-harvest-date datasets so
they can be ingested as CTSM streams for prognostic crop runs. See the
README inside the directory.

---

## `tools/contrib/`

User-contributed pre/post-processing scripts. They are not part of the
formally tested toolset; quality varies. Notable entries:

| Script | Purpose |
|--------|---------|
| `SpinupStability_BGC_v12_SE.ncl`, `SpinupStability_BGC_v11.ncl`, `SpinupStability_SP_v10.ncl` | check whether a BGC or SP spinup has reached equilibrium |
| `run_clmtowers/` | run a batch of single-site tower cases |
| `run_clm_historical.v11.csh` | template for a long historical case chain |
| `add_tillage_to_paramsfile.py` | add tillage parameters to a CLM `paramfile` |
| `tweak_latlons.py` | small lat/lon perturbation for sensitivity tests |
| `ssp_anomaly_forcing_smooth/` | smooth anomaly forcing for SSP cases |
| `popden.ncl`, `abm_raw.ncl`, `create_scrip_file.ncl` | NCL scripts for population density and grid creation |
| `modify_singlept_site/` | hand-modify a single-point fsurdat |

---

## `tools/param_utils` and `python/ctsm/param_utils`

Tools to query and modify CTSM `paramfile` (PFT and physics parameter
files). Useful for perturbed-parameter ensembles (PPE). Documentation:
https://escomp.github.io/CTSM/users_guide/using-clm-tools/paramfile-tools.html.

---

## Comparing CTSM output: `cprnc`

`cprnc` is the CESM NetCDF comparison tool (built as part of CIME under
`cime/tools/cprnc`). It reports field-by-field differences between two
NetCDF files, with statistics on RMS, min, max, and exact-match. The
standard CTSM regression test framework relies on `cprnc` to compare a
test against a baseline.

```bash
cprnc file_a.nc file_b.nc | tee cprnc.out
```

---

## Where to next

- Wire a freshly-made `fsurdat` into a case via `user_nl_clm`: `running-cases.md`
- Hand-build a single-site case end to end: `running-cases.md` plus this doc
- Submit a tools fix back to ESCOMP: `contributing-pr.md`
