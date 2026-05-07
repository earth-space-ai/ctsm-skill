# Debugging CTSM

CTSM failures cluster into a few canonical buckets:

1. **Sandbox-state failures**: missing or out-of-sync submodules, wrong
   tag, stale `ctsm_pylib` environment.
2. **Build failures**: ESMF/PIO/NetCDF mismatch, compiler version, missing
   machine port.
3. **Namelist build failures**: `build-namelist` cannot find a default for
   your physics + forcing + grid combination.
4. **Runtime crashes**: mesh vs `fsurdat` size mismatch, `finidat`
   resolution mismatch, NaN propagation, conservation violation, restart
   pointer mismatch.
5. **Scientific surprises**: cold start instead of restart, wrong tuning
   mode, wrong forcing window.

This document gives the diagnostic moves for each.

---

## 1. Sandbox state

### Symptoms

- `cime/scripts/create_newcase: No such file or directory`
- `src/fates/main/EDMainMod.F90: No such file or directory`
- Linker errors for routines you know are in CMEPS or CDEPS.

### Diagnosis

```bash
./bin/git-fleximod status
```

Any row with `s` in column 1 is out of sync. Any row with `e` is missing
entirely.

### Fix

```bash
./bin/git-fleximod update
```

If that does not work, your `.gitmodules` may be in a bad state from a
manual edit. Look for unresolved merge markers:

```bash
grep '<<<<<<<' .gitmodules
```

After resolving, rerun update.

For a stale `ctsm_pylib`:

```bash
./py_env_create -r ctsm_pylib_old
conda activate ctsm_pylib
```

---

## 2. Build failures

### Symptom: `case.build` exits with non-zero

```bash
cd $CASEROOT
./case.build 2>&1 | tee build.log
```

Look at the last 200 lines of `build.log`. The actual compiler error usually
ends up many lines above the final error message. Useful patterns:

| Error fragment | Likely cause |
|----------------|-------------|
| `cannot find -lesmf` | ESMF module not loaded or built against wrong compiler |
| `cannot find -lpio`, `cannot find -lpioc` | PIO build mismatch (`module load` the right PIO) |
| `Symbol not found: __netcdf_MOD_*` | NetCDF Fortran built with different compiler than the build is using |
| `Error: Type mismatch in argument` | Likely a stale build object from a previous tag; `./case.build --clean-all` and rebuild |
| `error #5208: Syntax error` | Some compiler choke on a Fortran construct, often after a manual source edit |

The CIME build logs are under `$EXEROOT/<component>/log` (e.g.,
`$EXEROOT/lnd/clm.bldlog.<date>`). Always inspect the per-component log,
not just the top-level build summary.

To clean:

```bash
./case.build --clean-all     # clean every component (slow but thorough)
./case.build --clean lnd     # clean only CTSM
./case.build
```

### Symptom: `mksurfdata_esmf` build fails on a non-Derecho machine

This is a known limitation. Use `subset_data` instead, or port `mksurfdata_esmf`. Track
https://github.com/ESCOMP/CTSM/issues/2341.

---

## 3. Namelist build failures

`./case.build` runs `build-namelist` for every component and writes
materialized namelists to `$RUNDIR` and `CaseDocs/`. If `build-namelist`
fails, you see a Perl traceback with a `Cannot find default for ...` or
`Cannot resolve ...` message.

Most common causes:

- The XML defaults database (`bld/namelist_files/namelist_defaults_ctsm.xml`)
  has no entry for your `(CLM_PHYSICS_VERSION, LND_TUNING_MODE, grid,
  compset)` combination. Either pick a supported combination, or set
  `--run-unsupported` in `create_newcase` and explicitly specify the
  required inputs in `user_nl_clm`.
- A missing `fsurdat`. Set explicitly:
  ```fortran
  fsurdat = '/path/to/your/surfdata_xxx.nc'
  ```
- A missing `paramfile`. Pick the right one for the physics version from
  the input data tree (`$DIN_LOC_ROOT/lnd/clm2/paramdata/`).

To regenerate namelists after editing `user_nl_clm` without rebuilding:

```bash
./preview_namelists
```

Then inspect `CaseDocs/lnd_in`.

---

## 4. Runtime crashes

### Symptom: case fails immediately with mesh / fsurdat mismatch

```
ERROR: Mesh element count (... ) does not match fsurdat dimension ...
```

The atmospheric mesh, land mesh, and `fsurdat` all need to have the same
number of land elements (or compatible ones). Cause is usually that you
mixed grids, e.g. `f09` mesh with an `f19` fsurdat, or set
`LND_DOMAIN_MESH` by hand without updating `fsurdat`.

Fix: pick one resolution and confirm `./xmlquery LND_DOMAIN_MESH` matches
the mesh that produced the `fsurdat`.

### Symptom: `finidat` resolution mismatch

```
ERROR: finidat ... has nlevsoi = ..., expected ...
```

The initial-condition file `finidat` was generated for a different
resolution or vertical layering. CTSM has online interpolation in
`src/init_interp/` for many cases, but it must be enabled and supported
for that field set.

To enable online init interp:

```fortran
use_init_interp = .true.
init_interp_method = 'general'
```

For an unsupported case, do a cold start and respin:

```bash
./xmlchange CLM_FORCE_COLDSTART=on
```

### Symptom: NaN propagates and the model halts

Look at `lnd.log.*` and `cesm.log.*` for the first occurrence of
`NaN`, `Infinity`, or `BalanceCheck`. Common causes:

- Bad forcing data (negative precipitation, missing radiation field).
  Check `datm.log.*` and `CaseDocs/datm.streams.xml`.
- Out-of-range surface dataset (e.g., a custom `fsurdat` with PCT_NATVEG
  not summing to 100).
- A new physics PR that introduced a divide-by-zero. Bisect with `git
  bisect` or compare `BalanceCheck` output against a known-good baseline.

For interactive debugging, rebuild with debug flags:

```bash
./xmlchange DEBUG=TRUE
./case.build --clean-all
./case.build
./case.submit
```

### Symptom: water or carbon balance check fails

CTSM has built-in conservation checks in
`src/biogeophys/BalanceCheckMod.F90` and the CN balance check in
`src/biogeochem/CNBalanceCheckMod.F90`. They print a column index and a
reservoir delta. To localize, run a 1-timestep case with the failing
column as a single-point case (`subset_data point` at that lat/lon).

### Symptom: restart pointer mismatch

```
ERROR: rpointer file expected ... but found ...
```

`$RUNDIR/rpointer.*` files are pointers to the most recent restart files
for each component. If they are out of date or were written by a different
case, restart fails. Fix:

```bash
cd $RUNDIR
ls -lt *.r.* | head           # find the right set of restart files
$EDITOR rpointer.cpl rpointer.lnd rpointer.atm ...
# Edit each pointer to name the actual restart file, no path
```

For CMEPS, the driver pointer is `rpointer.cpl`, **not** the legacy
`rpointer.drv`.

---

## 5. Scientific surprises

### Wrong tuning mode

`./xmlquery LND_TUNING_MODE` should match the (physics, forcing) pair you
intend. CLM6 with GSWP3 will run, but does not reproduce calibrated
behavior. The full table is in
`bld/namelist_files/namelist_defaults_ctsm.xml` under the `lnd_tuning_mode`
defaults block.

### Wrong forcing year window

`./xmlquery DATM_YR_ALIGN,DATM_YR_START,DATM_YR_END` plus
`grep stream_year_first CaseDocs/lnd_in` and the `_last` and
`model_year_align` lines. For an `IHist` compset the start year of streams
should equal `RUN_STARTDATE` year, the last year should be the last year
you want to simulate (or beyond), and `model_year_align` should be the same
as `stream_year_first` for natural cycling.

### Restart that secretly cold-started

Look at `lnd.log.*` for `arbinit` (means the column got an arbitrary
initial state because the requested `finidat` did not provide one for it).
A few orphan columns are normal for transient landuse. Many orphan columns
mean your `finidat` does not match your `fsurdat`.

### History tape says wrong fields

`CaseDocs/lnd_in` shows the actual `hist_fincl1` etc. that was assembled
from defaults plus your `user_nl_clm`. If a field you added does not
appear, check the spelling against the registration site:

```bash
grep -rn "hist_addfld.*'<FIELDNAME>'" src/
```

---

## Comparing against a baseline

For "did my change affect answers" questions, the standard tool is
`cprnc`:

```bash
cprnc baseline.clm2.h0a.YYYY-MM.nc current.clm2.h0a.YYYY-MM.nc | tee cprnc.out
grep -i "DIFFER" cprnc.out
```

For full regression testing, `run_sys_tests` (top level of CTSM) submits
the standard test suite for several compilers on the standard CTSM testing
machines (Derecho, Izumi). Register expected new failures in
`cime_config/testdefs/ExpectedTestFails.xml`.

---

## Where to look first

| Problem looks like | Where to look |
|--------------------|---------------|
| Build error | `$EXEROOT/<component>/<component>.bldlog.*` |
| Runtime error | `$RUNDIR/<case>.log.*`, `lnd.log.*`, `cesm.log.*`, `med.log.*` |
| Forcing read failure | `datm.log.*`, `CaseDocs/datm.streams.xml` |
| Restart confusion | `rpointer.*` in `$RUNDIR`, `RUN_TYPE`, `RUN_REFCASE`, `RUN_REFDATE` |
| Conservation violation | `lnd.log.*` for `BalanceCheck`, source in `src/biogeophys/BalanceCheckMod.F90` |
| Mesh / fsurdat mismatch | `./xmlquery LND_DOMAIN_MESH,ATM_DOMAIN_MESH`, `grep fsurdat CaseDocs/lnd_in` |
| Strange answers | `./xmlquery -p CLM`, `LND_TUNING_MODE`, `DATM_YR*`, baseline `cprnc` |

---

## Where to next

- Sandbox setup so the fleximod state is right next time: `getting-started.md`
- Wiring up history fields to verify the answer change: `output-history-fields.md`
- File a fix back upstream once you find one: `contributing-pr.md`
