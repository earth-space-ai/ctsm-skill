---
name: ctsm
description: >
  Self-contained guide to CTSM (Community Terrestrial Systems Model), the
  land component of CESM that subsumes the Community Land Model (CLM). Covers
  the git-fleximod sandbox, the CIME case workflow (create_newcase, case.setup,
  case.build, case.submit), the src/biogeochem and src/biogeophys module
  families, CLM6 vs CLM5 vs CLM4.5 physics, FATES ecosystem demography,
  LILAC standalone coupling, history output (h0a/h0i tapes, hist_fincl), the
  tools/ pre-processors (mksurfdata_esmf, subset_data, fsurdat_modifier), and
  the ESCOMP/CTSM fork-and-PR workflow with ChangeLog and ChangeSum.
  Progressive disclosure, start in this file for routing, drill into
  reference/ for depth.
version: 0.1.0
tags:
  - earth-science
  - land-surface-model
  - community-land-model
  - clm
  - ctsm
  - cesm
  - fortran
  - fates
  - lilac
  - cime
  - climate
---

# CTSM, Community Terrestrial Systems Model, Complete Guide

> **CTSM** = Community Terrestrial Systems Model (includes the Community Land Model, CLM, of CESM)
> Maintainers: NCAR/CGD CTSM software engineering and science teams
> Source: https://github.com/ESCOMP/CTSM
> Docs: https://escomp.github.io/CTSM/
> Forum: https://bb.cgd.ucar.edu/cesm/forums/ctsm-clm-mosart-rtm.134/
> Latest science version covered here: ctsm5.4.x (CLM6 physics default)

**What CTSM does:** Solves coupled land surface energy, water, carbon, and
nitrogen budgets on a subgrid hierarchy (gridcell, landunit, column, patch).
Represents vegetation phenology and biogeochemistry, snow and soil hydrology,
glaciers, lakes, urban canyons, crops with irrigation and prognostic phenology,
river routing (MOSART or mizuRoute), and optional ecosystem demography
(FATES). Runs offline with CDEPS data atmosphere (I compsets), coupled inside
CESM with CAM, or coupled to a host atmosphere model through LILAC.

**Who this skill is for:** Agents, students, and researchers who want to
clone, build, run, modify, debug, and contribute to CTSM.

---

## Quick Decision Tree

```
"What do I need?"
│
├─ First time, what is CTSM and how do I clone and build it?
│  └─ Read: reference/getting-started.md
│     (git clone, git-fleximod update, supported machines, py_env_create)
│
├─ I want to run a standard CTSM case (offline, I compset)
│  └─ Read: reference/running-cases.md
│     (create_newcase, case.setup, case.build, case.submit, xmlchange)
│
├─ I want to understand how the source tree is organized
│  └─ Read: reference/architecture.md
│     (src/biogeochem, src/biogeophys, src/main, src/dyn_subgrid, FATES, cpl)
│
├─ I want to pick a physics version, BGC vs SP vs FATES, or turn on crop / fire
│  └─ Read: reference/physics-and-options.md
│     (CLM6 vs CLM5 vs CLM4.5, BGC, SP, FATES, MOSART, urban, lake, glacier)
│
├─ I want to control history output, add fields, change frequency
│  └─ Read: reference/output-history-fields.md
│     (h0a/h0i tapes, hist_fincl, hist_nhtfrq, hist_mfilt, hist_dov2xy)
│
├─ I want to couple CTSM to my own atmosphere model (no CESM, no CAM)
│  └─ Read: reference/lilac-coupling.md
│     (Lightweight Infrastructure for Land Atmosphere Coupling, ESMF API)
│
├─ I want to make a surface dataset, subset to a site, or modify inputs
│  └─ Read: reference/python-tools.md
│     (mksurfdata_esmf, subset_data, fsurdat_modifier, mesh_maker, ctsm_pylib)
│
├─ I want to submit a pull request to ESCOMP/CTSM
│  └─ Read: reference/contributing-pr.md
│     (fork, feature branch, ChangeLog, ChangeSum, ExpectedTestFails, run_sys_tests)
│
└─ The model won't build, won't run, or gives wrong answers
   └─ Read: reference/debugging.md
      (build failures, externals desync, runtime crashes, restart issues, baselines)
```

---

## What's Inside CTSM

```
CTSM/                                        ← top-level (this is $CTSMROOT)
├── README, README.md, README_GITFLEXIMOD.rst
├── WhatsNewInCTSM5.4.md                     ← release notes for ctsm5.4.x
├── CONTRIBUTING.md                          ← how to open issues and PRs
├── bin/git-fleximod                         ← submodule package manager
├── py_env_create                            ← creates ctsm_pylib conda env
├── ccs_config/                              ← CIME grids, compsets, machines
├── cime/                                    ← CIME (Common Infrastructure for Modeling Earth)
├── cime_config/
│   ├── config_compsets.xml                  ← I compsets, etc. (~130 compsets)
│   ├── config_component.xml                 ← CTSM XML variables (CLM_*)
│   ├── config_pes.xml                       ← processor layouts
│   ├── config_tests.xml                     ← CTSM-specific test types
│   ├── user_nl_clm                          ← starter user namelist (copied into each case)
│   ├── usermods_dirs/clm/                   ← named user-mod directories (e.g., NEON sites)
│   └── testdefs/ExpectedTestFails.xml       ← known-failing tests
├── components/
│   ├── cmeps/                               ← NUOPC mediator (default driver)
│   ├── cdeps/                               ← data models (DATM, DICE, DOCN, DROF, DWAV)
│   ├── cism/                                ← Community Ice Sheet Model
│   ├── mosart/                              ← Model for Scale Adaptive River Transport
│   ├── mizuroute/                           ← reach-based river transport
│   └── rtm/                                 ← legacy River Transport Model
├── doc/
│   ├── ChangeLog, ChangeSum                 ← per-tag history (UpdateChangeLog.pl)
│   ├── IMPORTANT_NOTES.md                   ← unsupported and experimental options
│   ├── Quickstart.GUIDE.md                  ← tiny NUOPC quick start
│   ├── README.NUOPC_driver.md               ← MCT to NUOPC migration notes
│   ├── source/                              ← Sphinx docs source
│   └── design/                              ← software design docs
├── bld/
│   ├── build-namelist                       ← Perl driver
│   ├── CLMBuildNamelist.pm                  ← namelist generator
│   └── namelist_files/
│       ├── namelist_definition_ctsm.xml     ← every CLM namelist variable
│       └── namelist_defaults_ctsm.xml       ← per-physics defaults
├── lilac/                                   ← Lightweight Infrastructure for Land Atmosphere Coupling
│   ├── src/                                 ← LILAC Fortran source
│   ├── atm_driver/                          ← bundled atm driver for testing LILAC
│   ├── build_ctsm                           ← script to build CTSM as a static library
│   ├── make_runtime_inputs                  ← script to stage runtime inputs
│   └── docs/
├── src/                                     ← CTSM Fortran source (~370 .F90 files)
│   ├── main/                                ← clm_driver, clm_initializeMod, controlMod, histFileMod
│   ├── biogeochem/                          ← CN cycle, fire, FUN, MEGAN, satellite phenology
│   ├── biogeophys/                          ← canopy, soil, snow, urban, lake, photosynthesis
│   ├── soilbiogeochem/                      ← decomposition, MIMICS
│   ├── dyn_subgrid/                         ← landuse change, harvest, dynamic PFT
│   ├── init_interp/                         ← online interpolation of initial conditions
│   ├── fates/                               ← FATES submodule (NGEET/fates)
│   ├── cpl/                                 ← driver coupling caps (nuopc, lilac, share_esmf)
│   ├── utils/                               ← clm_time_manager, restUtilMod, MatrixMod
│   ├── self_tests/                          ← internal tests run from a CTSM system test
│   ├── unit_test_shr/, unit_test_stubs/     ← pFUnit test scaffolding
├── tools/
│   ├── mksurfdata_esmf/                     ← parallel ESMF-based surface dataset generator
│   ├── site_and_regional/                   ← subset_data, mesh_maker, NEON and PLUMBER2 wrappers
│   ├── modify_input_files/                  ← fsurdat_modifier, mesh_mask_modifier
│   ├── crop_calendars/                      ← GGCMI sowing/harvest date processor
│   ├── mkmapgrids/                          ← legacy SCRIP grid scripts
│   └── contrib/                             ← user-contributed pre/post scripts
├── python/
│   ├── ctsm/                                ← Python package: subset_data.py, mesh_maker.py, etc.
│   └── conda_env_ctsm_py.yml                ← ctsm_pylib spec
├── share/                                   ← CESM shared code
└── libraries/                               ← PIO (deprecated copy)
```

**For most users:** create a case with `create_newcase`, edit `user_nl_clm`
and `xmlchange` settings, build with `case.build`, submit with `case.submit`.
Source code edits live under `src/` (physics) and `bld/namelist_files/`
(namelist plumbing).

---

## Critical Rules

1. **Always run `./bin/git-fleximod update` after a fresh clone or after
   checking out a new tag.** CTSM uses git-fleximod (not plain git submodules)
   to manage CIME, CMEPS, CDEPS, CISM, MOSART, mizuRoute, RTM, FATES, ccs_config.
   Without this step the build fails with missing components.

   ```bash
   git clone https://github.com/ESCOMP/CTSM.git my_ctsm_sandbox
   cd my_ctsm_sandbox
   ./bin/git-fleximod update
   ```

2. **Switching tags requires a re-update.** After `git checkout ctsm5.4.002`,
   rerun `./bin/git-fleximod update`. The `.gitmodules` file pins each
   submodule to a `fxtag`, and your sandbox can drift if you forget.

3. **Never edit `cime_config/user_nl_clm` directly for a case.** That file is
   the template copied into each new case directory. Edit the
   per-case `user_nl_clm` inside the case directory, not the repo copy.

4. **CMEPS NUOPC is the only driver as of ctsm5.4.** The MCT driver was
   removed. Older docs that mention `rpointer.drv`, `cpl.log`, or
   `DATM_CLMNCEP_YR_*` apply to MCT. The current names are `rpointer.cpl`,
   `med.log` plus `drv.log`, and `DATM_YR_*`.

5. **History tapes are now split into `hXa` and `hXi` files (ctsm5.4+).**
   `h0` becomes `h0a` (averaged or non-instantaneous) and `h0i` (instantaneous).
   Post-processors that hard-code `h0` filenames must be updated. See
   `reference/output-history-fields.md`.

6. **Run cases under a scratch filesystem, not inside the source tree.**
   CIME creates `$CASEROOT` (case directory) and `$RUNDIR`/`$EXEROOT` (run and
   build) by default in user-specific scratch paths defined in
   `ccs_config/cesm/machines/config_machines.xml`. Do not override these to
   point inside the git checkout.

7. **`mksurfdata_esmf` is currently only fully validated on Derecho.** For
   other machines use `tools/site_and_regional/subset_data` to derive
   regional and single-point surface datasets from existing global ones.
   See https://github.com/ESCOMP/CTSM/issues/2341.

8. **For PRs touching FATES, update both repos.** FATES lives in `src/fates/`
   as a separate ESCOMP-managed submodule pinned to an `fxtag` in
   `.gitmodules`. Coordinate FATES tag bumps with the FATES team.

---

## Quick Start (CTSM offline, I compset, on a CIME-supported machine)

```bash
# 1. Clone and pull submodules
git clone https://github.com/ESCOMP/CTSM.git my_ctsm_sandbox
cd my_ctsm_sandbox
./bin/git-fleximod update

# 2. Create a case (CLM6 BGC with crop, 1850 control, 1 degree)
cd cime/scripts
./create_newcase --case ~/cases/i2000ClmBgcCrop \
                 --res f09_t232 \
                 --compset I2000Clm60BgcCrop \
                 --mach derecho

# 3. Configure, edit namelists, build, submit
cd ~/cases/i2000ClmBgcCrop
./case.setup
$EDITOR user_nl_clm                  # add hist_fincl, finidat overrides, etc.
./xmlchange STOP_OPTION=nyears,STOP_N=5
./case.build
./case.submit
```

Output appears under `$RUNDIR` (see `./xmlquery RUNDIR`). History files are
named `<case>.clm2.h0a.YYYY-MM.nc` and `<case>.clm2.h0i.YYYY-MM-DD-SSSSS.nc`.

Full walkthrough in `reference/getting-started.md` and
`reference/running-cases.md`.

---

## Reference Documents

| Document | What's inside |
|----------|---------------|
| `reference/getting-started.md` | Repos, `git clone`, `git-fleximod update`, supported machines (Derecho, Casper, Izumi, Perlmutter), `py_env_create`, `ctsm_pylib`, common clone pitfalls |
| `reference/architecture.md` | Top-level layout, `src/biogeochem`, `src/biogeophys`, `src/main`, `src/dyn_subgrid`, `src/utils`, `src/cpl`, FATES submodule, subgrid hierarchy (gridcell, landunit, column, patch) |
| `reference/running-cases.md` | `create_newcase`, compset naming convention, `case.setup`, `case.build`, `case.submit`, `xmlchange`, `xmlquery`, `user_nl_clm`, `CaseDocs/`, resubmit, branch and hybrid runs, the `README.CHECKLIST.new_case` checklist |
| `reference/physics-and-options.md` | CLM6 vs CLM5 vs CLM4.5, `CLM_PHYSICS_VERSION`, `CLM_BLDNML_OPTS` (`-bgc sp`, `-bgc bgc`, `-bgc fates`, `-crop`), `LND_TUNING_MODE`, FATES options, MOSART vs mizuRoute, urban (TBD/HD/MD, LCZ), lake, glacier (CISM), fire (Li2014/2016/2021/2024) |
| `reference/output-history-fields.md` | History file architecture, h0a/h0i split, `hist_fincl1..10` and `hist_fexcl*`, `hist_nhtfrq`, `hist_mfilt`, `hist_dov2xy` for 1D vector PFT/column output, single-point and PLUMBER2/NEON cases |
| `reference/lilac-coupling.md` | LILAC purpose, ESMF API, `lilac/build_ctsm`, `lilac/make_runtime_inputs`, `atm_driver` test, integration into a host atmosphere |
| `reference/python-tools.md` | `tools/mksurfdata_esmf` (gen_mksurfdata_build, gen_mksurfdata_namelist, gen_mksurfdata_jobscript), `tools/site_and_regional/subset_data` (point and region), `tools/modify_input_files/fsurdat_modifier`, `mesh_maker`, `ctsm_pylib` Python environment |
| `reference/debugging.md` | Build failures (missing externals, ESMF mismatch), runtime crashes (mesh / fsurdat mismatch, finidat resolution), `lnd.log` and `cesm.log`, restart pointer mismatches, `CLM_FORCE_COLDSTART`, baseline comparisons |
| `reference/contributing-pr.md` | Fork ESCOMP/CTSM, feature branch off master, `ctsm-software@ucar.edu`, opening an issue first, PR review, `doc/UpdateChangeLog.pl`, `ChangeSum`, `ExpectedTestFails.xml`, `run_sys_tests`, integrator workflow |
