# LILAC: Coupling CTSM to a Host Atmosphere

LILAC, the **Lightweight Infrastructure for Land-Atmosphere Coupling**,
provides a high-level Fortran API for embedding CTSM inside an atmospheric
model that is not CESM. It is built on ESMF (Earth System Modeling
Framework). The intended hosts are atmospheric models (WRF was the
prototype) that want CTSM as the land surface scheme without bringing in
CMEPS, CDEPS, and the rest of the CESM machinery.

This document covers what LILAC is, how it is laid out in `lilac/`, how to
build CTSM as a static library that LILAC can link against, and how to use
the bundled `atm_driver` to test the LILAC API end-to-end before wiring it
into a real atmospheric model.

---

## When to use LILAC

| Scenario | Use LILAC? |
|----------|-----------|
| Standalone offline CTSM with CDEPS DATM | No, use a CIME I compset |
| CTSM coupled inside CESM with CAM, CICE, POP, CISM | No, use the NUOPC cap and CMEPS |
| CTSM as the land surface in WRF, MPAS, or another non-CESM atmospheric model | Yes |
| Quick regional simulation driven by reanalysis | Probably no, CDEPS DATM with single-point or regional forcing is easier |

LILAC is the right tool when the host atmosphere has its own time loop and
its own MPI decomposition and you want CTSM to advance one timestep at a
time when the host calls into it.

---

## Repository layout

```
lilac/
├── src/                     ← LILAC Fortran source (the cap layer)
├── docs/                    ← Sphinx documentation source
│   ├── index.rst
│   ├── developers.rst
│   └── api.rst
├── atm_driver/              ← bundled fake atmosphere driver for end-to-end testing
│   ├── atm_driver.F90
│   ├── atm_driver_in        ← namelist for atm_driver
│   ├── Makefile
│   └── cheyenne.sub         ← legacy batch submission template
├── build_ctsm               ← script to build CTSM as a static library for LILAC
├── make_runtime_inputs      ← script to stage runtime inputs (lnd_in, surfdata, mesh) for a LILAC run
├── download_input_data      ← downloads the CTSM input datasets needed for a LILAC run
├── bld_templates/           ← namelist and stream template files for runtime input generation
├── cmake/                   ← CMake macros (LILAC uses CMake, not CIME, for its own build)
├── CMakeLists.txt
├── ci/, tests/              ← LILAC test infrastructure
├── stub_rof/                ← stub river runoff component for LILAC tests
└── docker-compose.yml, Dockerfile  ← containerized dev environment
```

The CTSM-side cap that LILAC actually calls is in `src/cpl/lilac/`.

---

## How LILAC is structured

LILAC exposes three procedures to the host:

1. **`lilac_init`**, called once at startup. Initializes ESMF, sets up the
   exchange grids, allocates CTSM derived types, reads `lnd_in`, and runs
   CTSM's own initialization (the same `clm_initializeMod` path as CESM).
2. **`lilac_run`**, called once per coupling timestep. Receives the host
   atmosphere's downward forcing (radiation, precipitation, temperature,
   humidity, wind), advances CTSM's internal time loop, and returns
   land-to-atmosphere fluxes.
3. **`lilac_final`**, called at shutdown.

Internally LILAC does the regridding between the atmospheric grid and the
land grid using ESMF mesh files. Regridding weights are computed online (no
offline mapping files), the same approach as the NUOPC cap.

For the API contract see `lilac/docs/api.rst`. For developer-oriented build
and test instructions see `lilac/docs/developers.rst`.

---

## Build CTSM as a static library

```bash
cd lilac
./build_ctsm /path/to/build/dir --machine derecho --compiler intel
```

`build_ctsm` is a Python wrapper that:

1. Detects (or accepts) the machine name and pulls the matching CIME
   machine config.
2. Builds CTSM and its dependencies (CIME share code, `csm_share`, ESMF,
   PIO) using the same toolchain as a CIME case.
3. Produces `libctsm.a` (or `.so` if requested) plus `lilac` headers in
   the build directory.

Run `./build_ctsm --help` for all options. For machines not known to CIME,
you must port CIME first (see `getting-started.md`).

---

## Stage runtime inputs

A LILAC run still needs a CTSM `lnd_in` namelist, a surface dataset
(`fsurdat`), a parameter file (`paramfile`), and ESMF mesh files for the
atmospheric and land grids. The helper script:

```bash
cd lilac
./make_runtime_inputs --rundir /path/to/rundir \
                      --compset I2000Clm60Sp \
                      --lnd-grid f09 \
                      --atm-mesh /path/to/atm.mesh.nc
```

writes a `lnd_in` based on the requested compset, copies the right
`fsurdat` and `paramfile`, and stages the mesh files into `rundir`. From
there the host application is expected to point its LILAC initialization
at `rundir`.

If the input datasets are not on the local file system, run
`./download_input_data` first to pull them from the CESM input data server.

---

## End-to-end test with `atm_driver`

The `atm_driver/` directory contains a minimal fake atmospheric model that
exercises the LILAC API the same way a real host would. It is the canonical
LILAC test:

```bash
cd lilac/atm_driver
$EDITOR atm_driver_in        # set namelist (number of timesteps, mesh files, etc.)
make                         # builds atm_driver against libctsm.a built earlier
./atm_driver                 # or qsub cheyenne.sub on a batch system
```

`atm_driver` produces standard CTSM history files in the run directory, so
you can compare against an equivalent CIME I-compset run to confirm that the
LILAC integration is correct.

---

## Containerized development

For developers who want to iterate on LILAC without an HPC environment,
`lilac/Dockerfile` and `lilac/docker-compose.yml` define a Linux image with
ESMF, NetCDF, Fortran, and CMake preinstalled:

```bash
cd lilac
docker-compose build
docker-compose run
```

Inside the container, `cd /lilac/build && cmake .. && make && ctest` runs
the LILAC unit and integration tests.

---

## What LILAC does NOT do

- It does not provide a coupling interface to a non-atmospheric host (e.g.,
  it is not designed for ocean-only or hydrology-model-only coupling).
- It does not include CMEPS (the NUOPC mediator). The host atmosphere is
  responsible for time stepping; LILAC just advances CTSM when called.
- It does not include CDEPS (the CESM data models). For DATM-driven
  configurations use CIME I compsets, not LILAC.
- It does not include MOSART or mizuRoute. River routing is the host
  application's responsibility (or use the `stub_rof/` placeholder for
  testing).

---

## Practical notes

- **LILAC is younger than the NUOPC cap.** Expect rougher edges. Track
  open issues at https://github.com/ESCOMP/CTSM/issues with the `lilac` label.
- **CTSM tag and LILAC compatibility.** LILAC ships in the same CTSM
  repository, so the LILAC source always matches the CTSM tag. There is no
  separate LILAC version to track.
- **ESMF version matters.** LILAC requires a relatively recent ESMF (8.4+).
  Mismatched ESMF builds against the LILAC cap produce link errors that
  look like missing `ESMF_*` symbols.
- **Regridding is online.** No offline mapping files are needed for the
  atmosphere-land remap. Just provide ESMF mesh files for both grids.

---

## Where to next

- See where the LILAC cap connects to CTSM: `architecture.md` (`src/cpl/lilac/`)
- Build a regular CIME case if you do not actually need a non-CESM host: `running-cases.md`
- LILAC-related build failures: `debugging.md`
