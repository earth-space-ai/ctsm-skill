# Physics and Options

CTSM exposes physics choices through three layers:

1. **Compset alias**: picks a coherent default bundle (physics version, mode, forcing, river model).
2. **`xmlchange CLM_*`**: changes the major switches that drive `build-namelist`.
3. **`user_nl_clm`**: fine-grained namelist overrides.

This document explains the physics versions, the `-bgc {sp,bgc,fates}` mode
switch, prognostic crop, FATES, the river models, the urban / lake / glacier
landunits, and the fire schemes.

---

## Physics version: `CLM_PHYSICS_VERSION`

| Value | When to use |
|-------|-------------|
| `clm6_0` | Current default in `ctsm5.4.x`. CMIP7 surface datasets, CRUJRA2024 forcing, `li2024` fire, Bytnerowicz N fixation, Sturm/Jordan1991 snow thermal conductivity. |
| `clm5_0` | The CLM5 of Lawrence et al. 2019. CMIP6 surface datasets, GSWP3 forcing, `li2016` fire. Use for CMIP6 reproducibility. |
| `clm4_5` | CLM4.5 of Oleson et al. 2013. Maintained for backward comparison only. Many newer features unavailable. |

Set with `./xmlchange CLM_PHYSICS_VERSION=clm6_0` (or pick a compset whose
`CLM<NN>` slot already encodes it). Defaults flow through
`bld/namelist_files/namelist_defaults_ctsm.xml`, which is keyed on physics
version.

The `LND_TUNING_MODE` XML variable selects a calibration set tied to a
physics version and forcing pair, for example `clm6_0_CRUJRA2024`. Mismatched
combinations (CLM6 physics with CLM5 tuning, or vice versa) will run but
will not reproduce published behavior, so always confirm
`./xmlquery LND_TUNING_MODE` against your chosen compset.

---

## Vegetation mode: `CLM_BLDNML_OPTS -bgc {sp,bgc,fates}`

Three mutually exclusive modes select how vegetation is represented:

### SP (Satellite Phenology)

Prescribed monthly LAI, SAI, and canopy height from MODIS-derived datasets.
No prognostic carbon or nitrogen. Cheap, useful for biogeophysics-only
studies (snow, soil temperature, energy fluxes).

```bash
./xmlchange CLM_BLDNML_OPTS="-bgc sp"
```

### BGC (Biogeochemistry)

Full prognostic CN cycle: vegetation grows leaves and roots based on
photosynthesis, allocates C and N among pools, dies and decomposes through
litter and soil organic matter. Includes:

- FUN (Fixation and Uptake of Nitrogen) for plant N uptake competition.
- `vcmax_opt` selects the photosynthesis acclimation scheme (LUNA is the default).
- Prognostic carbon isotopes via `use_c13` and `use_c14` (defaults differ by
  CMIP era; in `ctsm5.4`+ they are on for `HistClm60Bgc` with CRUJRA or CAM7
  forcing and off for SSP, single-point, and older forcings).

```bash
./xmlchange CLM_BLDNML_OPTS="-bgc bgc"
```

Add `-crop` to enable prognostic crop:

```bash
./xmlchange CLM_BLDNML_OPTS="-bgc bgc -crop"
```

### FATES

Replace CTSM's CN representation with the FATES ecosystem demography model
(NGEET/fates). FATES tracks individual plant cohorts in size-structured
patches, simulates competition for light and (optionally) water, and has its
own fire and disturbance schemes.

```bash
./xmlchange CLM_BLDNML_OPTS="-bgc fates"
```

Common FATES toggles in `user_nl_clm`:

```fortran
use_fates                  = .true.
use_fates_sp               = .false.   ! .true. for FATES with prescribed phenology
use_fates_planthydro       = .false.   ! plant hydraulics
use_fates_managed_fire     = .false.
use_fates_tree_damage      = .false.
use_fates_cohort_age_tracking = .false.
fates_parteh_mode          = 1         ! plant element handler mode
fates_seeddisp_cadence     = 0         ! seed dispersal
fates_hydro_solver         = 2DPicard  ! 2D Picard hydro solver (the new default)
```

Many FATES parameters live on a separate parameter file
(`fates_paramfile`) and the FATES parameter file format and contents change
between API tags. Check the
[HLM-FATES compatibility table](https://fates-users-guide.readthedocs.io/en/latest/user/release-tags-compat-table.html)
when changing the FATES tag.

`doc/IMPORTANT_NOTES.md` lists FATES options that are currently untested or
not turned on by default (e.g., `fates_spitfire_mode == 5`,
`fates_photosynth_acclimation == kumarathunge2019`,
`fates_stomatal_model == medlyn2011`).

---

## Crop: `-crop`

Adds a crop landunit to the gridcell with prognostic crop functional types
(corn, soy, spring wheat, etc.), planting and harvest from
crop calendars (`tools/crop_calendars/`), fertilizer application, and
irrigation. Crop is BGC-only (no SP+crop). With `clm6_0` defaults,
irrigation is automatically on for transient cases.

To override the crop calendar with prescribed sowing/harvest dates from
GGCMI, use `tools/crop_calendars/` to regrid the input data, then point at
it through `stream_fldfilename_swindow*`-family namelist variables.

---

## Fire schemes (`fire_method`)

Selectable factory in `src/biogeochem/CNFireFactoryMod.F90`. Available
backends:

| Value | Source |
|-------|--------|
| `nofire` | no fire |
| `li2014` | Li et al. 2014 |
| `li2016` | Li et al. 2016, default in CLM5 |
| `li2021` | Li et al. 2021 |
| `li2024` | Li et al. 2024, default in `clm6_0` |

In `user_nl_clm`:

```fortran
fire_method = 'li2024'
```

For FATES, fire is handled inside FATES (`fates_spitfire_mode`,
`use_fates_managed_fire`). The CTSM-side FATES fire bridge is
`src/biogeochem/FATESFire*Mod.F90`.

---

## Snow

Several knobs:

- `snow_thermal_cond_method`, `snow_thermal_cond_glc_method`,
  `snow_thermal_cond_lake_method`. ctsm5.4 uses `Sturm` over land and lake
  and `Jordan1991` over glaciers by default for `clm6_0`.
- `snicar_*` family in `bld/namelist_files/namelist_definition_ctsm.xml`
  controls the SNICAR snow albedo model: dust optics, snow grain shape,
  number of radiation bands, intermixing of black carbon and dust. Many
  combinations are not validated; see `doc/IMPORTANT_NOTES.md`.
- `SnowCoverFractionFactoryMod.F90` selects the fSCA scheme:
  `NiuYang2007` or `SwensonLawrence2012` (the default since CLM5).

---

## Urban

Three urban density classes (TBD = tall building district, HD = high
density, MD = medium density). In `clm6_0`+ the historical urban inputs are
partitioned into TBD, HD, and MD in proportion to GaoOneill present-day
classes. Urban canyon code lives in `src/biogeophys/Urban*.F90`. Each
urban landunit has 5 columns: roof, sunwall, shadewall, impervious road,
pervious road.

`URBAN_PARAM_TYPE` and the urban parameter file (`paramfile`) define
geometry and material properties. Local Climate Zone (LCZ) classes 1 to 10
are now part of the urban logic on `clm6_0`.

---

## Lake

Lakes are a separate landunit with their own column type and a
multi-layer thermal solver (`src/biogeophys/Lake*Mod.F90`). Most lake
options are off by default. `allowlakeprod` and `replenishlakec` are
listed as untested in `doc/IMPORTANT_NOTES.md`.

---

## Glacier (CISM and glacier-mec)

CTSM couples to CISM through the glacier-mec landunit, which has up to 10
elevation classes per gridcell (`maxpatch_glc = 10`, do not change). The
file family is `src/biogeophys/GlacierSurfaceMassBalanceMod.F90` plus
`src/main/glc2lndMod.F90`, `lnd2glcMod.F90`. The XML variable
`GLC_TWO_WAY_COUPLING` switches between one-way (CTSM tells CISM about
surface mass balance) and two-way coupling (CISM tells CTSM about glacier
extent changes).

In `clm6_0` the default for `glcmec_downscale_longwave` is `FALSE`, since
turning longwave downscaling off improves Greenland melt and runoff.

---

## River routing: MOSART vs mizuRoute vs RTM

| Component | When to use |
|-----------|-------------|
| `MOSART` | Default in CTSM compsets, gridded river routing on a 0.5 degree network |
| `mizuRoute` | Reach-based vector routing, supports Hydrologic Response Units (HRUs); useful for hydrology studies |
| `RTM` | Legacy CESM1 river model, kept for backward comparisons only |
| `SROF` | Stub (no river), use only when river behavior is irrelevant |

Pick at compset creation time (`MOSART` vs `RTM` vs `SROF` in the longname).
mizuRoute requires a custom compset and a HRU mesh.

---

## Photosynthesis and stomatal options

In `src/biogeophys/PhotosynthesisMod.F90`. Common knobs:

- `stomatal_conductance_method`, the choice between Ball-Berry-style and
  Medlyn-style stomatal conductance.
- `vcmax_opt`, the temperature acclimation scheme. `vcmax_opt = 4` is
  flagged as untested.
- `LUNA` (Leaf Utilization of Nitrogen for Assimilation), in
  `src/biogeophys/LunaMod.F90`, optimizes leaf N allocation between
  photosynthetic components. Activate with `use_luna = .true.`.

---

## Soil hydrology and water retention

`src/biogeophys/SoilWaterRetentionCurveFactoryMod.F90` switches between:

- `ClappHornberg1978`, the historical default.
- `VanGenuchten1980`, a more flexible curve, available since CLM5.

The factory is set by `soil_water_retention_curve_method` in `user_nl_clm`.

---

## Anomaly forcing

Recent additions (`ctsm5.4`) add automatic, more flexible use of anomaly
forcings for CMIP6 ISSP cases. Documentation:
https://escomp.github.io/CTSM/users_guide/running-special-cases/Running-with-anomaly-forcing.html.

---

## What is experimental

`doc/IMPORTANT_NOTES.md` is the authoritative list of options that are
either tested but off by default, or untested entirely. Spot-check before
turning on anything exotic. Notable unsupported settings include:

- `use_cndv` (deprecated in favor of FATES)
- `use_vichydro` (deprecated)
- `pertlim` (deprecated)
- `h2osfcflag` (deprecated)
- `urban_traffic` (not implemented)
- `reduce_dayl_factor` (commented out in source)
- `use_extralakelayers`
- Almost any `vcmax_opt = 4`

---

## Where to next

- Run a case once you have picked physics: `running-cases.md`
- Inspect what is in the source for any of the above: `architecture.md`
- Output a new variable for a non-default scheme: `output-history-fields.md`
- Submit a new physics PR: `contributing-pr.md`
