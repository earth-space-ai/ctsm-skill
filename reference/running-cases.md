# Running CTSM Cases (CIME workflow)

CTSM cases are managed through CIME, the Common Infrastructure for Modeling
Earth. The four-step pattern is universal across CESM components:

```
create_newcase  →  case.setup  →  case.build  →  case.submit
```

Everything else (changing settings, adding namelist mods, branching off a
spinup) is done by editing files inside the case directory between these
steps.

---

## Step 1: `create_newcase`

```bash
cd cime/scripts
./create_newcase --case ~/cases/i2000_bgc \
                 --res f09_t232 \
                 --compset I2000Clm60BgcCrop \
                 --mach derecho \
                 --project P12345678
```

Important flags:

| Flag | Meaning |
|------|---------|
| `--case` | absolute path to the case directory CIME will create |
| `--res` | grid alias (`f09` is 0.9x1.25 atmosphere, `t232` is the 0.25-degree TX2 ocean grid mesh, etc.); see `query_config --grids` |
| `--compset` | compset alias or longname; see below |
| `--mach` | machine name (must be in `ccs_config/cesm/machines/config_machines.xml`) |
| `--compiler` | optional, default for the machine |
| `--project` | accounting code on HPC machines |
| `--user-mods-dir` | path to a `usermods_dirs/clm/<name>/` directory that pre-stages namelist and xml mods |
| `--run-unsupported` | skip the warning for combinations not flagged `science_support` |

To list available choices:

```bash
./query_config --compsets clm    # all CLM-related compsets
./query_config --grids           # all grid aliases
./query_config --machines        # ported machines
```

There are about 130 CTSM compsets defined in
`cime_config/config_compsets.xml`.

---

## Compset naming convention

Compset longnames follow:

```
TIME_ATM[%phys]_LND[%phys]_ICE[%phys]_OCN[%phys]_ROF[%phys]_GLC[%phys]_WAV[%phys][_BGC%phys]
```

For an example, `I2000Clm60BgcCrop` has alias `I2000Clm60BgcCrop` and longname:

```
2000_DATM%GSWP3v1_CLM60%BGC-CROP_SICE_SOCN_MOSART_SGLC_SWAV
```

| Slot | Value | Meaning |
|------|-------|---------|
| TIME | `2000` | Year-2000 control conditions (constant CO2, etc.) |
| ATM | `DATM%GSWP3v1` | data atmosphere with GSWP3 v1 forcing |
| LND | `CLM60%BGC-CROP` | CLM6 physics, BGC mode, prognostic crop |
| ICE | `SICE` | stub ice |
| OCN | `SOCN` | stub ocean |
| ROF | `MOSART` | Model for Scale Adaptive River Transport |
| GLC | `SGLC` | stub glacier |
| WAV | `SWAV` | stub wave |

Common TIME prefixes:

| Prefix | Meaning |
|--------|---------|
| `2000` | Year-2000 control conditions |
| `1850` | Pre-industrial control |
| `HIST` | Historical (1850 to ~2014, dataset-dependent) |
| `SSP585`, `SSP370`, etc. | CMIP6 future scenarios |

Common LND modifiers:

| Suffix | Meaning |
|--------|---------|
| `%SP` | Satellite Phenology, prescribed LAI from MODIS, no carbon cycle |
| `%BGC` | Full prognostic biogeochemistry (CN cycle) |
| `%BGC-CROP` | BGC plus prognostic crop |
| `%FATES` | FATES ecosystem demography |
| `%FATES-SP` | FATES with prescribed phenology |

Common ATM data forcings:

| Suffix | Meaning |
|--------|---------|
| `%CRUJRA2024b` | CRUJRA 2024b reanalysis (1901 to 2023), default for CLM6 |
| `%GSWP3v1` | GSWP3 v1 (1901 to 2014), default for CLM5 |
| `%QIA` | Qian et al. 2006 (legacy) |
| `%1PT` | single-point forcing (PLUMBER2, NEON, or user-supplied) |
| `%CAM6.0` | CAM6 reanalysis-style forcing |

A handy starting list of CLM6 compsets:

| Alias | Use |
|-------|-----|
| `I2000Clm60Sp` | SP, year-2000 control |
| `I2000Clm60Bgc` | BGC, year-2000 control |
| `I2000Clm60BgcCrop` | BGC plus crop, year-2000 control |
| `I2000Clm60Fates` | FATES, year-2000 control |
| `IHistClm60Bgc`, `IHistClm60BgcCrop` | historical 1850 to 2014 |
| `I1PtClm60Bgc`, `I1PtClm60Fates` | single-point variants |

---

## Step 2: `case.setup`

```bash
cd ~/cases/i2000_bgc
./case.setup
```

This:

- Creates `$CASEROOT/CaseDocs/` with a snapshot of every namelist and stream file the build will use.
- Writes the `case.run` and `case.st_archive` batch scripts.
- Creates the `user_nl_*` files that you edit (`user_nl_clm`, `user_nl_datm`, `user_nl_datm_streams`, `user_nl_mosart`).

`./case.setup --help` shows reset (`--reset`) and clean (`--clean`) options.

---

## Step 3: edit XML and namelists

Two editable surfaces:

**(a) XML variables**, via `xmlchange` and `xmlquery`:

```bash
./xmlquery STOP_OPTION,STOP_N,RESUBMIT
./xmlchange STOP_OPTION=nyears,STOP_N=10,RESUBMIT=4
./xmlquery -p CLM            # show all CLM_* vars
./xmlquery LND_TUNING_MODE
```

Frequently changed XML variables:

| Variable | Purpose |
|----------|---------|
| `STOP_OPTION`, `STOP_N` | run length per submission (`nyears`, `nmonths`, `ndays`, `nsteps`) |
| `RESUBMIT` | number of times `case.submit` re-queues itself |
| `CONTINUE_RUN` | `FALSE` for first submit, `TRUE` for restarts |
| `RUN_TYPE` | `startup`, `hybrid`, `branch` |
| `RUN_REFCASE`, `RUN_REFDATE`, `RUN_REFTOD` | for hybrid and branch |
| `DATM_YR_ALIGN`, `DATM_YR_START`, `DATM_YR_END` | DATM streams cycling window |
| `CLM_PHYSICS_VERSION` | `clm4_5`, `clm5_0`, `clm6_0` |
| `CLM_BLDNML_OPTS` | `-bgc {sp,bgc,fates}`, `-crop`, `-clm_demand`, etc. |
| `LND_TUNING_MODE` | tuning preset, e.g., `clm6_0_CRUJRA2024` |
| `CLM_FORCE_COLDSTART` | `on` to skip `finidat` and start cold |
| `JOB_QUEUE`, `JOB_WALLCLOCK_TIME` | queue settings |
| `NTASKS_LND`, `NTHRDS_LND`, `ROOTPE_LND` | processor layout |

**(b) `user_nl_clm`** for CLM namelist mods. Each line is `name = value`.
Example:

```fortran
! user_nl_clm
! Override LAI input dataset for SP runs
fsurdat = '/glade/p/cesm/cseg/inputdata/lnd/clm2/surfdata_map/.../my_fsurdat.nc'

! Add an extra history tape with daily soil moisture
hist_fincl2  = 'H2OSOI', 'TSOI', 'TLAI'
hist_nhtfrq  = 0, -24
hist_mfilt   = 1, 365
hist_dov2xy  = .true., .true.

! Force a cold start for the first chunk
finidat = ' '
```

Do **not** edit `cime_config/user_nl_clm` (that is the template). Edit the
case directory copy.

For the full list of every namelist variable, see
`bld/namelist_files/namelist_definition_ctsm.xml`.

---

## Step 4: `case.build`

```bash
./case.build
```

This compiles CTSM, CMEPS, CDEPS, MOSART, and the data models for your
case. It also runs `build-namelist` to materialize the actual `lnd_in`,
`datm_in`, etc., from the XML state plus your `user_nl_*` mods. The
materialized namelists end up in `$RUNDIR` and a snapshot in
`CaseDocs/lnd_in`.

If you change `user_nl_clm` or any `xmlchange`, rerun `./case.build
--clean-all` only if the change requires a recompile. For a namelist-only
change, `./preview_namelists` regenerates `CaseDocs/` and `./case.submit`
stages the new namelists into `$RUNDIR` automatically.

---

## Step 5: `case.submit`

```bash
./case.submit
```

This queues the run on the machine batch system. The submission script is
`case.run`. For a multi-resubmit chain:

```bash
./xmlchange RESUBMIT=10,CONTINUE_RUN=FALSE
./case.submit                  # first chunk; CONTINUE_RUN flips to TRUE for the rest
```

Output goes to `$RUNDIR` (see `./xmlquery RUNDIR`). If short-term archiving is
on, completed chunks are moved to `$DOUT_S_ROOT` between resubmits.

---

## Pre-submit checklist

The repo ships a checklist at `README.CHECKLIST.new_case`. The essentials:

- `./xmlquery -p CLM` to confirm `CLM_*` settings.
- `./xmlquery -p CLM_PHYSICS_VERSION` matches what you wanted.
- `./xmlquery -p CLM_BLDNML_OPTS` shows the right `-bgc {sp,bgc,fates}` and `-crop`.
- `./xmlquery LND_TUNING_MODE` matches the physics and forcing combination.
- For an I compset, `./xmlquery -p DATM_YR` matches your forcing window.
- `grep stream_year_first CaseDocs/lnd_in` and the matching `_last` and `model_year_align` are sane.
- `./xmlquery RUN_TYPE`, then `grep finidat CaseDocs/lnd_in` (for startup or
  hybrid) or `grep nrevsn CaseDocs/lnd_in` (for branch) shows the right
  initial-condition file.
- Run a short test (a month or so), then inspect `lnd.log` and `atm.log` for
  warnings and to confirm the file paths read.

---

## Useful case-directory commands

| Command | Purpose |
|---------|---------|
| `./preview_run` | show the batch submission, processor layout, and runtime estimate |
| `./preview_namelists` | regenerate `CaseDocs/` after a namelist change without submitting |
| `./check_input_data` | list every input dataset and check whether it is staged |
| `./check_case` | sanity-check the case for common configuration errors |
| `./case.cmpgen_namelists` | compare current namelists against a baseline |
| `./case.test` | run the registered system test for this case |
| `./case.st_archive` | manually invoke short-term archiving |
| `./xmlchange BATCH_COMMAND_FLAGS+="--reservation=foo"` | append batch flags |

---

## Restarts, branches, and hybrids

Three `RUN_TYPE` modes:

- **startup**: cold start, or restart from a `finidat` provided in `user_nl_clm`.
- **hybrid**: pick up state from a different case at a given date, then reset the date to a new value.
- **branch**: continue a different case from its restart files exactly where it left off, including the date.

For branch and hybrid, set:

```bash
./xmlchange RUN_TYPE=branch,RUN_REFCASE=spinup_case_name,RUN_REFDATE=0301-01-01,RUN_REFTOD=00000
```

CIME will look for restart files at `$DIN_LOC_ROOT/restart/...` or in a
case-specified location. See https://escomp.github.io/CTSM/users_guide/.

---

## Where to next

- Pick which physics, BGC mode, fire scheme, and other options: `physics-and-options.md`
- Customize history output (which fields, what frequency, vector or gridded): `output-history-fields.md`
- Make a new surface dataset for a site: `python-tools.md`
- Things that go wrong at submit time: `debugging.md`
