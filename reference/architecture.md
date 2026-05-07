# CTSM Architecture

CTSM is a large Fortran codebase (about 370 `.F90` files in `src/` plus the
FATES submodule) organized along three axes:

1. **Physics family**: biogeophysics (energy, water, snow, urban, lake) vs
   biogeochemistry (carbon, nitrogen, fire, FUN, MEGAN, satellite phenology),
   plus soil biogeochemistry (decomposition, MIMICS) and dynamic subgrid.
2. **Driver coupling layer**: the same physics code is wrapped by either the
   NUOPC cap (for CESM and standalone CIME runs) or the LILAC cap (for direct
   coupling to a host atmosphere model). Both live under `src/cpl/`.
3. **Subgrid hierarchy**: gridcell, landunit, column, patch. Almost every
   physics call operates over a filter of indices into these levels.

This document maps the source tree, walks the subgrid hierarchy, names the
top-level driver routines, and points at the FATES boundary.

---

## Top-level layout

```
src/
├── main/                ← top-level control, initialization, history, restart
├── biogeochem/          ← CN cycle, fire, FUN, MEGAN, satellite phenology, dust, VOC
├── biogeophys/          ← canopy, soil, snow, urban, lake, photosynthesis, hydrology
├── soilbiogeochem/      ← decomposition, MIMICS, soil C and N pools
├── dyn_subgrid/         ← annual landuse change, harvest, dynamic PFT, transient
├── init_interp/         ← online interpolation of finidat to current grid
├── fates/               ← FATES submodule (NGEET/fates), ecosystem demography
├── cpl/                 ← driver coupling caps
│   ├── nuopc/           ← NUOPC cap for CESM CMEPS
│   ├── lilac/           ← LILAC cap for standalone host-atm coupling
│   ├── share_esmf/      ← shared ESMF helpers
│   └── utils/
├── utils/               ← clm_time_manager, restUtilMod, MatrixMod, spmdMod, domainMod
├── self_tests/          ← internal tests run as part of certain system tests
├── unit_test_shr/       ← shared mock modules for pFUnit unit tests
└── unit_test_stubs/     ← stub modules so unit tests can compile without full deps
```

---

## src/main: the heartbeat

The land model time loop is in `clm_driver.F90`. Initialization is in
`clm_initializeMod.F90` (called once at startup). Restart and history I/O are
in `restFileMod.F90` and `histFileMod.F90`.

Key files in `src/main/`:

| File | Role |
|------|------|
| `clm_driver.F90` | per-timestep driver, calls all physics in order |
| `clm_initializeMod.F90` | one-time init, reads `finidat`, builds subgrid, calls `init_*` from each module |
| `clm_instMod.F90` | declares the long list of derived-type instances (waterstate, energyflux, etc.) |
| `controlMod.F90` | parses the `clm_inparm` namelist |
| `histFileMod.F90` | history tape machinery (hist_fincl, hist_nhtfrq, etc.) |
| `restFileMod.F90.in` | restart file read and write |
| `clm_varctl.F90` | global runtime control flags (use_bgc, use_fates, finidat path) |
| `clm_varpar.F90` | model dimensions (nlevsoi, numpft, nlevdecomp) |
| `clm_varcon.F90` | physical constants |
| `decompMod.F90` | MPI decomposition |
| `filterMod.F90` | landunit/column/patch filters used everywhere |
| `GridcellType.F90`, `LandunitType.F90`, `ColumnType.F90`, `PatchType.F90` | the four subgrid derived types |
| `atm2lndMod.F90`, `lnd2atmMod.F90` | atmosphere coupling fluxes (in CESM directions) |
| `glc2lndMod.F90`, `lnd2glcMod.F90` | glacier (CISM) coupling |
| `init_hydrology.F90` | hydrology-specific init |
| `initVerticalMod.F90` | soil and snow vertical layering |

`clm_driver` is the right entry point when you want to find where a
particular flux is computed. It calls (roughly in order) the hydrology,
energy, photosynthesis, BGC, fire, dynamic subgrid, and history routines.

---

## Subgrid hierarchy

CTSM represents heterogeneity below the atmospheric grid through a four-level
hierarchy. Reading and modifying CTSM code without internalizing this is
painful, so know it cold:

```
gridcell  ──┐  one per atmospheric grid cell (or single point)
            ├── landunit  ──┐  vegetated, urban (TBD/HD/MD), lake, glacier, crop, ice
                            ├── column  ──┐  soil column with its own snow/soil state
                                          ├── patch  one PFT (or crop type) on the column
```

| Level | Typical count per gridcell | Examples |
|-------|---------------------------|----------|
| gridcell | 1 | the atmospheric cell |
| landunit | up to 9 | natural-veg, crop, urban TBD, urban HD, urban MD, lake, glacier, glacier-mec (multiple elevation classes), wetland |
| column | varies | one per landunit usually, urban has 5 (roof, sunwall, shadewall, impervious road, pervious road), glacier-mec has up to 10 elevation classes |
| patch | 1 to 17+ | one per PFT on the column (16 natural PFTs in CLM5/6 plus bare soil, crop landunit adds crop functional types) |

Filter arrays in `filterMod.F90` (e.g., `filter(nc)%num_lakec`,
`filter(nc)%lakec`) loop over only the indices that match a condition. Almost
every physics module begins with one or more filters.

The hierarchy types are defined in:

- `GridcellType.F90`, `LandunitType.F90`, `ColumnType.F90`, `PatchType.F90`
  in `src/main/`.
- Variable-constant modules `landunit_varcon.F90`, `column_varcon.F90`,
  `pftconMod.F90` define the integer codes (e.g.,
  `istcrop`, `isturb_tbd`, `nat_veg`).

History output by default averages everything to the gridcell (`hist_dov2xy
= .true.`). To get raw 1D vector output at the patch or column level, set
`hist_dov2xy = .false.` for that tape (see `output-history-fields.md`).

---

## src/biogeophys

Biogeophysics handles the energy and water budgets, snow physics, urban
canyons, lakes, and photosynthesis. About 89 files. Highlights:

| File / module family | Purpose |
|----------------------|---------|
| `CanopyFluxesMod.F90` | sensible and latent heat from vegetated columns |
| `BareGroundFluxesMod.F90` | same for bare soil |
| `PhotosynthesisMod.F90` | C3/C4 photosynthesis, stomatal conductance (Medlyn or Ball-Berry) |
| `LunaMod.F90` | LUNA leaf nitrogen optimization |
| `SoilHydrologyMod.F90`, `SoilWaterMovementMod.F90` | soil water, Richards equation |
| `SoilWaterRetentionCurve*Mod.F90` | factory: Clapp-Hornberger 1978 vs van Genuchten 1980 |
| `SnowHydrologyMod.F90`, `SnowSnicarMod.F90`, `SnowCoverFraction*Mod.F90` | snow layers, SNICAR albedo, fSCA scheme (NiuYang2007 vs SwensonLawrence2012) |
| `SoilTemperatureMod.F90`, `LakeTemperatureMod.F90` | thermal solver |
| `UrbanFluxesMod.F90`, `UrbanRadiationMod.F90`, `UrbBuildTempOleson2015Mod.F90` | urban canyon energy budget |
| `IrrigationMod.F90` | crop irrigation |
| `BalanceCheckMod.F90`, `TotalWaterAndHeatMod.F90` | conservation diagnostics |
| `WaterStateBulkType.F90`, `WaterFluxBulkType.F90`, `Wateratm2lndBulkType.F90` | bulk water type, plus `*TracerType.F90` versions for water isotopes |
| `OzoneFactoryMod.F90`, `OzoneMod.F90`, `OzoneOffMod.F90` | ozone damage, factory pattern |

The `*Factory*` naming is a deliberate Fortran-style polymorphic factory:
the `Factory` module returns a class-allocatable instance of one of the
sibling implementations based on a namelist switch.

---

## src/biogeochem

Biogeochemistry is everything carbon, nitrogen, and emissions related. About
73 files. Highlights:

| File / module family | Purpose |
|----------------------|---------|
| `CNDriverMod.F90` | CN cycle driver, called once per timestep |
| `CNVegCarbonStateType.F90`, `CNVegCarbonFluxType.F90`, etc. | vegetation C and N pools and fluxes |
| `CNAllocationMod.F90`, `CNFUNMod.F90` | allocation, including FUN (Fixation and Uptake of Nitrogen) |
| `CNPhenologyMod.F90` | leaf-on/leaf-off, evergreen vs deciduous |
| `CNFireBaseMod.F90`, `CNFireLi2014Mod.F90` ... `CNFireLi2024Mod.F90` | factory of fire schemes; `li2024` is the ctsm5.4 default |
| `CropType.F90`, `CropReprPoolsMod.F90` | prognostic crop module |
| `NutrientCompetitionMethodMod.F90` and `NutrientCompetitionFactoryMod.F90` | competition between roots and microbes for mineral N |
| `SatellitePhenologyMod.F90` | SP mode, prescribed LAI from MODIS |
| `MEGANFactorsMod.F90`, `VOCEmissionMod.F90` | biogenic VOC emissions for atmospheric chemistry |
| `DustEmisFactory.F90`, `DustEmisZender2003.F90`, `DustEmisLeung2023.F90` | dust emission factory |
| `FATESFireBase.F90`, `FATESFireFactoryMod.F90`, `FATESFireDataMod.F90` | bridge from CTSM to FATES fire driver |
| `dynCNDVMod.F90` | (deprecated) CN dynamic vegetation |

---

## src/dyn_subgrid

Annual updates that change subgrid weights or convert area between landunits.
This runs once per simulated year, not every timestep:

- `dynSubgridDriverMod.F90`, `dynSubgridControlMod.F90` orchestrate the year-end update.
- `dynpftFileMod.F90`, `dynpftFileMod`, `dynpftFileMod` read transient PFT, urban, lake, and harvest streams.
- `dynLandunitAreaMod.F90` updates landunit areas.
- `dynPatchStateUpdaterMod.F90`, `dynColumnStateUpdaterMod.F90` conserve state when areas change.
- `dynHarvestMod.F90` applies wood harvest.
- `dynFATESLandUseChangeMod.F90` translates landuse change for FATES.
- `dynConsBiogeochemMod.F90`, `dynConsBiogeophysMod.F90` enforce conservation across the area change.

---

## src/utils

Utilities that physics code depends on but that are not physics themselves:

- `clm_time_manager.F90`, model time and calendar (wraps `seq_time_mod`).
- `restUtilMod.F90.in` (a `.in` template that the build expands), low-level restart I/O.
- `MatrixMod.F90`, `SparseMatrixMultiplyMod.F90`, the sparse matrix carbon-cycle accelerator.
- `domainMod.F90`, gridded domain (legacy, mostly used inside tools).
- `spmdMod.F90`, MPI helpers.
- `IssueFixedMetadataHandler.F90`, machinery that records on history files which GitHub issues have been resolved relative to a known-bug baseline.

---

## src/cpl

Three caps wrap the same physics so it can run under different drivers:

- `cpl/nuopc/`, the NUOPC cap. Used for all CESM runs (CMEPS mediator) and
  for offline runs through CIME's data atmosphere CDEPS.
- `cpl/lilac/`, the LILAC cap. Used when an external atmosphere model wants
  to call CTSM directly through ESMF without all of CESM. See
  `lilac-coupling.md`.
- `cpl/share_esmf/`, ESMF helpers shared between caps.
- `cpl/utils/`, shared coupling utilities.

Both caps allocate the same `clm_inst*` derived-type instances and call
`clm_drv` (in `src/main/clm_driver.F90`) per timestep. The cap differences
are about how forcing fields arrive and where `lnd2atm`/`atm2lnd` data go.

---

## FATES (`src/fates`)

FATES, the **Functionally Assembled Terrestrial Ecosystem Simulator**, is an
ecosystem demography submodule maintained by the **NGEET** consortium at
https://github.com/NGEET/fates. CTSM pins a FATES tag in `.gitmodules` (e.g.,
`sci.1.89.0_api.43.0.0`).

The boundary is in:

- `src/utils/clmfates_interfaceMod.F90`, the CTSM-side facade.
- `src/biogeochem/CNVegetationFacade.F90`, abstracts vegetation state behind
  a single interface so the same `clm_driver` call works whether the
  vegetation is the standard CN representation or a FATES instance.
- `src/biogeochem/FATESFire*Mod.F90`, the fire bridge.

Activate with `use_fates = .true.` in `user_nl_clm` or by picking a
`%FATES` compset such as `I2000Clm60Fates`.

---

## Where to next

- Wire up a case and actually run this code: `running-cases.md`
- Pick a physics version and what `use_*` flags imply: `physics-and-options.md`
- Add an output field that lives on a column or patch: `output-history-fields.md`
- Where to put a new physics module that needs a PR: `contributing-pr.md`
